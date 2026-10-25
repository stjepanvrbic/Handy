# Architecture decisions

## ADR001

**Title:** Fork Handy rather than drive it from outside

Current revision: 1.

### Revision 1

Accepted 2026-10-25 from the hands-free design discussion.

**Decision:** Hands-free dictation and the editable pad are built in this fork of cjpais/Handy, in modules of their own, with `main` rebased on upstream.

**Reason:** Upstream's only external control on Windows is a toggle command with no start, stop or audio hand-off, so an outside process would lose the first syllables of each utterance and could not start a recording while the previous one pastes. Upstream is under a feature freeze, has closed every hands-free and auto-stop pull request, and tells contributors to fork.

**Consequences:** Every upstream release is a rebase. Fork code touches as few upstream files as possible. Nothing is contributed upstream.

## ADR002

**Title:** The pad appends committed text from the session's existing stream and is owned by the backend

Current revision: 1.

### Revision 1

Accepted 2026-10-25 from the hands-free design discussion and the transcribe.cpp source reading.

**Decision:** A hotkey session keeps upstream's single stream. The pad consumes the stream's committed text as append-only deltas; the owner edits the settled text; stop pastes the settled text plus the final tail. The backend owns the settled text and reconciles edits by version: an edit replaces the settled text as of its base version, and deltas consumed after that version are re-appended.

**Reason:** transcribe-cpp guarantees committed text is never rewritten within a stream, and the Nemotron family commits every decoded chunk, about every 1.1 s, with an empty tentative suffix. That makes committed text safe to edit without changing upstream's recording, streaming or paste pipeline. Rejected: segmenting hotkey sessions into one stream per utterance, which restarts punctuation and capitalization at every pause and adds a segmenter the pad does not need; a pad owned by the overlay, which would lose the dictation if the overlay failed and would make the backend wait on the webview at stop.

**Consequences:** Text becomes editable with the commit lag, about a second. Hands-free mode reuses the same stream and pad model, with one stream per turn finalized at the detected end of turn. The batch fallback applies only when the pad is empty.

## ADR003

**Title:** Keep Silero VAD v6.2 as the speech detector

Current revision: 1.

### Revision 1

Accepted 2026-10-25 from the hands-free design discussion and the VAD research.

**Decision:** Speech onset and silence detection stay on Silero VAD v6.2 as upstream runs it through transcribe-cpp.

**Reason:** FireRedVAD beats Silero only on its authors' unreleased test set. On Silero's noise-only benchmark, which includes keyboard typing and mouse clicks, Silero v6 scores 0.87 accuracy against 0.39 for FireRedVAD streaming; an independent Rust benchmark also ranks Silero higher at a seventeenth of the CPU. FireRedVAD would also need a second inference runtime or a community ggml port.

**Consequences:** No new VAD dependency. Upstream's VAD backend setting stays the switch if measurements ever favour another detector.

## ADR004

**Title:** Hands-free mode ends turns with Smart Turn v3.2 through ONNX Runtime

Current revision: 1.

### Revision 1

Accepted 2026-10-25 from the hands-free design discussion and the end-of-turn research.

**Decision:** In hands-free mode a turn ends when Smart Turn v3.2, asked at each short VAD pause over the last 8 s of audio, reports a complete turn, or when silence exceeds a settable limit. The model runs through the ONNX Runtime crate.

**Reason:** Each turn is pasted as it ends, so a premature end pastes half a sentence. On the public end-of-turn benchmark Smart Turn v3.2 cuts off 14.8% of turns at 600 ms of silence against 21.7% for a silence timer. Smart Turn ships only as ONNX; porting its Whisper Tiny encoder and head to ggml inside transcribe.cpp is more work for the same result, and the owner asked for the simplest path. Handy used ONNX Runtime before upstream removed it, so Windows bundling has precedent.

**Consequences:** A second inference runtime in the fork, also available for speaker verification later. The pad feature does not depend on it.

## ADR005

**Title:** Local post-processing runs Superwhisper S1-mini in Handy through a text session added to transcribe.cpp

Current revision: 1.

### Revision 1

Accepted 2026-10-25 from the local post-processing discussion and benchmarks on the owner's RTX 5090.

**Decision:** The Local provider runs Superwhisper S1-mini v1 at F16 in-process, through a text-generation session added to a fork of transcribe.cpp and exposed by its Rust crate, on the ggml backend Handy already uses (Vulkan on Windows). S1-mini takes its fixed system prompt and a style control line; Handy's prompts apply to the other providers only.

**Reason:** On the owner's ten real dictations (21 to 370 words), S1-mini F16 gave the best cleanup of every model tried and the lowest latency: 58 ms for 21 words and 751 ms for 370 words on CUDA, 857 ms on Vulkan, using 2.2 GB of VRAM. It never answered or executed dictated instructions. Rejected:

- Qwen3.8-27B NVFP4 with a DFlash drafter: good output but 19 GB resident and 1.8 s for 370 words.
- Nemotron 3.5 Bolt E4B: 6.9 GB; added no sentence punctuation to a raw lowercase input.
- Nemotron 3.5 Lightning 30B-A3B: 19 GB; returned a run-on transcript unchanged and added a preamble.
- S1-mini Q4_K_M: 1.4 GB but lost paragraph breaks and merged sentences on 8 of 15 cases.
- A bundled llama-server process: a second runtime and process to manage, which the owner rejected.
- The llama.cpp Rust bindings: a second copy of ggml linked beside transcribe-cpp's.
- CUDA instead of Vulkan: 10 to 15% faster on long inputs but adds about 600 MB of NVIDIA runtime DLLs.

Upstream's maintainer asked that local generation land in transcribe.cpp before any Handy integration, when closing upstream pull request 1763, and transcribe.cpp already has the Qwen3 decoder block, KV cache and byte-level BPE tokenizer S1-mini needs.

**Consequences:** The fork depends on a transcribe.cpp fork through a git dependency, which follows its upstream as this repository follows Handy. The Local provider is English only. A prompt-driven local model would need its own architecture support in the text session and a new decision. CUDA remains available through transcribe-cpp's `cuda` feature if Vulkan latency regresses.

# Architecture decisions

## ADR001

**Title:** Fork Handy rather than build outside it

Current revision: 2.

### Revision 2

Accepted 2026-10-27, when the owner narrowed the fork to local post-processing.

**Decision:** Local post-processing is built in this fork of cjpais/Handy, in modules of its own, with `main` rebased on upstream.

**Reason:** The Local provider has to run inside Handy's post-processing step and share its inference runtime, which an outside process cannot do. Upstream is under a feature freeze and closed both local post-processing pull requests offered to it, asking that local generation first land in transcribe.cpp and be built with care.

**Consequences:** Every upstream release is a rebase. Fork code touches as few upstream files as possible. Nothing is contributed upstream.

### Revision 1

Accepted 2026-10-25 from the hands-free design discussion.

**Decision:** Hands-free dictation and the editable pad are built in this fork of cjpais/Handy, in modules of their own, with `main` rebased on upstream.

**Reason:** Upstream's only external control on Windows is a toggle command with no start, stop or audio hand-off, so an outside process would lose the first syllables of each utterance and could not start a recording while the previous one pastes. Upstream is under a feature freeze, has closed every hands-free and auto-stop pull request, and tells contributors to fork.

**Consequences:** Every upstream release is a rebase. Fork code touches as few upstream files as possible. Nothing is contributed upstream.

## ADR002 to ADR004

Withdrawn 2026-10-27, with the editable pad and hands-free mode they governed: ADR002 (the pad appends committed stream text and is owned by the backend), ADR003 (keep Silero VAD v6.2), ADR004 (end hands-free turns with Smart Turn v3.2). Their accepted text is in git history. The identifiers are not reused.

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

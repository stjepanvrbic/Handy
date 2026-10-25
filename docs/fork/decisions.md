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

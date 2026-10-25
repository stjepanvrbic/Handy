# Spec

## Scope

The system is this fork of Handy, a desktop speech-to-text app, as described in the [intent](intent.md). Actors: the owner, who dictates, edits and pastes; the target application, which receives pasted text; the streaming speech model and the local text model, which Handy runs through transcribe-cpp.

This revision specifies the hotkey session with an editable pad and the Local post-processing provider. Hands-free mode and owner-only transcription are within the project but not yet specified.

Not supported: editing the live tail; voice commands for editing; the pad in sessions that use a non-streaming model or live long-form chunking, which behave as upstream Handy does; prompts with the Local provider; local text models other than S1-mini; languages other than English with the Local provider.

## Terms

- **Session**: the span from the transcribe hotkey starting a recording to its stop or cancel. Upstream calls it a recording.
- **Stream**: the model's live decode of a session, with append-only committed text and a volatile tentative suffix.
- **Delta**: committed text that the stream added since the pad last consumed it.
- **Pad**: the editable text region of the overlay holding a session's settled text.
- **Settled text**: the pad's content: consumed deltas, as edited by the owner.
- **Live tail**: the stream's tentative text, shown read-only after the settled text.
- **Final tail**: text the stream emits on finalize beyond the committed text the pad consumed.
- **Version**: a counter the backend increments on every change it makes to the settled text.
- **Activation**: the pad taking keyboard focus.
- **Target**: the window that had keyboard focus before the pad was last activated, or at session start if the pad was never activated.
- **Local provider**: the post-processing provider that runs the local text model inside Handy instead of calling an endpoint.
- **Local text model**: Superwhisper S1-mini v1, file `s1-mini-f16.gguf`, a model trained only to clean speech-to-text output.
- **Style settings**: S1-mini's three controls: Styling (`casual`, `semi-casual`, `semi-formal`, `formal`), Structure (`prose`, `lists`) and Context (`general`, `email`).
- **Chunk**: a run of whole sentences of one transcription, cleaned in one call of the local text model.

## Behavior

### The pad shows settled text and the live tail

- A session with a streaming model opens the overlay with an empty pad.
- Each delta is appended to the end of the settled text, with the stream's own whitespace.
- The live tail is rendered read-only after the settled text, visually distinct from it.
- A session with a non-streaming model shows the overlay as upstream does, without a pad.

### The owner edits settled text

- A click into the pad activates it. Keyboard input then goes to the pad until another window takes focus.
- The owner can change any part of the settled text. A delta that arrives during or after an edit is appended after the edited text; an edit is never lost, reordered or duplicated by a delta.
- Global hotkeys keep working while the pad is active, and the hotkey keystroke is not inserted into the pad.
- The live tail cannot be edited.

### Stop pastes the pad

- On stop, the text pasted is the settled text followed by the final tail. Upstream post-processing (custom words, filler-word removal, optional language-model post-processing) and history apply to that text as they apply to a transcription today.
- If the pad is active at stop, Handy returns keyboard focus to the target and pastes only once the target is the foreground window.
- If the target no longer exists at stop, the text is copied to the clipboard, nothing is typed, and the overlay reports it.
- The overlay is hidden only after the paste has been sent, and the pad is empty once the session ends.
- When the stream produced no text, the recording is transcribed in batch as upstream does. When the pad holds text, the pad is pasted even if the stream's finalize fails; the final tail is then empty.

### Cancel discards the session

- Cancel discards the pad and the recording, as upstream does. The cancel hotkey works while the pad is active.

### The overlay grows with the text

- The overlay's height grows to fit the settled text and live tail, up to the work area of the monitor the session started on, minus the existing margins. Beyond that cap, the pad scrolls.
- The overlay stays on the session's monitor for the whole session, anchored to the configured edge.

### The Local provider cleans transcriptions in Handy

- The post-processing provider list contains Local alongside upstream's providers. Upstream's providers, prompts and prompt selection are unchanged.
- With Local selected, the settings show the local text model's download state and the style settings in place of the prompt selector. Style settings default to `semi-formal`, `prose`, `general`.
- Post-processing with Local sends each chunk to the local text model with its fixed system prompt and a control line built from the style settings, decodes greedily, and joins the cleaned chunks with a single space. Custom words, history and paste then apply as they do for any provider.
- A transcription whose input fits the model's limit is one chunk. A longer one is split at sentence ends so that no chunk exceeds the limit; a single sentence over the limit is split at the last word boundary that fits.
- Output that is empty after trimming is pasted as empty: the model returns an empty string for transcriptions made only of filler or noise.
- If the model is not downloaded, fails to load, or fails to generate, the unprocessed transcription is pasted and the overlay reports that post-processing failed.

### The local text model is loaded only while needed

- The local text model is loaded when post-processing is enabled with Local selected and the model downloaded, and stays loaded between transcriptions.
- It is unloaded when post-processing is disabled, another provider is selected, or the owner presses the unload hotkey. Unloading frees its GPU memory.
- After the unload hotkey, the next transcription that needs post-processing loads the model again; the load time adds to that transcription's wait.
- The unload hotkey is a binding in upstream's binding registry, unbound by default.

## Contracts and invariants

- The backend owns the pad. It consumes deltas from the stream, increments the version on each change, and sends the overlay the full settled text, its version and the live tail on every change.
- The overlay sends an edit as the full edited settled text plus the version it was based on. The backend applies it as: edited text, followed by every delta consumed after that version, in order. The result is the new settled text with a new version.
- Committed text is consumed exactly once, in order. The backend tracks how much committed text it has consumed.
- The pasted text is the settled text, after any edit received before stop, followed by the final tail. The final tail is the finalized text beyond the consumed committed text when the finalized text starts with it, and empty otherwise.
- The target is captured at session start and again immediately before each activation. It is never the overlay or another Handy window.
- A paste is sent only while the target is the foreground window.
- Hotkey bindings and their activation modes are unchanged from upstream, apart from the added unload binding.
- transcribe-cpp exposes a text session: open a text model, generate from a system message and a user message with greedy decoding up to a token budget, cancel, close. It loads the published S1-mini GGUF unmodified, in llama.cpp's `qwen3` layout and tokenizer keys, and runs on the same ggml backend as the speech model (Vulkan on Windows).
- The text session's output for a given input is identical to llama.cpp's greedy output for the same GGUF, chat template and input, token for token.
- The local text model's input is the system prompt from its model card, verbatim, then a user message of `[Styling: <s>] [Structure: <t>] [Context: <c>]`, a newline, and the chunk. Thinking is disabled in the chat template.
- The chunk limit is 800 input tokens, under the model card's recommended maximum of about 1,000.
- The model file is downloaded from commit `34add00a48a2e5d24e5a4ee5405a99620a3a240c` of `superwhisper/s1-mini-GGUF` and verified by SHA-256 `0370da4f1bae19e3150bcafa33c5d396c15f97bf25519540a3e013db5cc00af4` before loading.

## Acceptance

1. **Deltas append.** Committed text "Hello" then "Hello world" yields settled text "Hello world". Boundary: unit test of the pad buffer.
2. **Edit survives a later delta.** Settled "Helo world" at version 3; an edit "Hello world" based on version 3 arrives after delta " again" produced version 4; result "Hello world again". Boundary: unit test of the pad buffer.
3. **Stale edit.** An edit based on version 2 arrives after versions 3 and 4 appended deltas; result is the edited text followed by both deltas in order. Boundary: unit test of the pad buffer.
4. **Paste composition.** Consumed committed "a b", finalized "a b c", settled "A b" yields paste "A b c". Finalized text that does not start with the consumed committed text yields the settled text alone. Boundary: unit test of the paste composition.
5. **Activation and hotkeys.** A click into the pad lets the owner type into it; the transcribe hotkey then stops the session and its keystroke does not appear in the pad. Boundary: scripted manual check on Windows 11.
6. **Focus restoration.** With the pad active after an edit, stop pastes the edited text into the target editor. Boundary: scripted manual check on Windows 11.
7. **Target gone.** Closing the target during the session and stopping leaves the text on the clipboard with nothing typed and a notice in the overlay. Boundary: scripted manual check on Windows 11.
8. **Growth and monitor pinning.** Dictating more lines than fit grows the overlay to the cap, after which the pad scrolls; moving the mouse to another monitor does not move the overlay. Boundary: scripted manual check on Windows 11.
9. **Non-streaming sessions unchanged.** A session with a non-streaming model shows the upstream overlay. Boundary: existing Playwright overlay test.
10. **Reference cleanup.** Input "so um i need to like send the the report by uh friday no wait make that thursday" with default style settings yields "I need to send the report by Thursday." Boundary: transcribe.cpp test of the text session.
11. **Parity with llama.cpp.** For a fixture set of at least 15 transcripts, including one over 300 words and two that are questions or instructions, the text session's output equals llama.cpp's greedy output. Boundary: transcribe.cpp test of the text session against recorded llama.cpp outputs.
12. **Latency.** A 370-word transcription is cleaned in under 1.0 s and a 40-word one in under 150 ms, model already loaded, on the owner's RTX 5090 with the Vulkan backend. Boundary: benchmark through the transcribe-cpp Rust crate.
13. **Chunking.** A transcription over the chunk limit splits at sentence ends, no chunk exceeds the limit, an over-limit sentence splits at a word boundary, and the joined output keeps chunk order. Boundary: unit test of the chunker.
14. **Settings.** Selecting Local shows the style settings and hides the prompt selector; selecting another provider restores upstream's prompt settings unchanged. Boundary: Playwright test of the post-processing settings.
15. **Lifecycle.** Enabling post-processing with Local loads the model; disabling it, switching provider, or pressing the unload hotkey frees the model's GPU memory; the next post-processed transcription after the hotkey loads it again. Boundary: unit test of the lifecycle state, plus a scripted manual check of GPU memory on Windows 11.
16. **Failure is visible.** With the model file missing or corrupt, a post-processed transcription pastes the unprocessed text and the overlay reports the failure. Boundary: scripted manual check on Windows 11.

## ADR appendix

### Current index

- [ADR001](decisions.md#adr001): Fork Handy rather than drive it from outside. Revision 1, applies to the whole fork.
- [ADR002](decisions.md#adr002): The pad appends committed text from the session's existing stream and is owned by the backend. Revision 1, applies to hotkey sessions.
- [ADR003](decisions.md#adr003): Keep Silero VAD v6.2 as the speech detector. Revision 1, applies to all modes.
- [ADR004](decisions.md#adr004): Hands-free mode ends turns with Smart Turn v3.2 through ONNX Runtime. Revision 1, applies to hands-free mode.
- [ADR005](decisions.md#adr005): Local post-processing runs Superwhisper S1-mini in Handy through a text session added to transcribe.cpp. Revision 1, applies to the Local provider.

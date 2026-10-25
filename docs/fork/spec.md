# Spec

## Scope

The system is this fork of Handy, a desktop speech-to-text app, as described in the [intent](intent.md). Actors: the owner, who dictates, edits and pastes; the target application, which receives pasted text; the streaming speech model, which Handy runs through transcribe-cpp.

This revision specifies the hotkey session with an editable pad. Hands-free mode and owner-only transcription are within the project but not yet specified.

Not supported: editing the live tail; voice commands for editing; the pad in sessions that use a non-streaming model or live long-form chunking, which behave as upstream Handy does.

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

## Contracts and invariants

- The backend owns the pad. It consumes deltas from the stream, increments the version on each change, and sends the overlay the full settled text, its version and the live tail on every change.
- The overlay sends an edit as the full edited settled text plus the version it was based on. The backend applies it as: edited text, followed by every delta consumed after that version, in order. The result is the new settled text with a new version.
- Committed text is consumed exactly once, in order. The backend tracks how much committed text it has consumed.
- The pasted text is the settled text, after any edit received before stop, followed by the final tail. The final tail is the finalized text beyond the consumed committed text when the finalized text starts with it, and empty otherwise.
- The target is captured at session start and again immediately before each activation. It is never the overlay or another Handy window.
- A paste is sent only while the target is the foreground window.
- Hotkey bindings and their activation modes are unchanged from upstream.

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

## ADR appendix

### Current index

- [ADR001](decisions.md#adr001): Fork Handy rather than drive it from outside. Revision 1, applies to the whole fork.
- [ADR002](decisions.md#adr002): The pad appends committed text from the session's existing stream and is owned by the backend. Revision 1, applies to hotkey sessions.
- [ADR003](decisions.md#adr003): Keep Silero VAD v6.2 as the speech detector. Revision 1, applies to all modes.
- [ADR004](decisions.md#adr004): Hands-free mode ends turns with Smart Turn v3.2 through ONNX Runtime. Revision 1, applies to hands-free mode.

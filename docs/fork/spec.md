# Spec

## Scope

The system is this fork of Handy, a desktop speech-to-text app, as described in the [intent](intent.md). It adds the Local post-processing provider. Actors: the owner, who dictates and chooses post-processing settings; the target application, which receives pasted text; the local text model, which Handy runs through transcribe-cpp.

Not supported: prompts with the Local provider; local text models other than S1-mini; languages other than English with the Local provider.

## Terms

- **Local provider**: the post-processing provider that runs the local text model inside Handy instead of calling an endpoint.
- **Local text model**: Superwhisper S1-mini v1, file `s1-mini-f16.gguf`, a model trained only to clean speech-to-text output.
- **Style settings**: S1-mini's three controls: Styling (`casual`, `semi-casual`, `semi-formal`, `formal`), Structure (`prose`, `lists`) and Context (`general`, `email`).
- **Chunk**: a run of whole sentences of one transcription, cleaned in one call of the local text model.

## Behavior

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

- Hotkey bindings and their activation modes are unchanged from upstream, apart from the added unload binding.
- transcribe-cpp exposes a text session: open a text model, generate from a system message and a user message with greedy decoding up to a token budget, cancel, close. It loads the published S1-mini GGUF unmodified, in llama.cpp's `qwen3` layout and tokenizer keys, and runs on the same ggml backend as the speech model (Vulkan on Windows).
- The text session's output for a given input is identical to llama.cpp's greedy output for the same GGUF, chat template and input, token for token.
- The local text model's input is the system prompt from its model card, verbatim, then a user message of `[Styling: <s>] [Structure: <t>] [Context: <c>]`, a newline, and the chunk. Thinking is disabled in the chat template.
- The chunk limit is 800 input tokens, under the model card's recommended maximum of about 1,000.
- The model file is downloaded from commit `34add00a48a2e5d24e5a4ee5405a99620a3a240c` of `superwhisper/s1-mini-GGUF` and verified by SHA-256 `0370da4f1bae19e3150bcafa33c5d396c15f97bf25519540a3e013db5cc00af4` before loading.

## Acceptance

1. **Reference cleanup.** Input "so um i need to like send the the report by uh friday no wait make that thursday" with default style settings yields "I need to send the report by Thursday." Boundary: transcribe.cpp test of the text session.
2. **Parity with llama.cpp.** For a fixture set of at least 15 transcripts, including one over 300 words and two that are questions or instructions, the text session's output equals llama.cpp's greedy output. Boundary: transcribe.cpp test of the text session against recorded llama.cpp outputs.
3. **Latency.** A 370-word transcription is cleaned in under 1.0 s and a 40-word one in under 150 ms, model already loaded, on the owner's RTX 5090 with the Vulkan backend. Boundary: benchmark through the transcribe-cpp Rust crate.
4. **Chunking.** A transcription over the chunk limit splits at sentence ends, no chunk exceeds the limit, an over-limit sentence splits at a word boundary, and the joined output keeps chunk order. Boundary: unit test of the chunker.
5. **Settings.** Selecting Local shows the style settings and hides the prompt selector; selecting another provider restores upstream's prompt settings unchanged. Boundary: Playwright test of the post-processing settings.
6. **Lifecycle.** Enabling post-processing with Local loads the model; disabling it, switching provider, or pressing the unload hotkey frees the model's GPU memory; the next post-processed transcription after the hotkey loads it again. Boundary: unit test of the lifecycle state, plus a scripted manual check of GPU memory on Windows 11.
7. **Failure is visible.** With the model file missing or corrupt, a post-processed transcription pastes the unprocessed text and the overlay reports the failure. Boundary: scripted manual check on Windows 11.

## ADR appendix

### Current index

- [ADR001](decisions.md#adr001): Fork Handy rather than build outside it. Revision 2, applies to the whole fork.
- [ADR005](decisions.md#adr005): Local post-processing runs Superwhisper S1-mini in Handy through a text session added to transcribe.cpp. Revision 1, applies to the Local provider.

ADR002 to ADR004 were withdrawn on 2026-10-27 with the editable pad and hands-free mode; their identifiers are not reused.

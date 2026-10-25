# Intent

## Problem

Stjepan dictates with Handy's streaming model and reads the text as it appears, which lets mistakes be caught at once. The live text cannot be edited, so a word with unusual spelling is fixed only after pasting. Every dictation also has to be started and stopped by hotkey, even a quick sentence into the current field. Other voices in the room would be transcribed too.

## Outcome

In a hotkey session, Stjepan can type into or correct the live text while dictation continues, and the finished text lands where they choose. With hands-free mode on, they speak without touching the keyboard and each finished utterance lands in the field that has focus, without losing its first syllable or being cut off mid-sentence. Only their voice is transcribed.

## Constraints

Windows 11 desktop; all processing stays local. Upstream cjpais/Handy accepts no new features, so this stays a fork that follows upstream. Handy's existing push-to-talk flow keeps working.

## Non-goals

Contributing the feature upstream. Wake words or spoken commands. Transcribing meetings or several speakers.

## Stakeholders

Stjepan Vrbic decides and is the only user.

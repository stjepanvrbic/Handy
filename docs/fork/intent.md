# Intent

## Problem

Stjepan dictates with Handy and wants each transcription cleaned before it is pasted: punctuation, filler words, false starts, numbers. Today that needs a cloud provider or a language model server run and managed outside Handy, and a capable general model holds most of the GPU's memory the whole time.

## Outcome

With post-processing on, Handy cleans each transcription on this machine before pasting it, without a noticeable wait and without holding most of the GPU's memory, and Stjepan can free that memory whenever they need it. Handy's existing post-processing providers and prompts keep working.

## Constraints

Windows 11 desktop; all processing stays local. Upstream cjpais/Handy accepts no new features, so this stays a fork that follows upstream.

## Non-goals

Contributing the feature upstream. Languages other than English. Changes to recording, transcription or pasting.

## Stakeholders

Stjepan Vrbic decides and is the only user.

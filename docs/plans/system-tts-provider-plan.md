# System TTS Provider Plan

## Decision

SpeakSwiftlyMobile should be a standalone iOS app with a speech synthesis provider Audio Unit extension. The app manages setup, model assets, local preview, diagnostics, and user-visible voice information. The extension is the system-facing renderer that exposes voices to iOS speech synthesis and accessibility surfaces.

This is a durable building-block change, not a temporary compatibility shim. It creates the correct iOS ownership boundary for system TTS: the host app owns user setup and resources, while the extension owns the render-time contract with the operating system.

## Apple API Grounding

Apple's system-facing third-party speech API is `AVSpeechSynthesisProviderAudioUnit`, an `AUAudioUnit` subclass that receives `AVSpeechSynthesisProviderRequest` values and renders speech audio buffers. Apple says the system scans speech synthesizer Audio Unit extensions, loads their voices, and makes those voices available to `AVSpeechSynthesizer` and accessibility technologies such as VoiceOver and Speak Screen.

Apple also states that network access is not allowed in speech synthesizers. That means the provider extension must be able to synthesize from local assets without contacting a service.

The relevant Apple docs are:

- `AVSpeechSynthesisProviderAudioUnit`: https://developer.apple.com/documentation/avfaudio/avspeechsynthesisprovideraudiounit
- `AVSpeechSynthesisProviderVoice`: https://developer.apple.com/documentation/avfaudio/avspeechsynthesisprovidervoice
- Speech synthesis overview: https://developer.apple.com/documentation/avfaudio/speech-synthesis
- Creating an Audio Unit extension: https://developer.apple.com/documentation/avfaudio/creating-an-audio-unit-extension
- `AUAudioUnit`: https://developer.apple.com/documentation/audiotoolbox/auaudiounit
- Core ML model format overview: https://developer.apple.com/documentation/coreml/getting-a-core-ml-model

## Target Shape

### `SpeakSwiftlyMobile`

The app target owns:

- onboarding and permission/status education
- local voice and model catalog UI
- model installation and deletion
- local preview using the same mobile speech engine
- diagnostics for model availability, latency, and render failures
- `TextForSpeech` profile editing once the dependency is mobile-ready

The app must not be the only synthesis surface. System TTS integration depends on the extension exposing voices that the OS can discover.

### `SpeakSwiftlySpeechProvider`

The extension target owns:

- `AVSpeechSynthesisProviderAudioUnit` subclass
- static or locally discoverable `AVSpeechSynthesisProviderVoice` list
- request-to-render state for `AVSpeechSynthesisProviderRequest`
- audio-buffer rendering through the Audio Unit render path
- cancellation through `cancelSpeechRequest()`
- marker metadata when the engine can provide word or sentence timing

The extension should start with one known voice and one small local model. Dynamic voice lists can come later after the resource-sharing model between app and extension is proven.

### `MobileSpeechCore`

Use a shared source group or framework target only if both the app and extension need the same code immediately. Its job would be narrow:

- normalize text through `TextForSpeech`
- load a local Core ML model
- convert model output into PCM buffers
- expose a small synchronous or bounded-asynchronous render state object for the extension
- return concrete errors that the app can display and the extension can log

Do not move desktop `SpeakSwiftly` runtime concepts here. No worker process, JSONL protocol, resident MLX model controls, server controls, generated-file job store, or macOS playback routing belongs in the mobile app.

## Model Runtime Direction

The default assumption is local models only. The provider extension cannot depend on network synthesis, and the app should be useful without a server once its model assets are installed.

Evaluate three runtime paths before picking the first model:

- Core ML model package: preferred first target when the chosen TTS model can be converted cleanly. This keeps the mobile engine aligned with Apple's on-device ML stack and avoids shipping a separate inference runtime inside the Audio Unit extension.
- ExecuTorch with Core ML delegation: useful if `optimum-executorch` can export the chosen Hugging Face model more directly than a pure Core ML path. Treat this as a model-conversion experiment first, not as a guaranteed app dependency.
- `mlx-audio-swift`: only consider this if Gale's fork can support iOS cleanly, the runtime and Metal resource story works inside an iOS app and provider extension, and it provides a concrete advantage over Core ML for the selected small model. Do not inherit the macOS `SpeakSwiftly` MLX worker shape.

For the first working slice, choose one small model, one voice, one sample rate, and one render format. A broader model catalog should come after the system provider can render reliably with one local model.

## Model Delivery Direction

Use App Store-managed asset delivery for large model packs when possible.

Apple's current direction is Background Assets. Apple-hosted Background Assets let App Store Connect host asset packs separately from the app build, and the system can manage download and update work. That fits the desired shape: a small initial app install, a default model that can download after install, and additional voices or models that the user can install from inside the app.

On-Demand Resources are the older App Store-hosted resource mechanism. Apple now describes On-Demand Resources as legacy and recommends migrating to Background Assets, so treat ODR as a fallback only if deployment-target or tooling constraints block Background Assets.

The app should model installed assets explicitly:

- bundled minimal fallback assets, if any
- default model asset pack
- optional model or voice asset packs
- download state
- install size and local disk usage
- provider-extension visibility after install

The provider extension should not assume every catalog model is present. It should expose only voices whose required local assets are installed and readable from the app/extension sharing location.

## TextForSpeech Dependency Plan

SpeakSwiftlyMobile should depend on `TextForSpeech` after a mobile-readiness pass.

That pass should audit and condition:

- summarization-provider defaults
- Foundation Models availability checks
- persistence defaults and Application Support paths
- filesystem assumptions
- platform-specific Apple framework imports
- tests that should prove iOS-compatible normalization behavior

`TextForSpeech` should remain the shared text-conditioning package. Speech generation, audio playback, Audio Unit rendering, and Core ML model ownership should stay in SpeakSwiftlyMobile.

## First Implementation Slices

1. Project structure

Add XcodeGen targets for the app, the speech provider extension, tests, and a narrow shared source target only if the app and extension both need it in the first pass.

2. Provider skeleton

Create a minimal `AVSpeechSynthesisProviderAudioUnit` subclass with one local voice, request capture, cancellation, and silence or fixture-tone rendering so the extension contract can be exercised before model integration.

3. Model delivery skeleton

Add a model catalog and installation state model before real model rendering. Prefer Background Assets for App Store-hosted model packs, with a development-only fixture path for local testing.

4. Engine prototype

Add a tiny Core ML loading boundary and a deterministic render-state API. Start with one model and one output format.

5. TextForSpeech integration

After the `TextForSpeech` mobile-conditioning pass, wire normalization into the app preview and provider request preparation.

6. System validation

Verify that the voice appears through system speech APIs and accessibility speech surfaces, then add diagnostics for the cases where the provider extension is installed but not selected.

## Open Questions

- Which iOS version should be the minimum once the provider extension and Core ML model requirements are tested?
- Which small Core ML TTS model is the first target, and what are its text input, acoustic output, sample rate, and latency constraints?
- Can `optimum-executorch` produce a Core ML-delegated or otherwise iOS-friendly artifact for the selected TTS model, or is a direct Core ML conversion simpler?
- Does Gale's `mlx-audio-swift` fork have a practical iOS runtime and resource story, and does it outperform a Core ML path for the chosen small model?
- Can the extension read app-installed model assets from an app group container, or should the first slice ship one bundled model inside the extension?
- Which minimum iOS version is acceptable if Background Assets becomes the primary model-delivery mechanism?
- How much `TextForSpeech` profile editing belongs in the mobile app before system voice rendering is proven?
- What voice markers can the first model provide: none, sentence boundaries, word boundaries, or approximate timings?

## Non-Goals

- Do not port the macOS `SpeakSwiftly` worker runtime into iOS.
- Do not add a server-backed synthesis path to the provider extension.
- Do not build voice cloning in the first provider slice.
- Do not require network access for any provider-extension synthesis path.
- Do not introduce a new shared package until duplication between app and extension, or between mobile and macOS, proves it is necessary.

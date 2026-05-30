# SpeakSwiftlyMobile

SpeakSwiftlyMobile is the planned iOS host app for small local speech models. It is separate from the macOS `SpeakSwiftly` worker and will use `TextForSpeech` for shared text conditioning after `TextForSpeech` has been audited and conditioned for mobile-safe use.

## Planned Shape

- iOS SwiftUI host app for voice setup, model installation, diagnostics, and local preview.
- Speech synthesis provider Audio Unit extension for system TTS integration.
- App-owned Core ML speech engine with small on-device models only.
- `TextForSpeech` dependency for normalization, built-in profiles, and custom pronunciation/profile state.
- No network-backed synthesis path in the provider extension.

## Current Status

The project is a fresh XcodeGen scaffold. The first implementation pass should define the system TTS extension target and a minimal local model/rendering boundary before adding app UI beyond setup and diagnostics.

See [docs/plans/system-tts-provider-plan.md](docs/plans/system-tts-provider-plan.md) for the initial architecture plan.

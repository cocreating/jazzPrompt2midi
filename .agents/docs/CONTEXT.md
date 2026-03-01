# Project Context: JazzPrompt2MIDI

## Overview
JazzPrompt2MIDI is a modern, web-based DAW-lite application designed for rapid jazz composition sketching. It takes a structured text prompt (the "Musical Lead Sheet") and converts it into real-time audio (high-quality piano) and downloadable multi-track MIDI files.

## Technical Stack
- **Framework**: Svelte 5 (Native Runes: `$state`, `$derived`, `$effect`).
- **Audio Engine**: Tone.js (Transport-based scheduling).
- **Instruments**: Salamander Grand Piano (multi-sample Sampler).
- **Music Theory**: Tonal.js (Chord resolution and interval math).
- **Styling**: Vanilla CSS (macOS Desktop/Terminal aesthetic).

## Architectural Design
### Reactivity Chain
1. **Source of Truth**: The `payload` variable (a large YAML-like string).
2. **Parsing Layer**: Derived variables use regex to scan the payload for metadata (`TEMPO`, `KEY`, `TIME`) and musical sections (`CHORDS`, `MELODY`).
3. **Audio Sync**: A debounced `$effect` monitors changes to the parsed music data. When changes occur during playback, it disposes of existing `Tone.Part` instances and re-constructs the musical sequence on the `Tone.Transport`.
4. **Bi-Directional Sync**: A `updateMeta` helper allows UI inputs (the status bar) to perform partial string replacements on the source `payload`, keeping the text and UI in perfect sync.

### Audio Pipeline
- **Tone.Transport**: Manages the master clock and relative beat positioning.
- **Tone.Part**: Handles event-driven playback for the Chord and Melody tracks.
- **Tone.Sampler**: Provides high-resolution acoustic piano playback using the Salamander set.
- **Tone.Draw**: Used to synchronize the UI (Virtual Piano) with the audio engine's precise timing.

### UI & Visualization
- **Virtual Piano**: A custom Svelte component that visualizes MIDI notes in real-time.
- **Real-time Sync**: Uses a `$state` Set of active MIDI numbers. Notes are added/removed via `Tone.Draw` callbacks scheduled within the `Tone.Part` loop, ensuring perfect visual-to-audio alignment.
- **Zen Mode**: A distraction-free layout that focuses strictly on the text editor, hiding all visualization and metadata panels.

### Options & Performance
- **Global Options**: A `$state` object manages `instrument` (Piano/Guitar), `voicingMode` (Basic/Jazz/Wide), and `zenMode`.
- **Dynamic Sampling**: Switching instruments triggers a dynamic reload of the `Tone.Sampler` with appropriate URL maps, ensuring high-fidelity playback without bloating the initial page load.

### MIDI Generation
- **Format**: MIDI Format 1 (Multi-track).
- **Tracks**: Track 1 contains Meta messages (Tempo, Time Sig) and Chords. Track 2 contains the Melody.
- **Timing**: 480 PPQ (Pulses Per Quarter Note) for high-resolution rhythmic mapping.

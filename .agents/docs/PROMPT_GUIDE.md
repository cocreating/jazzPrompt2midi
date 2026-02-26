# Musical Prompt Guide

To successfully generate compositions, followed the structured format below. The engine is case-insensitive for keywords.

## Metadata Section
Metadata must be listed at the top.
- `TITLE: [Name]` - Used for the MIDI filename.
- `TEMPO: [Number] BPM` - Controls playback speed.
- `TIME: [Num/Den]` - Controls bar length and time signature.
- `KEY: [Letter] [Mode]` - (e.g., `C minor`) used for tonal context.

## Musical Sections
Markers `CHORDS:` and `MELODY:` define the start of each track.

### Chords Syntax
Format: `bar [N]: [ChordName] ([Beats]) ...`
- **Default Multi-Chord**: `bar 1: Cmaj7 G7` (Splits the bar evenly, 2 beats each in 4/4).
- **Flexible Durations**: Support for `Chord (beats)` and attached `Chord(beats)` (e.g., `Fm9(4)`).
- **Complex Extensions**: Support for internal parentheses in chord names (e.g., `G13(b9)(4)` or `Fm(maj7)`).
- **Slash Chords**: Standard support for bass-voiced chords like `Ebmaj9/G`.
- **Chord Types**: Supports full extensions (e.g., `13b9`, `maj9#11`, `m11`, `ø7`, `dim7`).

### Melody Syntax
Format: `bar [N] beat [POS]: [NoteName][Octave] duration [D] velocity [V]`
- **NoteName**: A-G with `#` or `b`.
- **Octave**: Middle C is `C4`.
- **Position**: `1` is the start of the bar. `2.5` is the eighth-note upbeat of beat 2.
- **Duration**: Expressed in beats (e.g., `duration 1` is a quarter note, `duration 0.5` is an eighth note).
- **Velocity**: (Optional) MIDI Velocity from `0` to `127`. Defaults to `80` if omitted.

## Example Template
```text
TITLE: Smooth Glide
TEMPO: 120
TIME: 4/4

CHORDS:
bar 1: Dm7 (2) G13 (2)
bar 2: Cmaj9 (4)

MELODY:
bar 1 beat 1: A4 duration 0.5 velocity 72
bar 1 beat 1.5: C5 duration 0.5 velocity 80
bar 1 beat 2: D5 duration 1 velocity 90
bar 2 beat 1: E5 duration 2 velocity 60
```

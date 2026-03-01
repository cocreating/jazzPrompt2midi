<script>
    import { onMount } from 'svelte';

    /** @type {{ activeChordNotes: Set<number>, activeMelodyNotes: Set<number> }} */
    let { activeChordNotes, activeMelodyNotes } = $props();

    const OCTAVES = 4;
    const START_OCTAVE = 2;
    const NOTES_IN_OCTAVE = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];

    let keys = $derived.by(() => {
        let allKeys = [];
        for (let oct = START_OCTAVE; oct < START_OCTAVE + OCTAVES; oct++) {
            NOTES_IN_OCTAVE.forEach((note, i) => {
                const name = `${note}${oct}`;
                const midi = (oct + 1) * 12 + i;
                const isBlack = note.includes('#');
                allKeys.push({ name, midi, isBlack });
            });
        }
        // Add final C
        const topOctave = START_OCTAVE + OCTAVES;
        allKeys.push({ name: `C${topOctave}`, midi: (topOctave + 1) * 12, isBlack: false });
        return allKeys;
    });

    function getNoteState(midi) {
        const isChord = activeChordNotes.has(midi);
        const isMelody = activeMelodyNotes.has(midi);
        if (isChord && isMelody) return 'collision';
        if (isMelody) return 'harmony';
        if (isChord) return 'chord';
        return '';
    }
</script>

<div class="piano-container">
    <div class="piano-scroll">
        <div class="keys">
            {#each keys as key}
                {@const state = getNoteState(key.midi)}
                <div
                    class="key {key.isBlack ? 'black' : 'white'}"
                    class:chord={state === 'chord'}
                    class:harmony={state === 'harmony'}
                    class:collision={state === 'collision'}
                    data-note={key.name}
                >
                    {#if !key.isBlack && key.name.startsWith('C')}
                        <span class="label">{key.name}</span>
                    {/if}
                </div>
            {/each}
        </div>
    </div>
</div>

<style>
    .piano-container {
        width: 100%;
        height: 130px;
        background: #0a0a0a;
        border-top: 1px solid #222;
        position: relative;
        overflow: hidden;
        user-select: none;
    }

    .piano-scroll {
        width: 100%;
        height: 100%;
        overflow-x: auto;
        display: flex;
        scrollbar-width: thin;
        scrollbar-color: #333 #000;
    }

    .piano-scroll::-webkit-scrollbar {
        height: 4px;
    }
    .piano-scroll::-webkit-scrollbar-track {
        background: #000;
    }
    .piano-scroll::-webkit-scrollbar-thumb {
        background: #222;
        border-radius: 2px;
    }

    .keys {
        display: flex;
        height: 100%;
        margin: 0 auto;
        padding: 0 5px;
        position: relative;
    }

    .key {
        position: relative;
        flex-shrink: 0;
        transition: background-color 0.1s ease, transform 0.05s ease;
    }

    .key.white {
        width: 32px;
        height: 100%;
        background: #fdfdfd;
        border: 1px solid #ddd;
        border-bottom-left-radius: 4px;
        border-bottom-right-radius: 4px;
        z-index: 1;
    }

    .key.black {
        width: 22px;
        height: 60%;
        background: #111;
        border: 1px solid #000;
        margin-left: -11px;
        margin-right: -11px;
        z-index: 2;
        border-bottom-left-radius: 2px;
        border-bottom-right-radius: 2px;
    }

    /* Chord Active (Cyan) */
    .key.white.chord {
        background: #00ffcc;
        box-shadow: inset 0 0 10px rgba(0, 255, 204, 0.5), 0 0 20px rgba(0, 255, 204, 0.4);
        transform: translateY(2px);
    }
    .key.black.chord {
        background: #00ccaa;
        box-shadow: 0 0 15px rgba(0, 255, 204, 0.5);
        transform: translateY(1px);
    }

    /* Harmony Active (Red) */
    .key.white.harmony {
        background: #F44336;
        box-shadow: inset 0 0 10px rgba(244, 67, 54, 0.5), 0 0 20px rgba(244, 67, 54, 0.4);
        transform: translateY(2px);
        border-color: #d32f2f;
    }
    .key.black.harmony {
        background: #ef5350;
        box-shadow: 0 0 15px rgba(244, 67, 54, 0.5);
        transform: translateY(1px);
        border-color: #b71c1c;
    }

    /* Collision Active (Yellow) */
    .key.white.collision {
        background: #FFEB3B;
        box-shadow: inset 0 0 10px rgba(255, 235, 59, 0.5), 0 0 20px rgba(255, 235, 59, 0.4);
        transform: translateY(2px);
        border-color: #fbc02d;
    }
    .key.black.collision {
        background: #fff176;
        box-shadow: 0 0 15px rgba(255, 235, 59, 0.5);
        transform: translateY(1px);
        border-color: #f9a825;
    }

    .label {
        position: absolute;
        bottom: 5px;
        left: 50%;
        transform: translateX(-50%);
        font-size: 0.6rem;
        color: #999;
        pointer-events: none;
        font-family: inherit;
    }

    @media (max-width: 600px) {
        .key.white {
            width: 26px;
        }
        .key.black {
            width: 18px;
            margin-left: -9px;
            margin-right: -9px;
        }
        .piano-container {
            height: 110px;
        }
    }
</style>

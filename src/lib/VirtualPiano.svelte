<script>
    import { onMount } from 'svelte';

    /** @type {{ activeNotes: Set<string> }} */
    let { activeNotes } = $props();

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

    function isNoteActive(midi) {
        return activeNotes.has(midi);
    }
</script>

<div class="piano-container">
    <div class="piano-scroll">
        <div class="keys">
            {#each keys as key}
                <div
                    class="key {key.isBlack ? 'black' : 'white'}"
                    class:active={isNoteActive(key.midi)}
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
        height: 120px;
        background: #111;
        border-top: 1px solid #333;
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
        scrollbar-color: #333 #111;
    }

    /* Custom scrollbar for chrome/safari */
    .piano-scroll::-webkit-scrollbar {
        height: 4px;
    }
    .piano-scroll::-webkit-scrollbar-track {
        background: #111;
    }
    .piano-scroll::-webkit-scrollbar-thumb {
        background: #333;
        border-radius: 2px;
    }

    .keys {
        display: flex;
        height: 100%;
        margin: 0 auto;
        padding: 0 10px;
        position: relative;
    }

    .key {
        position: relative;
        flex-shrink: 0;
        transition: background-color 0.1s ease, transform 0.05s ease;
    }

    .key.white {
        width: 30px;
        height: 100%;
        background: #eee;
        border: 1px solid #ccc;
        border-bottom-left-radius: 4px;
        border-bottom-right-radius: 4px;
        z-index: 1;
    }

    .key.black {
        width: 20px;
        height: 60%;
        background: #222;
        border: 1px solid #000;
        margin-left: -10px;
        margin-right: -10px;
        z-index: 2;
        border-bottom-left-radius: 2px;
        border-bottom-right-radius: 2px;
    }

    .key.white.active {
        background: #00ffcc;
        box-shadow: inset 0 0 10px rgba(0, 255, 204, 0.5), 0 0 20px rgba(0, 255, 204, 0.3);
        transform: translateY(2px);
    }

    .key.black.active {
        background: #00ccaa;
        box-shadow: 0 0 15px rgba(0, 255, 204, 0.4);
        transform: translateY(1px);
    }

    .label {
        position: absolute;
        bottom: 5px;
        left: 50%;
        transform: translateX(-50%);
        font-size: 0.6rem;
        color: #888;
        pointer-events: none;
    }
</style>

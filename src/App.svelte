<script>
import defaultPayload from './defaultPayload.txt?raw';
import { Chord } from 'tonal';
import * as Tone from 'tone';

let payload = $state(defaultPayload);

let extractMeta = $derived((regex) => (payload.match(regex) || [])[1] || '-');
let bpm = $derived(parseInt(extractMeta(/TEMPO:\s*(\d+)/)) || 0);
let key = $derived(extractMeta(/KEY:\s*(.+)/));
let time = $derived(extractMeta(/TIME:\s*([\d/]+)/));
let bars = $derived(parseInt(extractMeta(/BARS:\s*(\d+)/)) || 0);

let bpb = $derived(parseInt((time || '4/4').split('/')[0]) || 4);

// Split payload into CHORDS and MELODY blocks
let sections = $derived(payload.split(/(?=CHORDS|MELODY)/));
let chordsText = $derived((sections.find(s => s.startsWith('CHORDS')) || '').trim());
let melodyText = $derived((sections.find(s => s.startsWith('MELODY')) || '').trim());

// Parse chords
let chordsMatches = $derived([...chordsText.matchAll(/bar (\d+):\s*(.+)/g)]
    .map(m => {
        const barNum = parseInt(m[1]);
        const tokens = m[2].split(/\s+/).filter(t => /^[A-G]/.test(t) || t.includes('('));
        const res = [];
        let currChord = null;
        let defaultDur = bpb / tokens.filter(t => !t.startsWith('(')).length;
        tokens.forEach(t => {
            if (t.startsWith('(')) {
                if (currChord) currChord.dur = parseFloat(t.slice(1, -1));
            } else {
                currChord = { name: t, dur: defaultDur };
                res.push(currChord);
            }
        });
        return { bar: barNum, chords: res };
    }));

let chordsCount = $derived(chordsMatches.reduce((acc, c) => acc + c.chords.length, 0));
let notesMatches = $derived([...melodyText.matchAll(/bar (\d+) beat ([\d.]+):\s*([A-G][b#]?\d)\s*duration\s*([\d.]+)/g)]);
let notesCount = $derived(notesMatches.length);

let isPlaying = $state(false);
let statusText = $state("Ready");
let sampler = null;
let playTimeout;

const stopSequence = () => {
    isPlaying = false;
    statusText = "Ready";
    clearTimeout(playTimeout);
    if (sampler) {
        sampler.releaseAll(Tone.now());
    }
};

const togglePlay = async () => {
    if (isPlaying) { stopSequence(); return; }
    if (!bpm || !time || (!notesCount && !chordsCount)) { statusText = "Invalid Payload Data"; return; }

    statusText = "Downloading Piano Samples...";

    if (!sampler) {
        sampler = new Tone.Sampler({
            urls: {
                A0: "A0.mp3",
                C1: "C1.mp3",
                "D#1": "Ds1.mp3",
                "F#1": "Fs1.mp3",
                A1: "A1.mp3",
                C2: "C2.mp3",
                "D#2": "Ds2.mp3",
                "F#2": "Fs2.mp3",
                A2: "A2.mp3",
                C3: "C3.mp3",
                "D#3": "Ds3.mp3",
                "F#3": "Fs3.mp3",
                A3: "A3.mp3",
                C4: "C4.mp3",
                "D#4": "Ds4.mp3",
                "F#4": "Fs4.mp3",
                A4: "A4.mp3",
                C5: "C5.mp3",
                "D#5": "Ds5.mp3",
                "F#5": "Fs5.mp3",
                A5: "A5.mp3",
                C6: "C6.mp3",
                "D#6": "Ds6.mp3",
                "F#6": "Fs6.mp3",
                A6: "A6.mp3",
                C7: "C7.mp3",
                "D#7": "Ds7.mp3",
                "F#7": "Fs7.mp3",
                A7: "A7.mp3",
                C8: "C8.mp3"
            },
            release: 1,
            baseUrl: "https://tonejs.github.io/audio/salamander/"
        }).toDestination();

        await Tone.loaded();
    }

    await Tone.start();

    isPlaying = true;
    statusText = `Playing (${notesCount} notes, ${chordsCount} chords)...`;

    const spb = 60.0 / bpm;
    const firstMelBar = notesMatches.length ? parseInt(notesMatches[0][1]) : 999;
    const firstChdBar = chordsMatches.length ? chordsMatches[0].bar : 999;
    const firstBar = Math.min(firstMelBar, firstChdBar);
    const startOff = Tone.now() + 0.1;

    let lastTotSecs = 0;

    // Melody
    notesMatches.forEach(m => {
        const note = m[3];
        const start = startOff + ((parseInt(m[1]) - firstBar) * bpb + (parseFloat(m[2]) - 1)) * spb;
        const dur = parseFloat(m[4]) * spb;
        sampler.triggerAttackRelease(note, dur, start, 1.0);
        lastTotSecs = Math.max(lastTotSecs, start + dur - startOff);
    });

    // Chords
    chordsMatches.forEach(m => {
        let beatOffset = 0;
        m.chords.forEach(ch => {
            const start = startOff + ((m.bar - firstBar) * bpb + beatOffset) * spb;
            const dur = ch.dur * spb;
            const notes = Chord.get(ch.name).notes.map(n => n + "3");
            sampler.triggerAttackRelease(notes, dur, start, 0.4); // lower volume
            beatOffset += ch.dur;
            lastTotSecs = Math.max(lastTotSecs, start + dur - startOff);
        });
    });

    playTimeout = setTimeout(() => {
        isPlaying = false;
        statusText = "Ready";
    }, lastTotSecs * 1000 + 200);
};

const midiNotNum = (name) => {
    const n = { 'C':0, 'C#':1, 'Db':1, 'D':2, 'D#':3, 'Eb':3, 'E':4, 'F':5, 'F#':6, 'Gb':6, 'G':7, 'G#':8, 'Ab':8, 'A':9, 'A#':10, 'Bb':10, 'B':11 };
    const m = name.match(/([A-G][b#]?)(\d)/);
    return m ? n[m[1]] + (parseInt(m[2]) + 1) * 12 : 60;
};

const exportMidi = () => {
    if (!bpm || !time || (!notesCount && !chordsCount)) {
        statusText = "Cannot export invalid payload.";
        return;
    }
    statusText = "MIDI Extracted! Download Starting...";

    const writeVarLen = (v) => { let b = [v & 0x7f]; while (v >>= 7) b.unshift((v & 0x7f) | 0x80); return b; };
    const pStr = (s) => [...s].map(c => c.charCodeAt(0));
    const p32 = (v) => [(v>>24)&0xff, (v>>16)&0xff, (v>>8)&0xff, v&0xff];
    const p16 = (v) => [(v>>8)&0xff, v&0xff];

    // Format 1 (multi-track), 2 tracks (chords, melody), 480 ticks
    const hdr = [...pStr('MThd'), ...p32(6), ...p16(1), ...p16(2), ...p16(480)];

    const uspqn = Math.floor(60000000 / bpm);
    const [num, den] = time.split('/').map(Number);

    const firstMelBar = notesMatches.length ? parseInt(notesMatches[0][1]) : 999;
    const firstChdBar = chordsMatches.length ? chordsMatches[0].bar : 999;
    const firstBar = Math.min(firstMelBar, firstChdBar);

    const makeTrk = (events, includesTempo, name) => {
        let trk = [];
        if (includesTempo) {
            trk.push(0, 0xFF, 0x51, 0x03, (uspqn>>16)&0xff, (uspqn>>8)&0xff, uspqn&0xff);
            trk.push(0, 0xFF, 0x58, 0x04, num || 4, Math.log2(den || 4), 24, 8);
        }
        trk.push(0, 0xFF, 0x03, name.length, ...pStr(name)); // Track name event

        events.sort((a,b) => a.t - b.t || (a.on ? 1 : -1));
        let lastT = 0;
        events.forEach(e => {
            const dt = e.t - lastT;
            trk.push(...writeVarLen(dt), e.on ? 0x90 : 0x80, e.nn, e.on ? 80 : 0);
            lastT = e.t;
        });
        trk.push(0, 0xFF, 0x2F, 0x00);
        return [...pStr('MTrk'), ...p32(trk.length), ...trk];
    }

    // Chords Trk
    let cEvents = [];
    chordsMatches.forEach(m => {
        let beatOffset = 0;
        m.chords.forEach(ch => {
            const stBeat = (m.bar - firstBar) * bpb + beatOffset;
            const stTick = Math.max(0, Math.round(stBeat * 480));
            const endTick = Math.max(stTick + 1, Math.round((stBeat + ch.dur) * 480));
            const notes = Chord.get(ch.name).notes;
            notes.forEach(note => {
                const nn = midiNotNum(note + "3");
                cEvents.push({ t: stTick, on: true, nn }, { t: endTick, on: false, nn });
            });
            beatOffset += ch.dur;
        });
    });

    // Melody Trk
    let mEvents = [];
    notesMatches.forEach(m => {
        const stBeat = (parseInt(m[1]) - firstBar) * bpb + (parseFloat(m[2]) - 1);
        const stTick = Math.max(0, Math.round(stBeat * 480));
        const endTick = Math.max(stTick + 1, Math.round((stBeat + parseFloat(m[4])) * 480));
        const nn = midiNotNum(m[3]);
        mEvents.push({ t: stTick, on: true, nn }, { t: endTick, on: false, nn });
    });

    // Add 2 tracks payload
    const buf = new Uint8Array([...hdr, ...makeTrk(cEvents, true, 'Chords'), ...makeTrk(mEvents, false, 'Melody')]);
    const blob = new Blob([buf], { type: 'audio/midi' });
    const u = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = u; a.download = 'jazz-composition.mid'; a.click();
    URL.revokeObjectURL(u);

    setTimeout(() => { if (!isPlaying) statusText = "Ready"; }, 2000);
};
</script>

<main class="app-container">
    <header class="window-header">
        <div class="window-controls">
            <div class="mac-btn close"></div>
            <div class="mac-btn minimize"></div>
            <div class="mac-btn maximize"></div>
        </div>
        <div class="titles">
            <h1 class="title">JAZZPROMPT2MIDI</h1>
        </div>
        <div class="subtitle-wrap">
            <h2 class="subtitle">THE MOST IMPORTANT</h2>
        </div>
    </header>

    <div class="status-bar">
        <div class="stat"><span>BPM</span> <strong>{bpm || '-'}</strong></div>
        <div class="stat"><span>KEY</span> <strong>{key || '-'}</strong></div>
        <div class="stat"><span>TIME</span> <strong>{time || '-'}</strong></div>
        <div class="stat"><span>BARS</span> <strong>{bars || '-'}</strong></div>
        <div class="stat"><span>NOTES</span> <strong>{notesCount}</strong></div>
        <div class="stat"><span>CHORDS</span> <strong>{chordsCount}</strong></div>
    </div>

    <section class="editor-section">
        <textarea bind:value={payload} spellcheck="false"></textarea>
    </section>

    <footer class="footer-controls">
        <div class="playback-controls">
            <button class="icon-btn" onclick={togglePlay} aria-label={isPlaying ? "Stop" : "Play"}>
                {#if isPlaying}
                <svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor"><path d="M6 6h12v12H6z" /></svg>
                {:else}
                <svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor"><path d="M8 5v14l11-7z" /></svg>
                {/if}
            </button>
            <button class="icon-btn" onclick={stopSequence} aria-label="Stop">
                <svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor"><path d="M6 6h12v12H6z" /></svg>
            </button>
            <button class="icon-btn" aria-label="Options" onclick={() => statusText = "Options menu toggled"}>
                <svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor"><path d="M6 10c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2zm12 0c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2zm-6 0c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2z" /></svg>
            </button>
        </div>
        <div class="sys-msg">>{statusText}</div>
        <button class="btn-primary" onclick={exportMidi}>MIDI</button>
    </footer>
</main>

<style>
:global(body) {
    background-color: #000;
    color: #e0e0e0;
    font-family: 'Courier New', Courier, monospace;
    margin: 0;
    padding: 0;
    height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
}

.app-container {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;
    background-color: #0d0d0d;
    border: 1px solid #333;
    box-shadow: 0 0 30px rgba(0,0,0,0.8);
    transition: all 0.3s ease;
}

.window-header {
    display: flex;
    align-items: center;
    padding: 0.75rem 1rem;
    background: linear-gradient(180deg, #2a2a2a, #1a1a1a);
    border-bottom: 1px solid #000;
}

.window-controls {
    display: flex;
    gap: 0.5rem;
    flex: 1;
}

.mac-btn {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    cursor: default;
}

.mac-btn.close { background-color: #ff5f56; }
.mac-btn.minimize { background-color: #ffbd2e; }
.mac-btn.maximize { background-color: #27c93f; }

.titles {
    flex: 2;
    text-align: center;
}

.title {
    margin: 0;
    font-size: 1rem;
    font-weight: bold;
    color: #fff;
    letter-spacing: 1px;
}

.subtitle-wrap {
    flex: 1;
    text-align: right;
}

.subtitle {
    margin: 0;
    font-size: 0.75rem;
    color: #888;
    text-transform: uppercase;
}

.status-bar {
    display: flex;
    flex-wrap: wrap;
    background-color: #1a1a24;
    border-bottom: 1px solid #333;
    padding: 0.5rem;
    font-size: 0.85rem;
}

.stat {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0.25rem;
    min-width: 60px;
    border-right: 1px solid #333;
}
.stat:last-child {
    border-right: none;
}

.stat span {
    color: #888;
    font-size: 0.65rem;
    margin-bottom: 0.15rem;
}
.stat strong {
    color: #00ffcc;
}

.editor-section {
    flex: 1;
    display: flex;
    overflow: hidden;
    position: relative;
    padding: 1rem;
}

textarea {
    width: 100%;
    height: 100%;
    background-color: #030303;
    color: #00ff00;
    border: 1px solid #222;
    padding: 1rem;
    font-family: inherit;
    font-size: 0.9rem;
    resize: none;
    outline: none;
    line-height: 1.5;
    border-radius: 4px;
}
textarea:focus {
    border-color: #00ffcc;
}

.footer-controls {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.75rem 1rem;
    background-color: #1a1a1a;
    border-top: 1px solid #333;
}

.playback-controls {
    display: flex;
    gap: 0.5rem;
}

.icon-btn {
    background: #2a2a2a;
    border: 1px solid #444;
    color: #ccc;
    border-radius: 4px;
    padding: 0.4rem;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.2s ease;
}
.icon-btn:hover {
    background: #444;
    color: #fff;
}

.sys-msg {
    color: #ff00ff;
    font-size: 0.85rem;
    flex: 1;
    text-align: center;
    padding: 0 1rem;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.btn-primary {
    background-color: transparent;
    border: 1px solid #00ffcc;
    color: #00ffcc;
    padding: 0.5rem 1rem;
    font-family: inherit;
    font-weight: bold;
    cursor: pointer;
    font-size: 0.8rem;
    text-transform: uppercase;
    border-radius: 4px;
    transition: all 0.2s ease;
}
.btn-primary:hover {
    background-color: #00ffcc;
    color: #000;
}

@media (min-width: 22.5em) { /* 360px */
    .title { font-size: 1.1rem; }
    .status-bar { padding: 0.5rem 1rem; }
}

@media (min-width: 37.5em) { /* 600px */
    .app-container {
        width: 95vw;
        height: 95vh;
        border-radius: 8px;
    }
    textarea { font-size: 1rem; }
    .stat { flex-direction: row; justify-content: space-between; padding: 0.5rem; }
    .stat span { margin-bottom: 0; margin-right: 0.5rem; }
}

@media (min-width: 50em) { /* 800px */
    .app-container {
        width: 90vw;
        height: 90vh;
    }
    .titles { flex: 3; }
}

@media (min-width: 56.25em) { /* 900px */
    .app-container {
        width: 80vw;
        height: 85vh;
    }
    textarea { font-size: 1.1rem; }
}

@media (min-width: 64em) { /* 1024px */
    .app-container {
        width: 960px;
        height: 640px;
    }
}
</style>

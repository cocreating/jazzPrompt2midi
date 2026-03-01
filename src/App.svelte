<script>
import { Chord } from 'tonal';
import * as Tone from 'tone';
import VirtualPiano from './lib/VirtualPiano.svelte';

// Load default payloads from assets folder dynamically
const rawDefaults = import.meta.glob('./assets/defaults/*.txt', { eager: true, query: '?raw', import: 'default' });

const defaultsMap = Object.entries(rawDefaults).reduce((acc, [path, content]) => {
    const filename = path.split('/').pop();
    const fileContent = String(content);
    const titleMatch = fileContent.match(/TITLE:\s*(.+)/i);
    const title = titleMatch ? titleMatch[1].trim() : filename;

    acc[filename] = {
        id: filename,
        content: fileContent,
        label: title
    };
    return acc;
}, {});

const defaultItems = Object.values(defaultsMap).sort((a, b) => a.label.localeCompare(b.label));

let selectedDefault = $state(defaultItems[0]?.id || '');
let payload = $state(defaultItems[0]?.content || '');

$effect(() => {
    if (selectedDefault && defaultsMap[selectedDefault]) {
        payload = defaultsMap[selectedDefault].content;
    }
});

let extractMeta = $derived((regex) => {
    const m = payload.match(regex);
    return m ? m[1].trim() : '-';
});
let title = $derived(extractMeta(/TITLE:\s*(.+)/i));
let bpm = $derived(parseInt(extractMeta(/TEMPO:\s*(\d+)/i)) || 120);
let key = $derived(extractMeta(/KEY:\s*(.+)/i));
let time = $derived(extractMeta(/TIME:\s*([\d/]+)/i));
let bars = $derived(parseInt(extractMeta(/BARS:\s*(\d+)/i)) || 0);

let bpb = $derived(parseInt((time || '4/4').split('/')[0]) || 4);

let sections = $derived(payload.split(/(?=CHORDS|MELODY)/i));
let chordsText = $derived((sections.find(s => s.match(/^CHORDS/i)) || '').trim());
let melodyText = $derived((sections.find(s => s.match(/^MELODY/i)) || '').trim());

let chordsMatches = $derived([...chordsText.matchAll(/bar (\d+):\s*(.+)/gi)]
    .map(m => {
        const barNum = parseInt(m[1]);
        let lineContent = m[2];

        const barVelMatch = lineContent.match(/velocity\s+(\d+)/i);
        const barVelocity = barVelMatch ? parseInt(barVelMatch[1]) : 80;
        lineContent = lineContent.replace(/velocity\s+\d+/i, '').trim();

        const chordRegex = /([A-G](?:(?!\(\s*\d)\S)*)(?:\s*\(([^)]+)\))?/g;
        const res = [];
        let match;

        const countRegex = new RegExp(chordRegex);
        let chordsInBar = 0;
        while (countRegex.exec(lineContent)) chordsInBar++;
        const defaultDur = bpb / (chordsInBar || 1);

        const parseRegex = new RegExp(chordRegex);
        while ((match = parseRegex.exec(lineContent)) !== null) {
            const name = match[1];
            const paramStr = match[2] || "";

            let dur = defaultDur;
            let velocity = barVelocity;

            if (paramStr) {
                const params = paramStr.split(/[\s,]+/).filter(p => p.trim());
                if (params.length > 0) {
                    const d = parseFloat(params[0]);
                    if (!isNaN(d)) dur = d;
                }

                const vIdx = params.findLastIndex(p => p.toLowerCase() === 'velocity');
                if (vIdx !== -1 && params[vIdx + 1]) {
                    const v = parseInt(params[vIdx + 1]);
                    if (!isNaN(v)) velocity = v;
                }
            }

            res.push({ name, dur, velocity });
        }
        return { bar: barNum, chords: res };
    }));

let chordsCount = $derived(chordsMatches.reduce((acc, c) => acc + c.chords.length, 0));
let notesMatches = $derived([...melodyText.matchAll(/bar (\d+) beat ([\d.]+):\s*([A-G][b#]?\d|REST)\s*duration\s*([\d.]+)(?:\s+velocity\s+(\d+))?/gi)]);
let notesCount = $derived(notesMatches.length);

let isPlaying = $state(false);
let isPaused = $state(false);
let statusText = $state("Ready");
let sampler = null;
let melodyPart = null;
let chordsPart = null;
let stopEventId = null;
let showOptions = $state(false);

let activeChordNotes = $state(new Set());
let activeMelodyNotes = $state(new Set());

const noteToMidi = (note) => {
    return Tone.Frequency(note).toMidi();
};

const addChordNote = (note) => {
    const midi = typeof note === 'string' ? noteToMidi(note) : note;
    activeChordNotes.add(midi);
    activeChordNotes = new Set(activeChordNotes);
};

const removeChordNote = (note) => {
    const midi = typeof note === 'string' ? noteToMidi(note) : note;
    activeChordNotes.delete(midi);
    activeChordNotes = new Set(activeChordNotes);
};

const addMelodyNote = (note) => {
    const midi = typeof note === 'string' ? noteToMidi(note) : note;
    activeMelodyNotes.add(midi);
    activeMelodyNotes = new Set(activeMelodyNotes);
};

const removeMelodyNote = (note) => {
    const midi = typeof note === 'string' ? noteToMidi(note) : note;
    activeMelodyNotes.delete(midi);
    activeMelodyNotes = new Set(activeMelodyNotes);
};

const clearActiveNotes = () => {
    activeChordNotes = new Set();
    activeMelodyNotes = new Set();
};

let instrument = $state('piano'); // piano, bright-piano, electric-piano, guitar
let voicingMode = $state('jazz');
let zenMode = $state(false);

const stopSequence = () => {
    isPlaying = false;
    isPaused = false;
    statusText = "Ready";
    Tone.Transport.stop();
    Tone.Transport.cancel();
    stopEventId = null;
    clearActiveNotes();
    if (sampler) {
        sampler.releaseAll(Tone.now());
    }
    if (melodyPart) { melodyPart.dispose(); melodyPart = null; }
    if (chordsPart) { chordsPart.dispose(); chordsPart = null; }
};

const setupParts = () => {
    if (melodyPart) { melodyPart.dispose(); melodyPart = null; }
    if (chordsPart) { chordsPart.dispose(); chordsPart = null; }
    if (stopEventId !== null) {
        Tone.Transport.clear(stopEventId);
        stopEventId = null;
    }
    if (!sampler) return;

    const firstMelBar = notesMatches.length ? parseInt(notesMatches[0][1]) : 999;
    const firstChdBar = chordsMatches.length ? chordsMatches[0].bar : 999;
    const firstBar = Math.min(firstMelBar, firstChdBar);

    const spb = 60.0 / bpm;
    const spbar = bpb * spb;

    const melEvents = notesMatches.map(m => {
        const barNum = parseInt(m[1]);
        const beatInBar = parseFloat(m[2]);
        const timeVal = (barNum - firstBar) * spbar + (beatInBar - 1) * spb;
        const velVal = m[5] ? parseInt(m[5]) / 127 : 0.63;
        return {
            time: timeVal,
            note: m[3],
            dur: parseFloat(m[4]) * spb,
            vel: velVal
        };
    });

    melodyPart = new Tone.Part((t, e) => {
        if (e.note.toUpperCase() !== 'REST') {
            sampler.triggerAttackRelease(e.note, e.dur, t, e.vel);
            Tone.Draw.schedule(() => {
                addMelodyNote(e.note);
            }, t);
            Tone.Draw.schedule(() => {
                removeMelodyNote(e.note);
            }, t + e.dur);
        }
    }, melEvents).start(0);

    const chdEvents = [];
    chordsMatches.forEach(m => {
        let beatOffset = 0;
        m.chords.forEach(ch => {
            const timeVal = (m.bar - firstBar) * spbar + beatOffset * spb;
            const durVal = ch.dur * spb;
            const notes = getVoicedNotes(ch.name);
            const velVal = ch.velocity / 127;
            chdEvents.push({ time: timeVal, notes, dur: durVal, vel: velVal });
            beatOffset += ch.dur;
        });
    });

    chordsPart = new Tone.Part((t, e) => {
        sampler.triggerAttackRelease(e.notes, e.dur, t, e.vel);
        Tone.Draw.schedule(() => {
            e.notes.forEach(n => addChordNote(n));
        }, t);
        Tone.Draw.schedule(() => {
            e.notes.forEach(n => removeChordNote(n));
        }, t + e.dur);
    }, chdEvents).start(0);

    let lastTime = 0;
    [...melEvents, ...chdEvents].forEach(e => {
        const end = Tone.Time(e.time).toSeconds() + Tone.Time(e.dur).toSeconds();
        if (end > lastTime) lastTime = end;
    });

    stopEventId = Tone.Transport.scheduleOnce((t) => {
        Tone.Draw.schedule(() => {
            isPlaying = false;
            isPaused = false;
            statusText = "Ready";
            clearActiveNotes();
        }, t);
    }, lastTime + 0.1);
};

const getVoicedNotes = (chordName) => {
    const c = Chord.get(chordName);
    const notes = c.notes;

    if (voicingMode === 'basic') {
        return notes.map(n => n + "3");
    }

    if (voicingMode === 'jazz') {
        return notes.map((n, i) => {
            if (i === 0) return n + "3";
            if (i === 1) return n + "4";
            if (i === 2) return n + "3";
            return n + "4";
        });
    }

    if (voicingMode === 'wide') {
        return notes.map((n, i) => {
            return n + (i % 2 === 0 ? "2" : "4");
        });
    }

    return notes.map(n => n + "3");
};

let syncTimeout = null;
$effect(() => {
    if ((isPlaying || isPaused) && (notesMatches || chordsMatches || bpm || time)) {
        if (syncTimeout) clearTimeout(syncTimeout);
        syncTimeout = setTimeout(() => {
            Tone.Transport.bpm.value = bpm;
            setupParts();
        }, 200);
    }
});

const togglePlay = async () => {
    if (isPlaying) {
        Tone.Transport.pause();
        isPlaying = false;
        isPaused = true;
        statusText = "Paused";
        return;
    }

    if (!bpm || !time || (!notesCount && !chordsCount)) { statusText = "Invalid Payload Data"; return; }

    if (!sampler) {
        const pianoUrls = {
            A0: "A0.mp3", C1: "C1.mp3", "D#1": "Ds1.mp3", "F#1": "Fs1.mp3",
            A1: "A1.mp3", C2: "C2.mp3", "D#2": "Ds2.mp3", "F#2": "Fs2.mp3",
            A2: "A2.mp3", C3: "C3.mp3", "D#3": "Ds3.mp3", "F#3": "Fs3.mp3",
            A3: "A3.mp3", C4: "C4.mp3", "D#4": "Ds4.mp3", "F#4": "Fs4.mp3",
            A4: "A4.mp3", C5: "C5.mp3", "D#5": "Ds5.mp3", "F#5": "Fs5.mp3",
            A5: "A5.mp3", C6: "C6.mp3", "D#6": "Ds6.mp3", "F#6": "Fs6.mp3",
            A6: "A6.mp3", C7: "C7.mp3", "D#7": "Ds7.mp3", "F#7": "Fs7.mp3",
            A7: "A7.mp3", C8: "C8.mp3"
        };

        const selectedUrls = {
            piano: {
                A0: "A0.mp3", C1: "C1.mp3", "D#1": "Ds1.mp3", "F#1": "Fs1.mp3",
                A1: "A1.mp3", C2: "C2.mp3", "D#2": "Ds2.mp3", "F#2": "Fs2.mp3",
                A2: "A2.mp3", C3: "C3.mp3", "D#3": "Ds3.mp3", "F#3": "Fs3.mp3",
                A3: "A3.mp3", C4: "C4.mp3", "D#4": "Ds4.mp3", "F#4": "Fs4.mp3",
                A4: "A4.mp3", C5: "C5.mp3", "D#5": "Ds5.mp3", "F#5": "Fs5.mp3",
                A5: "A5.mp3", C6: "C6.mp3", "D#6": "Ds6.mp3", "F#6": "Fs6.mp3",
                A6: "A6.mp3", C7: "C7.mp3", "D#7": "Ds7.mp3", "F#7": "Fs7.mp3",
                A7: "A7.mp3", C8: "C8.mp3"
            },
            'bright-piano': {
                A0: "A0.mp3", C1: "C1.mp3", "D#1": "Ds1.mp3", "F#1": "Fs1.mp3",
                A1: "A1.mp3", C2: "C2.mp3", "D#2": "Ds2.mp3", "F#2": "Fs2.mp3",
                A2: "A2.mp3", C3: "C3.mp3", "D#3": "Ds3.mp3", "F#3": "Fs3.mp3",
                A3: "A3.mp3", C4: "C4.mp3", "D#4": "Ds4.mp3", "F#4": "Fs4.mp3",
                A4: "A4.mp3", C5: "C5.mp3", "D#5": "Ds5.mp3", "F#5": "Fs5.mp3",
                A5: "A5.mp3", C6: "C6.mp3", "D#6": "Ds6.mp3", "F#6": "Fs6.mp3",
                A6: "A6.mp3", C7: "C7.mp3", "D#7": "Ds7.mp3", "F#7": "Fs7.mp3",
                A7: "A7.mp3", C8: "C8.mp3"
            },
            'electric-piano': {
                A1: "A1.mp3", C2: "C2.mp3", "D#2": "Ds2.mp3", "F#2": "Fs2.mp3",
                A2: "A2.mp3", C3: "C3.mp3", "D#3": "Ds3.mp3", "F#3": "Fs3.mp3",
                A3: "A3.mp3", C4: "C4.mp3", "D#4": "Ds4.mp3", "F#4": "Fs4.mp3",
                A4: "A4.mp3", C5: "C5.mp3", "D#5": "Ds5.mp3", "F#5": "Fs5.mp3",
                A5: "A5.mp3", C6: "C6.mp3", "D#6": "Ds6.mp3", "F#6": "Fs6.mp3",
                A6: "A6.mp3", C7: "C7.mp3", "D#7": "Ds7.mp3", "F#7": "Fs7.mp3"
            },
            guitar: {
                "F#2": "Fs2.mp3", "F#3": "Fs3.mp3", "F#4": "Fs4.mp3", "F#5": "Fs5.mp3",
                G2: "G2.mp3", G3: "G3.mp3", G4: "G4.mp3", G5: "G5.mp3",
                "G#2": "Gs2.mp3", "G#3": "Gs3.mp3", "G#4": "Gs4.mp3", "G#5": "Gs5.mp3",
                A2: "A2.mp3", A3: "A3.mp3", A4: "A4.mp3", A5: "A5.mp3",
                "A#2": "As2.mp3", "A#3": "As3.mp3", "A#4": "As4.mp3", "A#5": "As5.mp3",
                B2: "B2.mp3", B3: "B3.mp3", B4: "B4.mp3", B5: "B5.mp3",
                C3: "C3.mp3", C4: "C4.mp3", C5: "C5.mp3", C6: "C6.mp3",
                "C#3": "Cs3.mp3", "C#4": "Cs4.mp3", "C#5": "Cs5.mp3", "C#6": "Cs6.mp3",
                D3: "D3.mp3", D4: "D4.mp3", D5: "D5.mp3", D6: "D6.mp3",
                "D#3": "Ds3.mp3", "D#4": "Ds4.mp3", "D#5": "Ds5.mp3", "D#6": "Ds6.mp3",
                E2: "E2.mp3", E3: "E3.mp3", E4: "E4.mp3", E5: "E5.mp3"
            }
        };

        const baseUrls = {
            piano: "https://tonejs.github.io/audio/salamander/",
            'bright-piano': "https://raw.githubusercontent.com/nbrosowsky/tonejs-instruments/master/samples/piano/",
            'electric-piano': "https://raw.githubusercontent.com/Alexander-T-S/Tone.js-Instruments/master/samples/piano-electric/",
            guitar: "https://raw.githubusercontent.com/nbrosowsky/tonejs-instruments/master/samples/guitar-acoustic/"
        };

        const selectedUrlsMap = selectedUrls[instrument] || selectedUrls.piano;
        const selectedBase = baseUrls[instrument] || baseUrls.piano;

        statusText = `Downloading ${instrument} Samples...`;
        try {
            sampler = new Tone.Sampler({
                urls: selectedUrlsMap,
                release: 1,
                baseUrl: selectedBase
            }).toDestination();
            await Tone.loaded();
        } catch (err) {
            console.error("Sampler loading failed:", err);
            statusText = `Error loading ${instrument} samples. Check connection.`;
            sampler = null;
            return;
        }
    }

    await Tone.start();

    if (!isPaused) {
        Tone.Transport.cancel();
        Tone.Transport.bpm.value = bpm;
        setupParts();
    }

    isPlaying = true;
    isPaused = false;
    statusText = `Playing (${notesCount} notes, ${chordsCount} chords)...`;
    Tone.Transport.start();
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
        trk.push(0, 0xFF, 0x03, name.length, ...pStr(name));

        events.sort((a,b) => a.t - b.t || (a.on ? 1 : -1));
        let lastT = 0;
        events.forEach(e => {
            const dt = e.t - lastT;
            trk.push(...writeVarLen(dt), e.on ? 0x90 : 0x80, e.nn, (e.on ? (e.vel || 80) : 0));
            lastT = e.t;
        });
        trk.push(0, 0xFF, 0x2F, 0x00);
        return [...pStr('MTrk'), ...p32(trk.length), ...trk];
    }

    let cEvents = [];
    chordsMatches.forEach(m => {
        let beatOffset = 0;
        m.chords.forEach(ch => {
            const stBeat = (m.bar - firstBar) * bpb + beatOffset;
            const stTick = Math.max(0, Math.round(stBeat * 480));
            const endTick = Math.max(stTick + 1, Math.round((stBeat + ch.dur) * 480));
            const notes = getVoicedNotes(ch.name);
            const vel = ch.velocity || 60;
            notes.forEach(note => {
                const nn = midiNotNum(note);
                cEvents.push({ t: stTick, on: true, nn, vel }, { t: endTick, on: false, nn });
            });
            beatOffset += ch.dur;
        });
    });

    let mEvents = [];
    notesMatches.forEach(m => {
        if (m[3].toUpperCase() === 'REST') return;
        const stBeat = (parseInt(m[1]) - firstBar) * bpb + (parseFloat(m[2]) - 1);
        const stTick = Math.max(0, Math.round(stBeat * 480));
        const endTick = Math.max(stTick + 1, Math.round((stBeat + parseFloat(m[4])) * 480));
        const vel = m[5] ? parseInt(m[5]) : 80;
        mEvents.push({ t: stTick, on: true, nn: midiNotNum(m[3]), vel }, { t: endTick, on: false, nn: midiNotNum(m[3]) });
    });

    const slug = (title || 'jazz-composition').toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '');
    const buf = new Uint8Array([...hdr, ...makeTrk(cEvents, true, 'Chords'), ...makeTrk(mEvents, false, 'Melody')]);
    const blob = new Blob([buf], { type: 'audio/midi' });
    const u = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = u; a.download = `${slug}.mid`; a.click();
    URL.revokeObjectURL(u);

    setTimeout(() => { if (!isPlaying) statusText = "Ready"; }, 2000);
};

const updateMeta = (tag, val) => {
    if (tag === 'TEMPO') {
        const regex = /(TEMPO:\s*)(\d+)(.*)/i;
        if (payload.match(regex)) {
            payload = payload.replace(regex, `$1${val}$3`);
        } else {
            payload = `TEMPO: ${val} BPM\n` + payload;
        }
    } else {
        const regex = new RegExp(`(${tag}:\\s*)([^\\n\\r]+)`, 'i');
        if (payload.match(regex)) {
            payload = payload.replace(regex, `$1${val}`);
        } else {
            payload = `${tag.toUpperCase()}: ${val}\n` + payload;
        }
    }
};

const templates = {
    'Blues in Bb': `TITLE: Blues in Bb\nTEMPO: 120 BPM\nKEY: Bb\nTIME: 4/4\nBARS: 12\n\nCHORDS:\nbar 1: Bb7\nbar 2: Eb7\nbar 3: Bb7\nbar 4: Bb7\nbar 5: Eb7\nbar 6: Edim7\nbar 7: Bb7\nbar 8: G7alt\nbar 9: Cm7\nbar 10: F7alt\nbar 11: Bb7 G7alt\nbar 12: Cm7 F7alt\n\nMELODY:\nbar 1 beat 1: F4 duration 0.5\nbar 1 beat 1.5: D4 duration 0.5\n`,
    'II-V-I in C': `TITLE: II-V-I in C\nTEMPO: 110 BPM\nKEY: C\nTIME: 4/4\nBARS: 4\n\nCHORDS:\nbar 1: Dm7\nbar 2: G7\nbar 3: Cmaj7\nbar 4: Cmaj7\n\nMELODY:\nbar 1 beat 1: F4 duration 1.0\nbar 2 beat 1: B3 duration 1.0\nbar 3 beat 1: C4 duration 2.0\n`,
    'Rhythm Changes (A)': `TITLE: Rhythm Changes A Section\nTEMPO: 180 BPM\nKEY: Bb\nTIME: 4/4\nBARS: 8\n\nCHORDS:\nbar 1: Bb6 G7\nbar 2: Cm7 F7\nbar 3: Bb6 G7\nbar 4: Cm7 F7\nbar 5: Fm7 Bb7\nbar 6: Eb7 Ebm7\nbar 7: Bb6 G7\nbar 8: Cm7 F7\n\nMELODY:\nbar 1 beat 1: D4 duration 0.5\n`
};

const loadTemplate = (name) => {
    payload = templates[name];
    showOptions = false;
    statusText = `Loaded template: ${name}`;
};

const toggleOptions = () => {
    showOptions = !showOptions;
};
</script>

<main class="app-container" class:zen={zenMode}>
    <header class="window-header">
        <div class="playback-controls header-controls">
            <button class="icon-btn" onclick={togglePlay} aria-label={isPlaying ? "Pause" : "Play"}>
                {#if isPlaying}
                <svg viewBox="0 0 24 24" width="16" height="16" fill="currentColor">
                    <path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/>
                </svg>
                {:else}
                <svg viewBox="0 0 24 24" width="16" height="16" fill="currentColor">
                    <path d="M8 5v14l11-7z" />
                </svg>
                {/if}
            </button>
            <button class="icon-btn" onclick={stopSequence} aria-label="Stop/Restart">
                <svg viewBox="0 0 24 24" width="16" height="16" fill="currentColor">
                    <path d="M6 6h12v12H6z" />
                </svg>
            </button>
        </div>
        <div class="titles">
            <h1 class="title">JAZZPROMPT2MIDI</h1>
        </div>
        <div class="header-right">
            <div class="default-select-wrap">
                <select bind:value={selectedDefault} aria-label="Select default payload">
                    {#each defaultItems as item}
                        <option value={item.id}>{item.label}</option>
                    {/each}
                </select>
            </div>
            <button class="icon-btn options-btn" aria-label="Options" onclick={toggleOptions} class:active={showOptions}>
                <svg viewBox="0 0 24 24" width="18" height="18" fill="currentColor"><path d="M6 10c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2zm12 0c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2zm-6 0c-1.1 0-2 .9-2 2s.9 2 2 2 2-.9 2-2-.9-2-2-2z" /></svg>
            </button>
        </div>
    </header>

    <div class="main-content">
        <div class="status-bar-outer">
            <div class="status-bar">
                <div class="stat">
                    <span>BPM</span>
                    <input
                        type="text"
                        value={bpm}
                        onchange={(e) => updateMeta('TEMPO', e.currentTarget.value)}
                        spellcheck="false"
                    />
                </div>
                <div class="stat">
                    <span>KEY</span>
                    <input
                        type="text"
                        value={key}
                        onchange={(e) => updateMeta('KEY', e.currentTarget.value)}
                        spellcheck="false"
                    />
                </div>
                <div class="stat">
                    <span>TIME</span>
                    <input
                        type="text"
                        value={time}
                        onchange={(e) => updateMeta('TIME', e.currentTarget.value)}
                        spellcheck="false"
                    />
                </div>
                <div class="stat">
                    <span>BARS</span>
                    <input
                        type="text"
                        value={bars}
                        onchange={(e) => updateMeta('BARS', e.currentTarget.value)}
                        spellcheck="false"
                    />
                </div>
                <div class="stat"><span>NOTES</span> <strong>{notesCount}</strong></div>
                <div class="stat"><span>CHORDS</span> <strong>{chordsCount}</strong></div>
            </div>
        </div>

        <section class="editor-section">
            <div class="editor-container">
                <textarea bind:value={payload} spellcheck="false"></textarea>
            </div>
        </section>
    </div>

    {#if !zenMode}
        <VirtualPiano {activeChordNotes} {activeMelodyNotes} />
    {/if}

    <footer class="footer-controls">
        <div class="footer-inner">
            <div class="sys-msg">>{statusText}</div>
            <button class="btn-primary" onclick={exportMidi}>MIDI</button>
        </div>
    </footer>

    {#if showOptions}
    <div
        class="options-overlay"
        onclick={toggleOptions}
        onkeydown={(e) => e.key === 'Escape' && toggleOptions()}
        role="button"
        tabindex="-1"
        aria-label="Close options"
    >
        <div class="options-content" onclick={(e) => e.stopPropagation()} role="presentation">
            <h3>OPTIONS</h3>

            <div class="option-group">
                <label for="instr-select">Instrument</label>
                <select id="instr-select" bind:value={instrument} onchange={() => { stopSequence(); if(sampler) { sampler.dispose(); sampler = null; } }}>
                    <option value="piano">Classic Grand (Salamander)</option>
                    <option value="bright-piano">Bright Piano (VSO2)</option>
                    <option value="electric-piano">Electric Piano (Rhodes)</option>
                    <option value="guitar">Jazz Guitar (Acoustic)</option>
                </select>
            </div>

            <div class="option-group">
                <label for="voicing-select">Voicing Mode</label>
                <select id="voicing-select" bind:value={voicingMode}>
                    <option value="basic">Basic (1-3-5-7)</option>
                    <option value="jazz">Jazz (Rootless/3-7)</option>
                    <option value="wide">Wide (Open Position)</option>
                </select>
            </div>

            <div class="option-group">
                <label for="zen-toggle">Layout</label>
                <button id="zen-toggle" class="toggle-btn" class:active={zenMode} onclick={() => zenMode = !zenMode}>
                    {zenMode ? 'Exit Zen Mode' : 'Enter Zen Mode'}
                </button>
            </div>

            <div class="option-group">
                <span class="group-label">Templates</span>
                <div class="template-grid">
                    {#each Object.keys(templates) as name}
                        <button onclick={() => loadTemplate(name)}>{name}</button>
                    {/each}
                </div>
            </div>

            <button class="close-options" onclick={toggleOptions}>CLOSE</button>
        </div>
    </div>
    {/if}
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
width: 100vw;
    height: 100vh;
    background-color: #0d0d0d;
    border: 1px solid #333;
    box-shadow: 0 0 30px rgba(0,0,0,0.8);
}

.window-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.5rem 1rem;
    background: linear-gradient(180deg, #2a2a2a, #1a1a1a);
    border-bottom: 1px solid #000;
    gap: 1rem;
}

.playback-controls.header-controls {
    display: flex;
    gap: 0.5rem;
}

.titles {
    flex: 1;
    text-align: center;
}

.title {
    margin: 0;
    font-size: 0.9rem;
    font-weight: bold;
    color: #fff;
    letter-spacing: 2px;
}

.header-right {
    display: flex;
    align-items: center;
    gap: 1rem;
}

.default-select-wrap select {
    background: #000;
    color: #00ffcc;
    border: 1px solid #444;
    padding: 2px 5px;
    font-size: 0.75rem;
    border-radius: 4px;
    outline: none;
}

.options-btn {
    background: transparent;
    border: 1px solid transparent;
    color: #ccc;
    padding: 4px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
}
.options-btn:hover { color: #fff; }
.options-btn.active { color: #00ffcc; border-color: #00ffcc; }

.status-bar-outer {
    width: 100%;
    background-color: #1a1a24;
    border-bottom: 1px solid #333;
}

.status-bar {
    display: flex;
    flex-wrap: wrap;
    padding: 0.5rem;
    font-size: 0.85rem;
    max-width: 1200px;
    margin: 0 auto;
}

.stat {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0.25rem;
    min-width: 70px;
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

.stat input {
    background: transparent;
    border: none;
    color: #00ffcc;
    font-family: inherit;
    font-weight: bold;
    text-align: center;
    font-size: 0.85rem;
    width: 100%;
    outline: none;
}

.main-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    overflow: hidden;
}

.editor-section {
    flex: 1;
    display: flex;
    justify-content: center;
    overflow: hidden;
    padding: 1rem;
    background-color: #080808;
}

.editor-container {
    width: 100%;
    max-width: 900px;
    height: 100%;
}

textarea {
    width: 100%;
    height: 100%;
    background-color: #030303;
    color: #00ff00;
    border: 1px solid #222;
    padding: 1rem;
    font-family: inherit;
    font-size: 0.95rem;
    resize: none;
    outline: none;
    line-height: 1.6;
    border-radius: 4px;
    box-shadow: inset 0 0 10px rgba(0,0,0,0.5);
}

textarea:focus {
    border-color: #444;
}

.footer-controls {
    width: 100%;
    background-color: #1a1a1a;
    border-top: 1px solid #333;
}

.footer-inner {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.75rem 1rem;
    max-width: 1200px;
    margin: 0 auto;
}

.playback-controls {
    display: flex;
    gap: 0.8rem;
}

.icon-btn {
    background: #2a2a2a;
    color: #ccc;
    border-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 8px;
    cursor: pointer;
    border: 1px solid #444;
}

.icon-btn:hover { background: #3a3a3a; color: #fff; }
.icon-btn.active { color: #00ffcc; border-color: #ff00ff; background: #ff00ff; color: #000; }

.sys-msg {
    flex: 1;
    margin: 0 1.5rem;
    font-size: 0.8rem;
    color: #ff00ff;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    text-align: center;
}

.btn-primary {
    background: transparent;
    border: 1px solid #00ffcc;
    color: #00ffcc;
    font-weight: bold;
    border-radius: 4px;
    padding: 0.5rem 1.2rem;
    cursor: pointer;
    transition: all 0.2s;
    text-transform: uppercase;
}

.btn-primary:hover {
    background: #00ffcc;
    color: #000;
}

/* Options Overlay */
.options-overlay {
    position: fixed;
    top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(0,0,0,0.85);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
}
.options-content {
    background: #1a1a24;
    border: 1px solid #444;
    padding: 2rem;
    border-radius: 8px;
    width: 90%;
    max-width: 500px;
}
.options-content h3 { margin-top: 0; border-bottom: 1px solid #333; padding-bottom: 0.5rem; color: #ff00ff; }

.option-group { margin-bottom: 1.5rem; display: flex; flex-direction: column; gap: 0.5rem; }
.option-group label { font-size: 0.75rem; color: #888; text-transform: uppercase; }
.option-group select { background: #0d0d12; color: #fff; border: 1px solid #333; padding: 0.5rem; border-radius: 4px; outline: none; }

.toggle-btn { background: #2a2a35; color: #fff; border: 1px solid #444; padding: 0.5rem; border-radius: 4px; cursor: pointer; }
.toggle-btn.active { background: #00ffcc; color: #000; border-color: #00ffcc; }

.template-grid { display: grid; grid-template-columns: 1fr; gap: 0.5rem; }
.template-grid button { background: #222; border: 1px solid #333; color: #ccc; padding: 0.4rem; font-size: 0.75rem; border-radius: 4px; text-align: left; }
.template-grid button:hover { border-color: #00ffcc; color: #fff; }

.close-options { width: 100%; padding: 0.75rem; margin-top: 1rem; background: transparent; border: 1px solid #ff00ff; color: #ff00ff; border-radius: 4px; cursor: pointer; }
.close-options:hover { background: #ff00ff; color: #000; }

.zen .window-header, .zen .status-bar-outer {
    display: none !important;
}

/* Responsive adjustments */
@media (max-width: 768px) {
    .window-header {
        flex-direction: column;
        padding: 0.75rem;
        gap: 0.75rem;
    }
    .titles { order: -1; width: 100%; }
    .title { font-size: 0.85rem; }

    .status-bar .stat { min-width: 30%; border: none; }

    .footer-inner { flex-direction: column; gap: 1rem; text-align: center; }
    .sys-msg { margin: 0; }

    .editor-container { max-width: 100%; }
}

@media (max-width: 480px) {
    .status-bar .stat { min-width: 45%; }
}
</style>

<script>
    import { onMount, onDestroy } from 'svelte';
    import * as Tone from 'tone';

    let { totalBars = 0, bpb = 4, onSeek } = $props();

    let canvas;
    let ctx;
    let width = $state(0);
    let height = 40;
    let isDragging = false;
    let animationFrame;

    let totalBeats = $derived(totalBars * bpb);
    let playheadPos = $state(0); // in pixels

    const updatePlayhead = () => {
        if (!canvas || isDragging) {
            animationFrame = requestAnimationFrame(updatePlayhead);
            return;
        }

        const seconds = Tone.Transport.seconds;
        const bpm = Tone.Transport.bpm.value;
        const secondsPerBeat = 60 / bpm;
        const currentBeat = seconds / secondsPerBeat;

        playheadPos = (currentBeat / totalBeats) * width;
        draw();
        animationFrame = requestAnimationFrame(updatePlayhead);
    };

    const draw = () => {
        if (!ctx) return;
        ctx.clearRect(0, 0, width, height);

        // Background
        ctx.fillStyle = '#1e1e1e';
        ctx.fillRect(0, 0, width, height);

        // Grid lines (Bars)
        ctx.strokeStyle = '#333';
        ctx.lineWidth = 1;
        for (let i = 0; i <= totalBars; i++) {
            const x = (i / totalBars) * width;
            ctx.beginPath();
            ctx.moveTo(x, 0);
            ctx.lineTo(x, height);
            ctx.stroke();

            if (i < totalBars) {
                ctx.fillStyle = '#666';
                ctx.font = '10px Inter, sans-serif';
                ctx.fillText(i + 1, x + 4, 12);
            }
        }

        // Beats (Sub-grid)
        ctx.strokeStyle = '#252525';
        for (let i = 0; i < totalBeats; i++) {
            if (i % bpb === 0) continue;
            const x = (i / totalBeats) * width;
            ctx.beginPath();
            ctx.moveTo(x, height * 0.6);
            ctx.lineTo(x, height);
            ctx.stroke();
        }

        // Playhead
        ctx.strokeStyle = '#ff3e00';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(playheadPos, 0);
        ctx.lineTo(playheadPos, height);
        ctx.stroke();

        // Playhead diamond top
        ctx.fillStyle = '#ff3e00';
        ctx.beginPath();
        ctx.moveTo(playheadPos - 4, 0);
        ctx.lineTo(playheadPos + 4, 0);
        ctx.lineTo(playheadPos, 6);
        ctx.closePath();
        ctx.fill();
    };

    onMount(() => {
        ctx = canvas.getContext('2d');
        const resize = () => {
            const rect = canvas.getBoundingClientRect();
            width = rect.width;
            canvas.width = width;
            canvas.height = height;
            draw();
        };
        window.addEventListener('resize', resize);
        resize();
        animationFrame = requestAnimationFrame(updatePlayhead);

        return () => {
            window.removeEventListener('resize', resize);
            cancelAnimationFrame(animationFrame);
        };
    });

    const handleMouseDown = (e) => {
        isDragging = true;
        seek(e);
        window.addEventListener('mousemove', handleMouseMove);
        window.addEventListener('mouseup', handleMouseUp);
    };

    const handleMouseMove = (e) => {
        if (isDragging) seek(e);
    };

    const handleMouseUp = () => {
        isDragging = false;
        window.removeEventListener('mousemove', handleMouseMove);
        window.removeEventListener('mouseup', handleMouseUp);
    };

    const seek = (e) => {
        if (width <= 0) return;
        const rect = canvas.getBoundingClientRect();
        let x = e.clientX - rect.left;
        x = Math.max(0, Math.min(x, width));
        playheadPos = x;

        const ratio = x / width;
        const targetBeat = ratio * totalBeats;

        onSeek(targetBeat);
        draw();
    };

    let currentBeatForAria = $derived(((playheadPos / (width || 1)) * totalBeats) / bpb);

    $effect(() => {
        if (totalBars || bpb) draw();
    });
</script>

<div class="timeline-container" bind:clientWidth={width}>
    <canvas
        bind:this={canvas}
        onmousedown={handleMouseDown}
        role="slider"
        aria-label="Timeline"
        aria-valuemin="0"
        aria-valuemax={totalBars}
        aria-valuenow={currentBeatForAria}
        tabindex="0"
    ></canvas>
</div>

<style>
    .timeline-container {
        width: 100%;
        height: 40px;
        background: #1e1e1e;
        border-top: 1px solid #333;
        border-bottom: 1px solid #333;
        cursor: crosshair;
        overflow: hidden;
    }
    canvas {
        display: block;
        width: 100%;
        height: 100%;
    }
</style>

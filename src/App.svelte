<script>
  import { onDestroy } from "svelte";

  // ── Random palette on every visit ────────────────────────────────────────
  const palettes = [
    { primary: "#440d49", accent: "#c3ddc0", bg: "#d4bff9" }, // lavender
    { primary: "#1a3d5c", accent: "#b8ddf0", bg: "#cde8f5" }, // ocean blue
    { primary: "#3b1f0a", accent: "#f5d9a0", bg: "#fcefd4" }, // warm sand
    { primary: "#1b3d2e", accent: "#a8dfc0", bg: "#c8f0da" }, // forest green
    { primary: "#5c1a1a", accent: "#f0c4b8", bg: "#f5dcd4" }, // terracotta
    { primary: "#1e1e5c", accent: "#c4c8f0", bg: "#d8dbf5" }, // midnight indigo
    { primary: "#4a2060", accent: "#e0c8f5", bg: "#eeddf7" }, // deep violet
    { primary: "#0d3d3d", accent: "#a0ddd8", bg: "#c4eeec" }, // teal
    { primary: "#5c3d00", accent: "#f0dda0", bg: "#faefd4" }, // amber
    { primary: "#2d0d3d", accent: "#d0b8f0", bg: "#e5d4f5" }, // plum
    { primary: "#3d1a00", accent: "#ffd6a0", bg: "#ffe8c8" }, // burnt orange
    { primary: "#003d3d", accent: "#a0f0e0", bg: "#c0f5ee" }, // aqua
    { primary: "#1a1a1a", accent: "#e0e0e0", bg: "#f0f0f0" }, // monochrome
    { primary: "#3d0033", accent: "#f5b8e8", bg: "#fad4f2" }, // rose
    { primary: "#003d1a", accent: "#b8f5c8", bg: "#d4fae0" }, // mint
    { primary: "#1a2d3d", accent: "#b8cce8", bg: "#d4e0f5" }, // slate blue
    { primary: "#2d2d00", accent: "#e8e0a0", bg: "#f5f0c8" }, // olive
    { primary: "#3d001a", accent: "#f5b8c8", bg: "#fad4dc" }, // crimson blush
    { primary: "#001a3d", accent: "#b8d0f5", bg: "#d4e4fa" }, // royal navy
    { primary: "#1a3d00", accent: "#c8f0a0", bg: "#dff5c4" }, // lime
  ];
  const palette = palettes[Math.floor(Math.random() * palettes.length)];
  const root = document.documentElement;
  root.style.setProperty("--primary", palette.primary);
  root.style.setProperty("--accent",  palette.accent);
  root.style.setProperty("--bg",      palette.bg);

  // ── Audio refs ────────────────────────────────────────────────────────────
  let audio, notifyaudio;

  // ── Timer state ───────────────────────────────────────────────────────────
  let slidervalue = 25;
  let timervalue  = "00:00";
  let interval    = null;
  let seconds     = 0;

  // ══════════════════════════════════════════════════════════════════════════
  //  SPEECH ENGINE
  //  - Single queue → zero overlap between modules
  //  - Cycles through all available TTS voices for variety
  //  - Modules A & B are kept ≥ 10 s apart
  // ══════════════════════════════════════════════════════════════════════════
  let speechQueue    = [];
  let isSpeechActive = false;
  let voices         = [];
  let voiceIndex     = 0;

  // Called by Svelte once DOM is ready; also fires on voiceschanged
  function loadVoices() {
    const v = speechSynthesis.getVoices();
    if (v.length) voices = v.filter(v => v.lang.startsWith("en"));
  }
  loadVoices();
  if (typeof speechSynthesis !== "undefined")
    speechSynthesis.onvoiceschanged = loadVoices;

  function drainQueue() {
    if (isSpeechActive || speechQueue.length === 0) return;
    isSpeechActive = true;

    const text   = speechQueue.shift();
    const speech = new SpeechSynthesisUtterance(text);

    // Cycle through voices for variety
    if (voices.length > 0) {
      speech.voice  = voices[voiceIndex % voices.length];
      voiceIndex++;
    }
    speech.pitch  = 0.75;
    speech.rate   = 0.95;
    speech.volume = SpeakVolume;
    speech.onend  = () => { isSpeechActive = false; drainQueue(); };
    speech.onerror = () => { isSpeechActive = false; drainQueue(); };
    speechSynthesis.speak(speech);
  }

  function speak(text) { speechQueue.push(text); drainQueue(); }

  // ── 10-second gap enforcement between Module A & B ────────────────────────
  let lastClockSpoke = 0;
  let lastTimerSpoke = 0;

  function speakClock(text) {
    const gap  = 10_000;
    const wait = Math.max(0, gap - (Date.now() - lastTimerSpoke));
    setTimeout(() => { lastClockSpoke = Date.now(); speak(text); }, wait);
  }

  function speakTimer(text) {
    const gap  = 10_000;
    const wait = Math.max(0, gap - (Date.now() - lastClockSpoke));
    setTimeout(() => { lastTimerSpoke = Date.now(); speak(text); }, wait);
  }

  // ── Presets — productivity-first ─────────────────────────────────────────
  //  Short focus: 5 10 15 20
  //  Pomodoro:    25
  //  Deep work:   30 45 60
  //  Long:        90 120
  let presetvalues = [5, 10, 15, 20, 25, 30, 45, 60, 90, 120];

  // ── localStorage ──────────────────────────────────────────────────────────
  const lsGet = (k) => localStorage.getItem(k);
  const lsSave = () => {
    localStorage.setItem("SoundChosen",         SoundChosen);
    localStorage.setItem("NoiseVolume",         NoiseVolume.toString());
    localStorage.setItem("SpeakVolume",         SpeakVolume.toString());
    localStorage.setItem("ClockOn",             clockOn.toString());
    localStorage.setItem("ClockIntervalMins",   clockIntervalMins.toString());
    localStorage.setItem("TimerSpeakOn",        timerSpeakOn.toString());
    localStorage.setItem("TimerAnnounceEvery",  timerAnnounceEvery.toString());
  };

  // ── Sound / volume ────────────────────────────────────────────────────────
  const notifysound = "https://www.soundjay.com/clock/sounds/alarm-clock-01.mp3";
  const soundlist = [
    { link: "https://www.soundjay.com/nature/sounds/rain-04.mp3",     title: "Rain"      },
    { link: "https://www.soundjay.com/nature/sounds/waterfall-1.mp3", title: "Waterfall" },
    { link: "https://www.soundjay.com/nature/sounds/fire-1.mp3",      title: "Fire"      },
    { link: "https://www.soundjay.com/nature/sounds/stream-3.mp3",    title: "Stream"    },
  ];
  const volumelists = [
    { volume: 0.1, title: "Very Low"  },
    { volume: 0.2, title: "Low"       },
    { volume: 0.6, title: "Medium"    },
    { volume: 0.8, title: "High"      },
    { volume: 1.0, title: "Very High" },
  ];

  let SoundChosen = lsGet("SoundChosen") || soundlist[0].link;
  let NoiseVolume = lsGet("NoiseVolume") ? parseFloat(lsGet("NoiseVolume")) : 0.2;
  let SpeakVolume = lsGet("SpeakVolume") ? parseFloat(lsGet("SpeakVolume")) : 0.8;

  // Reload + resume audio when source changes
  let audioPlaying = false;
  $: if (audio && SoundChosen) {
    audio.src = SoundChosen;
    audio.load();
    if (audioPlaying) audio.play().catch(() => {});
  }

  // ══════════════════════════════════════════════════════════════════════════
  //  MODULE A — Speaking Clock
  // ══════════════════════════════════════════════════════════════════════════
  const clockIntervalOptions = [1, 2, 5, 10, 15, 20, 30, 60];
  let clockOn           = lsGet("ClockOn") === "true";
  let clockIntervalMins = lsGet("ClockIntervalMins") ? parseInt(lsGet("ClockIntervalMins")) : 30;
  let clockTimer        = null;
  let currentTimeDisplay = "";

  const displayTick = setInterval(() => {
    currentTimeDisplay = new Date().toLocaleTimeString([], {
      hour: "2-digit", minute: "2-digit", hour12: true,
    });
  }, 1000);

  function timeToWords() {
    const d = new Date(), h = d.getHours(), m = d.getMinutes();
    let s = h === 0 ? "12" : h > 12 ? String(h - 12) : String(h);
    if      (m === 0) s += " o'clock";
    else if (m < 10)  s += " oh " + m;
    else              s += " " + m;
    s += h < 12 ? " AM" : " PM";
    return s;
  }

  function startClock() {
    stopClock();
    speakClock(timeToWords());
    clockTimer = setInterval(() => speakClock(timeToWords()), clockIntervalMins * 60_000);
  }
  function stopClock() { if (clockTimer) { clearInterval(clockTimer); clockTimer = null; } }
  function toggleClock() { clockOn = !clockOn; lsSave(); clockOn ? startClock() : stopClock(); }
  function onClockIntervalChange() { lsSave(); if (clockOn) startClock(); }

  if (clockOn) setTimeout(() => startClock(), 200);

  // ══════════════════════════════════════════════════════════════════════════
  //  MODULE B — Timer Speech
  // ══════════════════════════════════════════════════════════════════════════
  let timerSpeakOn = lsGet("TimerSpeakOn") !== "false";
  // How often to announce remaining time (every N minutes)
  const timerAnnounceOptions = [1, 2, 5, 10, 15, 20, 30];
  let timerAnnounceEvery = lsGet("TimerAnnounceEvery")
    ? parseInt(lsGet("TimerAnnounceEvery"))
    : 1;

  const tick = () => {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    seconds--;
    timervalue = `${mins >= 10 ? mins : "0" + mins}:${secs >= 10 ? secs : "0" + secs}`;

    if (secs === 0 && seconds !== 0 && timerSpeakOn && mins % timerAnnounceEvery === 0)
      speakTimer(`${mins} minute${mins !== 1 ? "s" : ""} remaining`);

    if (seconds === 0) {
      resetTimer();
      if (timerSpeakOn) speakTimer("Timer finished");
      notifyaudio.currentTime = 0;
      notifyaudio.play().catch(() => {});
      setTimeout(() => { notifyaudio.pause(); notifyaudio.currentTime = 0; }, 10_000);
    }
  };

  const startTimer = () => {
    lsSave();
    if (seconds === 0) seconds = slidervalue * 60;
    if (interval) return;
    interval = setInterval(tick, 1000);
    audio.volume = NoiseVolume;
    audioPlaying = true;
    audio.play().catch(() => {});
  };

  const stopTimer = () => {
    clearInterval(interval);
    interval = null;
    audioPlaying = false;
    audio.pause();
  };

  const resetTimer = () => { stopTimer(); seconds = 0; timervalue = "00:00"; };

  const choosePreset = (e) => {
    slidervalue = parseInt(e.target.value);
    resetTimer();
    startTimer();
  };

  onDestroy(() => { stopClock(); clearInterval(interval); clearInterval(displayTick); });
</script>

<svelte:head>
  <title>Speaktimer</title>
  <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
  <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png" />
  <link rel="manifest" href="/site.webmanifest" />
</svelte:head>

<div class="shell">
  <div class="grid">

    <!-- ═══ MODULE A : Speaking Clock ═══ -->
    <section class="panel clock-panel" class:active={clockOn}>
      <h3>🕐 Speaking Clock <span class="tag">A</span></h3>
      <div class="clock-face">{currentTimeDisplay}</div>
      <p class="label">Announce every</p>
      <select bind:value={clockIntervalMins} on:change={onClockIntervalChange}>
        {#each clockIntervalOptions as m}
          <option value={m}>{m} min</option>
        {/each}
      </select>
      <button class="toggle {clockOn ? 'is-on' : 'is-off'}" on:click={toggleClock}>
        {clockOn ? "🔔 ON" : "🔕 OFF"}
      </button>
    </section>

    <!-- ═══ MODULE B : Timer ═══ -->
    <section class="panel timer-panel">
      <h3>⏱ Timer <span class="tag">B</span></h3>
      <div class="countdown">{timervalue}</div>
      <p class="label">{slidervalue} min &nbsp;·&nbsp;
        <span class="voices-note">{voices.length} voice{voices.length !== 1 ? "s" : ""} loaded</span>
      </p>
      <input type="range" class="slider" min="1" max="120" bind:value={slidervalue} />
      <div class="btn-row">
        <button on:click={startTimer}>▶ Start</button>
        <button on:click={stopTimer}>⏸ Stop</button>
        <button on:click={resetTimer}>↺ Reset</button>
      </div>
      <label class="check-label">
        <input type="checkbox" bind:checked={timerSpeakOn} on:change={lsSave} />
        Speak remaining — every
      </label>
      <select bind:value={timerAnnounceEvery} on:change={lsSave} disabled={!timerSpeakOn}>
        {#each timerAnnounceOptions as m}
          <option value={m}>{m} min</option>
        {/each}
      </select>
    </section>

    <!-- ═══ Presets ═══ -->
    <section class="panel presets-panel">
      <h3>⚡ Presets</h3>
      <p class="label">Tap to start instantly</p>
      <div class="presets-grid">
        {#each presetvalues as p}
          <button
            class="preset-btn"
            class:pomodoro={p === 25}
            value={p}
            on:click={(e) => choosePreset(e)}
            title={p === 25 ? "Pomodoro" : p >= 90 ? "Deep work" : p <= 10 ? "Short break" : ""}
          >{p}</button>
        {/each}
      </div>
      <p class="preset-hint">🍅 25 = Pomodoro &nbsp;·&nbsp; bold = deep work</p>
    </section>

    <!-- ═══ Settings ═══ -->
    <section class="panel prefs-panel">
      <h3>⚙ Settings</h3>
      <p class="label">Background sound</p>
      <select bind:value={SoundChosen} on:change={lsSave}>
        {#each soundlist as s}<option value={s.link}>{s.title}</option>{/each}
      </select>
      <p class="label">Noise volume</p>
      <select bind:value={NoiseVolume} on:change={() => { if (audio) audio.volume = NoiseVolume; lsSave(); }}>
        {#each volumelists as v}<option value={v.volume}>{v.title}</option>{/each}
      </select>
      <p class="label">Speech volume</p>
      <select bind:value={SpeakVolume} on:change={lsSave}>
        {#each volumelists as v}<option value={v.volume}>{v.title}</option>{/each}
      </select>
      <p class="queue-note">
        {#if isSpeechActive}🔉 Speaking…{:else if speechQueue.length > 0}⏳ {speechQueue.length} queued{:else}✔ Ready{/if}
        &nbsp;·&nbsp; A↔B gap: 10 s
      </p>
    </section>

  </div>
</div>

<!-- bind:this lets JS control play/pause; src set reactively in script -->
<audio bind:this={audio} loop />
<audio bind:this={notifyaudio} src={notifysound} />

<style>
  @import url("https://fonts.googleapis.com/css2?family=Rubik:wght@400;500;700&display=swap");

  :root {
    --primary:  #440d49;
    --accent:   #c3ddc0;
    --bg:       #d4bff9;
    --muted:    rgba(68,13,73,0.55);
  }

  * { box-sizing: border-box; margin: 0; padding: 0; }

  :global(body) {
    background: var(--bg);
    font-family: "Rubik", sans-serif;
    font-size: 13px;
    overflow: hidden;
  }

  .shell {
    height: 100vh;
    padding: 0.55rem;
    display: flex;
    align-items: stretch;
  }

  .grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    grid-template-rows: 1fr 1fr;
    gap: 0.45rem;
    width: 100%;
  }

  .panel {
    background: var(--bg);
    border: 2px dashed var(--primary);
    border-radius: 6px;
    padding: 0.6rem 0.75rem;
    display: flex;
    flex-direction: column;
    gap: 0.32rem;
    overflow: hidden;
  }

  .panel.active {
    border-style: solid;
    box-shadow: 0 0 0 2px var(--primary);
  }

  h3 {
    font-size: 0.82rem;
    font-weight: 700;
    color: var(--primary);
    display: flex;
    align-items: center;
    gap: 0.35rem;
  }

  .tag {
    font-size: 0.58rem;
    font-weight: 700;
    background: var(--primary);
    color: var(--accent);
    padding: 0.1rem 0.35rem;
    border-radius: 3px;
    letter-spacing: 0.06em;
  }

  .label {
    font-size: 0.7rem;
    font-weight: 500;
    color: var(--muted);
  }

  .voices-note { font-weight: 400; }

  /* Clock face */
  .clock-face {
    font-size: 1.8rem;
    font-weight: 700;
    color: var(--primary);
    letter-spacing: 0.03em;
  }

  /* Countdown */
  .countdown {
    font-size: 2rem;
    font-weight: 700;
    color: var(--primary);
    background: var(--accent);
    border: 2px solid var(--primary);
    border-radius: 5px;
    text-align: center;
    padding: 0.3rem;
    letter-spacing: 0.08em;
    font-variant-numeric: tabular-nums;
  }

  /* Slider */
  .slider {
    width: 100%;
    height: 8px;
    -webkit-appearance: none;
    background: var(--accent);
    border: 2px solid var(--primary);
    border-radius: 4px;
    outline: none;
    cursor: pointer;
    overflow: hidden;
  }
  .slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 14px; height: 14px;
    border-radius: 50%;
    background: var(--accent);
    border: 3px solid var(--primary);
    cursor: pointer;
    box-shadow: -300px 0 0 296px var(--primary);
  }

  /* Selects */
  select {
    width: 100%;
    background: var(--accent);
    color: var(--primary);
    border: 2px solid var(--primary);
    border-radius: 5px;
    padding: 0.28rem 0.5rem;
    font-size: 0.76rem;
    font-family: inherit;
    font-weight: 500;
    outline: none;
    cursor: pointer;
  }

  /* Buttons */
  button {
    font-family: inherit;
    font-size: 0.8rem;
    font-weight: 700;
    background: var(--accent);
    color: var(--primary);
    border: 2px solid var(--primary);
    border-radius: 5px;
    padding: 0.32rem 0.65rem;
    cursor: pointer;
    transition: opacity 0.15s;
  }
  button:hover { opacity: 0.7; }

  .btn-row { display: flex; gap: 0.35rem; }
  .btn-row button { flex: 1; }

  /* Toggle */
  .toggle { width: fit-content; font-size: 0.82rem; padding: 0.32rem 1rem; }
  .is-on  { background: var(--primary); color: var(--accent); }
  .is-off { background: var(--accent);  color: var(--primary); }

  /* Checkbox */
  .check-label {
    display: flex; align-items: center; gap: 0.35rem;
    font-size: 0.7rem; color: var(--muted); cursor: pointer;
  }
  .check-label input { accent-color: var(--primary); cursor: pointer; }

  /* Presets */
  .presets-grid {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    gap: 0.28rem;
    overflow-y: auto;
    flex: 1;
  }
  .preset-btn {
    padding: 0.32rem 0.1rem;
    font-size: 0.78rem;
    text-align: center;
  }
  .preset-btn:hover         { background: var(--primary); color: var(--accent); }
  .preset-btn.pomodoro      { border-style: solid; border-width: 2px; }
  .preset-hint {
    font-size: 0.62rem;
    color: var(--muted);
    margin-top: 0.1rem;
  }

  /* Queue note */
  .queue-note {
    font-size: 0.67rem;
    color: var(--muted);
    margin-top: auto;
    border-top: 1px dashed var(--primary);
    padding-top: 0.35rem;
  }

  /* Mobile */
  @media (max-width: 540px) {
    :global(body) { overflow: auto; }
    .grid { grid-template-columns: 1fr; grid-template-rows: auto; }
    .shell { height: auto; }
    .clock-face, .countdown { font-size: 1.5rem; }
    .presets-grid { grid-template-columns: repeat(5, 1fr); }
  }
</style>

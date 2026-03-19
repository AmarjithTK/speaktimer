<script>
  import { onDestroy } from "svelte";

  // ── Random palette on every visit ────────────────────────────────────────
  const palettes = [
    { primary: "#1A051D", accent: "#E5D4F5", bg: "#F4EFFF" }, // HC lavender
    { primary: "#0A1824", accent: "#CDE8F5", bg: "#EAF6FF" }, // HC ocean blue
    { primary: "#201004", accent: "#FCEFD4", bg: "#FFF8EA" }, // HC warm sand
    { primary: "#0D1E16", accent: "#C8F0DA", bg: "#EEFAF3" }, // HC forest green
    { primary: "#2A0B0B", accent: "#F5DCD4", bg: "#FFF0EB" }, // HC terracotta
    { primary: "#0D0D2A", accent: "#D8DBF5", bg: "#F1F2FF" }, // HC midnight indigo
    { primary: "#210D2C", accent: "#EEDDF7", bg: "#F8F0FF" }, // HC deep violet
    { primary: "#051A1A", accent: "#C4EEEC", bg: "#EFFFFD" }, // HC teal
    { primary: "#2A1C00", accent: "#FAEFD4", bg: "#FFFBEA" }, // HC amber
    { primary: "#14051D", accent: "#E5D4F5", bg: "#F6EEFF" }, // HC plum
    { primary: "#200D00", accent: "#FFE8C8", bg: "#FFF4EA" }, // HC burnt orange
    { primary: "#001E1E", accent: "#C0F5EE", bg: "#EAFFFC" }, // HC aqua
    { primary: "#000000", accent: "#E0E0E0", bg: "#FFFFFF" }, // HC monochrome
    { primary: "#1C0017", accent: "#FAD4F2", bg: "#FFF0FA" }, // HC rose
    { primary: "#001C0C", accent: "#D4FAE0", bg: "#EEFFF3" }, // HC mint
    { primary: "#0B141C", accent: "#D4E0F5", bg: "#F0F5FF" }, // HC slate blue
    { primary: "#1A1A00", accent: "#F5F0C8", bg: "#FEFFEA" }, // HC olive
    { primary: "#25000F", accent: "#FAD4DC", bg: "#FFF0F4" }, // HC crimson blush
    { primary: "#000B1C", accent: "#D4E4FA", bg: "#F0F6FF" }, // HC royal navy
    { primary: "#0B1C00", accent: "#DFF5C4", bg: "#F4FFEA" }, // HC lime
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
  //  - Cycles voices for variety (web: by voice object; native: by pitch/rate)
  //  - Modules A & B are kept ≥ 10 s apart
  // ══════════════════════════════════════════════════════════════════════════
  let speechQueue    = [];
  let isSpeechActive = false;
  let voices         = [];
  let voiceIndex     = 0;

  // Web Speech API voice loading
  function loadVoices() {
    const v = speechSynthesis.getVoices();
    if (v.length) voices = v.filter(v => v.lang.startsWith("en"));
  }
  loadVoices();
  if (typeof speechSynthesis !== "undefined")
    speechSynthesis.onvoiceschanged = loadVoices;

  // ── Pleasant voice picker ─────────────────────────────────────────────────
  // Ranks available English voices: Google > known-pleasant names > first available
  const PLEASANT_NAMES = [
    "google", "samantha", "karen", "victoria", "moira", "fiona",
    "veena", "tessa", "allison", "ava", "susan", "zira",
  ];
  function pleasantVoice() {
    if (!voices.length) return null;
    for (const name of PLEASANT_NAMES) {
      const v = voices.find(v => v.name.toLowerCase().includes(name));
      if (v) return v;
    }
    return voices[0];
  }

  // ── Motivational time-awareness quotes (cyclic) ───────────────────────────
  const quotes = [
    "Use this moment well — it won't come back.",
    "Small steps every hour build the life you want.",
    "Your attention is your most valuable currency.",
    "Time is the only resource you cannot earn back.",
    "What you do right now shapes who you become.",
    "Every minute of focus is an investment in your future.",
    "Be present. This hour is a gift.",
    "Clarity comes to those who use their time with intention.",
    "Progress, not perfection, is what time rewards.",
    "An hour of deep work is worth a day of distraction.",
    "Don't count the hours; make the hours count.",
    "Your future self will thank you for the work you do now.",
    "One focused hour can change a whole day.",
    "Time flies — but you are the pilot.",
    "Do something today that your future self will be proud of.",
    "Momentum is built one intentional moment at a time.",
    "The best time to start was yesterday. The second best is now.",
    "Each hour is a fresh canvas. Paint it well.",
    "Discipline is choosing what you want most over what you want now.",
    "Greatness is built minute by minute.",
    "A year from now you'll wish you had started today.",
    "Your work right now is compounding silently.",
    "Focused effort now creates freedom later.",
    "Every hour of rest is fuel. Every hour of work is progress.",
    "Time is the great equaliser — what matters is what you do with it.",
    "Stay the course. The results are coming.",
    "Consistency over time is unstoppable.",
    "You have enough time for what truly matters.",
    "Let this hour be better than the last.",
    "Breathe, focus, and make this moment count.",
  ];
  let quoteIndex = 0;

  function drainQueue() {
    if (isSpeechActive || speechQueue.length === 0) return;
    isSpeechActive = true;
    
    // Dequeue item and parse string vs object (for backwards compatibility if just a string)
    const item = speechQueue.shift();
    const isObj = typeof item === "object";
    const text  = isObj ? item.text : item;
    const isQuote = isObj ? item.isQuote : false;
    const delay = isObj ? (item.delay || 0) : 0;

    setTimeout(() => {
      const speech = new SpeechSynthesisUtterance(text);
      if (isQuote) {
        const pv = pleasantVoice();
        if (pv) speech.voice = pv;
        speech.pitch  = 1.05;
        speech.rate   = 0.9;
      } else {
        if (voices.length > 0) {
          speech.voice = voices[voiceIndex % voices.length];
          voiceIndex++;
        }
        speech.pitch  = 0.75;
        speech.rate   = 0.95;
      }
      speech.volume = SpeakVolume;
      speech.onend  = () => { isSpeechActive = false; drainQueue(); };
      speech.onerror = () => { isSpeechActive = false; drainQueue(); };
      speechSynthesis.speak(speech);
    }, delay);
  }

  function speak(text) { speechQueue.push({ text }); drainQueue(); }

  // ── 10-second gap enforcement between Module A & B ────────────────────────
  let lastClockSpoke = 0;
  let lastTimerSpoke = 0;

  function speakClock(text) {
    const gap  = 10_000;
    const wait = Math.max(0, gap - (Date.now() - lastTimerSpoke));
    setTimeout(() => {
      lastClockSpoke = Date.now();
      speak(text);
      
      // Queue quote 5 seconds after time announcement finishes
      if (motivationOn) {
        const quoteText = quotes[quoteIndex % quotes.length];
        quoteIndex++;
        speechQueue.push({ text: quoteText, isQuote: true, delay: 5_000 });
      }
      drainQueue();
    }, wait);
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
    localStorage.setItem("MotivationOn",        motivationOn.toString());
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
  let motivationOn      = lsGet("MotivationOn") !== "false";
  let clockTimer        = null;
  let currentTimeDisplay = "";

  const displayTick = setInterval(() => {
    const now = new Date();
    const h = now.getHours();
    const m = now.getMinutes().toString().padStart(2, '0');
    const s = now.getSeconds().toString().padStart(2, '0');
    const ms = now.getMilliseconds().toString().padStart(3, '0');
    const ampm = h >= 12 ? 'PM' : 'AM';
    const h12 = h % 12 || 12;
    const hStr = h12.toString().padStart(2, '0');
    currentTimeDisplay = `${hStr}:${m}:${s}.${ms} ${ampm}`;
  }, 30);

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
      <label class="check-label">
        <input type="checkbox" bind:checked={motivationOn} on:change={lsSave} />
        Speak motivational quote after time
      </label>
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
    font-variant-numeric: tabular-nums;
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

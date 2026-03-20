<script>
  import { onDestroy } from "svelte";

  const palettes = [
    { primary: "#1A051D", accent: "#E5D4F5", bg: "#F4EFFF" },
    { primary: "#0A1824", accent: "#CDE8F5", bg: "#EAF6FF" },
    { primary: "#201004", accent: "#FCEFD4", bg: "#FFF8EA" },
    { primary: "#0D1E16", accent: "#C8F0DA", bg: "#EEFAF3" },
    { primary: "#2A0B0B", accent: "#F5DCD4", bg: "#FFF0EB" },
    { primary: "#0D0D2A", accent: "#D8DBF5", bg: "#F1F2FF" },
    { primary: "#210D2C", accent: "#EEDDF7", bg: "#F8F0FF" },
    { primary: "#051A1A", accent: "#C4EEEC", bg: "#EFFFFD" },
    { primary: "#2A1C00", accent: "#FAEFD4", bg: "#FFFBEA" },
    { primary: "#14051D", accent: "#E5D4F5", bg: "#F6EEFF" },
    { primary: "#200D00", accent: "#FFE8C8", bg: "#FFF4EA" },
    { primary: "#001E1E", accent: "#C0F5EE", bg: "#EAFFFC" },
    { primary: "#000000", accent: "#E0E0E0", bg: "#FFFFFF" },
    { primary: "#1C0017", accent: "#FAD4F2", bg: "#FFF0FA" },
    { primary: "#001C0C", accent: "#D4FAE0", bg: "#EEFFF3" },
    { primary: "#0B141C", accent: "#D4E0F5", bg: "#F0F5FF" },
    { primary: "#1A1A00", accent: "#F5F0C8", bg: "#FEFFEA" },
    { primary: "#25000F", accent: "#FAD4DC", bg: "#FFF0F4" },
    { primary: "#000B1C", accent: "#D4E4FA", bg: "#F0F6FF" },
    { primary: "#0B1C00", accent: "#DFF5C4", bg: "#F4FFEA" },
  ];

  const palette = palettes[Math.floor(Math.random() * palettes.length)];
  const root = document.documentElement;
  root.style.setProperty("--primary", palette.primary);
  root.style.setProperty("--accent", palette.accent);
  root.style.setProperty("--bg", palette.bg);

  let audio, notifyaudio;
  let wakeLock = null;

  const lsGet = (k) => localStorage.getItem(k);
  const lsSet = (k, v) => localStorage.setItem(k, String(v));

  let slidervalue = 25;
  let timervalue = "00:00";
  let interval = null;
  let seconds = 0;

  let speechQueue = [];
  let isSpeechActive = false;
  let voices = [];
  let voiceIndex = 0;

  const PLEASANT_NAMES = [
    "google", "samantha", "karen", "victoria", "moira", "fiona",
    "veena", "tessa", "allison", "ava", "susan", "zira",
  ];

  function loadVoices() {
    if (typeof speechSynthesis === "undefined") return;
    const all = speechSynthesis.getVoices();
    voices = all.filter((v) => {
      const lang = v.lang?.toLowerCase() || "";
      return lang.startsWith("en") || lang.startsWith("ml");
    });
  }

  loadVoices();
  if (typeof speechSynthesis !== "undefined") {
    speechSynthesis.onvoiceschanged = loadVoices;
  }

  let voiceListMode = lsGet("VoiceListMode") || "pleasant";
  let favoriteVoiceKey = lsGet("FavoriteVoiceKey") || "__auto__";
  let speechLanguage = lsGet("SpeechLanguage") || "en";
  let languageVoices = [];
  let availableVoices = [];

  $: languageVoices =
    speechLanguage === "ml"
      ? voices.filter((voice) => (voice.lang || "").toLowerCase().startsWith("ml"))
      : voices.filter((voice) => (voice.lang || "").toLowerCase().startsWith("en"));

  $: availableVoices =
    voiceListMode === "all"
      ? languageVoices
      : languageVoices.filter((voice) =>
          PLEASANT_NAMES.some((name) => voice.name.toLowerCase().includes(name))
        );

  $: if (voiceListMode === "pleasant" && availableVoices.length === 0) {
    availableVoices = languageVoices;
  }

  function preferredVoice() {
    const scopedByLanguage = languageVoices.length ? languageVoices : voices;
    if (!scopedByLanguage.length) return null;
    if (favoriteVoiceKey !== "__auto__") {
      const selected = scopedByLanguage.find((v) => `${v.name}|${v.lang}` === favoriteVoiceKey);
      if (selected) return selected;
    }
    const scoped = availableVoices.length ? availableVoices : scopedByLanguage;
    return scoped[0] || scopedByLanguage[0];
  }

  const motivationCategories = [
    "General",
    "Focus",
    "Discipline",
    "Calm",
    "Positivity",
    "Historic Figures",
  ];

  const motivationDelayOptions = [5, 10, 20, 30, 40, 60];

  const quotesByCategory = {
    General: [
      "Use this moment well — it won't come back.",
      "Small steps every hour build the life you want.",
      "Time is the only resource you cannot earn back.",
      "Progress, not perfection, is what time rewards.",
      "Your work right now is compounding silently.",
    ],
    Focus: [
      "Your attention is your most valuable currency.",
      "An hour of deep work is worth a day of distraction.",
      "One focused hour can change a whole day.",
      "Clarity comes to those who use their time with intention.",
      "Focused effort now creates freedom later.",
    ],
    Discipline: [
      "Discipline is choosing what you want most over what you want now.",
      "Greatness is built minute by minute.",
      "The best time to start was yesterday. The second best is now.",
      "Consistency over time is unstoppable.",
      "Stay the course. The results are coming.",
    ],
    Calm: [
      "Be present. This hour is a gift.",
      "Breathe, focus, and make this moment count.",
      "Each hour is a fresh canvas. Paint it well.",
      "You have enough time for what truly matters.",
      "Let this hour be better than the last.",
    ],
    Positivity: [
      "What you do right now shapes who you become.",
      "Your future self will thank you for the work you do now.",
      "Do something today that your future self will be proud of.",
      "Momentum is built one intentional moment at a time.",
      "A year from now you'll wish you had started today.",
    ],
    "Historic Figures": [
      "Aristotle said: We are what we repeatedly do. Excellence, then, is a habit.",
      "Leonardo da Vinci said: Time stays long enough for anyone who will use it.",
      "Benjamin Franklin said: Lost time is never found again.",
      "Maya Angelou said: Nothing will work unless you do.",
      "Bruce Lee said: The successful warrior is the average person, with laser-like focus.",
    ],
  };

  const quoteIndexByCategory = {};

  function nextQuote(category) {
    const safe = quotesByCategory[category] ? category : "General";
    const group = quotesByCategory[safe];
    const idx = quoteIndexByCategory[safe] || 0;
    quoteIndexByCategory[safe] = idx + 1;
    return group[idx % group.length];
  }

  function drainQueue() {
    if (isSpeechActive || speechQueue.length === 0) return;
    isSpeechActive = true;

    const item = speechQueue.shift();
    const isObj = typeof item === "object";
    const text = isObj ? item.text : item;
    const isQuote = isObj ? item.isQuote : false;
    const delay = isObj ? (item.delay || 0) : 0;

    setTimeout(() => {
      const speech = new SpeechSynthesisUtterance(text);
      const voice = preferredVoice();
      if (voice) {
        speech.voice = voice;
      } else if (voices.length > 0) {
        speech.voice = voices[voiceIndex % voices.length];
        voiceIndex++;
      }

      if (isQuote) {
        speech.pitch = 1.05;
        speech.rate = 0.9;
      } else {
        speech.pitch = 0.75;
        speech.rate = 0.95;
      }

      speech.volume = SpeakVolume;
      speech.onend = () => {
        isSpeechActive = false;
        drainQueue();
      };
      speech.onerror = () => {
        isSpeechActive = false;
        drainQueue();
      };
      speechSynthesis.speak(speech);
    }, delay);
  }

  function speak(text) {
    speechQueue.push({ text });
    drainQueue();
  }

  let lastClockSpoke = 0;
  let lastTimerSpoke = 0;

  let motivationOn = lsGet("MotivationOn") !== "false";
  let motivationCategory = lsGet("MotivationCategory") || "General";
  if (!motivationCategories.includes(motivationCategory)) {
    motivationCategory = "General";
  }
  let motivationDelaySeconds = lsGet("MotivationDelaySeconds")
    ? parseInt(lsGet("MotivationDelaySeconds"))
    : 10;

  function speakClock(text) {
    const gap = 10_000;
    const wait = Math.max(0, gap - (Date.now() - lastTimerSpoke));
    setTimeout(() => {
      lastClockSpoke = Date.now();
      speak(text);
      if (motivationOn) {
        speechQueue.push({
          text: nextQuote(motivationCategory),
          isQuote: true,
          delay: motivationDelaySeconds * 1000,
        });
        drainQueue();
      }
    }, wait);
  }

  function speakTimer(text) {
    const gap = 10_000;
    const wait = Math.max(0, gap - (Date.now() - lastClockSpoke));
    setTimeout(() => {
      lastTimerSpoke = Date.now();
      speak(text);
    }, wait);
  }

  let presetvalues = [5, 10, 15, 20, 25, 30, 45, 60, 90, 120];
  let timerSpeakOn = lsGet("TimerSpeakOn") !== "false";
  let timerNoiseOn = lsGet("TimerNoiseOn") !== "false";
  const timerAnnounceOptions = [1, 2, 5, 10, 15, 20, 30];
  let timerAnnounceEvery = lsGet("TimerAnnounceEvery")
    ? parseInt(lsGet("TimerAnnounceEvery"))
    : 1;

  let chainModeOn = lsGet("ChainModeOn") === "true";
  const chainPresets = {
    "Pomodoro 25-5x4": [25, 5, 25, 5, 25, 5, 25, 15],
    "Sprint 50-10x2": [50, 10, 50, 10],
    "Quick 15-3x3": [15, 3, 15, 3, 15, 3],
  };
  let chainPresetKey = lsGet("ChainPresetKey") || "Pomodoro 25-5x4";
  if (!chainPresets[chainPresetKey]) {
    chainPresetKey = "Pomodoro 25-5x4";
  }
  let chainIndex = 0;

  const clockIntervalOptions = [1, 2, 5, 10, 15, 20, 30, 60];
  let clockOn = lsGet("ClockOn") === "true";
  let clockIntervalMins = lsGet("ClockIntervalMins")
    ? parseInt(lsGet("ClockIntervalMins"))
    : 30;
  let clockTimer = null;
  let currentTimeDisplay = "";

  const displayTick = setInterval(() => {
    const now = new Date();
    const h = now.getHours();
    const m = now.getMinutes().toString().padStart(2, "0");
    const s = now.getSeconds().toString().padStart(2, "0");
    const ms = now.getMilliseconds().toString().padStart(3, "0");
    const ampm = h >= 12 ? "PM" : "AM";
    const h12 = h % 12 || 12;
    const hStr = h12.toString().padStart(2, "0");
    currentTimeDisplay = `${hStr}:${m}:${s}.${ms} ${ampm}`;
  }, 30);

  function timeToWords() {
    const d = new Date();
    const h = d.getHours();
    const m = d.getMinutes();
    let s = h === 0 ? "12" : h > 12 ? String(h - 12) : String(h);
    if (m === 0) s += " o'clock";
    else if (m < 10) s += " oh " + m;
    else s += " " + m;
    s += h < 12 ? " AM" : " PM";
    return s;
  }

  function startClock() {
    stopClock();
    speakClock(timeToWords());
    clockTimer = setInterval(() => speakClock(timeToWords()), clockIntervalMins * 60_000);
  }

  function stopClock() {
    if (clockTimer) {
      clearInterval(clockTimer);
      clockTimer = null;
    }
  }

  function toggleClock() {
    clockOn = !clockOn;
    lsSave();
    if (clockOn) startClock();
    else stopClock();
  }

  function formatTimer(v) {
    const mins = Math.floor(v / 60)
      .toString()
      .padStart(2, "0");
    const secs = (v % 60).toString().padStart(2, "0");
    return `${mins}:${secs}`;
  }

  function handleTimerFinished() {
    if (chainModeOn) {
      const sequence = chainPresets[chainPresetKey] || [25];
      if (chainIndex < sequence.length - 1) {
        chainIndex += 1;
        const nextMinutes = sequence[chainIndex];
        seconds = nextMinutes * 60;
        timervalue = `${String(nextMinutes).padStart(2, "0")}:00`;
        if (timerSpeakOn) {
          speakTimer(`Starting next timer: ${nextMinutes} minute${nextMinutes !== 1 ? "s" : ""}`);
        }
        return;
      }
      chainIndex = 0;
    }

    resetTimer();
    if (timerSpeakOn) speakTimer("Timer finished");
    notifyaudio.currentTime = 0;
    notifyaudio.play().catch(() => {});
    setTimeout(() => {
      notifyaudio.pause();
      notifyaudio.currentTime = 0;
    }, 10_000);
  }

  function tick() {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    timervalue = formatTimer(seconds);
    seconds -= 1;

    if (secs === 0 && seconds !== 0 && timerSpeakOn && mins % timerAnnounceEvery === 0) {
      speakTimer(`${mins} minute${mins !== 1 ? "s" : ""} remaining`);
    }

    if (seconds <= 0) {
      handleTimerFinished();
    }
  }

  const startTimer = () => {
    lsSave();
    if (seconds === 0) {
      if (chainModeOn) {
        const sequence = chainPresets[chainPresetKey] || [25];
        if (chainIndex >= sequence.length) chainIndex = 0;
        seconds = sequence[chainIndex] * 60;
      } else {
        seconds = slidervalue * 60;
      }
      timervalue = formatTimer(seconds);
    }

    if (interval) return;

    interval = setInterval(tick, 1000);
    updateNoisePlayback();
  };

  const stopTimer = () => {
    clearInterval(interval);
    interval = null;
    audioPlaying = false;
    audio.pause();
  };

  const resetTimer = () => {
    stopTimer();
    seconds = 0;
    timervalue = "00:00";
    chainIndex = 0;
  };

  const choosePreset = (e) => {
    slidervalue = parseInt(e.target.value);
    resetTimer();
    startTimer();
  };

  function onClockIntervalChange() {
    lsSave();
    if (clockOn) startClock();
  }

  const notifysound = "https://www.soundjay.com/clock/sounds/alarm-clock-01.mp3";
  const soundlist = [
    { link: "https://www.soundjay.com/nature/sounds/rain-04.mp3", title: "Rain" },
    { link: "https://www.soundjay.com/nature/sounds/waterfall-1.mp3", title: "Waterfall" },
    { link: "https://www.soundjay.com/nature/sounds/fire-1.mp3", title: "Fire" },
    { link: "https://www.soundjay.com/nature/sounds/stream-3.mp3", title: "Stream" },
  ];

  const volumelists = [
    { volume: 0.1, title: "Very Low" },
    { volume: 0.2, title: "Low" },
    { volume: 0.6, title: "Medium" },
    { volume: 0.8, title: "High" },
    { volume: 1.0, title: "Very High" },
  ];

  let SoundChosen = lsGet("SoundChosen") || soundlist[0].link;
  let NoiseVolume = lsGet("NoiseVolume") ? parseFloat(lsGet("NoiseVolume")) : 0.2;
  let SpeakVolume = lsGet("SpeakVolume") ? parseFloat(lsGet("SpeakVolume")) : 0.8;
  let audioPlaying = false;

  let fullscreenDarkTheme = lsGet("FullscreenDarkTheme") !== "false";
  let fullscreenDimBrightness = lsGet("FullscreenDimBrightness") === "true";
  let fullscreenStartLandscape = lsGet("FullscreenStartLandscape") === "true";

  let showFullscreenFocus = false;
  let fullscreenShowTimer = false;
  let showFullscreenControls = true;
  let fullscreenControlsHideTimer = null;

  function restartFullscreenControlsTimer() {
    clearTimeout(fullscreenControlsHideTimer);
    fullscreenControlsHideTimer = setTimeout(() => {
      showFullscreenControls = false;
    }, 5000);
  }

  async function enterFullscreenApi() {
    if (document.fullscreenElement) return;
    if (document.documentElement.requestFullscreen) {
      await document.documentElement.requestFullscreen().catch(() => {});
    }
  }

  async function exitFullscreenApi() {
    if (document.fullscreenElement && document.exitFullscreen) {
      await document.exitFullscreen().catch(() => {});
    }
  }

  async function requestWakeLock() {
    if (!("wakeLock" in navigator)) return;
    try {
      wakeLock = await navigator.wakeLock.request("screen");
    } catch (_) {}
  }

  function releaseWakeLock() {
    if (wakeLock) {
      wakeLock.release().catch(() => {});
      wakeLock = null;
    }
  }

  async function openFullscreenFocus() {
    showFullscreenFocus = true;
    fullscreenShowTimer = interval != null;
    showFullscreenControls = true;
    restartFullscreenControlsTimer();
    await enterFullscreenApi();
    await requestWakeLock();
  }

  async function closeFullscreenFocus() {
    showFullscreenFocus = false;
    showFullscreenControls = true;
    clearTimeout(fullscreenControlsHideTimer);
    releaseWakeLock();
    await exitFullscreenApi();
  }

  function toggleFullscreenOverlayControls() {
    showFullscreenControls = !showFullscreenControls;
    if (showFullscreenControls) restartFullscreenControlsTimer();
    else clearTimeout(fullscreenControlsHideTimer);
  }

  function handleFullscreenControlInteraction() {
    showFullscreenControls = true;
    restartFullscreenControlsTimer();
  }

  function lsSave() {
    lsSet("SoundChosen", SoundChosen);
    lsSet("NoiseVolume", NoiseVolume);
    lsSet("SpeakVolume", SpeakVolume);
    lsSet("ClockOn", clockOn);
    lsSet("ClockIntervalMins", clockIntervalMins);
    lsSet("MotivationOn", motivationOn);
    lsSet("MotivationCategory", motivationCategory);
    lsSet("MotivationDelaySeconds", motivationDelaySeconds);
    lsSet("TimerNoiseOn", timerNoiseOn);
    lsSet("TimerSpeakOn", timerSpeakOn);
    lsSet("TimerAnnounceEvery", timerAnnounceEvery);
    lsSet("ChainModeOn", chainModeOn);
    lsSet("ChainPresetKey", chainPresetKey);
    lsSet("VoiceListMode", voiceListMode);
    lsSet("FavoriteVoiceKey", favoriteVoiceKey);
    lsSet("SpeechLanguage", speechLanguage);
    lsSet("FullscreenDarkTheme", fullscreenDarkTheme);
    lsSet("FullscreenDimBrightness", fullscreenDimBrightness);
    lsSet("FullscreenStartLandscape", fullscreenStartLandscape);
  }

  function updateNoisePlayback() {
    if (!audio) return;
    if (interval && timerNoiseOn) {
      audio.volume = NoiseVolume;
      if (!audioPlaying) {
        audioPlaying = true;
      }
      audio.play().catch(() => {});
    } else {
      audioPlaying = false;
      audio.pause();
    }
  }

  if (clockOn) setTimeout(() => startClock(), 200);

  function onChainModeToggle() {
    chainIndex = 0;
    lsSave();
  }

  function onChainPresetChange() {
    chainIndex = 0;
    lsSave();
  }

  onDestroy(() => {
    stopClock();
    clearInterval(interval);
    clearInterval(displayTick);
    clearTimeout(fullscreenControlsHideTimer);
    releaseWakeLock();
  });
</script>

<svelte:head>
  <title>Speaktimer</title>
  <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
  <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png" />
  <link rel="manifest" href="/site.webmanifest" />
</svelte:head>

<div class="shell">
  <div class="toolbar">
    <div class="app-title">Lifer</div>
    <button on:click={openFullscreenFocus} title="Fullscreen Focus">⛶ Fullscreen Focus</button>
  </div>

  <div class="grid">
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

      <p class="label">Motivation category</p>
      <select bind:value={motivationCategory} disabled={!motivationOn} on:change={lsSave}>
        {#each motivationCategories as c}
          <option value={c}>{c}</option>
        {/each}
      </select>

      <p class="label">Motivation delay after time</p>
      <select bind:value={motivationDelaySeconds} disabled={!motivationOn} on:change={lsSave}>
        {#each motivationDelayOptions as sec}
          <option value={sec}>{sec} sec</option>
        {/each}
      </select>

      <button class="toggle {clockOn ? 'is-on' : 'is-off'}" on:click={toggleClock}>
        {clockOn ? "🔔 ON" : "🔕 OFF"}
      </button>
    </section>

    <section class="panel timer-panel">
      <h3>⏱ Timer <span class="tag">B</span></h3>
      <div class="countdown">{timervalue}</div>

      <p class="label">{slidervalue} min · <span class="voices-note">{voices.length} voice{voices.length !== 1 ? "s" : ""} loaded</span></p>
      <input type="range" class="slider" min="1" max="120" bind:value={slidervalue} />

      <div class="btn-row">
        <button on:click={startTimer}>▶ Start</button>
        <button on:click={stopTimer}>⏸ Stop</button>
        <button on:click={resetTimer}>↺ Reset</button>
      </div>

      <label class="check-label">
        <input type="checkbox" bind:checked={timerNoiseOn} on:change={() => { lsSave(); updateNoisePlayback(); }} />
        Play background noise during timer
      </label>

      <label class="check-label">
        <input type="checkbox" bind:checked={timerSpeakOn} on:change={lsSave} />
        Speak remaining — every
      </label>
      <select bind:value={timerAnnounceEvery} on:change={lsSave} disabled={!timerSpeakOn}>
        {#each timerAnnounceOptions as m}
          <option value={m}>{m} min</option>
        {/each}
      </select>

      <label class="check-label">
        <input type="checkbox" bind:checked={chainModeOn} on:change={onChainModeToggle} />
        Enable chain timers
      </label>

      <select bind:value={chainPresetKey} on:change={onChainPresetChange} disabled={!chainModeOn}>
        {#each Object.keys(chainPresets) as key}
          <option value={key}>{key}</option>
        {/each}
      </select>

      {#if chainModeOn}
        <p class="label">Chain step {chainIndex + 1}/{chainPresets[chainPresetKey]?.length || 1}</p>
      {/if}
    </section>

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
      <p class="preset-hint">🍅 25 = Pomodoro · bold = deep work</p>
    </section>

    <section class="panel prefs-panel">
      <h3>⚙ Settings</h3>

      <p class="label">Background sound</p>
      <select bind:value={SoundChosen} on:change={lsSave}>
        {#each soundlist as s}
          <option value={s.link}>{s.title}</option>
        {/each}
      </select>

      <p class="label">Noise volume</p>
      <select bind:value={NoiseVolume} on:change={() => { lsSave(); updateNoisePlayback(); }}>
        {#each volumelists as v}
          <option value={v.volume}>{v.title}</option>
        {/each}
      </select>

      <p class="label">Speech volume</p>
      <select bind:value={SpeakVolume} on:change={lsSave}>
        {#each volumelists as v}
          <option value={v.volume}>{v.title}</option>
        {/each}
      </select>

      <p class="label">Voice list</p>
      <select bind:value={voiceListMode} on:change={lsSave}>
        <option value="pleasant">Pleasant voices</option>
        <option value="all">All English voices</option>
      </select>

      <p class="label">Speech language</p>
      <select bind:value={speechLanguage} on:change={() => { favoriteVoiceKey = "__auto__"; lsSave(); }}>
        <option value="en">English</option>
        <option value="ml">Malayalam (if available)</option>
      </select>

      <p class="label">Favorite voice</p>
      <select bind:value={favoriteVoiceKey} on:change={lsSave}>
        <option value="__auto__">Auto select</option>
        {#each availableVoices as v}
          <option value={`${v.name}|${v.lang}`}>{v.name} ({v.lang})</option>
        {/each}
      </select>

      <label class="check-label">
        <input type="checkbox" bind:checked={fullscreenDarkTheme} on:change={lsSave} />
        Use dark theme in fullscreen by default
      </label>
      <label class="check-label">
        <input type="checkbox" bind:checked={fullscreenDimBrightness} on:change={lsSave} />
        Dim screen brightness in fullscreen
      </label>
      <label class="check-label">
        <input type="checkbox" bind:checked={fullscreenStartLandscape} on:change={lsSave} />
        Start fullscreen in horizontal orientation
      </label>

      <p class="queue-note">
        {#if isSpeechActive}
          🔉 Speaking…
        {:else if speechQueue.length > 0}
          ⏳ {speechQueue.length} queued
        {:else}
          ✔ Ready
        {/if}
        · A↔B gap: 10 s
      </p>
    </section>
  </div>
</div>

{#if showFullscreenFocus}
  <div
    class="fullscreen-focus {fullscreenDarkTheme ? 'dark' : 'light'} {fullscreenDimBrightness ? 'dim' : ''} {fullscreenStartLandscape ? 'landscape' : ''}"
    on:click={toggleFullscreenOverlayControls}
  >
    <div class="focus-card">
      {#if fullscreenShowTimer}
        <div class="focus-label">REMAINING</div>
        <div class="focus-value">{timervalue}</div>
      {:else}
        <div class="focus-label">CURRENT TIME</div>
        <div class="focus-value clock">{currentTimeDisplay}</div>
      {/if}
    </div>

    {#if showFullscreenControls}
      <div class="focus-controls" on:click|stopPropagation={handleFullscreenControlInteraction}>
        <button on:click={() => { fullscreenShowTimer = !fullscreenShowTimer; }}>Toggle Clock/Timer</button>
        <button on:click={() => { fullscreenDarkTheme = !fullscreenDarkTheme; lsSave(); }}>Toggle Theme</button>
        <button on:click={closeFullscreenFocus}>Exit</button>
      </div>
    {/if}
  </div>
{/if}

<audio bind:this={audio} src={SoundChosen} loop />
<audio bind:this={notifyaudio} src={notifysound} />

<style>
  @import url("https://fonts.googleapis.com/css2?family=Rubik:wght@400;500;700&display=swap");

  :root {
    --primary: #440d49;
    --accent: #c3ddc0;
    --bg: #d4bff9;
    --muted: rgba(68, 13, 73, 0.55);
  }

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

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
    flex-direction: column;
    gap: 0.45rem;
    align-items: stretch;
  }

  .toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.5rem;
  }

  .app-title {
    color: var(--primary);
    font-weight: 700;
    font-size: 1rem;
  }

  .grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    grid-template-rows: 1fr 1fr;
    gap: 0.45rem;
    width: 100%;
    min-height: 0;
    flex: 1;
  }

  .panel {
    background: var(--bg);
    border: 2px dashed var(--primary);
    border-radius: 6px;
    padding: 0.6rem 0.75rem;
    display: flex;
    flex-direction: column;
    gap: 0.32rem;
    overflow: auto;
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

  .clock-face {
    font-size: 1.8rem;
    font-weight: 700;
    color: var(--primary);
    letter-spacing: 0.03em;
    font-variant-numeric: tabular-nums;
  }

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
    width: 14px;
    height: 14px;
    border-radius: 50%;
    background: var(--accent);
    border: 3px solid var(--primary);
    cursor: pointer;
    box-shadow: -300px 0 0 296px var(--primary);
  }

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

  button:hover {
    opacity: 0.7;
  }

  .btn-row {
    display: flex;
    gap: 0.35rem;
  }

  .btn-row button {
    flex: 1;
  }

  .toggle {
    width: fit-content;
    font-size: 0.82rem;
    padding: 0.32rem 1rem;
  }

  .is-on {
    background: var(--primary);
    color: var(--accent);
  }

  .is-off {
    background: var(--accent);
    color: var(--primary);
  }

  .check-label {
    display: flex;
    align-items: center;
    gap: 0.35rem;
    font-size: 0.7rem;
    color: var(--muted);
    cursor: pointer;
  }

  .check-label input {
    accent-color: var(--primary);
    cursor: pointer;
  }

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

  .preset-btn:hover {
    background: var(--primary);
    color: var(--accent);
  }

  .preset-btn.pomodoro {
    border-style: solid;
    border-width: 2px;
  }

  .preset-hint {
    font-size: 0.62rem;
    color: var(--muted);
    margin-top: 0.1rem;
  }

  .queue-note {
    font-size: 0.67rem;
    color: var(--muted);
    margin-top: auto;
    border-top: 1px dashed var(--primary);
    padding-top: 0.35rem;
  }

  .fullscreen-focus {
    position: fixed;
    inset: 0;
    z-index: 2000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 1rem;
  }

  .fullscreen-focus.dark {
    background: #000;
    color: #fff;
  }

  .fullscreen-focus.light {
    background: #fff;
    color: #000;
  }

  .fullscreen-focus.dim::after {
    content: "";
    position: absolute;
    inset: 0;
    background: rgba(0, 0, 0, 0.2);
    pointer-events: none;
  }

  .fullscreen-focus.landscape {
    writing-mode: horizontal-tb;
  }

  .focus-card {
    border: 2px solid currentColor;
    border-radius: 14px;
    padding: 1.25rem;
    width: min(95vw, 1100px);
    display: flex;
    flex-direction: column;
    gap: 0.6rem;
    text-align: center;
    z-index: 2;
    background: color-mix(in srgb, currentColor 8%, transparent);
  }

  .focus-label {
    opacity: 0.75;
    font-size: 1rem;
    letter-spacing: 0.08em;
    font-weight: 700;
  }

  .focus-value {
    font-size: clamp(3rem, 13vw, 10rem);
    font-weight: 800;
    font-variant-numeric: tabular-nums;
    line-height: 1;
  }

  .focus-value.clock {
    font-size: clamp(2.2rem, 8vw, 6rem);
  }

  .focus-controls {
    position: absolute;
    top: 1rem;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 0.5rem;
    z-index: 3;
  }

  @media (max-width: 540px) {
    :global(body) {
      overflow: auto;
    }

    .grid {
      grid-template-columns: 1fr;
      grid-template-rows: auto;
    }

    .shell {
      height: auto;
    }

    .clock-face,
    .countdown {
      font-size: 1.5rem;
    }

    .presets-grid {
      grid-template-columns: repeat(5, 1fr);
    }

    .focus-controls {
      flex-direction: column;
      width: calc(100% - 2rem);
      left: 1rem;
      transform: none;
    }
  }
</style>

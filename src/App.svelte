<script>
  let audio, notifyaudio;
  let slidervalue = 13;

  let timervalue = "00 : 00";
  let interval = null;
  let speakCurrentTimeInterval = null;
  let secs = 0,
    mins = 0;
  let seconds = 0;

  // speeech api not speecking ==> speechsynthesis.cancel() to

  console.log(seconds);

  let presetvalues = [
    2,
    4,
    5,
    6,
    7,
    10,
    15,
    20,
    25,
    30,
    40,
    50,
    60,
    70,
    12,
    16,
    80,
    90,
    120,
    // 160,
    180,
  ];

  presetvalues.sort((a, b) => a - b);

  const localstorageset = () => {
    localStorage.setItem("SoundChoosen", SoundChoosen.toString());
    localStorage.setItem("NoiseVolume", NoiseVolume.toString());
    localStorage.setItem("SpeakVolume", SpeakVolume.toString());
    localStorage.setItem("SpeechAlways", SpeechAlways.toString());
  };

  const localstorageget = (key) => {
    if (localStorage.getItem(key)) {
      return localStorage.getItem(key);
    }
    return "false";
  };

  // let Sriver= 'https://www.soundjay.com/nature/sounds/stream-3.mp3'
  // let Swaterfall = 'https://www.soundjay.com/nature/sounds/waterfall-1.mp3'
  // let Sfire = ' https://www.soundjay.com/nature/sounds/fire-1.mp3'
  // let Srain = 'https://www.soundjay.com/nature/sounds/rain-04.mp3'

  let notifysound = "https://www.soundjay.com/clock/sounds/alarm-clock-01.mp3";

  let soundlist = [
    {
      link: "https://www.soundjay.com/nature/sounds/rain-04.mp3",
      title: "Rain",
    },
    {
      link: "https://www.soundjay.com/nature/sounds/waterfall-1.mp3",
      title: "Water Fall",
    },

    {
      link: "https://www.soundjay.com/nature/sounds/fire-1.mp3",
      title: "Fire",
    },

    {
      link: "https://www.soundjay.com/nature/sounds/stream-3.mp3",
      title: "Stream",
    },
  ];

  let SoundChoosen =
    localstorageget("SoundChoosen") != "false"
      ? localstorageget("SoundChoosen")
      : soundlist[0]["link"];
  let volumelists = [
    { volume: 0.1, title: "Very Low" },
    { volume: 0.2, title: "Low" },
    { volume: 0.6, title: "Medium" },
    { volume: 0.8, title: "High" },
    { volume: 1, title: "Very High" },
  ];
  let NoiseVolume =
    localstorageget("NoiseVolume") != "false"
      ? parseFloat(localstorageget("NoiseVolume"))
      : volumelists[1]["volume"];

  let SpeakVolume =
    localstorageget("SpeakVolume") != "false"
      ? parseFloat(localstorageget("SpeakVolume"))
      : volumelists[1]["volume"];

  let SpeechAlways =
    localstorageget("SpeechAlways") != "false"
      ? localstorageget("SpeechAlways")
      : "yes";
  // NoiseVolume = parseFloat(NoiseVolume);

  function speakCurrentTime() {
    speakCurrentTimeInterval = setInterval(() => {
      // let date = new Date();
      // let hh = date.getHours();

      // let amPm = hh >= 12 ? "PM" : "AM";

      // hh = hh % 12 || 12;

      // console.log("im here");

      // let mm = date.getMinutes();
      // speak(`${hh} ${mm} ${amPm}`);
      speak(getCurrentTimeString());
    }, 91 * 1000);
  }

  console.log(`this is ${SpeechAlways}`);

  function getCurrentTimeString() {
    const now = new Date();
    const hours = now.getHours();
    const minutes = now.getMinutes();
    let timeString = "";

    // Convert hours to spoken phrase
    if (hours === 0) {
      timeString += "12";
    } else if (hours > 12) {
      timeString += hours - 12;
    } else {
      timeString += hours;
    }

    // Convert minutes to spoken phrase
    if (minutes === 0) {
      timeString += " o'clock";
    } else if (minutes < 10) {
      timeString += " oh " + minutes;
    } else {
      timeString += " " + minutes;
    }

    // Add AM/PM depending on the time of day
    if (hours < 12) {
      timeString += " AM";
    } else {
      timeString += " PM";
    }

    return timeString;
  }

  function speak(text) {
    let synth = speechSynthesis;
    // speechSynthesis.cancel();
    if (synth.speaking) {
      synth.cancel();
    }

    let speech = new SpeechSynthesisUtterance(text);

    // speech.rate = 10;
    speech.pitch = 0.7;
    speech.volume = SpeakVolume;

    synth.speak(speech);
  }

  window.onload = () => {
    if (SpeechAlways == "yes") {
      speakCurrentTime();
    }
  };

  const timer = () => {
    // notifyaudio.play()
    // console.log('im here');

    mins = Math.floor(seconds / 60);
    secs = seconds % 60;
    seconds--;
    timervalue = `${mins >= 10 ? mins : "0" + mins} : ${
      secs >= 10 ? secs : "0" + secs
    }`;
    if (secs == 0 && seconds != 0) {
      // speechSynthesis.?
      // speechSynthesis.speak(new SpeechSynthesisUtterance(`${mins} minutes`));
      speak(`${mins} minutes`);
    }
    // if (secs == 30 && seconds != 0 && mins % 2 == 0) {
    // }
    if (seconds == 0) {
      reset();

      if (SpeechAlways == "no") {
        clearInterval(speakCurrentTimeInterval);
        speakCurrentTimeInterval = null;
      }
      notifyaudio.play();
      // speechSynthesis.speak(
      //   new SpeechSynthesisUtterance("Timer is now finished Amarjith")
      // );

      // setTimeout(() => {
      //   // let out = null;
      //   let choice = confirm("Timer finished");
      //   if (choice == true || choice == false) {
      //     notifyaudio.pause();
      //   }
      // }, 3000);
    }
  };

  const chooseButton = (e) => {
    slidervalue = e.target.value;
    start();
  };

  const start = () => {
    localstorageset();
    if (seconds == 0) seconds = slidervalue * 60;

    if (interval) return;

    interval = setInterval(timer, 1000);
    audio.volume = NoiseVolume;
    console.log(NoiseVolume);
    audio.play();
    clearInterval(speakCurrentTimeInterval);
    speakCurrentTimeInterval = null;
    speakCurrentTime();
  };

  const stop = () => {
    clearInterval(interval);
    interval = null;
    audio.pause();
  };

  const reset = () => {
    stop();
    seconds = 0;
    timervalue = "00 : 00";
  };
</script>

<svelte:head>
  <!-- <link rel="stylesheet" href="https://unpkg.com/@picocss/pico@1.*/css/pico.min.css"> -->

  <title>Speaktimer</title>
  <link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
  <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png" />
  <link rel="icon" type="image/png" sizes="16x16" href="/favicon-16x16.png" />
  <link rel="manifest" href="/site.webmanifest" />
</svelte:head>

<!-- one way binding vs two way binding -->

<div class="container">
  <div class="flex-item flex-item1">
    <h2>Preferences</h2>

    <br />

    <p>Choose the Noise sound</p>
    <select name="Sound" id="cars" bind:value={SoundChoosen}>
      {#each soundlist as sound}
        <option value={sound["link"]}>{sound["title"]}</option>
      {/each}
    </select>
    <p>Choose the Noise volume level</p>

    <select name="NoiseVolume" id="NoiseVolume" bind:value={NoiseVolume}>
      {#each volumelists as volume}
        <option value={volume["volume"]}>{volume["title"]}</option>
      {/each}
    </select>
    <p>Choose the Speech volume level</p>

    <select name="speakVolume" id="speakVolume" bind:value={SpeakVolume}>
      {#each volumelists as volume}
        <option value={volume["volume"]}>{volume["title"]}</option>
      {/each}
    </select>
    <p>Do you want the Speech to be always on</p>

    <select name="speechalwayson" id="speechalwayson" bind:value={SpeechAlways}>
      <option value="yes">Yes</option>
      <option value="no">No</option>
    </select>
  </div>
  <div class="flex-item flex-item2">
    <h2>Timer</h2>

    <!-- <div class="tooltip"> -->

    <p>
      Current slider value is &nbsp;<b style="font-size:2rem;">{slidervalue}</b>
      &nbsp; minutes
    </p>

    <input
      type="range"
      class="slider "
      min="1"
      max="60"
      bind:value={slidervalue}
    />

    <!-- <span class="tooltiptext">Tooltip text</span> -->
    <!-- </div> -->

    <!-- <p>Create a timer of {slidervalue} minutes </p> -->

    <div class="time-block">
      <h1>{timervalue}</h1>
    </div>

    <div class="buttons-block">
      <button on:click={start}>Start</button>
      <button on:click={stop}>Stop</button>
      <button on:click={reset}>Reset</button>
    </div>
  </div>
  <div class="flex-item flex-item3">
    <h2>Presets</h2>
    <p>Use Slider or presets to start timer</p>

    <div class="flex-container">
      {#each presetvalues as preset}
        <div class="preset">
          <!-- <p><b>{preset}</b> minutes</p> -->
          <button
            value={preset}
            on:click={(e) => {
              chooseButton(e);
            }}>{preset}</button
          >
        </div>
      {/each}
    </div>
  </div>
</div>

<!-- <audio src="./rain.mp3" loop bind:this={audio} ></audio> -->
<audio src={SoundChoosen} loop bind:this={audio} />
<audio src={notifysound} loop bind:this={notifyaudio} />

<style>
  @import url("https://fonts.googleapis.com/css2?family=Rubik&display=swap");

  :root {
    /* --primarycolor:#9c1b1b;
      --accentcolor:#f89d9d; */

    --primarycolor: #440d49;
    --accentcolor: #c3ddc0;
    --backgroundcolor: #d4bff9;
    /* font-size: 16px; */

    /* 


light 
    --primarycolor: #440d49;
    --accentcolor:#defabb;
    --backgroundcolor: #d4bff9; */
    /* --font200:rgb(52, 21, 78);

    --light:black;
    --dark:black;
    --accent:red; */
  }

  #app {
    margin: 0;
  }
  * {
    margin: 0;
    padding: 0;
  }

  h2 {
    font-size: 3rem;
    margin: 2rem 1rem;
    /* text-decoration: underline; */
  }
  /* :global(body) { this will apply to <body> margin: 0; padding: 0; } */

  .container {
    display: flex;
    flex-wrap: wrap;
    margin: 0;
    font-family: "Rubik", sans-serif;

    color: var(--font200);
  }

  .flex-item {
    /* background:red; */
    border: 2px dashed white;
    border: 2px var(--primarycolor) dashed;

    flex-basis: 400px;
    flex-grow: 1;
    background-color: var(--backgroundcolor);
    padding: 1rem;
  }

  .flex-container {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
  }

  .flex-container .preset {
    width: 20%;

    margin: 1rem 2.5%;
    display: flex;
    justify-content: center;
    align-items: center;
    /* align-items:center; */
  }

  .preset h1,
  .preset button {
    display: block;
  }

  .buttons-block {
    width: fit-content;
    margin: 2rem auto;
  }

  .buttons-block button {
    margin: 1rem;
    padding: 1rem 2.5rem;
    border: 2px var(--primarycolor) solid;
  }

  select {
    font-weight: bold;
    margin: 1rem auto;
    display: block;
    width: 80%;
    background: var(--accentcolor);
    color: var(--primarycolor);
    border: var(--primarycolor);
    border-radius: 7px;
    outline: none;
    padding: 1rem 1rem;
    border: 2px var(--primarycolor) solid;
    font-size: 1.3rem;
  }

  button:hover {
    opacity: 0.7;
  }

  main {
    color: var(--primarycolor);
  }
  p {
    font-weight: bold;
    padding: 0.5rem;
    /* box-shadow: 1rem 1rem 1rem red; */
  }
  .slider {
    /* max-width: 400px;
     */

    width: 70%;
    margin: 2rem 15%;
    height: 15px;
    -webkit-appearance: none;
    background: var(--accentcolor);
    outline: none;
    border-radius: 4px;
    overflow: hidden;
    box-shadow: inset 0 0 5px var(--accentcolor);
    border: 2px var(--primarycolor) solid;
  }
  .slider::-webkit-slider-thumb {
    -webkit-appearance: none;
    width: 15px;
    height: 15px;
    border-radius: 50%;
    background: var(--accentcolor);
    cursor: pointer;
    border: 4px solid #4c4747;
    box-shadow: -407px 0 0 400px var(--primarycolor);
  }

  .time-block {
    /* font-family: Arial, Helvetica, sans-serif; */
    font-size: 3rem;
    display: flex;
    background-color: var(--accentcolor);
    color: var(--primarycolor);
    min-height: 150px;
    border-radius: 4px;
    align-items: center;
    justify-content: center;
    margin: 1rem;
    padding: 5rem 2rem;
    border: 2px var(--primarycolor) solid;
  }

  button {
    border-radius: 4px;
    border: none;
    /* padding: .6rem 2rem; */
    outline: none;
    font-weight: bold;
    /* fontsie */

    font-size: 1.3rem;

    /* box-sizing: content-box; */
    background-color: var(--accentcolor);
    color: var(--primarycolor);
  }

  .preset button {
    width: 90%;
    padding: 2rem 0.5rem;
    font-size: 1.5rem;
    border: 2px var(--primarycolor) solid;
  }

  @media (max-width: 500px) {
    /* 
  .time-block{
    font-size: 2rem;
  } */

    .flex-item1 {
      order: 3;
    }
    .flex-item2 {
      order: 2;
    }
    .flex-item3 {
      order: 1;
    }

    :root {
      font-size: 12px;
    }
    /* Your CSS styles here */
  }
</style>

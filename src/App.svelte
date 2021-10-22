<script>
  import { onMount } from "svelte";
  import { serverUrl } from "./stores.js";
  import { fade } from "svelte/transition";
  import { lang } from "./lang.js";
  import { defaultParams } from "./params.js";
  import Gallery from "./Gallery.svelte";

  let params = defaultParams;
  let displayGallery = false;
  let sound = false;
  let oscillator, gain;
  let selectedLang = "fr";
  let word = lang[selectedLang].greeting;
  let text, material;
  let captured = true;
  let recognition;
  let recognizing = false;
  let transition = false;

  onMount(() => {
    initText();
    initVoiceRecognition();
  });

  function initText() {
    material = new Blotter.LiquidDistortMaterial();

    text = new Blotter.Text(word, {
      family: "temeraire, serif",
      style: "italic",
      size: 300,
      fill: "white",
      paddingRight: 60,
      paddingLeft: 60,
    });

    const blotter = new Blotter(material, {
      texts: text,
    });

    const scope = blotter.forText(text);
    scope.appendTo(document.getElementById("word-container"));

    animate();
  }

  function initVoiceRecognition() {
    recognition = new webkitSpeechRecognition();
    recognition.lang = selectedLang;

    recognition.onstart = () => {
      console.log("recognition start");
      recognizing = true;
    };

    recognition.onend = () => {
      console.log("recognition end");
      recognizing = false;
    };

    recognition.onspeechstart = () => {
      console.log("speech start");
      if (!captured) {
        takeScreenshot();
      }
      params.seed.max = getNewSeed();
      transition = true;
    };

    recognition.onresult = (event) => {
      console.log("result");
      console.log(event.results);
      if (event.results[0].isFinal) {
        word = event.results[0][0].transcript
          .trim()
          .split(" ")
          .slice(0, 5)
          .join(" ")
          .toLowerCase();
        changeText(word);
        params.seed.min = params.seed.max;
        captured = false;
        transition = false;
        if (recognizing) {
          recognition.stop();
        }
      }
    };

    recognition.onerror = (event) => {
      console.log(event);
    };
  }

  function changeText(string) {
    text.value = string;
    text.needsUpdate = true;
  }

  function resumeExperience() {
    displayGallery = false;
    captured = true;
    params = defaultParams;
    if (sound) {
      gain.gain.value = 0.3;
    }
    word = lang[selectedLang].greeting;
    animate();
    initText();
  }

  function initSound() {
    const audioContext = new (window.AudioContext ||
      window.webkitAudioContext)();
    gain = audioContext.createGain();
    oscillator = audioContext.createOscillator();
    oscillator.connect(gain);
    gain.connect(audioContext.destination);
    oscillator.type = "triangle";
    oscillator.frequency.value = params.freq.value;
    gain.gain.value = sound ? 0.3 : 0;
    oscillator.start();
  }

  function takeScreenshot() {
    const img = document
      .querySelector("#word-container canvas")
      .toDataURL("image/png");
    console.log("capture d'écran !!", word);
    captured = true;
    let filename = word.replace(/'/g, "").replace(/ /g, "");
    const newScreenshot = {
      title: filename,
      file: img,
    };
    postScreenshot(newScreenshot);
  }

  const postScreenshot = async (screenshot) => {
    const res = await fetch(serverUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(screenshot),
    });
  };

  function openGallery() {
    if (recognizing) {
      recognition.stop();
    }
    transition = true;
    setTimeout(() => {
      if (sound) {
        gain.gain.value = 0;
      }
      displayGallery = true;
      transition = false;
      document.getElementById("word-container").innerHTML = "";
    }, 400);
  }

  function getNewSeed() {
    return (Math.random() * 20).toFixed(2);
  }

  function interpolate(min, max, fraction) {
    return (max - min) * fraction + min;
  }

  function handleKeydown(event) {
    if (recognizing) {
      return;
    }
    if (event.keyCode != 83) {
      return;
    }
    recognition.start();
    console.log("on t'écoute");
  }

  function handleKeyup(event) {
    if (event.keyCode != 83) {
      return;
    }
    recognition.stop();
    transition = false;
    console.log("fini de parler");
  }

  function toggleSound() {
    if (!oscillator) {
      initSound();
    }
    sound = !sound;
    gain.gain.value = sound ? 0.3 : 0;
  }

  function animate() {
    if (displayGallery) {
      return;
    }

    for (const key in params) {
      if (transition) {
        params[key].value = interpolate(
          params[key].value,
          params[key].max,
          params[key].in
        );
      } else {
        params[key].value = interpolate(
          params[key].value,
          params[key].min,
          params[key].out
        );
      }
    }

    if (!captured && !transition && params.speed.value < 0.8) {
      takeScreenshot();
    }

    if (oscillator) {
      oscillator.frequency.value = params.freq.value;
    }
    material.uniforms.uVolatility.value = params.volatility.value;
    material.uniforms.uSpeed.value = params.speed.value;
    material.uniforms.uSeed.value = params.seed.value;

    requestAnimationFrame(animate);
  }

  function changeLanguage(event) {
    selectedLang = event.target.dataset.lang;
    recognition.lang = lang[selectedLang].code;
    transition = true;
    setTimeout(() => {
      changeText(lang[selectedLang].greeting);
      transition = false;
    }, 600);
  }
</script>

<svelte:window on:keydown={handleKeydown} on:keyup={handleKeyup} />

<main>
  {#if !displayGallery}
    <div transition:fade>
      <p class="lien to-gallery" on:click={openGallery}>voir la galerie</p>
      <ul class="lang">
        {#each Object.values(lang) as el}
          <li
            on:click={changeLanguage}
            class={el.code.slice(0, 2) === selectedLang ? "selected" : ""}
            data-lang={el.code.slice(0, 2)}
          >
            {el.code.slice(0, 2)}
          </li>
        {/each}
      </ul>
      <p on:click={toggleSound} class="sound {sound ? 'selected' : ''}">
        {sound ? "sound on" : "sound off"}
      </p>
      {#if !recognition}
        <p class="error">
          la reconnaissance vocale n'est pas supportée par votre navigateur
        </p>
      {/if}
    </div>
  {/if}
  <div id="wrapper">
    <div
      style="filter: blur({params ? params.blur.value : 0}px); opacity: {params
        ? params.opacity.value
        : 0}; transform: scaleY({params ? params.scale.value : 1});"
      id="word-container"
    />
    <div id="grain" />
  </div>
  {#if displayGallery}
    <div transition:fade>
      <Gallery on:close={resumeExperience} />
    </div>
  {/if}
</main>

<style>
  #wrapper {
    height: 100vh;
    width: 100vw;
    overflow: hidden;
    position: absolute;
  }

  #word-container {
    height: 100vh;
    width: 100vw;
    display: grid;
    align-items: center;
    justify-items: center;
    position: absolute;
  }

  :global(#word-container canvas) {
    position: absolute;
    width: 100vw !important;
    height: 80vh !important;
  }

  #grain {
    background: url("/assets/img/grain.png");
    mix-blend-mode: color-dodge;
    height: 100vh;
    width: 100vw;
    opacity: 0.75;
  }

  .to-gallery {
    position: absolute;
    top: 2vw;
    left: 2vw;
  }

  .lang {
    display: flex;
    z-index: 2;
    position: absolute;
    top: 2vw;
    right: 2vw;
  }

  .sound {
    border-bottom: 1px solid transparent;
    z-index: 2;
    position: absolute;
    bottom: 2vw;
    left: 2vw;
    cursor: pointer;
  }

  ul li {
    cursor: pointer;
  }

  ul li + li {
    margin-left: 1em;
  }

  .error {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 2;
    background-color: #fef4f6;
    color: #f0506e;
    display: inline-block;
    font-size: 0.8em;
    width: max-content;
    padding: 8px;
    border-radius: 8px;
    border: 2px solid #f0506e;
  }
</style>

<script>
  import { onMount } from "svelte";
  import { defaultParams } from "./params.js";
  import Gallery from "./Gallery.svelte";
  import { serverUrl } from "./stores";

  let params;
  let word = "hello";
  let oscillator;
  let displayGallery;

  let text, material;
  let captured = true;

  let recognition;

  let transition = false;

  onMount(() => {
    displayGallery = false;
    params = defaultParams;
    // addSound();
    initText();
    initVoiceRecognition();
  });

  function initText() {
    word = "hello";

    material = new Blotter.LiquidDistortMaterial();

    text = new Blotter.Text(word, {
      family: "temeraire, serif",
      style: "italic",
      size: calculateWidth(word),
      fill: "white",
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
    recognition.continuous = true;
    recognition.interimResults = false;

    recognition.lang = "fr-FR";
    recognition.start();

    recognition.onstart = function () {
      console.log("recognition onstart");
    };

    recognition.onend = function () {
      console.log("recognition onend");
    };

    recognition.onspeechstart = (event) => {
      console.log("speech start");
      console.log("DÉBUT TRANSITION !!!");
      params.seed.max = getNewSeed();
      transition = true;
    };

    recognition.onspeechend = (event) => {
      console.log("speech end");
      console.log("FIN TRANSITION !!!");
      params.seed.min = params.seed.max;
      transition = false;
    };

    recognition.onresult = function (event) {
      console.log("result");
      console.log(event.results);
      // let word = event.results[0][0].transcript.trim().split(" ")[0];
      if (event.results[0].isFinal) {
        word = event.results[0][0].transcript.replace(" ", "");
        word = word.toLowerCase();
        console.log("change text");
        changeText(word);
        console.log("stop recognition");
        recognition.stop();
        setTimeout(() => {
          console.log("restart recognition");
          recognition.start();
        }, 200);
      }
    };

    recognition.onnomatch = function () {
      console.log("speech not recognized");
    };

    recognition.onerror = function (event) {
      console.log(event);

      // setTimeout(() => {
      //   console.log("restart recognition");
      //   recognition.start();
      // }, 200);
    };
  }

  function changeText(string) {
    if (!captured) {
      takeScreenshot();
    }
    text.value = string;
    text.size = calculateWidth(string);
    text.needsUpdate = true;
    captured = false;
  }

  function stopExperience() {
    document.getElementById("word-container").innerHTML = "";
    recognition.stop();
  }

  function restartExperience() {
    displayGallery = false;
    animate();
    params = defaultParams;
    initText();
    console.log("eeeet ça redémarre");
    recognition.start();
  }

  function addSound() {
    let audioContext = new (window.AudioContext || window.webkitAudioContext)();
    let gain = audioContext.createGain();
    oscillator = audioContext.createOscillator();
    oscillator.connect(gain);
    gain.connect(audioContext.destination);
    oscillator.type = "sawtooth";
    oscillator.frequency.value = params.freq.value;
    gain.gain.value = volume;
    oscillator.start();
    console.log(oscillator);
  }

  function takeScreenshot() {
    let img = document
      .querySelector("#word-container canvas")
      .toDataURL("image/png");
    console.log("capture d'écran !!!!");
    console.log(img);

    captured = true;

    let newScreenshot = {
      title: word,
      file: img,
    };

    postScreenshot(newScreenshot);

    // to do : envoyer sur le serveur
  }

  let postScreenshot = async (screenshot) => {
    console.log("allo");
    const res = await fetch(serverUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(screenshot),
    });
  };

  function openGallery() {
    console.log("openGallery");
    displayGallery = true;
    console.log(displayGallery);
    stopExperience();
  }

  function getNewSeed() {
    return (Math.random() * 20).toFixed(2);
  }

  function interpolate(min, max, fraction) {
    return (max - min) * fraction + min;
  }

  function calculateWidth(string) {
    return (window.innerWidth * 2) / string.length;
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

    // console.log("~~~~~~~~~~~~~~");
    // console.log("transition", params.freq.value);
    // console.log("volatility", params.volatility.value);
    // console.log("speed", params.speed.value);
    // console.log("blur", params.blur.value);
    // oscillator.frequency.value = params.freq.value;

    material.uniforms.uVolatility.value = params.volatility.value;
    material.uniforms.uSpeed.value = params.speed.value;
    material.uniforms.uSeed.value = params.seed.value;

    requestAnimationFrame(animate);
  }
</script>

<main>
  <p class="lien" on:click={openGallery}>voir la galerie</p>
  <div
    style="filter: blur({params ? params.blur.value : 0}px); opacity: {params
      ? params.opacity.value
      : 0}; transform: scaleY({params ? params.scale.value : 1});"
    id="word-container"
  />
  <div id="grain" />
  {#if displayGallery}
    <Gallery on:restart={restartExperience} />
  {/if}
</main>

<style>
  main {
    background-color: black;
  }

  #word-container {
    height: 100vh;
    width: 100vw;
    display: grid;
    align-items: center;
    justify-items: center;
    position: absolute;
    top: 0;
    left: 0;
    transition: background-color 200ms;
  }

  :global(#word-container canvas) {
    position: absolute;
    width: 100vw !important;
    height: 100vh !important;
  }

  #grain {
    background: url("/assets/img/grain.png");
    mix-blend-mode: color-dodge;
    height: 100vh;
    width: 100vw;
    opacity: 0.75;
  }
</style>

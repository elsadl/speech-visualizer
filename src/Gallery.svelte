<script>
  import { onMount, createEventDispatcher } from "svelte";
  import { fade } from 'svelte/transition';
  import { serverUrl } from "./stores.js";

  const dispatch = createEventDispatcher();

  const step = 10;
  let screenshots = [];
  let count = step;

  $: screenshotsToDisplay = [];

  onMount(async () => {
    const res = await fetch(serverUrl);
    screenshots = await res.json();
    screenshots = screenshots.experiments.reverse();
    screenshotsToDisplay = screenshots.slice(0, step);
  });

  function displayMore() {
    count = count + step;
    screenshotsToDisplay = screenshots.slice(0, count);
  }

  function closeGallery() {
    dispatch("close");
  }
</script>

<div id="gallery">
  <p class="lien close" on:click={closeGallery}>← retour</p>
  <div id="screenshots">
    {#each screenshotsToDisplay as screenshot}
      <div transition:fade class="screenshot">
        <p class="filename">{screenshot.title}.png // {screenshot.createdAt}</p>
        <img src={screenshot.file} alt={screenshot.title} />
      </div>
    {/each}
    {#if count < screenshots.length}
      <p class="lien plus" on:click={displayMore}>+ voir plus</p>
    {/if}
  </div>
</div>

<style>
  #gallery {
    width: 100vw;
    height: 100vh;
    background-color: black;
    position: absolute;
    top: 0;
    left: 0;
    z-index: 10;
  }

  #screenshots {
    margin: 6vw auto;
    margin-top: 80px;
    max-width: 600px;
  }

  .screenshot {
    position: relative;
    border-bottom: 1px solid #1b1b1b;
  }

  .screenshot:last-child {
    border-bottom: 0;
    padding-bottom: 100px;
  }

  .screenshot img {
    height: 40vh;
  }

  .screenshot + .screenshot {
    margin-top: 2em;
  }

  .filename {
    font-family: monospace;
    font-size: 0.65em;
    display: inline;
    position: absolute;
    top: 0;
    left: 0;
  }

  .lien.close {
    position: absolute;
    top: 2vw;
    left: 2vw;
  }

  .lien.plus {
    margin: 40px 0 140px;
  }
</style>

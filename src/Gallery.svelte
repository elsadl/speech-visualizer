<script>
  import { onMount, createEventDispatcher } from "svelte";

  const dispatch = createEventDispatcher();

  const baseUrl = "http://localhost:3001/screenshots";
  let screenshots = [];

  onMount(async () => {
    let res = await fetch(baseUrl);
    screenshots = await res.json();
    screenshots = screenshots.experiments;
    console.log(screenshots);
  });

  function closeGallery() {
    dispatch("restart");
  }
</script>

<div id="gallery">
  <p class="lien" on:click={closeGallery}>quitter la galerie</p>
  <div id="screenshots">
    {#each screenshots as screenshot}
      <div class="screenshot">
        <p>{screenshot.title}.png</p>
        <img src={screenshot.img} alt={screenshot.title}>
      </div>
    {/each}
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

  .screenshot + .screenshot {
    padding-top: 2em;
  }
</style>

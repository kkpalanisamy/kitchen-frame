<script>
  // Home card: crossfades through the shared album every 20 s; tap to open the Photos tab.
  // Only two images exist at a time (current + next), so it stays light on the A33.
  import { onMount, onDestroy } from 'svelte';
  import { api } from '../lib/api.js';
  import { src } from '../lib/slides.js';
  import { ui } from '../lib/state.svelte.js';

  const EVERY_MS = 20000;
  let items = [];
  let n = 0;
  let front = $state(0);
  let layers = $state([null, null]);
  let timer;

  function shuffle(a) {
    for (let i = a.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [a[i], a[j]] = [a[j], a[i]]; }
    return a;
  }

  function next() {
    n = (n + 1) % items.length;
    const url = src(items[n]);
    const pre = new Image();           // download first, then fade, so it never flashes blank
    pre.onload = () => { const back = 1 - front; layers[back] = url; front = back; };
    pre.src = url;
  }

  onMount(async () => {
    let all = [];
    try { all = (await api('/api/photos')).photos || []; } catch {}
    // The card is tall and narrow: prefer portrait photos when there are enough of them.
    const portraits = all.filter((p) => p.h > p.w);
    items = shuffle(portraits.length >= 10 ? portraits : all);
    if (!items.length) return;
    layers[0] = src(items[0]);
    timer = setInterval(next, EVERY_MS);
  });
  onDestroy(() => clearInterval(timer));
</script>

<button class="photo" onclick={() => (ui.tab = 'photos')} aria-label="Open Photos">
  {#each layers as url, k}
    {#if url}<img src={url} alt="" class:on={k === front} />{/if}
  {/each}
</button>

<style>
  .photo { position: relative; width: 100%; height: 100%; padding: 0; border-radius: 24px; overflow: hidden; background: var(--track); }
  img { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; opacity: 0; transition: opacity 1.2s ease; }
  img.on { opacity: 1; }
  .photo:active { opacity: .85; }
</style>

<script>
  // Photos tab: a grid of the shared album (~320px thumbs), tap for full screen,
  // swipe or tap the edges to move, tap the middle or Back to close.
  import { onMount } from 'svelte';
  import Icon from '../components/Icon.svelte';
  import { api, withKey } from '../lib/api.js';
  import { src } from '../lib/slides.js';

  let items = $state([]);
  let status = $state('loading');
  let open = $state(-1);

  const thumb = (p) => {
    const u = withKey('/api/photos/' + p.id);
    return u + (u.includes('?') ? '&' : '?') + 'size=thumb';
  };
  const day = (d) => (d ? new Date(d).toLocaleDateString(undefined, { month: 'short', day: 'numeric', year: 'numeric' }) : '');

  onMount(async () => {
    try {
      items = (await api('/api/photos')).photos || [];
      status = items.length ? 'ok' : 'empty';
    } catch {
      status = 'error';
    }
  });

  function go(d) { if (items.length) open = (open + d + items.length) % items.length; }

  // Swipe left/right to move; tap the outer quarters to step; tap the middle to close.
  let x0 = null;
  function down(e) { x0 = e.clientX; }
  function up(e) {
    if (x0 === null) return;
    const dx = e.clientX - x0;
    x0 = null;
    if (Math.abs(dx) > 60) go(dx < 0 ? 1 : -1);
    else if (e.clientX < innerWidth * 0.25) go(-1);
    else if (e.clientX > innerWidth * 0.75) go(1);
    else open = -1;
  }
  const stop = (e) => e.stopPropagation();

  let current = $derived(open >= 0 ? items[open] : null);
  let next = $derived(open >= 0 && items.length > 1 ? items[(open + 1) % items.length] : null);
</script>

<section class="photos">
  {#if status === 'loading'}<p class="note">Loading photos…</p>
  {:else if status === 'error'}<p class="note">Couldn't load the album. Check the album link.</p>
  {:else if status === 'empty'}<p class="note">No photos in the shared album yet.</p>
  {:else}
    <div class="grid">
      {#each items as p, i (p.id)}
        <button class="cell" onclick={() => (open = i)} aria-label="Open photo {i + 1}">
          <img src={thumb(p)} alt="" loading="lazy" decoding="async" />
        </button>
      {/each}
    </div>
  {/if}
</section>

{#if current}
  <div class="viewer" role="dialog" aria-label="Photo" onpointerdown={down} onpointerup={up}>
    {#key current.id}<img class="full" src={src(current)} alt="" />{/key}
    <!-- Load the next photo in the background so swiping is instant. -->
    {#if next}<img class="pre" src={src(next)} alt="" />{/if}
    <div class="bar" onpointerdown={stop} onpointerup={stop}>
      <button class="btn" onclick={() => (open = -1)}><Icon name="chevronLeft" size={26} />Photos</button>
      <span class="meta">{day(current.date)} · {open + 1} of {items.length}</span>
    </div>
  </div>
{/if}

<style>
  .photos { height: 100%; overflow-y: auto; padding: 4px 28px 28px; }
  .grid { display: grid; grid-template-columns: repeat(6, 1fr); gap: 10px; }
  .cell { aspect-ratio: 1; padding: 0; border-radius: 14px; overflow: hidden; background: var(--track); }
  .cell img { width: 100%; height: 100%; object-fit: cover; display: block; }
  .cell:active { opacity: .7; }
  .note { color: var(--secondary); font-size: 20px; padding: 24px 0; }

  .viewer { position: fixed; inset: 0; z-index: 20; background: #000; display: grid; place-items: center; touch-action: none; }
  .full { max-width: 100%; max-height: 100%; object-fit: contain; }
  .pre { position: absolute; width: 1px; height: 1px; opacity: 0; pointer-events: none; }
  .bar { position: absolute; top: 0; left: 0; right: 0; display: flex; justify-content: space-between; align-items: center;
         padding: 18px 24px; color: #fff; background: linear-gradient(rgba(0, 0, 0, .55), transparent); }
  .btn { display: flex; align-items: center; gap: 2px; height: 48px; padding: 0 14px 0 6px; border-radius: 24px; color: #fff; font-size: 20px; font-weight: 600; }
  .btn:active { background: rgba(255, 255, 255, .15); }
  .meta { font-size: 18px; opacity: .85; }
</style>

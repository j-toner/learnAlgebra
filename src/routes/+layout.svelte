<script lang="ts">
  import type { Snippet } from 'svelte';

  let { children }: { children: Snippet } = $props();
</script>

<svelte:head>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link
    href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;700&family=Source+Code+Pro:wght@400;600&family=Lora:ital,wght@0,400;0,600;1,400&display=swap"
    rel="stylesheet"
  />
</svelte:head>

<div class="app">
  {@render children()}
</div>

<style>
  :global(*) {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  :global(:root) {
    --bg: #1a1f2e;
    --bg-surface: #222840;
    --bg-raised: #2a3252;
    --chalk: #e8e4d9;
    --chalk-dim: #9a95899e;
    --accent-gold: #f0c060;
    --accent-teal: #5ec4b0;
    --accent-red: #e05a5a;
    --accent-green: #5ec47a;
    --font-display: 'Playfair Display', serif;
    --font-body: 'Lora', serif;
    --font-mono: 'Source Code Pro', monospace;
    --radius: 6px;
    --shadow: 0 4px 24px rgba(0, 0, 0, 0.4);
  }

  :global(body) {
    background-color: var(--bg);
    color: var(--chalk);
    font-family: var(--font-body);
    min-height: 100vh;
  }

  .app {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
  }
</style>

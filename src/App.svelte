<script lang="ts">
  import DiceCanvas from './lib/DiceCanvas.svelte';
  let count = 2;
</script>

<div class="app">
  
  <DiceCanvas bind:count />
</div>

<style>
  :global(html, body, #app) { height: 100%; margin: 0; background: #0b0d10; }
  .app { height: 100%; padding: 14px; color: #e9eef3; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
  h1 { font-size: 18px; font-weight: 700; letter-spacing: .3px; opacity: .9; margin: 6px 0 12px; }
</style>


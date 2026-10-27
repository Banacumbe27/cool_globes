<script setup lang="ts">
import PlanetaryGlobe from './components/PlanetaryGlobe.vue'
</script>

<template>
  <main class="app-root">
    <PlanetaryGlobe />
  </main>
</template>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body, #app, .app-root {
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0;
  overflow: hidden;
  background-color: #05070f;
}
</style>

<script setup>
import BackToTop from './components/BackToTop.vue'
</script>

<template>
  <div class="mhome">
    <RouterView />
  </div>
  <BackToTop />
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family:
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans',
    'Helvetica Neue', sans-serif;
}
</style>
<script setup>
import { ref } from 'vue'
const menuOpen = ref(false)
const toggleMenu = () => {
  menuOpen.value = !menuOpen.value
}
const closeMenu = () => {
  menuOpen.value = false
}
</script>

<template>
  <header class="mh-nav" id="mhNav">
    <div class="mh-nav-inner">
      <a class="mh-brand" href="/">
        <img src="/logo-mark-transparent.png" alt="MelbCool" />
        <span>MelbCool</span>
      </a>
      <nav class="mh-links" aria-label="Main navigation" style="display:flex; justify-content:space-evenly; flex:1;">

        <RouterLink to="/heatmap" exact-active-class="active">Heat Map</RouterLink>
        <RouterLink to="/cool-refuges" exact-active-class="active">Cool Refuge</RouterLink>
        <RouterLink to="/trip-coach" exact-active-class="active">Trip Coach</RouterLink>
        <RouterLink to="/outfit-advisor" exact-active-class="active">Outfit Advisor</RouterLink>
        <RouterLink to="/safety-plan" exact-active-class="active">Safety Plan</RouterLink>
      </nav>
      
    </div>
  </header>
</template>
<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <div class="gen">
    <RouterView />
  </div>
</template>

<style scoped>
.gen{
  background-color: rgba(127, 255, 212, 0.267);
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>

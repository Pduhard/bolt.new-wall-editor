<script setup lang="ts">
import RoomConfigurator from './components/RoomConfigurator.vue';
</script>

<template>
  <div class="app-container">
    <h1>2D Room Wall Configurator</h1>
    <RoomConfigurator />
  </div>
</template>

<style scoped>
.app-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}
</style>
<script setup lang="ts">
import BabylonScene from './components/BabylonScene.vue'
import GlbViewer from './components/GlbViewer.vue';
// import BabylonViewer from './components/BabylonViewer.vue'
</script>

<template>
  <div class="app-container">
    <h1>Babylon.js GLB Viewer</h1>
    <!-- <BabylonScene /> -->
    <!-- Use with a model URL: -->
    <!-- <BabylonViewer modelUrl="./models/z.glb" /> -->
    <!-- Or without a model URL to show the Add Model button: -->
    <!-- <BabylonViewer /> -->
     <GlbViewer />
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  height: 100%;
  overflow: hidden; /* Prevent scrolling on the entire page */
}

body {
  font-family: Arial, sans-serif;
  overscroll-behavior: none; /* Prevent bounce effects on some browsers */
}

.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100%;
  overflow: hidden; /* Prevent scrolling in the container */
}

h1 {
  text-align: center;
  padding: 16px;
  background-color: #2c3e50;
  color: white;
  margin: 0;
  user-select: none; /* Prevent text selection */
}
</style>

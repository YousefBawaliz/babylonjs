<template>
    <div class="babylon-container">
      <canvas ref="babylonCanvas" class="babylon-canvas"></canvas>
      <div v-if="isLoading" class="loading-overlay">Loading Model...</div>
      <div v-if="errorLoading" class="error-overlay">Error loading model: {{ errorLoading }}</div>
      <div v-if="!hasModel && !isLoading" class="add-model-overlay">
        <button class="add-model-button" @click="triggerFileInput">Add Model</button>
        <input
          ref="fileInput"
          type="file"
          accept=".glb"
          class="file-input"
          @change="handleFileSelected"
        />
      </div>
    </div>
  </template>

  <script setup>
  import { ref, onMounted, onUnmounted, watch } from 'vue';
  import {
    Engine,
    Scene,
    ArcRotateCamera,
    Vector3,
    HemisphericLight,
    SceneLoader,
    Color4
  } from '@babylonjs/core';
  // Import the GLTF loader side effect (registers the loader)
  import '@babylonjs/loaders/glTF'; // Don't remove this even if unused directly

  // --- Props ---
  const props = defineProps({
    modelUrl: {
      type: String,
      required: false, // Now optional to allow for file upload
      default: ''
    }
  });

  // --- Emits ---
  const emit = defineEmits(['model-loaded', 'model-error']);

  // --- Refs ---
  const babylonCanvas = ref(null); // Ref to the canvas element
  const fileInput = ref(null); // Ref to the file input element
  const isLoading = ref(false);
  const errorLoading = ref(null);
  const hasModel = ref(false); // Track if a model is currently loaded

  // --- Babylon.js Variables ---
  let engine = null;
  let scene = null;
  let camera = null;

  // --- Functions ---

  const initializeBabylon = async () => {
    if (!babylonCanvas.value) {
      console.error("Canvas element not found!");
      return;
    }

    // Cleanup previous instance if any (useful for HMR or dynamic changes)
    disposeBabylon();

    isLoading.value = true;
    errorLoading.value = null;
    hasModel.value = false;

    try {
      // Create Engine
      engine = new Engine(babylonCanvas.value, true, { // Antialiasing enabled
        preserveDrawingBuffer: true,
        stencil: true,
        disableWebGL2Support: false // Use WebGL2 if available
      });

      // Create Scene
      scene = new Scene(engine);
      scene.clearColor = new Color4(0.9, 0.9, 0.95, 1); // Light gray background

      // Create Camera
      // Parameters: name, alpha, beta, radius, target position, scene
      camera = new ArcRotateCamera(
        "camera",
        -Math.PI / 2,        // Initial horizontal rotation
        Math.PI / 2.5,       // Initial vertical rotation
        10,                  // Initial distance from target
        Vector3.Zero(),      // Target point (center of the scene)
        scene
      );
      camera.attachControl(babylonCanvas.value, true); // Attach camera controls to the canvas
      camera.wheelPrecision = 50; // Adjust zoom sensitivity
      camera.lowerRadiusLimit = 2; // Prevent zooming too close
      camera.upperRadiusLimit = 50; // Prevent zooming too far out

      // Create Light
      // Parameters: name, direction, scene
      const light = new HemisphericLight("light", new Vector3(0, 1, 0), scene);
      light.intensity = 1.0; // Adjust light intensity

      // Start the render loop even if no model is loaded yet
      engine.runRenderLoop(() => {
        if (scene && scene.activeCamera) {
          scene.render();
        }
      });

      // Handle window resize
      window.addEventListener('resize', handleResize);

      // Only try to load a model if a URL is provided
      if (props.modelUrl) {
        await loadModel(props.modelUrl);
      } else {
        // No model to load, just finish initialization
        isLoading.value = false;
      }
    } catch (e) {
      console.error("Error initializing Babylon:", e);
      errorLoading.value = e.message || 'Unknown error occurred.';
      emit('model-error', e.message || 'Unknown error occurred.');
    } finally {
      isLoading.value = false;
    }
  };

  // New function to load a model from URL
  const loadModel = async (url) => {
    if (!scene) {
      console.error("Scene not initialized");
      return;
    }

    isLoading.value = true;
    errorLoading.value = null;

    try {
      // Load the GLB Model
      // SceneLoader.ImportMeshAsync returns a promise with { meshes, particleSystems, skeletons, animationGroups }
      // The first parameter "" means import all meshes
      // The second parameter is the root URL (directory)
      // The third is the filename
      let result;

      // Check if the URL is a file path or a blob URL
      if (url.startsWith('blob:')) {
        // For blob URLs, we need to handle differently (but this shouldn't happen anymore)
        console.warn('Blob URL detected in loadModel. This path should not be used anymore.');
        errorLoading.value = 'Please use the Add Model button to load models.';
        return;
      } else {
        // For file paths
        result = await SceneLoader.ImportMeshAsync(
          "", // Load all meshes in the file
          "", // Root URL (use empty string if modelUrl is absolute or relative to public)
          url, // The path to your model
          scene,
          (event) => {
            // Optional: Progress feedback
            // const percentComplete = event.lengthComputable ? (event.loaded / event.total * 100).toFixed(2) : 'N/A';
            // console.log(`Loading progress: ${percentComplete}%`);
          }
        );
      }

      // Optional: Auto-center and frame the loaded model
      if (result && result.meshes && result.meshes.length > 0) {
          // Find the bounding box of all loaded meshes
          let min = result.meshes[0].getBoundingInfo().boundingBox.minimumWorld;
          let max = result.meshes[0].getBoundingInfo().boundingBox.maximumWorld;

          for(let i = 1; i < result.meshes.length; i++){
              const meshMin = result.meshes[i].getBoundingInfo().boundingBox.minimumWorld;
              const meshMax = result.meshes[i].getBoundingInfo().boundingBox.maximumWorld;

              min = Vector3.Minimize(min, meshMin);
              max = Vector3.Maximize(max, meshMax);
          }

          const center = Vector3.Center(min, max);
          const extents = max.subtract(min);
          const size = Math.max(extents.x, extents.y, extents.z);
          const radius = size * 1.5; // Adjust multiplier for padding

          camera.setTarget(center);
          camera.radius = radius;
          // Ensure the camera distance limits make sense for the model size
          camera.lowerRadiusLimit = size * 0.1;
          camera.upperRadiusLimit = radius * 5;
      }

      // Mark that we have a model loaded
      hasModel.value = true;
      emit('model-loaded', url);

    } catch (e) {
      console.error("Error loading model:", e);
      errorLoading.value = e.message || 'Unknown error occurred.';
      emit('model-error', e.message || 'Unknown error occurred.');
    } finally {
      isLoading.value = false;
    }
  };

  const handleResize = () => {
    if (engine) {
      engine.resize();
    }
  };

  const disposeBabylon = () => {
    window.removeEventListener('resize', handleResize);
    if (engine) {
      engine.stopRenderLoop(); // Stop the render loop
      if (scene) {
          scene.dispose(); // Dispose scene resources
          scene = null;
      }
      engine.dispose(); // Dispose engine resources
      engine = null;
      camera = null; // Clear references
    }
  };

  // --- File Input Handling ---

  const triggerFileInput = () => {
    if (fileInput.value) {
      fileInput.value.click();
    }
  };

  const handleFileSelected = (event) => {
    const file = event.target.files[0];
    if (!file || !scene) return;

    isLoading.value = true;
    errorLoading.value = null;

    // Use FileReader to read the file as ArrayBuffer
    const reader = new FileReader();

    reader.onload = async (fileEvent) => {
      const fileContent = fileEvent.target?.result;
      if (!fileContent) {
        errorLoading.value = 'Error: Could not read file';
        isLoading.value = false;
        return;
      }

      try {
        // Import the mesh from the ArrayBuffer
        const result = await SceneLoader.ImportMeshAsync(
          "", // Load all meshes
          "", // Root URL (empty for ArrayBuffer)
          new Uint8Array(fileContent), // Pass the ArrayBuffer data
          scene,
          undefined, // onProgress callback
          ".glb" // Explicitly specify the file extension
        );

        // Process the loaded model (center it, etc.)
        if (result.meshes.length > 0) {
          // Find the bounding box of all loaded meshes
          let min = result.meshes[0].getBoundingInfo().boundingBox.minimumWorld;
          let max = result.meshes[0].getBoundingInfo().boundingBox.maximumWorld;

          for(let i = 1; i < result.meshes.length; i++){
            const meshMin = result.meshes[i].getBoundingInfo().boundingBox.minimumWorld;
            const meshMax = result.meshes[i].getBoundingInfo().boundingBox.maximumWorld;

            min = Vector3.Minimize(min, meshMin);
            max = Vector3.Maximize(max, meshMax);
          }

          const center = Vector3.Center(min, max);
          const extents = max.subtract(min);
          const size = Math.max(extents.x, extents.y, extents.z);
          const radius = size * 1.5; // Adjust multiplier for padding

          camera.setTarget(center);
          camera.radius = radius;
          // Ensure the camera distance limits make sense for the model size
          camera.lowerRadiusLimit = size * 0.1;
          camera.upperRadiusLimit = radius * 5;
        }

        // Mark that we have a model loaded
        hasModel.value = true;
        emit('model-loaded', file.name);

      } catch (e) {
        console.error("Error loading model from file:", e);
        errorLoading.value = e.message || 'Unknown error occurred.';
        emit('model-error', e.message || 'Unknown error occurred.');
      } finally {
        isLoading.value = false;
      }
    };

    reader.onerror = () => {
      errorLoading.value = 'Error: Failed to read the file';
      isLoading.value = false;
    };

    // Read the file as an ArrayBuffer
    reader.readAsArrayBuffer(file);

    // Reset the file input so the same file can be selected again if needed
    event.target.value = '';
  };

  // --- Lifecycle Hooks ---

  onMounted(() => {
    // Initialize Babylon.js once the component is mounted and the canvas is available
    initializeBabylon();
  });

  onUnmounted(() => {
    // Clean up Babylon.js resources when the component is unmounted
    disposeBabylon();
  });

  // Optional: Watch for changes in the modelUrl prop to reload the model
  watch(() => props.modelUrl, (newUrl, oldUrl) => {
    if (newUrl && newUrl !== oldUrl && babylonCanvas.value) {
      console.log(`Model URL changed to: ${newUrl}. Reloading...`);
      initializeBabylon(); // Re-initialize everything
    }
  });

  </script>

  <style scoped>
  .babylon-container {
    position: relative; /* Needed for absolute positioning of overlays */
    width: 800px;
    height: 800px;
    border: 1px solid #ccc; /* Optional: adds a border */
    overflow: hidden; /* Prevent content spillover */
  }

  .babylon-canvas {
    width: 100%;
    height: 100%;
    display: block; /* Prevents potential extra space below canvas */
    outline: none; /* Removes the default focus outline */
  }

  .loading-overlay,
  .error-overlay,
  .add-model-overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    background-color: rgba(255, 255, 255, 0.8);
    font-size: 1.2em;
    color: #333;
    z-index: 10; /* Ensure it's above the canvas */
  }

  .error-overlay {
      color: red;
      background-color: rgba(255, 200, 200, 0.8);
      padding: 20px;
      text-align: center;
      white-space: pre-wrap; /* Allow error message line breaks */
  }

  .add-model-button {
    padding: 12px 24px;
    background-color: #4CAF50;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 16px;
    cursor: pointer;
    transition: background-color 0.3s;
  }

  .add-model-button:hover {
    background-color: #45a049;
  }

  .file-input {
    display: none; /* Hide the actual file input */
  }
  </style>
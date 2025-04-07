<template>
  <div class="glb-viewer-wrapper">
    <h2>BabylonJS GLB Viewer</h2>

    <div class="file-input-container">
      <label for="glbFile">Select GLB File:</label>
      <input
        id="glbFile"
        type="file"
        accept=".glb"
        @change="handleFileChange"
        :disabled="isLoading || !isViewerReady"
      />
      <button @click="clearModel" :disabled="!isModelLoaded || isLoading || !isViewerReady" style="margin-left: 10px;">Clear Model</button>
    </div>

    <babylon-viewer ref="viewerElement" class="viewer-instance" environment="https://assets.babylonjs.com/environments/environmentSpecular.env"></babylon-viewer>

    <div v-if="isLoading" class="loading-overlay">Loading...</div> <div v-if="!isViewerReady && !loadError" class="loading-overlay">Initializing Viewer...</div>
    <div v-if="loadError" class="error-message">Error: {{ loadError }}</div>

  </div>
</template>

<script setup lang="ts">
import { ref, shallowRef, onMounted, onBeforeUnmount } from 'vue';

// Import just registers the element
import '@babylonjs/viewer';

// --- Refs ---
const viewerElement = ref<HTMLElement | any>(null);
const viewerInstance = shallowRef<any>(null);
let currentFile: File | null = null;
const isModelLoaded = ref(false);
const isLoading = ref(false);
const isViewerReady = ref(false);
const loadError = ref<string | null>(null);

// --- Event Listener Handlers ---
const onViewerReady = () => {
    console.log("viewerready event fired!");
    if (!viewerElement.value) return;

    const details = viewerElement.value.viewerDetails;
    if (details && details.viewer) {
        console.log("Successfully accessed viewer instance via viewerDetails");
        viewerInstance.value = details.viewer;
        isViewerReady.value = true;
        loadError.value = null;

        // Attach Model Status Listeners
        viewerElement.value.addEventListener('modelchange', onModelChange);
        viewerElement.value.addEventListener('modelerror', onModelError);
        console.log("Attached modelchange and modelerror listeners.");

    } else {
        console.error("viewerready event fired, but viewerDetails or viewerDetails.viewer is missing.", details);
        loadError.value = "Viewer initialized, but API instance could not be accessed.";
        isViewerReady.value = false;
    }
};

const onModelChange = (event: CustomEvent) => {
    const loadedSource = event.detail;
    console.log("modelchange event fired. Detail:", loadedSource);
    isLoading.value = false; // Stop loading indicator on any model change
    if (loadedSource) {
        isModelLoaded.value = true;
        loadError.value = null;
    } else {
        // This case handles if modelchange *does* fire for null
        isModelLoaded.value = false;
    }
};

const onModelError = (event: ErrorEvent) => {
    console.error("modelerror event fired:", event.error || event.message);
    isLoading.value = false; // Stop loading indicator on error
    isModelLoaded.value = false;
    loadError.value = event.message || (event.error ? event.error.toString() : "Unknown model loading error.");
};

// --- File Handling and Loading ---
const handleFileChange = async (event: Event) => {
  const input = event.target as HTMLInputElement;
  if (input.files && input.files[0]) {
    const file = input.files[0];
    currentFile = file;
    await loadModelIntoViewer(file);
  } else {
    // Don't necessarily clear if user cancels - maybe do nothing or provide feedback
    console.log("File selection cancelled or no file chosen.");
  }
  input.value = '';
};

// --- Programmatic Loading ---
const loadModelIntoViewer = async (file: File) => {
  if (!isViewerReady.value || !viewerInstance.value) {
    console.error("Viewer instance is not available or not ready yet.");
    loadError.value = "Viewer not ready. Cannot load model.";
    return;
  }

  console.log(`Attempting to load model via instance: ${file.name}`);
  isLoading.value = true; // Start loading indicator
  loadError.value = null; // Clear previous errors

  try {
    await viewerInstance.value.loadModel(file);
    console.log("loadModel called successfully. Waiting for events...");
    // State updates (isLoading=false, isModelLoaded) handled by onModelChange/onModelError
  } catch (error: any) {
    console.error("Error calling loadModel:", error);
    loadError.value = error.message || "Error initiating model load.";
    isLoading.value = false; // Stop loading on immediate error
    isModelLoaded.value = false;
  }
};

// --- Clear Model ---
const clearModel = async () => {
  if (!isViewerReady.value || !viewerInstance.value) {
      console.warn("Cannot clear model, viewer instance not available or ready.");
      return;
  }

  console.log("Attempting to clear model...");
  isLoading.value = true; // Show loading briefly
  loadError.value = null;
  currentFile = null;

  try {
      if (typeof viewerInstance.value.loadModel === 'function') {
          await viewerInstance.value.loadModel(null); // Pass null to unload
          console.log("loadModel(null) called.");
          // *** Manually reset state here after the call completes ***
          isModelLoaded.value = false;
          isLoading.value = false; // <-- Reset loading state directly
      } else {
          console.warn("loadModel() method not found on viewer instance to clear.");
          isLoading.value = false; // Ensure loading stops if method is missing
      }
  } catch(error: any) {
       console.error("Error trying to clear model:", error);
       loadError.value = "Failed to clear model.";
       isLoading.value = false; // Ensure loading stops on error
       isModelLoaded.value = false;
  }
  // No finally block needed as state is set directly after await
};


// --- Lifecycle Hooks ---
onMounted(() => {
  console.log("GlbViewer component mounted. Adding viewerready listener...");
  if (viewerElement.value) {
    viewerElement.value.addEventListener('viewerready', onViewerReady);
  } else {
      console.error("Cannot add viewerready listener, viewer element ref is not set.");
      loadError.value = "Viewer element could not be found in DOM.";
  }
});

onBeforeUnmount(() => {
  if (viewerElement.value) {
    viewerElement.value.removeEventListener('viewerready', onViewerReady);
    viewerElement.value.removeEventListener('modelchange', onModelChange);
    viewerElement.value.removeEventListener('modelerror', onModelError);
    console.log("Removed viewer event listeners.");
  }
  if (viewerInstance.value && typeof viewerInstance.value.dispose === 'function') {
    console.log("Disposing viewer instance...");
    viewerInstance.value.dispose();
  }
});

</script>

<style scoped>
/* Styles remain the same */
.glb-viewer-wrapper { border: 1px solid #ccc; padding: 15px; border-radius: 5px; display: flex; flex-direction: column; gap: 15px; position: relative; }
.file-input-container { display: flex; align-items: center; gap: 10px; }
.viewer-instance { width: 100%; height: 500px; display: block; border: 1px solid #eee; background-color: #f0f0f0; }
.loading-overlay { position: absolute; top: 50px; left: 0; right: 0; bottom: 0; background-color: rgba(255, 255, 255, 0.7); display: flex; justify-content: center; align-items: center; font-size: 1.2em; z-index: 10; }
.error-message { margin-top: 10px; color: red; font-weight: bold; }
</style>
<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue';
// Import Babylon.js core
import * as BABYLON from 'babylonjs';
// Import the GLB/GLTF loader explicitly
import { GLTFFileLoader } from 'babylonjs-loaders';

const canvasRef = ref<HTMLCanvasElement | null>(null);
const loadingStatus = ref<string>('');
let engine: any = null;
let scene: any = null;
let loadedModel: any[] = [];
let resizeHandler: any = null;

// Prevent default scrolling when interacting with the canvas
const preventScroll = (event: Event) => {
  event.preventDefault();
};

// Initialize Babylon.js scene
const initScene = () => {
  if (!canvasRef.value) return;

  // Add event listeners to prevent scrolling
  canvasRef.value.addEventListener('wheel', preventScroll, { passive: false });
  canvasRef.value.addEventListener('touchmove', preventScroll, { passive: false });

  // Create engine with anti-aliasing and adaptive quality
  engine = new BABYLON.Engine(canvasRef.value, true, {
    preserveDrawingBuffer: true,
    stencil: true,
    antialias: true,
    adaptToDeviceRatio: true
  });

  // Create scene with optimizations
  scene = new BABYLON.Scene(engine);
  scene.clearColor = new BABYLON.Color4(0.9, 0.9, 0.9, 1); // Light gray background
  scene.useRightHandedSystem = true; // Better for imported models

  // Enable hardware scaling to improve performance
  scene.setRenderingAutoClearDepthStencil(0, false, false);
  scene.setRenderingAutoClearDepthStencil(1, false, false);

  // Create camera with improved settings
  const camera = new BABYLON.ArcRotateCamera(
    'camera',
    -Math.PI / 2,
    Math.PI / 2.5,
    10,
    BABYLON.Vector3.Zero(),
    scene
  );

  // Improve camera controls
  camera.attachControl(canvasRef.value, true); // Attach with default settings
  camera.lowerRadiusLimit = 0.1;
  camera.upperRadiusLimit = 100;
  camera.wheelPrecision = 50; // Smoother zooming
  camera.pinchPrecision = 50;
  camera.panningSensibility = 50; // Smoother panning
  camera.inertia = 0.7; // Add inertia for smoother rotation
  camera.angularSensibilityX = 1000; // Reduce sensitivity to prevent jittering
  camera.angularSensibilityY = 1000;
  camera.useFramingBehavior = true; // Help with framing the model

  // Create primary light - hemispheric light for ambient lighting without shadows
  const hemisphericLight = new BABYLON.HemisphericLight(
    'hemisphericLight',
    new BABYLON.Vector3(0, 1, 0),
    scene
  );
  hemisphericLight.intensity = 0.7;
  hemisphericLight.specular = new BABYLON.Color3(0.1, 0.1, 0.1); // Reduce specular highlights

  // Add directional light for better illumination (without shadows)
  const directionalLight = new BABYLON.DirectionalLight(
    'directionalLight',
    new BABYLON.Vector3(0.5, -1, 0.5),
    scene
  );
  directionalLight.intensity = 0.5;

  // Disable shadows globally
  scene.shadowsEnabled = false;

  // We're not adding a ground plane to avoid shadows intersecting with the model

  // Configure scene for better rendering quality
  scene.skipFrustumClipping = false;
  scene.autoClear = true; // Enable auto clearing for proper rendering
  scene.autoClearDepthAndStencil = true;

  // Enable anti-aliasing post process
  const defaultPipeline = new BABYLON.DefaultRenderingPipeline(
    'defaultPipeline',
    true, // HDR not needed for most models
    scene,
    [scene.activeCamera]
  );

  // Configure anti-aliasing
  defaultPipeline.samples = 4; // MSAA sample count
  defaultPipeline.fxaaEnabled = true; // Enable FXAA

  // Add subtle bloom for better visual quality
  defaultPipeline.bloomEnabled = true;
  defaultPipeline.bloomThreshold = 0.8;
  defaultPipeline.bloomWeight = 0.3;
  defaultPipeline.bloomKernel = 64;
  defaultPipeline.bloomScale = 0.5;

  // Enable image processing for better colors
  defaultPipeline.imageProcessingEnabled = true;
  defaultPipeline.imageProcessing.contrast = 1.1;
  defaultPipeline.imageProcessing.exposure = 1.0;

  // Start render loop with optimizations
  engine.runRenderLoop(() => {
    if (scene && scene.activeCamera) {
      scene.render();
    }
  });

  // Handle window resize with debouncing
  let resizeTimeout: number | null = null;
  resizeHandler = () => {
    if (resizeTimeout) {
      clearTimeout(resizeTimeout);
    }

    resizeTimeout = setTimeout(() => {
      if (engine) {
        engine.resize();
      }
      resizeTimeout = null;
    }, 100) as unknown as number;
  };

  window.addEventListener('resize', resizeHandler);
};

// Handle file import
const handleFileImport = (event: Event) => {
  const target = event.target as HTMLInputElement;
  if (!target.files || !target.files[0] || !scene) return;

  const file = target.files[0];
  loadingStatus.value = `Loading ${file.name}...`;

  // Create a blob URL for the file
  const url = URL.createObjectURL(file);

  // Remove previously loaded model if exists
  if (loadedModel && loadedModel.length > 0) {
    loadedModel.forEach(mesh => {
      if (mesh && typeof mesh.dispose === 'function') {
        mesh.dispose();
      }
    });
    loadedModel = [];
  }

  // Make sure the GLTF loader is registered
  if (!BABYLON.SceneLoader.IsPluginForExtensionAvailable('.glb')) {
    console.log('Registering GLB loader plugin');
    BABYLON.SceneLoader.RegisterPlugin(new GLTFFileLoader());
  }

  // Load the file directly using FileReader instead of blob URL
  const reader = new FileReader();

  reader.onload = (fileEvent) => {
    const fileContent = fileEvent.target?.result;
    if (!fileContent) {
      loadingStatus.value = 'Error: Could not read file';
      return;
    }

    try {
      // Load from ArrayBuffer
      BABYLON.SceneLoader.ImportMesh(
        '',
        '',
        new Uint8Array(fileContent as ArrayBuffer),
        scene,
        (meshes) => {
          console.log('Model loaded successfully:', meshes);
          loadingStatus.value = 'Model loaded successfully';

          // Store loaded meshes for later disposal if needed
          loadedModel = meshes;

          if (meshes.length === 0) {
            console.warn('No meshes were loaded');
            loadingStatus.value = 'Warning: No meshes were loaded';
            return;
          }

          // Create a parent container for all meshes to make manipulation easier
          const modelRoot = new BABYLON.TransformNode('modelRoot', scene);

          // Parent all meshes to the container and disable shadows
          meshes.forEach(mesh => {
            if (mesh.parent === null) {
              mesh.parent = modelRoot;
            }

            // Disable shadows on all meshes
            if (mesh.receiveShadows !== undefined) {
              mesh.receiveShadows = false;
            }
          });

          // Center the model
          if (meshes.length > 0) {
            // Create a bounding box info for the entire model
            let min = new BABYLON.Vector3(Number.MAX_VALUE, Number.MAX_VALUE, Number.MAX_VALUE);
            let max = new BABYLON.Vector3(Number.MIN_VALUE, Number.MIN_VALUE, Number.MIN_VALUE);

            // Find the bounding box for all meshes
            meshes.forEach(mesh => {
              if (mesh.getBoundingInfo && typeof mesh.getBoundingInfo === 'function') {
                const boundingInfo = mesh.getBoundingInfo();
                const meshMin = boundingInfo.boundingBox.minimumWorld;
                const meshMax = boundingInfo.boundingBox.maximumWorld;

                min = BABYLON.Vector3.Minimize(min, meshMin);
                max = BABYLON.Vector3.Maximize(max, meshMax);
              }
            });

            // Calculate center and size
            const center = BABYLON.Vector3.Center(min, max);
            const size = max.subtract(min);
            const maxSize = Math.max(size.x, size.y, size.z);

            console.log('Model center:', center);
            console.log('Model size:', size);
            console.log('Max size:', maxSize);

            // Move the model root to center it at origin
            modelRoot.position = center.scale(-1);

            // Adjust camera distance based on model size
            if (scene.activeCamera && scene.activeCamera instanceof BABYLON.ArcRotateCamera) {
              const camera = scene.activeCamera as BABYLON.ArcRotateCamera;
              camera.setTarget(BABYLON.Vector3.Zero()); // Look at origin where model is now centered
              camera.radius = maxSize * 2;

              // Ensure the camera is looking at the model from a good angle
              camera.alpha = Math.PI / 4; // 45 degrees
              camera.beta = Math.PI / 3;  // 60 degrees

              // Apply smooth transition to camera
              camera.rebuildAnglesAndRadius();

              // Enable camera framing behavior for this model
              if (camera.framingBehavior) {
                camera.framingBehavior.framingTime = 500; // ms
                camera.framingBehavior.elevationReturnTime = 1500; // ms
                camera.framingBehavior.zoomStopsAnimation = false;
                camera.framingBehavior.framingTime = 500;

                // Frame the model
                setTimeout(() => {
                  camera.framingBehavior?.zoomOnMeshesHierarchy(meshes);
                }, 100);
              }
            }
          }
        },
        (event) => {
          // Progress handling
          if (event.lengthComputable) {
            const percentage = Math.round((event.loaded / event.total) * 100);
            loadingStatus.value = `Loading: ${percentage}%`;
            console.log(`Loading progress: ${percentage}%`);
          }
        },
        (_, message, exception) => {
          // Error handling
          console.error('Error loading model:', message, exception);
          loadingStatus.value = `Error: ${message}`;
        },
        '.glb' // Explicitly specify the file extension
      );
    } catch (error) {
      console.error('Exception during model loading:', error);
      loadingStatus.value = `Exception: ${error instanceof Error ? error.message : String(error)}`;
    }
  };

  reader.onerror = () => {
    loadingStatus.value = 'Error: Failed to read the file';
  };

  // Read the file as an ArrayBuffer
  reader.readAsArrayBuffer(file);

  // Clean up the blob URL since we're not using it
  URL.revokeObjectURL(url);
};

// Lifecycle hooks
onMounted(() => {
  // Initialize the scene after the component is mounted and DOM is fully rendered
  window.addEventListener('load', () => {
    // Force a layout recalculation
    if (canvasRef.value) {
      const width = canvasRef.value.clientWidth;
      const height = canvasRef.value.clientHeight;
      console.log(`Canvas dimensions: ${width}x${height}`);

      // Set explicit dimensions
      canvasRef.value.width = width;
      canvasRef.value.height = height;
    }

    // Initialize the scene
    initScene();
  });

  // Fallback initialization if the load event already fired
  setTimeout(() => {
    if (!engine) {
      console.log('Fallback initialization');
      initScene();
    }
  }, 500);
});

onBeforeUnmount(() => {
  // Clean up resources
  if (loadedModel && loadedModel.length > 0) {
    loadedModel.forEach(mesh => {
      if (mesh && typeof mesh.dispose === 'function') {
        mesh.dispose();
      }
    });
  }

  if (scene) {
    scene.dispose();
  }

  if (engine) {
    engine.dispose();
  }

  // Remove event listeners
  if (canvasRef.value) {
    canvasRef.value.removeEventListener('wheel', preventScroll);
    canvasRef.value.removeEventListener('touchmove', preventScroll);
  }

  // Remove the resize event listener
  if (resizeHandler) {
    window.removeEventListener('resize', resizeHandler);
  }
});
</script>

<template>
  <div class="babylon-container">
    <div class="controls-bar">
      <label for="file-input" class="import-button">Import GLB File</label>
      <input
        id="file-input"
        ref="fileInputRef"
        type="file"
        accept=".glb"
        @change="handleFileImport"
        class="file-input"
      />
      <span v-if="loadingStatus" class="loading-status">{{ loadingStatus }}</span>
    </div>
    <div class="canvas-wrapper">
      <canvas ref="canvasRef" class="render-canvas"></canvas>
    </div>
  </div>
</template>

<style scoped>
.babylon-container {
  display: flex;
  flex-direction: column;
  width: 100%;
  height: calc(100vh - 42px); /* Adjust for header height */
}

.controls-bar {
  padding: 10px;
  background-color: #f5f5f5;
  text-align: center;
  border-bottom: 1px solid #ddd;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-wrap: wrap;
  gap: 10px;
}

.import-button {
  display: inline-block;
  padding: 10px 20px;
  background-color: #4CAF50;
  color: white;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.import-button:hover {
  background-color: #45a049;
}

.file-input {
  display: none;
}

.loading-status {
  font-weight: bold;
  color: #333;
  background-color: rgba(255, 255, 255, 0.8);
  padding: 5px 10px;
  border-radius: 4px;
}

.canvas-wrapper {
  flex: 1;
  position: relative;
  overflow: hidden;
  min-height: 400px; /* Ensure minimum height */
  background-color: #e0e0e0; /* Visual indicator for canvas area */
  touch-action: none; /* Prevent touch actions from propagating */
}

.render-canvas {
  width: 100%;
  height: 100%;
  touch-action: none;
  display: block;
  background-color: #f0f0f0; /* Visual indicator for canvas */
  /* Prevent default browser behaviors */
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}
</style>

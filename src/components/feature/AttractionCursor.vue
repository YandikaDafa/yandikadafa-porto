<template>
  <div :class="['attraction-cursor', { 'is-fullscreen': fullscreen }]">
    <canvas ref="canvasEl" class="attraction-canvas"></canvas>

    <div class="attraction-slot">
      <slot />
    </div>

    <div v-if="showControls && fullscreen" class="attraction-controls">
      <button type="button" class="control-btn" @click="randomizeColors">
        Random colors
      </button>
      <a
        class="control-btn"
        href="https://www.framer.com/@kevin-levron/"
        target="_blank"
        rel="noopener"
      >
        Framer Component
      </a>
    </div>
  </div>
</template>

<script setup>
import { onBeforeUnmount, onMounted, ref, shallowRef } from "vue";

const props = defineProps({
  fullscreen: {
    type: Boolean,
    default: true,
  },
  attractionIntensity: {
    type: Number,
    default: 0.75,
  },
  particleSize: {
    type: Number,
    default: 1.5,
  },
  showControls: {
    type: Boolean,
    default: true,
  },
});

const canvasEl = ref(null);
const cursorApp = shallowRef(null);

const randomizeColors = () => {
  const instance = cursorApp.value;
  if (!instance?.particles) return;

  instance.particles.light1.color.set(Math.random() * 0xffffff);
  instance.particles.light2.color.set(Math.random() * 0xffffff);
};

onMounted(async () => {
  const canvas = canvasEl.value;
  if (!canvas) return;

  const mod = await import(
    "https://cdn.jsdelivr.net/npm/threejs-components@0.0.26/build/cursors/attraction1.min.js"
  );

  const AttractionCursor = mod.default || mod;
  cursorApp.value = AttractionCursor(canvas, {
    particles: {
      attractionIntensity: props.attractionIntensity,
      size: props.particleSize,
    },
  });
});

onBeforeUnmount(() => {
  cursorApp.value?.destroy?.();
});
</script>

<style scoped>
.attraction-cursor {
  position: relative;
  width: 100%;
  height: 100%;
  pointer-events: auto;
  z-index: 0;
}

.attraction-cursor.is-fullscreen {
  position: fixed;
  inset: 0;
}

.attraction-canvas {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  pointer-events: none;
}

.attraction-slot {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1;
  pointer-events: auto;
}

.attraction-controls {
  position: fixed;
  width: 100%;
  bottom: 15px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 15px;
  z-index: 2;
  pointer-events: auto;
}

.control-btn {
  color: black;
  font-size: 12px;
  text-decoration: none;
  background: rgba(255, 255, 255, 0.5);
  border-radius: 5px;
  border: 1px solid grey;
  padding: 6px 10px;
  font-family: inherit;
}

.control-btn:hover {
  background: rgba(255, 255, 255, 0.75);
}
</style>

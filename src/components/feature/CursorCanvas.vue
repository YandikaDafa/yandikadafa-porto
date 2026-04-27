<template>
  <canvas ref="canvasEl" id="cursorCanvas"></canvas>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from "vue";

const canvasEl = ref(null);
let tubesApp = null;
let clickHandler = null;

onMounted(async () => {
  const cursorCanvas = canvasEl.value;
  if (!cursorCanvas) return;

  const mod = await import(
    "https://cdn.jsdelivr.net/npm/threejs-components@0.0.19/build/cursors/tubes1.min.js"
  );
  const TubesCursor = mod.default || mod;

  tubesApp = TubesCursor(cursorCanvas, {
    tubes: {
      colors: ["#f967fb", "#53bc28", "#6958d5"],
      lights: {
        intensity: 200,
        colors: ["#83f36e", "#fe8a2e", "#ff008a", "#60aed5"],
      },
    },
  });

  const randomColors = (count) =>
    new Array(count)
      .fill(0)
      .map(
        () =>
          "#" +
          Math.floor(Math.random() * 16777215)
            .toString(16)
            .padStart(6, "0")
      );

  clickHandler = () => {
    if (!tubesApp) return;
    const colors = randomColors(3);
    const lightsColors = randomColors(4);
    tubesApp.tubes.setColors(colors);
    tubesApp.tubes.setLightsColors(lightsColors);
  };

  document.body.addEventListener("click", clickHandler);
});

onBeforeUnmount(() => {
  if (clickHandler) {
    document.body.removeEventListener("click", clickHandler);
  }

  if (tubesApp && typeof tubesApp.destroy === "function") {
    tubesApp.destroy();
  }
});
</script>

<style scoped>
#cursorCanvas {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  overflow: hidden;
  z-index: 0;
}
</style>

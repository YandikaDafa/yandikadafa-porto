<template>
  <Teleport to="body">
    <div
      ref="cursorRef"
      class="circle-cursor"
      :style="{
        transform: `translate(${position.x}px, ${position.y}px)`,
      }"
    />
  </Teleport>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";

const cursorRef = ref(null);
const position = ref({ x: -100, y: -100 });
const targetPosition = ref({ x: -100, y: -100 });

let animationFrameId = null;

const onMouseMove = (e) => {
  targetPosition.value = {
    x: e.clientX,
    y: e.clientY,
  };
};

const animate = () => {
  // Smooth easing towards target position
  const ease = 0.15;
  position.value.x += (targetPosition.value.x - position.value.x) * ease;
  position.value.y += (targetPosition.value.y - position.value.y) * ease;

  animationFrameId = requestAnimationFrame(animate);
};

onMounted(() => {
  window.addEventListener("mousemove", onMouseMove);
  animate();
});

onUnmounted(() => {
  window.removeEventListener("mousemove", onMouseMove);
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
});
</script>

<style scoped>
.circle-cursor {
  position: fixed;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #000;
  pointer-events: none;
  z-index: 10000;
  top: -20px;
  left: -20px;
  mix-blend-mode: difference;
  will-change: transform;
}

/* Hide on touch devices */
@media (hover: none) {
  .circle-cursor {
    display: none;
  }
}
</style>

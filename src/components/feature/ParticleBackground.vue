<template>
  <canvas ref="canvas" class="particle-background"></canvas>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";

const canvas = ref<HTMLCanvasElement | null>(null);
let ctx: CanvasRenderingContext2D | null = null;
let animationId: number | null = null;
let particles: Particle[] = [];

interface Particle {
  x: number;
  y: number;
  ox: number; // original x
  oy: number; // original y
  vx: number; // velocity x
  vy: number; // velocity y
}

const mouse = {
  x: 0,
  y: 0,
  prevX: 0,
  prevY: 0,
};

const config = {
  particleCount: 1500,
  repulsionRadius: 150,
  repulsionForce: 0.8,
  returnForce: 0.02,
  damping: 0.95,
  particleSize: 1.5,
  particleColor: "rgba(51, 51, 51, 0.4)",
};

function createParticles(width: number, height: number) {
  particles = [];
  for (let i = 0; i < config.particleCount; i++) {
    const x = Math.random() * width;
    const y = Math.random() * height;
    particles.push({
      x,
      y,
      ox: x,
      oy: y,
      vx: 0,
      vy: 0,
    });
  }
}

function updateParticles() {
  particles.forEach((particle) => {
    // Calculate distance from mouse
    const dx = particle.x - mouse.x;
    const dy = particle.y - mouse.y;
    const distance = Math.sqrt(dx * dx + dy * dy);

    // Apply repulsion force if within radius
    if (distance < config.repulsionRadius && distance > 0) {
      const force =
        (1 - distance / config.repulsionRadius) * config.repulsionForce;

      // Add cursor velocity to the repulsion (for "terhempas" effect)
      const mouseDx = mouse.x - mouse.prevX;
      const mouseDy = mouse.y - mouse.prevY;
      const mouseSpeed = Math.sqrt(mouseDx * mouseDx + mouseDy * mouseDy);
      const velocityBoost = Math.min(mouseSpeed * 0.1, 2);

      const angle = Math.atan2(dy, dx);
      particle.vx += Math.cos(angle) * (force + velocityBoost);
      particle.vy += Math.sin(angle) * (force + velocityBoost);
    }

    // Apply return force (spring back to original position)
    const returnDx = particle.ox - particle.x;
    const returnDy = particle.oy - particle.y;
    particle.vx += returnDx * config.returnForce;
    particle.vy += returnDy * config.returnForce;

    // Apply damping
    particle.vx *= config.damping;
    particle.vy *= config.damping;

    // Update position
    particle.x += particle.vx;
    particle.y += particle.vy;
  });

  // Update previous mouse position
  mouse.prevX = mouse.x;
  mouse.prevY = mouse.y;
}

function drawParticles() {
  if (!ctx || !canvas.value) return;

  // Clear canvas
  ctx.clearRect(0, 0, canvas.value.width, canvas.value.height);

  // Draw particles
  ctx.fillStyle = config.particleColor;
  particles.forEach((particle) => {
    ctx!.beginPath();
    ctx!.arc(particle.x, particle.y, config.particleSize, 0, Math.PI * 2);
    ctx!.fill();
  });
}

function animate() {
  updateParticles();
  drawParticles();
  animationId = requestAnimationFrame(animate);
}

function handleMouseMove(e: MouseEvent) {
  const rect = canvas.value?.getBoundingClientRect();
  if (rect) {
    mouse.x = e.clientX - rect.left;
    mouse.y = e.clientY - rect.top;
  }
}

function handleResize() {
  if (!canvas.value) return;

  canvas.value.width = window.innerWidth;
  canvas.value.height = window.innerHeight;

  createParticles(canvas.value.width, canvas.value.height);
}

onMounted(() => {
  if (!canvas.value) return;

  ctx = canvas.value.getContext("2d");
  if (!ctx) return;

  // Set canvas size
  canvas.value.width = window.innerWidth;
  canvas.value.height = window.innerHeight;

  // Create particles
  createParticles(canvas.value.width, canvas.value.height);

  // Start animation
  animate();

  // Add event listeners
  window.addEventListener("mousemove", handleMouseMove);
  window.addEventListener("resize", handleResize);
});

onUnmounted(() => {
  if (animationId !== null) {
    cancelAnimationFrame(animationId);
  }
  window.removeEventListener("mousemove", handleMouseMove);
  window.removeEventListener("resize", handleResize);
});
</script>

<style scoped>
.particle-background {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  pointer-events: none;
}
</style>

<template>
  <div class="magic-sphere-wrapper">
    <Transition :name="slideDirection">
      <div 
        :key="activeTitle"
        class="magic-sphere-container"
      >
        <div class="sphere" :style="getSphereStyle(activeColor)">
          <div class="sphere-highlight"></div>
          <div class="sphere-reflection"></div>
          <span class="sphere-text">{{ activeTitle }}</span>
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { computed, ref, watch } from 'vue';

const props = defineProps({
  activeTitle: {
    type: String,
    default: 'Design'
  },
  sections: {
    type: Array,
    default: () => [
      { title: 'Design', color: '#8A2BE2' },
      { title: 'Development', color: '#FFFF00' },
      { title: 'Content', color: '#DC143C' }
    ]
  }
});

const slideDirection = ref('slide-left');
const previousIndex = ref(0);

// Get current index
const currentIndex = computed(() => {
  return props.sections.findIndex(s => s.title === props.activeTitle);
});

// Get active color
const activeColor = computed(() => {
  const section = props.sections.find(s => s.title === props.activeTitle);
  return section?.color || '#8A2BE2';
});

// Watch for section changes to determine direction
watch(currentIndex, (newIndex, oldIndex) => {
  if (newIndex > oldIndex) {
    // Scrolling down - slide left
    slideDirection.value = 'slide-left';
  } else {
    // Scrolling up - slide right
    slideDirection.value = 'slide-right';
  }
  previousIndex.value = newIndex;
});

function getSphereStyle(color) {
  return {
    '--sphere-color': color,
    '--sphere-light': hexToRgba(color, 0.8),
    '--sphere-dark': hexToRgba(darkenColor(color, 40), 1)
  };
}

function hexToRgba(hex, alpha = 1) {
  const r = parseInt(hex.slice(1, 3), 16);
  const g = parseInt(hex.slice(3, 5), 16);
  const b = parseInt(hex.slice(5, 7), 16);
  return `rgba(${r}, ${g}, ${b}, ${alpha})`;
}

function darkenColor(hex, percent) {
  const num = parseInt(hex.replace("#", ""), 16);
  const amt = Math.round(2.55 * percent);
  const R = Math.max((num >> 16) - amt, 0);
  const G = Math.max((num >> 8 & 0x00FF) - amt, 0);
  const B = Math.max((num & 0x0000FF) - amt, 0);
  return "#" + (0x1000000 + R * 0x10000 + G * 0x100 + B).toString(16).slice(1);
}
</script>

<style scoped>
.magic-sphere-wrapper {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
}

.magic-sphere-container {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  position: absolute;
  top: 0;
  left: 0;
}

.sphere {
  width: 220px;
  height: 220px;
  border-radius: 50%;
  background: radial-gradient(
    circle at 30% 30%,
    var(--sphere-light),
    var(--sphere-color) 40%,
    var(--sphere-dark) 100%
  );
  box-shadow: 
    inset -20px -20px 40px rgba(0, 0, 0, 0.4),
    inset 10px 10px 30px rgba(255, 255, 255, 0.2),
    0 20px 40px rgba(0, 0, 0, 0.5),
    0 0 60px var(--sphere-color);
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.sphere-highlight {
  position: absolute;
  width: 60%;
  height: 60%;
  top: 10%;
  left: 15%;
  background: radial-gradient(
    ellipse at 50% 50%,
    rgba(255, 255, 255, 0.6) 0%,
    rgba(255, 255, 255, 0.2) 40%,
    transparent 70%
  );
  border-radius: 50%;
  transform: rotate(-30deg);
  pointer-events: none;
}

.sphere-reflection {
  position: absolute;
  width: 30%;
  height: 15%;
  bottom: 15%;
  right: 20%;
  background: radial-gradient(
    ellipse at 50% 50%,
    rgba(255, 255, 255, 0.3) 0%,
    transparent 70%
  );
  border-radius: 50%;
  filter: blur(3px);
  pointer-events: none;
}

.sphere-text {
  color: white;
  font-size: 1.3rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 2px;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
  z-index: 10;
  text-align: center;
}

/* Slide LEFT transition (scroll down) */
.slide-left-enter-active,
.slide-left-leave-active {
  transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
}

.slide-left-enter-from {
  transform: translateX(100%);
  opacity: 0;
}

.slide-left-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}

/* Slide RIGHT transition (scroll up) */
.slide-right-enter-active,
.slide-right-leave-active {
  transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
}

.slide-right-enter-from {
  transform: translateX(-100%);
  opacity: 0;
}

.slide-right-leave-to {
  transform: translateX(100%);
  opacity: 0;
}
</style>

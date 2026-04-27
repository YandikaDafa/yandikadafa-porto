<template>
  <Transition name="intro">
    <div class="intro-screen" ref="introScreenRef" v-show="visible">
      <div class="intro-overlay" ref="overlayRef">
        <svg class="hole-mask" ref="holeMaskRef">
          <defs>
            <mask id="hole-mask">
              <rect width="100%" height="100%" fill="white" />
              <circle
                ref="holeCircleRef"
                cx="50%"
                cy="50%"
                r="0"
                fill="black"
              />
            </mask>
          </defs>
          <rect
            width="100%"
            height="100%"
            fill="black"
            mask="url(#hole-mask)"
          />
        </svg>
      </div>

      <div class="portal-rings-layer">
        <div class="portal-ring" ref="portalRing1"></div>
        <div class="portal-ring-2" ref="portalRing2"></div>
        <div class="portal-ring-3" ref="portalRing3"></div>
      </div>

      <div class="d-flex align-end justify-center ga-3 brand-container">
        <div class="text-white text-h2 text-sm-h1 font-weight-medium d-flex">
          <div class="brand-text-animation">
            <span>N</span>
          </div>
          <div class="brand-text-animation">
            <span>A</span>
          </div>
          <div class="brand-text-animation" :style="$vuetify.display.xs ? 'margin-left: -10px' : 'margin-left: -15px'">
            <span>V</span>
          </div>
          <div class="brand-text-animation">
            <span>I</span>
          </div>
          <div class="brand-text-animation">
            <span>A</span>
          </div>
        </div>
        <div class="d-flex" style="margin-bottom: 8px">
          <div class="brand-text-one-animation">
            <span>navigate&nbsp;</span>
          </div>
          <div class="brand-text-two-animation">
            <span>your&nbsp;</span>
          </div>
          <div class="brand-text-three-animation">
            <span>dreams.</span>
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import gsap from "gsap";

const introScreenRef = ref(null);
const overlayRef = ref(null);
const holeCircleRef = ref(null);
const portalRing1 = ref(null);
const portalRing2 = ref(null);
const portalRing3 = ref(null);

const visible = ref(false);

let tlIn = null;
let tlOut = null;

// ===============================
// PLAY IN (sampai slogan muncul)
// ===============================
async function playIn() {
  if (!introScreenRef.value) return;

  visible.value = true;

  // kill timeline lama kalau ada
  if (tlIn) {
    tlIn.kill();
    tlIn = null;
  }
  if (tlOut) {
    tlOut.kill();
    tlOut = null;
  }

  const screen = introScreenRef.value;
  const overlay = overlayRef.value;
  const circle = holeCircleRef.value;
  const rings = [portalRing1.value, portalRing2.value, portalRing3.value];

  // kill tweens yang menempel ke elemen-elemen ini
  gsap.killTweensOf([
    screen,
    overlay,
    circle,
    rings,
    ".brand-container",
    ".brand-text-animation",
    ".brand-text-one-animation",
    ".brand-text-two-animation",
    ".brand-text-three-animation",
    "#page-animation",
  ]);

  // BASELINE STATE — penting untuk bisa diulang tanpa reset external
  gsap.set(".brand-container", {
    scale: 1,
    rotation: 0,
    x: 0,
    y: 0,
    xPercent: 0,
    transformOrigin: "center center",
  });

  gsap.set(".brand-text-animation", {
    x: 0,
    y: 0,
    scale: 1,
    opacity: 1,
    rotation: 0,
  });

  gsap.set(
    ".brand-text-one-animation, .brand-text-two-animation, .brand-text-three-animation",
    {
      x: 0,
      y: 0,
      scale: 1,
      opacity: 1,
      rotation: 0,
    }
  );

  if (circle) {
    gsap.set(circle, { attr: { r: 0 } });
  }

  if (overlay) {
    gsap.set(overlay, {
      scale: 1,
      x: 0,
      y: 0,
      rotation: 0,
      transformOrigin: "center center",
    });
  }

  rings.forEach((ring) => {
    if (!ring) return;
    gsap.set(ring, {
      opacity: 0,
      scale: 1,
      x: 0,
      y: 0,
      rotation: 0,
      transform: "translate(-50%, -50%)",
    });
  });

  // zoom page content di belakang (opsional)
  // gsap.utils.toArray("#page-animation").forEach((el) => {
  //   gsap.to(el, {
  //     scale: 5,
  //     duration: 1,
  //     ease: "power4.inOut",
  //   });
  // });

  tlIn = gsap.timeline();

  // transisi awal
  tlIn.fromTo(
    overlay,
    { opacity: 0 },
    { opacity: 1, duration: 1, ease: "power4.out" }
  );

  // NAVIA
  tlIn.from(
    ".brand-text-animation",
    {
      y: 200,
      x: 1000,      
      scale: 10,
      opacity: 0,
      duration: 0.65,
      stagger: 0.1,
      ease: "power4.out",
    },
    0
  );

  // slogan
  tlIn.from(
    ".brand-text-one-animation, .brand-text-two-animation, .brand-text-three-animation",
    {
      y: 100,
      opacity: 0,
      duration: 0.65,
      stagger: 0.2,
      ease: "power4.out",
    },
    1.25
  );

  return new Promise((resolve) => {
    tlIn.eventCallback("onComplete", () => resolve());
  });
}

// ===============================
// PLAY OUT (portal, ring, hole, fade keluar ke route baru)
// ===============================
async function playOut() {
  if (!introScreenRef.value) {
    visible.value = false;
    return;
  }

  const portalRings = [portalRing1.value, portalRing2.value, portalRing3.value];

  if (tlOut) {
    tlOut.kill();
    tlOut = null;
  }

  tlOut = gsap.timeline();

  const maxRadius = Math.hypot(window.innerWidth, window.innerHeight);
  const portalStartTime = 0;

  // gsap.utils.toArray("#page-animation").forEach((el) => {
  //   gsap.set(el, {
  //     scale: 1,
  //   });
  // });

  // brand-container zoom + spin (NAVIA masuk portal)
  tlOut.fromTo(
    ".brand-container",
    {
      scale: 1,
      rotation: 0,      
    },
    {
      scale: 200,
      rotation: -70,
      xPercent: 350,
      duration: 1.75,
      ease: "power4.inOut",
    },
    portalStartTime
  );

  // ring muncul
  tlOut.to(
    portalRings,
    {
      opacity: 1,
      duration: 0.65,
    },
    portalStartTime + 0.5
  );

  // ring1
  if (portalRings[0]) {
    tlOut.to(
      portalRings[0],
      {
        scale: 60,
        opacity: 0,
        rotation: -360,
        duration: 1.5,
        ease: "power2.in",
      },
      portalStartTime
    );
  }

  // ring2
  if (portalRings[1]) {
    tlOut.to(
      portalRings[1],
      {
        scale: 55,
        opacity: 0,
        rotation: 360,
        duration: 1.45,
        ease: "power2.in",
      },
      portalStartTime + 0.05
    );
  }

  // ring3
  if (portalRings[2]) {
    tlOut.to(
      portalRings[2],
      {
        scale: 50,
        opacity: 0,
        rotation: -360,
        duration: 1.4,
        ease: "power2.in",
      },
      portalStartTime + 0.1
    );
  }

  // hole mask membesar
  if (holeCircleRef.value) {
    tlOut.to(
      holeCircleRef.value,
      {
        attr: { r: maxRadius },
        duration: 1.5,
        ease: "power2.inOut",
      },
      portalStartTime + 0.25
    );
  }

  // overlay scale
  if (overlayRef.value) {
    tlOut.to(
      overlayRef.value,
      {
        scale: 50,
        duration: 1.5,
        ease: "power2.inOut",
      },
      portalStartTime + 0.25
    );
  }

  tlOut.add(() => {
    visible.value = false;
  }, portalStartTime + 1.75);

  return new Promise((resolve) => {
    tlOut.eventCallback("onComplete", () => resolve());
  });
}

// ===============================
// MOUNT / UNMOUNT
// ===============================
onMounted(() => {
  // dikontrol dari router
  window.__routeIntro = {
    playIn,
    playOut,
    visible,
  };
});

function cleanup() {
  if (tlIn) {
    tlIn.kill();
    tlIn = null;
  }
  if (tlOut) {
    tlOut.kill();
    tlOut = null;
  }

  gsap.killTweensOf([
    introScreenRef.value,
    overlayRef.value,
    holeCircleRef.value,
    portalRing1.value,
    portalRing2.value,
    portalRing3.value,
    ".brand-container",
    ".brand-text-animation",
    ".brand-text-one-animation",
    ".brand-text-two-animation",
    ".brand-text-three-animation",
    "#page-animation",
  ]);
}

onUnmounted(() => {
  cleanup();
  if (window.__routeIntro) {
    delete window.__routeIntro;
  }
});
</script>

<style scoped>
.intro-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  overflow: hidden;
  pointer-events: none;
}

.intro-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  pointer-events: none;
  transform-origin: center center;
  will-change: auto;
}

.portal-rings-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  pointer-events: none;
}

.hole-mask {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}

.brand-container {
  z-index: 1;
  pointer-events: auto;
  will-change: auto;
}

.portal-ring,
.portal-ring-2,
.portal-ring-3 {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border-radius: 50%;
  border-style: solid;
  pointer-events: none;
  opacity: 0;
  will-change: auto;
}

.portal-ring {
  width: 80px;
  height: 80px;
  border: 4px solid rgba(255, 255, 255, 0.7);
  /* box-shadow: 0 0 40px rgba(255, 255, 255, 0.8),
    inset 0 0 40px rgba(255, 255, 255, 0.4); */
}

.portal-ring-2 {
  width: 140px;
  height: 140px;
  border: 3px solid rgba(255, 255, 255, 0.5);
  /* box-shadow: 0 0 30px rgba(255, 255, 255, 0.6),
    inset 0 0 30px rgba(255, 255, 255, 0.3); */
}

.portal-ring-3 {
  width: 200px;
  height: 200px;
  border: 2px solid rgba(255, 255, 255, 0.35);
  /* box-shadow: 0 0 25px rgba(255, 255, 255, 0.4),
    inset 0 0 25px rgba(255, 255, 255, 0.2); */
}
</style>

<template>
  <v-container
    fluid
    :class="$vuetify.display.smAndDown ? undefined : 'px-16'"
    class="d-flex flex-column align-center justify-center bg-grey-lighten-4"    
    min-height="100vh"
  >
    <div class="overflow-hidden pb-2">
      <p class="text-h3 text-sm-h2 text-md-h1 opacity-80 line-animation">
        my code works,
      </p>
    </div>
    <div class="overflow-hidden pb-2">
      <p class="text-h3 text-sm-h2 text-md-h1 opacity-80 line-animation">
        I don't know why.
      </p>
    </div>
    <div class="overflow-hidden pb-2">
      <p class="text-h3 text-sm-h2 text-md-h1 opacity-80 line-animation">
        hire me.😅
      </p>
    </div>
  </v-container>
</template>

<script setup>
import gsap from 'gsap';
import ScrollTrigger from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

const animation = async () => {
  gsap.utils.toArray(".line-animation").forEach((el, index) => {
    gsap.fromTo(
      el,
      {
        y: 300,
      },
      {
        y: 0,
        duration: 1,
        ease: "power4.out",
        delay: index * 0.01,
        scrollTrigger: {
          trigger: el,
          start: "top 150%",
          toggleActions: "play none none reverse",
        },
      }
    );
  });
}

let gsapCtx;

onMounted(async () => {
  gsapCtx = gsap.context(async () => {
    await animation();
  });
});

onUnmounted(() => {
  gsapCtx.revert();
});
</script>
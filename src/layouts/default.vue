<template>  
  <div id="page-content">
    <Navigation />
    <v-main class="bg-grey-lighten-4 pa-0">
      <router-view />
    </v-main>

    <PreFooter />    
  </div>
  <LoadingScreen />
</template>

<script setup>
// import CursorCanvas from "@/components/feature/CursorCanvas.vue";
import Navigation from "@/components/Navigation.vue";
import PreFooter from "@/components/PreFooter.vue";
import LoadingScreen from "@/components/LoadingScreen.vue";
import { onBeforeUnmount } from "vue";
import Lenis from "lenis";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
gsap.registerPlugin(ScrollTrigger);

const lenis = new Lenis({ autoRaf: false });
window.lenis = lenis;

lenis.on('scroll', ScrollTrigger.update);
ScrollTrigger.addEventListener('refresh', () => lenis.resize());
ScrollTrigger.defaults({ scroller: document.documentElement });

const lenisTicker = (time) => {
  lenis.raf(time * 1000);
};

gsap.ticker.add(lenisTicker);
gsap.ticker.lagSmoothing(0);

onBeforeUnmount(() => {
  gsap.ticker.remove(lenisTicker);
  lenis.destroy();
});
</script>

<style>
html,
body,
#app {
  margin: 0;
  width: 100%;
  height: 100%;
}

</style>

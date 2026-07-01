<template>
  <v-app-bar
    color="transparent"
    elevation="0"
    :height="$vuetify.display.mdAndUp ? 100 : 80"
    :scroll-behavior="'hide'"
    scroll-threshold="10"
    :class="$vuetify.display.mdAndUp ? 'px-16' : 'px-4'"
    style="mix-blend-mode: difference; color: white !important"
  >
    <template #prepend>           
      <div class="d-flex flex-column">
        <p>I Putu Yandika Dafa Pratama</p>
        <p class="text-body-2 opacity-50">Software Engineer</p>
      </div> 
    </template>       
    <template #append>
      <div class="d-flex align-center ga-2">
  <v-btn 
    icon="mdi-linkedin"
    size="large"
    variant="text"
    density="compact"
    href="https://www.linkedin.com/in/i-putu-yandika-dafa-pratama-214857331/"
    target="_blank"
  ></v-btn>

  <v-btn 
    icon="mdi-github" 
    size="large" 
    variant="text"
    density="compact"
    href="https://github.com/YandikaDafa"
    target="_blank"
  ></v-btn>

  <v-btn 
    icon="mdi-tiktok"
    size="large"
    variant="text"
    density="compact"
    href="https://www.tiktok.com/@dafff.999"
    target="_blank"
  ></v-btn>
</div>
    </template>
  </v-app-bar>
</template>

<script setup>
import { DotLottieVue } from "@lottiefiles/dotlottie-vue";
import { ref, onMounted, onBeforeUnmount } from "vue";
import gsap from "gsap";

const menu = ref(false);
const isPlayingMusic = ref(false);
let audioCtx;
let audioEl;
let track;
let gainNode;

const rampGain = (target, durationSec) => {
  if (!gainNode || !audioCtx) return;
  const t0 = audioCtx.currentTime;
  const start = gainNode.gain.value;
  gainNode.gain.cancelScheduledValues(t0);
  gainNode.gain.setValueAtTime(start, t0);
  gainNode.gain.linearRampToValueAtTime(target, t0 + durationSec);
};

const playMusic = async () => {
  if (!audioCtx) return;
  await audioCtx.resume();
  audioEl.loop = true;
  audioEl.play();
  rampGain(0.5, 2); // fade in 2 detik ke volume 70%
  isPlayingMusic.value = true;
};

const stopMusic = () => {
  if (!audioCtx) return;
  rampGain(0, 1.5); // fade out 1.5 detik
  setTimeout(() => {
    audioEl.pause();
    audioEl.currentTime = 0;
    isPlayingMusic.value = false;
  }, 1500);
};

const toggleMusic = () => {
  if (!isPlayingMusic.value) playMusic();
  else stopMusic();
};

const scrollToBottom = () => {
  window.scrollTo({ top: document.body.scrollHeight, behavior: "smooth" });
};

onMounted(() => {
  audioCtx = new AudioContext();
  audioEl = new Audio("/NAVIA.wav");
  track = audioCtx.createMediaElementSource(audioEl);
  gainNode = audioCtx.createGain();
  gainNode.gain.value = 0;
  track.connect(gainNode).connect(audioCtx.destination);
});

onBeforeUnmount(() => {});
</script>

<style scoped>

</style>

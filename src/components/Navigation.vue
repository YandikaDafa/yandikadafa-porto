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
          <v-icon>
            <svg xmlns="http://www.w3.org/2000/svg" width="1em" height="1em" viewBox="0 0 640 640" fill="currentColor">
              <path d="M427.5 299.7C429.7 300.6 431.7 301.6 433.8 302.5C463 316.6 484.4 337.7 495.6 363.9C511.3 400.4 512.8 459.7 465.3 507.1C429.1 543.3 385 559.6 322.7 560.1L322.4 560.1C252.2 559.6 198.3 536 162 489.9C129.7 448.9 113.1 391.8 112.5 320.3L112.5 319.8C113 248.3 129.6 191.2 161.9 150.2C198.2 104.1 252.2 80.5 322.4 80L322.7 80C393 80.5 447.6 104 485 149.9C503.4 172.6 517 199.9 525.6 231.6L485.2 242.4C478.1 216.6 467.4 194.6 453 177C423.8 141.2 380 122.8 322.5 122.4C265.5 122.9 222.4 141.2 194.3 176.8C168.1 210.1 154.5 258.3 154 320C154.5 381.7 168.1 429.9 194.3 463.3C222.3 498.9 265.5 517.2 322.5 517.7C373.9 517.3 407.9 505.1 436.2 476.8C468.5 444.6 467.9 405 457.6 380.9C451.5 366.7 440.5 354.9 425.7 346C422 372.9 413.9 394.3 401 410.8C383.9 432.6 359.6 444.4 328.3 446.1C304.7 447.4 282 441.7 264.4 430.1C243.6 416.3 231.4 395.3 230.1 370.8C227.6 322.5 265.8 287.8 325.3 284.4C346.4 283.2 366.2 284.1 384.5 287.2C382.1 272.4 377.2 260.6 369.9 252C359.9 240.3 344.3 234.3 323.7 234.2L323 234.2C306.4 234.2 284 238.8 269.7 260.5L235.3 236.9C254.5 207.8 285.6 191.8 323.1 191.8L323.9 191.8C386.5 192.2 423.8 231.3 427.6 299.5L427.4 299.7L427.5 299.7zM271.5 368.5C272.8 393.6 299.9 405.3 326.1 403.8C351.7 402.4 380.7 392.4 385.6 330.6C372.4 327.7 357.8 326.2 342.2 326.2C337.4 326.2 332.6 326.3 327.8 326.6C284.9 329 270.6 349.8 271.6 368.4L271.5 368.5z"/>
            </svg>
          </v-icon>
        </v-btn>                
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

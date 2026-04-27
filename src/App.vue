<template>
  <v-app>
    <!-- <ParticleBackground /> -->
    <!-- <CircleCursor /> -->
    <OilFluidCursor />
    <router-view />
    <!-- <RouteTransition /> -->
  </v-app>
</template>

<script setup>
import RouteTransition from "@/components/RouteTransition.vue";
import OilFluidCursor from "@/components/feature/OilFluidCursor.vue";
import CircleCursor from "@/components/feature/CircleCursor.vue";
import ParticleBackground from "@/components/feature/ParticleBackground.vue";
</script>

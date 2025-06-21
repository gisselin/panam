<script lang="ts" setup>
import { useIntervalFn } from "@vueuse/core"
import dayjs from "dayjs"
import { ref } from "vue"

const currentTime = ref(dayjs())

useIntervalFn(() => {
  currentTime.value = dayjs()
}, 1000)
</script>

<template>
  <time :datetime="currentTime.format()">
    <span>{{ currentTime.format("HH") }}</span>
    <span>:</span>
    <span>{{ currentTime.format("mm") }}</span>
  </time>
</template>

<style scoped>
time {
  --border-radius: 0.4vw;
  padding: var(--panel-top-gap) 1vw 0.5vw 1vw;
  background-color: black;
  color: var(--waiting-time-color);
  font-size: 4.2vw;
  line-height: 4vw;
  border-radius: 0 0 var(--border-radius) var(--border-radius);
}

span:nth-child(2) {
  animation: blink 1.4s infinite;
}

@keyframes blink {
  50% {
    opacity: 0.5;
  }
}

/* @media (max-height: 40vw) {
  time {
    font-size: 3.5vw;
  }
} */
</style>

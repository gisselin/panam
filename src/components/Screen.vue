<script setup lang="ts">
import { useIntervalFn } from "@vueuse/core"
import { computed, onMounted, ref } from "vue"
import type {
  SimpleDeparture,
  SimpleDisruption,
  SimpleLine,
} from "../services/Wagon"
import { Wagon } from "../services/Wagon"
import CurrentTime from "./CurrentTime.vue"
import DepartureRow from "./DepartureRow.vue"
import EmptyState from "./EmptyState.vue"
import Header from "./Header.vue"
import WaitingTime from "./WaitingTime.vue"
import Slider from "./disruptions/Slider.vue"
import SuspendedService from "./SuspendedService.vue"

const props = defineProps<{
  lat: number
  lon: number
  lineNumber: string
  directionTextHint: string | null
}>()

const departures = ref<SimpleDeparture[]>([])
const disruptions = ref<SimpleDisruption[]>([])
const line = ref<SimpleLine>()

async function updateDepartures() {
  const data = await Wagon.getDeparturesNear(
    props.lat,
    props.lon,
    props.lineNumber
  )

  const branchHashOfDirectionHint = data.departures.find(
    (x) =>
      props.directionTextHint !== null &&
      x.destination
        .toLocaleLowerCase()
        .includes(props.directionTextHint.toLocaleLowerCase())
  )?.branchHash

  departures.value = data.departures
    .filter((x) =>
      branchHashOfDirectionHint
        ? x.branchHash === branchHashOfDirectionHint
        : true
    )
    .sort((a, b) => a.leavesAt.diff(b.leavesAt))
    .slice(0, 3)
  line.value = data.line
}

const mostCommonDestination = computed<string>(() => {
  const destinationCounts = {} as Record<string, number>

  for (const departure of departures.value) {
    if (destinationCounts[departure.destination]) {
      destinationCounts[departure.destination]++
    } else {
      destinationCounts[departure.destination] = 1
    }
  }

  const proportion =
    Math.max(...Object.values(destinationCounts)) / departures.value.length

  if (proportion < 0.5) {
    return [...Object.keys(destinationCounts)].join(" • ")
  }

  return Object.entries(destinationCounts).reduce(
    (a, b) => (b[1] > a[1] ? b : a),
    ["", 0]
  )[0]
})

const nextDeparturesGoesToSameDestination = computed<boolean>(() => {
  return departures.value.every(
    (departure) => departure.destination === mostCommonDestination.value
  )
})

async function updateDisruptions() {
  const lineId = line.value?.id

  if (!lineId) {
    return
  }

  disruptions.value = await Wagon.getDisruptions(
    props.lat,
    props.lon,
    lineId,
    "",
    ""
  )
}

const serviceIsSuspended = computed<boolean>(() => {
  // check if a disruption is stopped service on current line
  return (
    disruptions.value.some(
      (disruption) =>
        disruption.type === "stoppedService" &&
        disruption.line.id === line.value?.id
    ) && departures.value.length === 0
  )
})

const isLastTrain = computed<boolean>(() => {
  return departures.value.length === 1
})

useIntervalFn(async () => {
  await updateDepartures()
}, 61 * 1000)

useIntervalFn(async () => {
  await updateDisruptions()
}, 5 * 60 * 1000)

onMounted(async () => {
  await updateDepartures()
  await updateDisruptions()
})
</script>

<template>
  <main>
    <CurrentTime class="clock"></CurrentTime>
    <Header
      class="header"
      v-if="line && !serviceIsSuspended"
      :line="line"
      :direction="mostCommonDestination"
    ></Header>
    <SuspendedService v-if="serviceIsSuspended"></SuspendedService>
    <EmptyState v-else-if="departures.length === 0"></EmptyState>
    <div class="withDisruptions" v-else>
      <TransitionGroup
        tag="section"
        name="horizontal"
        v-if="nextDeparturesGoesToSameDestination"
      >
        <article
          v-for="(departure, i) in departures.slice(0, 2)"
          :class="{ labelsHidden: isLastTrain }"
          :key="departure.id"
        >
          <span class="label" v-if="i == 0">1<sup>er</sup> métro</span>
          <span class="label" v-if="i == 1">2<sup>e</sup> métro</span>
          <WaitingTime font-size="18vw" :at="departure.leavesAt"></WaitingTime>
        </article>
        <article class="lastTrain" v-if="isLastTrain">
          <svg
            width="102"
            height="118"
            viewBox="0 0 102 118"
            fill="none"
            xmlns="http://www.w3.org/2000/svg"
          >
            <path d="M0 59.0001L102 0.110352V117.89L0 59.0001Z" fill="white" />
          </svg>
          <span>Dernier<br />métro</span>
        </article>
      </TransitionGroup>
      <TransitionGroup name="vertical" tag="ul" v-else>
        <DepartureRow
          v-for="departure in departures"
          :departure="departure"
          :key="departure.id"
        ></DepartureRow>
      </TransitionGroup>
      <Slider class="slider" :disruptions="disruptions"></Slider>
    </div>
  </main>
</template>

<style scoped>
main {
  background-color: black;
  height: 100vh;
  display: grid;
  grid-template-rows: auto 1fr;
}

.clock {
  position: absolute;
  top: 0;
  right: calc(env(safe-area-inset-left) + 1.5vw);
  z-index: 1;
}

ul {
  padding: 1rem;
  margin: 0;
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

section {
  height: 100%;
  padding: 0 0.5rem;
  padding-left: env(safe-area-inset-left);
  display: flex;
  align-items: center;
  overflow: hidden;
}

article {
  position: relative;
  display: flex;
  flex-direction: column;
  width: 50%;
}

article.labelsHidden .label {
  opacity: 0;
}

article.lastTrain {
  flex-direction: row;
  align-items: center;
  transform: translateX(-3vw);
}

article.lastTrain span {
  padding-top: 3vw;
  font-size: 4.2vw;
  font-weight: bold;
}

article.lastTrain svg {
  width: 6vw;
  height: 6vw;
  padding-top: 3vw;
}

article:last-child:not(.lastTrain)::before {
  --offset: 22%;
  content: "";
  position: absolute;
  top: var(--offset);
  left: 0;
  bottom: var(--offset);
  width: 0.6vw;
  background-color: white;
  border-radius: 999px;
  transform: translateY(12%);
}

article span {
  margin-left: 2vw;
  color: white;
  font-size: 2vw;
}

article time {
  align-self: center;
}

.withDisruptions {
  height: 100%;
  display: grid;
  grid-template-columns: 1fr;
  grid-template-rows: 1fr;
}

.slider {
  margin-top: var(--panel-top-gap);
}

@media (max-height: 40vw) {
  main {
    grid-template-rows: 1fr;
  }

  .header {
    display: none;
  }
}

@media (max-height: 60vw) {
  .withDisruptions {
    grid-template-columns: 1fr 1fr;
  }
}
</style>

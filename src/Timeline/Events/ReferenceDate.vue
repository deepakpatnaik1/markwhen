<script setup lang="ts">
import { computed, onMounted, ref, watch } from "vue";
import { useTimelineStore } from "../timelineStore";

const timelineStore = useTimelineStore();
const reference = computed(() => {
  return timelineStore.distanceFromBaselineLeftmostDate(
    timelineStore.referenceDate
  );
});
</script>

<template>
  <div
    class="absolute bg-th-ref-line top-0"
    :style="`width: 1px; left: ${reference}px; height: max(100vh, 100%);`"
  ></div>
</template>

<style scoped></style>

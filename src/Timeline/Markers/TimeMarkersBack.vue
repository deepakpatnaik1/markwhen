<script setup lang="ts">
import { viewportLeftMarginPixels } from "../utilities/dateTimeUtilities";
import { computed, ref } from "vue";
import { dateScale } from "@/Timeline/utilities/dateTimeUtilities";
import {
  useMarkersStore,
  Weight,
  type TimeMarker,
} from "@/Timeline/Markers/markersStore";
import { useWeekdayCache } from "../utilities/weekdayCache";
import {
  clamp,
  timeMarkerWeightMinimum,
  useTimelineStore,
} from "../timelineStore";
import { EventGroup, isEvent, iter, Event } from "@markwhen/parser";
import { toDateRange } from "@markwhen/parser";
import { useEventColor } from "../Events/composables/useEventColor";
import { equivalentPaths } from "../paths";
import { useNodeStore } from "../useNodeStore";
import { DateTime } from "luxon";
import { granularities } from "../utilities/DateTimeDisplay";
import type { Sourced } from "@/Markwhen/useLpc";
import { theme } from "@/config/palette";

const markersStore = useMarkersStore();
const timelineStore = useTimelineStore();
const { getWeekday } = useWeekdayCache();
const dark = computed(() => timelineStore.darkMode);
const colors = computed(() => dark.value ? theme.dark : theme.light);
const nodeStore = useNodeStore();

const leftMargin = viewportLeftMarginPixels;

const backgroundColor = computed(() => (tm: TimeMarker) => {
  if (timelineStore.weights[Weight.DAY] > 0.3) {
    const weekday = getWeekday(tm.dateTime);
    const a = Math.min(timelineStore.weights[Weight.DAY] * 0.3, 0.5);
    if (
      (typeof weekday === "number" && weekday === 6) ||
      weekday === 7 ||
      // @ts-ignore
      weekday.weekday === 6 ||
      // @ts-ignore
      weekday.weekday === 7
    ) {
      return `rgba(${colors.value.gridBorder}, ${a})`;
    }
  }
  return "unset";
});

const hovering = computed(() => (tm: TimeMarker) => {
  const h = markersStore.hoveringMarker;
  if (!h) {
    return false;
  }
  return +tm.dateTime === +h.dateTime;
});

const alpha = computed(
  () => (tm: TimeMarker) => timelineStore.weights[dateScale(tm.dateTime)]
);

const borderColor = computed(() => (tm: TimeMarker) => {
  const a = hovering.value(tm) ? 1 : (alpha.value(tm) - 0.3) * 2;
  return `rgba(${colors.value.gridLine}, ${a})`;
});

const eras = computed(() => {
  const erasAndMilestoneEvents = [] as any[];
  if (!timelineStore.transformedEvents) {
    return [];
  }
  for (const { eventy, path } of iter(timelineStore.transformedEvents)) {
    if (
      isEvent(eventy) &&
      ["era", "milestone"].some((t) =>
        eventy.tags.map((t) => t.toLowerCase()).includes(t)
      )
    ) {
      const { fromDateTime, toDateTime } = toDateRange(eventy.dateRangeIso);
      const color = useEventColor(eventy as Sourced<Event>).color.value;
      const isHovering = equivalentPaths(
        timelineStore.hoveringEventPaths,
        path
      );
      const styleObj = {
        left: timelineStore.distanceFromBaselineLeftmostDate(fromDateTime),
        width: Math.max(
          2,
          timelineStore.distanceBetweenDates(fromDateTime, toDateTime)
        ),
      } as any;
      if (color) {
        styleObj.backgroundColor = `rgba(${color}, ${isHovering ? 0.15 : 0.1})`;
        styleObj.borderColor = `rgba(${color}, ${isHovering ? 0.75 : 0.3})`;
      }
      erasAndMilestoneEvents.push(styleObj);
    }
  }
  return erasAndMilestoneEvents;
});
const scaleForThisDate = computed(
  () => (timeMarker: TimeMarker) => dateScale(timeMarker.dateTime)
);

const currentDateResolution = computed(() => {
  for (let i = 0; i < timelineStore.weights.length; i++) {
    if (timelineStore.weights[i] > timeMarkerWeightMinimum) {
      return i;
    }
  }
  return Weight.DECADE;
});
const text = computed(
  () => (timeMarker: TimeMarker) =>
    granularities[currentDateResolution.value][
      scaleForThisDate.value(timeMarker)
    ](timeMarker.dateTime)
);
const opacity = computed(
  () => (timeMarker: TimeMarker) => clamp((alpha.value(timeMarker) - 0.45) * 5)
);
const isHovering = computed(
  () => (timeMarker: TimeMarker) =>
    markersStore.hoveringMarker &&
    +markersStore.hoveringMarker?.dateTime === +timeMarker.dateTime
);
const hoveringText = computed(() => (timeMarker: TimeMarker) => {
  const dt = timeMarker.dateTime;
  if (currentDateResolution.value > 5) {
    return dt.year;
  }
  if (currentDateResolution.value > 3) {
    return dt.toLocaleString(DateTime.DATE_HUGE);
  }
  return dt.toLocaleString(DateTime.DATETIME_HUGE_WITH_SECONDS);
});
</script>

<template>
  <div
    class="fixed top-0 left-0 right-0 h-6 bg-th-surface/95 z-30 border-b border-th-border"
  ></div>
  <div
    v-for="timeMarker in markersStore.markers"
    :id="'' + timeMarker.ts"
    :key="timeMarker.ts"
    class="h-full flex-shrink-0 absolute top-0 bottom-0 pointer-events-none"
    :style="{
      backgroundColor: backgroundColor(timeMarker),
      width: `${timelineStore.pageScaleBy24 * timeMarker.size}px`,
      left: `${
        timelineStore.pageScaleBy24 * (timeMarker.accumulated - timeMarker.size)
      }px`,
      borderLeft: `1px ${
        hovering(timeMarker) ? 'solid' : 'dashed'
      } ${borderColor(timeMarker)}`,
      height: `max(${nodeStore.viewHeight}, 100%)`,
    }"
  >
    <div
      class="sticky top-0 -m-px"
      :class="{
        'font-bold z-50 border-th-border-strong':
          isHovering(timeMarker),
        'z-40 border-th-border': !isHovering(timeMarker),
      }"
    >
      <h6
        class="timeMarkerTitle text-sm whitespace-nowrap text-th-text pl-1 border-l border-inherit"
        :style="{
          opacity: isHovering(timeMarker) ? 1 : opacity(timeMarker),
        }"
      >
        {{ text(timeMarker) }}
      </h6>
      <div v-if="currentDateResolution <= 6" class="flex flex-row">
        <h6
          class="whitespace-nowrap text-xs font-bold bg-th-surface border-l p-1 border-th-border-strong"
          v-if="isHovering(timeMarker)"
        >
          {{ hoveringText(timeMarker) }}
        </h6>
        <h6 class="whitespace-nowrap text-xs font-bold">&nbsp;</h6>
      </div>
    </div>
  </div>
  <div
    v-for="era in eras"
    class="absolute top-0 bottom-0 h-full border-l border-r transition"
    :class="
      !era.backgroundColor
        ? `bg-th-section-bg/50 border-th-section-border/50`
        : ''
    "
    :style="{
      left: `${era.left}px`,
      width: `max(${era.width}px, 10px)`,
      height: `max(${nodeStore.viewHeight}, 100%)`,
      backgroundColor: era.backgroundColor,
      borderColor: era.borderColor,
    }"
  ></div>
</template>

<style>
.timeMarkerShader {
  z-index: -1;
  background: linear-gradient(to bottom, rgb(var(--color-header-gradient-start)), 85%, rgba(var(--color-header-gradient-end)));
}
</style>

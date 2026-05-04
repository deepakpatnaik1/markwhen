<script setup lang="ts">
import HoverHint from "@/Timeline/Settings/HoverHint.vue";
import { ref } from "vue";
import { useIsTouchscreen } from "@/Timeline/composables/useIsTouchscreen";

defineProps<{
  hoverHintTitle?: string;
  hoverHintShortcut?: string;
  selected?: boolean;
  hoverHintLeft?: number;
}>();
const emit = defineEmits<{ (event: "click"): void }>();

const hovering = ref(false);
const { canHover } = useIsTouchscreen();

const events = canHover.value
  ? {
      mouseover: () => {
        hovering.value = true;
      },
      mouseleave: () => {
        hovering.value = false;
      },
    }
  : {};

const click = () => {
  hovering.value = false;
  emit("click");
};
</script>

<template>
  <button
    @click="click"
    class="p-1 flex flex-row items-center hover:bg-th-hover-bg transition text-sm lg:text-base font-bold relative shrink-0"
    v-on="events"
    @mousedown.stop=""
    @touchstart.stop=""
    :class="{
      'text-th-accent outline-2 outline outline-th-accent':
        selected,
    }"
  >
    <slot></slot>
    <HoverHint
      :hover-position="'top'"
      :hovering="hovering"
      :shortcut="hoverHintShortcut"
      :left="hoverHintLeft"
      ><slot name="hoverHint" v-if="$slots['hoverHint']"></slot
    ><span v-else>{{ hoverHintTitle}}</span></HoverHint>
  </button>
</template>

<style scoped></style>

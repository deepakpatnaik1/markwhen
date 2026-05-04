<script setup lang="ts">
import { computed } from "vue";

const props = defineProps<{
  left: number;
  mouseDownTouchStartListener: (e: MouseEvent | TouchEvent) => void;
}>();

const styleObj = computed(() => ({ left: `${props.left}px` }));
</script>

<template>
  <button
    class="absolute flex items-center justify-center cursor-ew-resize touch-none top-0 bottom-0 z-[4]"
    :style="styleObj"
    @touchstart.passive="mouseDownTouchStartListener"
    @mousedown.prevent.stop="mouseDownTouchStartListener"
  >
    <div
      class="absolute w-[2px] h-4 bg-th-bar-empty rounded"
    ></div>
    <div class="w-4 h-4 absolute flex items-center justify-center">
      <div
        class="w-2 h-2 rounded-full border border-th-surface bg-th-bar-empty shadow"
      ></div>
    </div>
  </button>
</template>

<style scoped></style>

<script setup lang="ts">
import { computed } from 'vue'
import { useNav } from '@slidev/client'

const { clicks, clicksTotal, currentPage, go } = useNav()

const steps = computed(() => clicksTotal.value + 1)
// Reserve room for the widest label (`xx/xx`, digits derived from the total)
// so the rail does not shift when the number grows, e.g. 9 -> 10.
const labelMinWidth = computed(() => `${String(clicksTotal.value).length * 2 + 1}ch`)

function goToStep(step: number) {
  go(currentPage.value, step)
}
</script>

<template>
  <div v-if="clicksTotal" class="clicks-slider">
    <span class="clicks-slider__count" :style="{ minWidth: labelMinWidth }">
      <span class="clicks-slider__current">{{ clicks }}</span>
      <span class="clicks-slider__separator">/</span>
      <span class="clicks-slider__total">{{ clicksTotal }}</span>
    </span>
    <div class="clicks-slider__track">
      <button
        v-for="i in steps"
        :key="i"
        type="button"
        class="clicks-slider__step"
        :class="{ 'clicks-slider__step--active': i - 1 <= clicks }"
        :title="`Vai allo step ${i - 1}`"
        @mousedown.prevent
        @click="goToStep(i - 1)"
      />
    </div>
  </div>
</template>

<style scoped>
/* Default look — overridable through the --clicks-slider-* variables. */
.clicks-slider {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  font-family: var(--clicks-slider-font, var(--slidev-code-font-family, ui-monospace, monospace));
  user-select: none;
}

.clicks-slider__count {
  display: inline-block;
  font-size: 0.875em;
  text-align: right;
  font-variant-numeric: tabular-nums;
  white-space: nowrap;
}

.clicks-slider__current {
  color: var(--clicks-slider-accent, #7aa2ff);
}

.clicks-slider__separator {
  opacity: 0.25;
}

.clicks-slider__total {
  font-size: 0.75em;
  opacity: 0.5;
}

.clicks-slider__track {
  display: flex;
  flex: 1;
  gap: 0.25rem;
}

.clicks-slider__step {
  flex: 1;
  height: 0.375rem;
  padding: 0;
  border: 0;
  border-radius: var(--clicks-slider-radius, 999px);
  background: var(--clicks-slider-track, #c9c9d0);
  cursor: pointer;
  transition: opacity 150ms ease;
}

.clicks-slider__step:hover {
  opacity: 0.7;
}

.clicks-slider__step--active {
  background: var(--clicks-slider-accent, #7aa2ff);
}
</style>

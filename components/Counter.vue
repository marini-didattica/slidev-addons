<script setup lang="ts">
import { ref } from 'vue'

const props = defineProps({
  count: {
    default: 0,
  },
})

const counter = ref(props.count)
</script>

<template>
  <div class="counter">
    <button class="counter__button" type="button" @click="counter -= 1">
      -
    </button>
    <span class="counter__value">{{ counter }}</span>
    <button class="counter__button" type="button" @click="counter += 1">
      +
    </button>
  </div>
</template>

<style scoped>
/* Default look — overridable through the --counter-* variables. */
.counter {
  display: inline-flex;
  width: min-content;
  border: 1px solid var(--counter-border, #e4e4e8);
  border-radius: var(--counter-radius, 6px);
  overflow: hidden;
}

.counter__button {
  padding: var(--counter-padding, 0.5rem);
  border: 0;
  background: transparent;
  color: inherit;
  font-family: var(--counter-font, var(--slidev-code-font-family, ui-monospace, monospace));
  cursor: pointer;
  outline: none;
}

.counter__button:first-child {
  border-right: 1px solid var(--counter-border, #e4e4e8);
}

.counter__button:last-child {
  border-left: 1px solid var(--counter-border, #e4e4e8);
}

.counter__button:hover {
  background: var(--counter-hover-bg, rgba(128, 128, 128, 0.2));
}

.counter__value {
  margin: auto;
  padding: var(--counter-padding, 0.5rem);
  font-family: var(--counter-font, var(--slidev-code-font-family, ui-monospace, monospace));
}
</style>

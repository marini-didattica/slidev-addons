<script setup lang="ts">
withDefaults(defineProps<{
  accent?: boolean
  round?: boolean
}>(), {
  accent: false,
  round: false,
})
</script>

<template>
  <span class="badge" :class="{ 'badge-accent': accent, 'badge-round': round }">
    <slot />
  </span>
</template>

<style scoped>
/* Default look — overridable through the --badge-* variables. */
.badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.4em;
  padding: var(--badge-padding, 0.15em 0.65em);
  border: 1px solid var(--badge-border, #e4e4e8);
  border-radius: var(--badge-radius, 999px);
  background: var(--badge-bg, #f1f1f4);
  color: var(--badge-color, #6b6b73);
  font-family: var(--badge-font, var(--slidev-code-font-family, ui-monospace, monospace));
  font-size: var(--badge-font-size, 0.7em);
  font-weight: var(--badge-font-weight, 500);
  line-height: var(--badge-line-height, 1.7);
  letter-spacing: var(--badge-letter-spacing, 0.06em);
  text-transform: uppercase;
  vertical-align: middle;
}

.badge-accent {
  --badge-accent: var(--badge-accent-color, #3b82f6);
  border-color: color-mix(in srgb, var(--badge-accent) 45%, transparent);
  background: color-mix(in srgb, var(--badge-accent) 14%, transparent);
  color: var(--badge-accent);
}

.badge-round {
  aspect-ratio: 1;
  min-width: 3em;
  padding: var(--badge-round-padding, 0.5em);
  font-size: var(--badge-round-font-size, 0.8em);
}
</style>

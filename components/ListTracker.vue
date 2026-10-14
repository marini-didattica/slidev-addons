<script setup lang="ts">
import { computed } from 'vue'
import { useNav } from '@slidev/client'

/**
 * One pointer following an index into `items` over the clicks of a slide.
 * `values` maps an absolute click number to the tracked index: at click `c`
 * the pointer shows the value of the greatest key `<= c`. `null` hides it.
 */
interface Tracker {
  name: string
  values: Record<number, number | null>
  color?: string
}

const props = withDefaults(
  defineProps<{
    /** Cells drawn in order, one per list element. */
    items: (string | number)[]
    /** Pointers to draw; either named object or an ordered array. */
    trackers: Tracker[] | Record<string, Record<number, number | null>>
    /** Highlight the cell currently pointed at. */
    highlight?: boolean
    /** Draw the small index ruler under the grid. */
    showIndexes?: boolean
    /**
     * How negative indices are interpreted:
     * - `'invalid'`: out of bounds (pinned to the start edge, marked as such).
     * - `'wrap'`: valid Python indexing, shown at `items.length + index`.
     */
    negative?: 'invalid' | 'wrap'
    /** Mark out-of-bounds pointers in red and turn their arrow towards the edge. */
    markOutOfBounds?: boolean
  }>(),
  {
    highlight: false,
    showIndexes: true,
    negative: 'invalid',
    markOutOfBounds: true,
  },
)

const { clicks } = useNav()

const trackers = computed<Tracker[]>(() =>
  Array.isArray(props.trackers)
    ? props.trackers
    : Object.entries(props.trackers).map(([name, values]) => ({ name, values })),
)

const columns = computed(() => props.items.length)

function rawIndexAt(values: Record<number, number | null>, click: number): number | null {
  let bestKey = -Infinity
  let index: number | null = null
  for (const [key, value] of Object.entries(values)) {
    const clickKey = Number(key)
    if (clickKey <= click && clickKey > bestKey) {
      bestKey = clickKey
      index = value
    }
  }
  return index
}

interface Pointer {
  name: string
  color?: string
  label: string
  index: number | null
  left: number
  edge: 'start' | 'end' | null
  out: boolean
}

// One lane per tracker, always rendered, so the lane height can animate between
// hidden (0) and visible instead of making the grid jump when a pointer appears.
const pointers = computed<Pointer[]>(() =>
  trackers.value.map((tracker) => {
    const raw = rawIndexAt(tracker.values, clicks.value)
    if (raw === null) {
      return {
        name: tracker.name,
        color: tracker.color,
        label: '',
        index: null,
        left: 0,
        edge: null,
        out: false,
      }
    }

    // Resolve Python-style negative indices when enabled.
    const index = raw < 0 && props.negative === 'wrap' ? columns.value + raw : raw
    const inRange = index >= 0 && index < columns.value

    // Out-of-bounds pointers are pinned to the nearest grid edge and their
    // label is anchored inwards, so nothing can overflow the component.
    const edge = inRange ? null : index < 0 ? 'start' : 'end'
    const left = inRange
      ? ((index + 0.5) / columns.value) * 100
      : edge === 'start' ? 0 : 100

    return {
      name: tracker.name,
      color: tracker.color,
      label: `${tracker.name}=${raw}`,
      index,
      left,
      edge,
      out: !inRange,
    }
  }),
)
</script>

<template>
  <div class="list-tracker">
    <div class="list-tracker__lanes">
      <div
        v-for="pointer in pointers"
        :key="pointer.name"
        class="list-tracker__lane"
        :class="{ 'list-tracker__lane--visible': pointer.index !== null }"
      >
        <div
          v-if="pointer.index !== null"
          class="list-tracker__pointer"
          :class="{
            'list-tracker__pointer--out': pointer.out && markOutOfBounds,
            'list-tracker__pointer--start': pointer.edge === 'start',
            'list-tracker__pointer--end': pointer.edge === 'end',
          }"
          :style="{ left: `${pointer.left}%`, color: pointer.color }"
        >
          <span class="list-tracker__label">{{ pointer.label }}</span>
          <span
            class="list-tracker__arrow"
            :class="{
              'list-tracker__arrow--left': pointer.out && markOutOfBounds && pointer.edge === 'start',
              'list-tracker__arrow--right': pointer.out && markOutOfBounds && pointer.edge === 'end',
            }"
          />
        </div>
      </div>
    </div>

    <div
      class="list-tracker__grid"
      :style="{ gridTemplateColumns: `repeat(${columns}, minmax(0, 1fr))` }"
    >
      <div
        v-for="(item, index) in items"
        :key="index"
        class="list-tracker__cell"
        :class="{ 'list-tracker__cell--active': highlight && pointers.some(t => t.index === index) }"
      >
        {{ item }}
      </div>
    </div>

    <div
      v-if="showIndexes"
      class="list-tracker__grid list-tracker__ruler"
      :style="{ gridTemplateColumns: `repeat(${columns}, minmax(0, 1fr))` }"
    >
      <div v-for="(item, index) in items" :key="index" class="list-tracker__index">
        {{ index }}
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Default look — overridable through the --tracker-* variables. */
.list-tracker {
  position: relative;
}

.list-tracker__lanes {
  display: flex;
  flex-direction: column;
}

.list-tracker__lane {
  position: relative;
  height: 0;
  overflow: hidden;
  transition: height var(--tracker-transition, 220ms) ease;
}

.list-tracker__lane--visible {
  height: var(--tracker-lane-height, 2.1rem);
}

.list-tracker__pointer {
  position: absolute;
  top: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  color: var(--tracker-pointer-color, #7aa2ff);
  transform: translateX(-50%);
  transition: left 260ms ease, transform 260ms ease;
  white-space: nowrap;
}

/* Edge pointers anchor inwards so the label never leaves the component. */
.list-tracker__pointer--start {
  transform: translateX(0);
}

.list-tracker__pointer--end {
  transform: translateX(-100%);
}

.list-tracker__pointer--out {
  color: var(--tracker-out-of-bounds, #e5484d);
}

.list-tracker__label {
  font-family: var(--tracker-font, var(--slidev-code-font-family, ui-monospace, monospace));
  font-size: 0.82em;
  line-height: 1.2;
}

.list-tracker__arrow {
  width: 0;
  height: 0;
  margin-top: 0.1rem;
  border-left: 0.4rem solid transparent;
  border-right: 0.4rem solid transparent;
  border-top: 0.5rem solid currentColor;
}

.list-tracker__arrow--left {
  border-left: 0;
  border-right: 0.5rem solid currentColor;
  border-top: 0.4rem solid transparent;
  border-bottom: 0.4rem solid transparent;
}

.list-tracker__arrow--right {
  border-right: 0;
  border-left: 0.5rem solid currentColor;
  border-top: 0.4rem solid transparent;
  border-bottom: 0.4rem solid transparent;
}

.list-tracker__grid {
  display: grid;
}

.list-tracker__cell {
  padding: 0.32rem 0.5rem;
  border: 1px solid var(--tracker-border, #e4e4e8);
  border-right: 0;
  background: var(--tracker-cell-bg, #f7f7f9);
  color: var(--tracker-cell-fg, #18181b);
  font-family: var(--tracker-font, var(--slidev-code-font-family, ui-monospace, monospace));
  text-align: center;
}

.list-tracker__cell:first-child {
  border-top-left-radius: var(--tracker-radius, 5px);
  border-bottom-left-radius: var(--tracker-radius, 5px);
}

.list-tracker__cell:last-child {
  border-top-right-radius: var(--tracker-radius, 5px);
  border-bottom-right-radius: var(--tracker-radius, 5px);
  border-right: 1px solid var(--tracker-border, #e4e4e8);
}

.list-tracker__cell--active {
  background: var(--tracker-active-bg, rgba(122, 162, 255, 0.14));
}

.list-tracker__ruler {
  margin-top: 0.15rem;
}

.list-tracker__index {
  color: var(--tracker-index-color, #6b6b73);
  font-family: var(--tracker-font, var(--slidev-code-font-family, ui-monospace, monospace));
  font-size: 0.7em;
  text-align: center;
}

@media (prefers-reduced-motion: reduce) {
  .list-tracker__pointer,
  .list-tracker__lane {
    transition: none;
  }
}
</style>

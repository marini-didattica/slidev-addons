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
    /** Mark out-of-bounds pointers in red. */
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

// Geometry of the single pointer lane, in rem. Labels sit on top, then a leader
// line drops to the top-centre of the target cell, stopping a small margin
// above the grid. The arrow head completes the line and ends exactly at that
// margin. These values are mirrored by the matching rules below.
const LABEL_HEIGHT = 1.0
const ARROW_HEIGHT = 1.1
const MARGIN_TO_CELL = 0.2
const HEAD_WIDTH = 0.4
const LANE_HEIGHT = LABEL_HEIGHT + ARROW_HEIGHT

// Rough width of one monospace character (and the padding around a label), used
// to spread the pointers that share a target so their labels do not overlap.
const CHAR_WIDTH = 0.56
const LABEL_PADDING = 0.5
const LABEL_GAP = 0.4

interface Resolved {
  name: string
  color?: string
  label: string
  raw: number
  index: number
  edge: 'start' | 'end' | null
  out: boolean
}

interface Pointer {
  name: string
  color: string
  label: string
  /** Index the pointer highlights, or `null` when it is out of bounds. */
  index: number | null
  /** CSS `left` of the label and the start of the leader line. */
  left: string
  /** Inline style for the leader line (rotated and length-truncated for the head). */
  lineStyle: Record<string, string>
}

const pointers = computed<Pointer[]>(() => {
  const cols = columns.value

  const resolved = trackers.value.map<Resolved | null>((tracker) => {
    const raw = rawIndexAt(tracker.values, clicks.value)
    if (raw === null) return null

    // Resolve Python-style negative indices when enabled.
    const index = raw < 0 && props.negative === 'wrap' ? cols + raw : raw
    const inRange = cols > 0 && index >= 0 && index < cols
    const edge: 'start' | 'end' | null = inRange ? null : index < 0 ? 'start' : 'end'

    return {
      name: tracker.name,
      color: tracker.color,
      label: `${tracker.name}=${raw}`,
      raw,
      index,
      edge,
      out: !inRange,
    }
  })

  // Group the visible pointers by the cell (or edge) they point at, keeping the
  // trackers' declaration order so pointers that land on the same cell stay in
  // their left/right order instead of swapping.
  const groups = new Map<string, number[]>()
  resolved.forEach((entry, i) => {
    if (!entry) return
    const key = entry.out ? `edge:${entry.edge}` : `cell:${entry.index}`
    const members = groups.get(key)
    if (members) members.push(i)
    else groups.set(key, [i])
  })

  const placed = resolved.map<Pointer | null>((entry) =>
    entry
      ? {
          name: entry.name,
          color: 'var(--tracker-pointer-color, #7aa2ff)',
          label: entry.label,
          index: entry.out ? null : entry.index,
          left: '0%',
          lineStyle: {},
        }
      : null,
  )

  for (const [key, members] of groups) {
    const first = resolved[members[0]]!
    // Out-of-bounds pointers are pinned to the nearest grid edge.
    const target = first.out
      ? first.edge === 'start'
        ? 0
        : 100
      : ((first.index + 0.5) / cols) * 100

    // Lay the group out from estimated label widths, then turn each label's
    // offset from the group centre into its distance from the target.
    const widths = members.map((i) => resolved[i]!.label.length * CHAR_WIDTH + LABEL_PADDING)
    const total = widths.reduce((sum, width) => sum + width, 0) + LABEL_GAP * (members.length - 1)

    members.forEach((memberIndex, slot) => {
      const entry = resolved[memberIndex]!
      const width = widths[slot]
      const before = widths.slice(0, slot).reduce((sum, w) => sum + w, 0) + LABEL_GAP * slot
      const fromCentre = before + width / 2 - total / 2
      // Edge groups are pushed inwards so the outermost label stays inside the
      // component; cell groups are centred on the cell.
      const offset = first.out
        ? (first.edge === 'start' ? total / 2 : -total / 2) + fromCentre
        : fromCentre

      // The leader line runs from the label toward the target, leaving room for
      // the arrow head and stopping a small margin above the cell.
      const dx = -offset
      const reach = ARROW_HEIGHT - MARGIN_TO_CELL
      const targetLength = Math.hypot(dx, reach)
      const lineLength = Math.max(targetLength - HEAD_WIDTH, 0)
      const angle = (Math.atan2(reach, dx) * 180) / Math.PI

      placed[memberIndex] = {
        ...placed[memberIndex]!,
        color:
          entry.out && props.markOutOfBounds
            ? 'var(--tracker-out-of-bounds, #e5484d)'
            : entry.color ?? 'var(--tracker-pointer-color, #7aa2ff)',
        left: `calc(${target}% + ${offset}rem)`,
        lineStyle: {
          left: `calc(${target}% + ${offset}rem)`,
          width: `${lineLength}rem`,
          transform: `rotate(${angle}deg)`,
        },
      }
    })
  }

  return placed.filter((pointer): pointer is Pointer => pointer !== null)
})
</script>

<template>
  <div class="list-tracker">
    <div
      class="list-tracker__lane"
      :class="{ 'list-tracker__lane--visible': pointers.length > 0 }"
      :style="{ '--tracker-lane-height': `${LANE_HEIGHT}rem` }"
    >
      <div
        v-for="pointer in pointers"
        :key="pointer.name"
        class="list-tracker__pointer"
        :style="{ color: pointer.color }"
      >
        <span class="list-tracker__label" :style="{ left: pointer.left }">
          {{ pointer.label }}
        </span>
        <span class="list-tracker__arrow-line" :style="pointer.lineStyle">
          <svg
            class="list-tracker__arrow-head"
            viewBox="0 0 10 9"
            aria-hidden="true"
          >
            <polygon points="0,0 10,4.5 0,9" fill="currentColor" />
          </svg>
        </span>
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

/* A single lane holds every pointer; those that share a cell sit side by side.
   It fades in as it expands when a pointer first appears. */
.list-tracker__lane {
  position: relative;
  height: 0;
  opacity: 0;
  transition:
    height var(--tracker-transition, 220ms) ease,
    opacity var(--tracker-transition, 220ms) ease;
}

.list-tracker__lane--visible {
  /* Overridden inline from the component to stay in sync with the geometry. */
  height: var(--tracker-lane-height, 2.1rem);
  opacity: 1;
}

/* Transparent wrapper: lets the label, line and head share the lane. */
.list-tracker__pointer {
  display: contents;
}

.list-tracker__label {
  position: absolute;
  top: 0;
  display: flex;
  align-items: center;
  height: 1.0rem;
  transform: translateX(-50%);
  transition: left var(--tracker-transition, 220ms) ease;
  white-space: nowrap;
  font-family: var(--tracker-font, var(--slidev-code-font-family, ui-monospace, monospace));
  font-size: 0.82em;
  line-height: 1;
}

/* The leader line rotates from the label toward the target. */
.list-tracker__arrow-line {
  position: absolute;
  top: 1.0rem;
  height: 1.5px;
  background: currentColor;
  transform-origin: 0 50%;
  transition:
    left var(--tracker-transition, 220ms) ease,
    width var(--tracker-transition, 220ms) ease,
    transform var(--tracker-transition, 220ms) ease;
}

/* Real SVG arrow head, not a border hack. It sits at the end of the line and
   inherits the parent's rotation so it always points along the line. */
.list-tracker__arrow-head {
  position: absolute;
  left: 100%;
  top: 50%;
  width: 0.4rem;
  height: 0.36rem;
  transform: translateY(-50%);
  pointer-events: none;
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
  .list-tracker__label,
  .list-tracker__arrow-line,
  .list-tracker__lane {
    transition: none;
  }
}
</style>

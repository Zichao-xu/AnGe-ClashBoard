<template>
  <!-- Direct and reject as two halves of one mark split by a diagonal: a two-way
       arrow (one long diagonal, a short barb at each end) above, the same arrow
       struck through below. Each half is its own hit area and always wears its
       colour; the active half glows. -->
  <svg
    viewBox="0 0 24 24"
    fill="none"
    stroke-linecap="round"
    stroke-linejoin="round"
  >
    <path
      class="text-base-content/50"
      stroke="currentColor"
      stroke-width="0.9"
      d="M2.6 21.4L21.4 2.6"
    />
    <g
      v-for="half in HALVES"
      :key="half.kind"
      role="button"
      :aria-label="half.kind"
      :aria-pressed="active === half.kind"
      :aria-disabled="!available[half.kind]"
      :class="[
        'verdict-half transition duration-150',
        half.color,
        !interactive && 'pointer-events-none',
        !available[half.kind]
          ? 'pointer-events-none opacity-15'
          : active === half.kind
            ? 'verdict-glow'
            : 'opacity-60 hover:opacity-100',
      ]"
      @click.stop="emit('pick', half.kind)"
    >
      <path
        :d="half.hit"
        fill="transparent"
        stroke="none"
        class="cursor-pointer"
      />
      <g
        :transform="half.transform"
        stroke="currentColor"
        :stroke-width="half.width"
        class="pointer-events-none"
      >
        <path
          v-for="d in half.paths"
          :key="d"
          :d="d"
        />
      </g>
    </g>
  </svg>
</template>

<script setup lang="ts">
type Verdict = 'direct' | 'reject'

defineProps<{
  active: Verdict | null
  available: Partial<Record<Verdict, string>>
  interactive: boolean
}>()

const emit = defineEmits<{
  pick: [kind: Verdict]
}>()

// The mark covers the whole logo. Each glyph sits near the centre of the
// largest circle its half of the square can hold, so both
// keep the same distance to the divider; the arrow and the cross share one stroke width.
const HALVES = [
  {
    kind: 'direct' as const,
    hit: 'M0 0H24L0 24Z',
    // turned 5° flatter: the horizontal barbs make the shaft read steeper than
    // the divider, this brings it back to looking parallel
    // a touch shorter and heavier than the struck arrow below, so the lone
    // arrow carries the same visual weight as arrow plus strike
    transform: 'translate(7.4 7.4) rotate(5) scale(1.4)',
    width: 1.55 / 1.4,
    paths: ['M1.2 -2.8H2.8L-2.8 2.8H-1.2'],
    color: 'text-success',
  },
  {
    kind: 'reject' as const,
    hit: 'M24 0V24H0Z',
    // the same two-way arrow, mirrored across the divider and struck through
    // square to it; the strike stays clear of the divider. Crossed out, the
    // shaft needs only half the arrow's compensation (2.5°) to read parallel.
    transform: 'translate(17 17) rotate(2.5) scale(1.5)',
    width: 1.4 / 1.5,
    paths: ['M1.2 -2.8H2.8L-2.8 2.8H-1.2', 'M-1.6 -1.6L2.2 2.2'],
    color: 'text-error',
  },
]
</script>

<style scoped>
.verdict-glow {
  filter: drop-shadow(0 0 1.2px currentColor) drop-shadow(0 0 3px currentColor);
}
</style>

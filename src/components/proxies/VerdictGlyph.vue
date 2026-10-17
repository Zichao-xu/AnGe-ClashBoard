<template>
  <!-- Direct and reject as two halves of one mark split by a diagonal: a two-way
       transfer above, a cross below. Each half is its own hit area and always
       wears its colour; the active half glows. -->
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
      d="M3.5 20.5L20.5 3.5"
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
// keep the same distance to the divider; the transfer is drawn finer than the
// cross because its two strokes and barbs carry more ink.
const HALVES = [
  {
    kind: 'direct' as const,
    hit: 'M0 0H24L0 24Z',
    transform: 'translate(7.6 7.6) rotate(45) scale(1.5)',
    width: 1.05 / 1.5,
    paths: ['M-1.05 3V-3L-2.35 -1.7', 'M1.05 -3V3L2.35 1.7'],
    color: 'text-success',
  },
  {
    kind: 'reject' as const,
    hit: 'M24 0V24H0Z',
    transform: 'translate(16.4 16.4) scale(1.5)',
    width: 1.4 / 1.5,
    paths: ['M-2.05 -2.05L2.05 2.05M2.05 -2.05L-2.05 2.05'],
    color: 'text-error',
  },
]
</script>

<style scoped>
.verdict-glow {
  filter: drop-shadow(0 0 1.2px currentColor) drop-shadow(0 0 3px currentColor);
}
</style>

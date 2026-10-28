<template>
  <!-- Direct or reject as an exit badge: no disc, only the verdict drawn in its
       colour and glowing, as the lit half of the verdict mark does -->
  <svg
    :width="size"
    :height="size"
    viewBox="0 0 24 24"
    fill="none"
    stroke-linecap="round"
    stroke-linejoin="round"
    aria-hidden="true"
    :class="kind === 'direct' ? 'text-success' : 'text-error'"
  >
    <!-- direct: traffic both ways, two diagonal arrows side by side, opposite -->
    <g
      v-if="kind === 'direct'"
      class="badge-glow"
      fill="currentColor"
      stroke="currentColor"
    >
      <g
        v-for="turn in [-45, 135]"
        :key="turn"
        :transform="`translate(12 12) rotate(${turn}) translate(0 -2.9)`"
      >
        <path
          d="M-6.2 0H2.4"
          stroke-width="1.9"
        />
        <polygon
          points="6.4,0 2,-2.8 2,2.8"
          stroke-width="1"
        />
      </g>
    </g>
    <g
      v-else
      class="badge-glow"
      transform="translate(12 12) scale(2)"
      stroke="currentColor"
      :stroke-width="1.5 / 2"
    >
      <path d="M1.2 -2.8H2.8L-2.8 2.8H-1.2" />
      <path d="M-2.6 -2.6L2.6 2.6" />
    </g>
  </svg>
</template>

<script setup lang="ts">
defineProps<{
  kind: 'direct' | 'reject'
  size: number
}>()
</script>

<style scoped>
.badge-glow {
  filter: drop-shadow(0 0 1px currentColor) drop-shadow(0 0 2.5px currentColor);
}
</style>

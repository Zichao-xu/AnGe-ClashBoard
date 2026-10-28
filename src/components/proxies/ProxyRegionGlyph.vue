<template>
  <!-- Round badges: the background colour is part of the mark, not a backdrop -->
  <img
    v-if="BADGE_IMAGES[glyph]"
    :src="BADGE_IMAGES[glyph]"
    :width="size"
    :height="size"
    class="rounded-full"
    alt=""
    draggable="false"
  />
  <svg
    v-else
    :width="size"
    :height="size"
    viewBox="0 0 24 24"
    aria-hidden="true"
  >
    <defs>
      <clipPath :id="clipId">
        <circle
          cx="12"
          cy="12"
          r="12"
        />
      </clipPath>
    </defs>
    <g :clip-path="`url(#${clipId})`">
      <!-- United States: stars and stripes -->
      <g v-if="glyph === 'us'">
        <rect
          width="24"
          height="24"
          fill="#fff"
        />
        <rect
          v-for="index in 7"
          :key="index"
          x="0"
          :y="(index - 1) * 3.7"
          width="24"
          height="1.85"
          fill="#e0312a"
        />
        <rect
          width="11.5"
          height="12.9"
          fill="#1f4693"
        />
        <circle
          v-for="(star, index) in usStars"
          :key="index"
          :cx="star[0]"
          :cy="star[1]"
          r="0.62"
          fill="#fff"
        />
      </g>

      <!-- Hong Kong: white bauhinia on red -->
      <g v-else-if="glyph === 'hk'">
        <rect
          width="24"
          height="24"
          fill="#de2910"
        />
        <g fill="#fff">
          <path
            v-for="angle in [0, 72, 144, 216, 288]"
            :key="angle"
            d="M12 12.4C9.6 10.6 9.1 6.6 11.4 4.2C12.6 3 14.3 3.1 15.2 3.9C13.3 4.6 12.2 6.5 12.6 8.6C12.9 10 13 11.3 12 12.4Z"
            :transform="`rotate(${angle} 12 12)`"
          />
        </g>
        <polygon
          v-for="angle in [0, 72, 144, 216, 288]"
          :key="`s${angle}`"
          :points="starPoints(12.3, 7.2, 5, 0.9, 0.38)"
          fill="#de2910"
          :transform="`rotate(${angle} 12 12)`"
        />
      </g>

      <!-- Taiwan: white sun on blue canton over red -->
      <g v-else-if="glyph === 'tw'">
        <rect
          width="24"
          height="24"
          fill="#d52b1e"
        />
        <rect
          width="13"
          height="12.5"
          fill="#1e3b8f"
        />
        <polygon
          :points="starPoints(6.6, 6.4, 12, 4.4, 2.7)"
          fill="#fff"
        />
        <circle
          cx="6.6"
          cy="6.4"
          r="2.1"
          fill="#1e3b8f"
        />
        <circle
          cx="6.6"
          cy="6.4"
          r="1.7"
          fill="#fff"
        />
      </g>

      <!-- Japan: red disc on white -->
      <g v-else-if="glyph === 'jp'">
        <rect
          width="24"
          height="24"
          fill="#fff"
        />
        <circle
          cx="12"
          cy="12"
          r="6.2"
          fill="#bc002d"
        />
      </g>

      <!-- Singapore: crescent and five stars on red over white -->
      <g v-else-if="glyph === 'sg'">
        <rect
          width="24"
          height="24"
          fill="#fff"
        />
        <rect
          width="24"
          height="12"
          fill="#ef3340"
        />
        <circle
          cx="8"
          cy="6.6"
          r="4.1"
          fill="#fff"
        />
        <circle
          cx="9.6"
          cy="6.6"
          r="3.8"
          fill="#ef3340"
        />
        <polygon
          v-for="(point, index) in pentagon(13.4, 6.6, 2.3)"
          :key="index"
          :points="starPoints(point[0], point[1], 5, 0.95, 0.4)"
          fill="#fff"
        />
      </g>

      <!-- Every region: a coloured globe -->
      <g v-else-if="glyph === 'globe'">
        <rect
          width="24"
          height="24"
          fill="#4aa8e8"
        />
        <path
          d="M1.5 6.5C4 5 7 5.6 7.6 8C8.2 10.4 5.6 11 6.4 13.4C7 15 4 15.5 2.5 14Z"
          fill="#7cb342"
        />
        <path
          d="M8.5 15.5C10.5 14.8 12 16.4 11.2 18.6C10.6 20.4 11.6 22 10 23.5L7.6 23C8.6 20.8 7 18.6 8.5 15.5Z"
          fill="#7cb342"
        />
        <g
          fill="none"
          stroke="#0d3b66"
          stroke-width="1.1"
        >
          <path d="M12 0V24" />
          <path d="M12 0C17 3 17 21 12 24" />
          <path d="M12 7.5H24M12 16.5H24" />
          <circle
            cx="12"
            cy="12"
            r="11.4"
          />
        </g>
      </g>

      <!-- Every node: ALL in dark rounded strokes on teal -->
      <g v-else-if="glyph === 'all'">
        <rect
          width="24"
          height="24"
          fill="#5ec8a6"
        />
        <path
          d="M4.4 15.6L7.2 8.4L10 15.6M5.5 13H8.9M12 8.4V15.6H15M16.6 8.4V15.6H19.6"
          fill="none"
          stroke="#16202a"
          stroke-width="1.7"
          stroke-linecap="round"
          stroke-linejoin="round"
        />
      </g>

      <!-- Dedicated line: route between two endpoints on navy -->
      <g v-else-if="glyph === 'line'">
        <rect
          width="24"
          height="24"
          fill="#1c2f6e"
        />
        <path
          d="M7 17C7 11 17 13 17 7"
          fill="none"
          stroke="#fff"
          stroke-width="2"
          stroke-linecap="round"
        />
        <circle
          cx="7"
          cy="17"
          r="2.4"
          fill="#fff"
        />
        <circle
          cx="17"
          cy="7"
          r="2.4"
          fill="#e0312a"
        />
      </g>

      <!-- Direct: yellow hammer and sickle on red -->
      <g v-else-if="glyph === 'direct'">
        <rect
          width="24"
          height="24"
          fill="#e60012"
        />
        <g
          fill="none"
          stroke="#ffd400"
          stroke-linecap="round"
        >
          <path
            d="M8.6 6.6A6.3 6.3 0 1 1 7.7 16.6"
            stroke-width="2.4"
          />
          <path
            d="M7.7 16.6L5.6 18.7"
            stroke-width="2.8"
          />
          <path
            d="M8.8 9.6L17 17.8"
            stroke-width="2.2"
          />
        </g>
        <rect
          x="5.3"
          y="7"
          width="6.4"
          height="2.9"
          rx="0.5"
          fill="#ffd400"
          transform="rotate(-45 8.5 8.45)"
        />
      </g>

      <!-- Reject: white cross on dark red -->
      <g v-else-if="glyph === 'reject'">
        <rect
          width="24"
          height="24"
          fill="#7a1f1f"
        />
        <path
          d="M7.5 7.5L16.5 16.5M16.5 7.5L7.5 16.5"
          stroke="#fff"
          stroke-width="2.6"
          stroke-linecap="round"
        />
      </g>
    </g>
  </svg>
</template>

<script setup lang="ts">
import directBadge from '@/assets/region-badges/direct.png'
import globeBadge from '@/assets/region-badges/globe.png'
import hkBadge from '@/assets/region-badges/hk.png'
import lineBadge from '@/assets/region-badges/line.png'
import sgBadge from '@/assets/region-badges/sg.png'
import twBadge from '@/assets/region-badges/tw.png'
import usBadge from '@/assets/region-badges/us.png'
import { useId } from 'vue'

// Supplied artwork takes precedence; regions without one fall back to the SVG below.
const BADGE_IMAGES: Record<string, string> = {
  us: usBadge,
  hk: hkBadge,
  tw: twBadge,
  sg: sgBadge,
  globe: globeBadge,
  direct: directBadge,
  line: lineBadge,
}

defineProps<{
  glyph: string
  size: number
}>()

const clipId = `region-glyph-${useId()}`

// Star or sun polygon with `rays` points between the outer and inner radius.
const starPoints = (cx: number, cy: number, rays: number, outer: number, inner: number) => {
  const points: string[] = []

  for (let i = 0; i < rays * 2; i++) {
    const radius = i % 2 === 0 ? outer : inner
    const angle = (Math.PI * i) / rays - Math.PI / 2

    points.push(
      `${(cx + radius * Math.cos(angle)).toFixed(2)},${(cy + radius * Math.sin(angle)).toFixed(2)}`,
    )
  }

  return points.join(' ')
}

const pentagon = (cx: number, cy: number, radius: number) =>
  Array.from({ length: 5 }, (_, i) => {
    const angle = (Math.PI * 2 * i) / 5 - Math.PI / 2

    return [cx + radius * Math.cos(angle), cy + radius * Math.sin(angle)]
  })

// A sparse grid suggests the canton's stars at badge size.
const usStars = [1.6, 4, 6.4, 8.8].flatMap((x) => [1.8, 4.6, 7.4, 10.2].map((y) => [x, y]))
</script>

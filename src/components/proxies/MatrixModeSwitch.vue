<template>
  <!-- Automatic and manual as two halves of one square split by a diagonal, the
       same language as the direct / reject mark on the logos. The active half
       glows; a half that cannot be picked stays dim. -->
  <div class="relative aspect-square w-full select-none">
    <svg
      class="text-base-content/25 pointer-events-none absolute inset-0 h-full w-full"
      viewBox="0 0 100 100"
      preserveAspectRatio="none"
      aria-hidden="true"
    >
      <path
        d="M14 86L86 14"
        stroke="currentColor"
        stroke-width="1.5"
        stroke-linecap="round"
        vector-effect="non-scaling-stroke"
      />
    </svg>
    <button
      v-for="half in HALVES"
      :key="half.key"
      type="button"
      :disabled="!switchable && active !== half.key"
      :aria-pressed="active === half.key"
      :style="{ clipPath: half.clip }"
      :class="[
        'absolute inset-0 flex flex-col gap-1 p-2.5 transition duration-150',
        half.align,
        active === half.key
          ? 'text-primary mode-glow'
          : switchable
            ? 'text-base-content/35 hover:text-base-content/70 cursor-pointer'
            : 'text-base-content/15 cursor-default',
        active === half.key && !switchable ? 'cursor-default' : '',
      ]"
      @click="active !== half.key && switchable && emit('toggle')"
    >
      <component
        :is="half.icon"
        class="h-7 w-7 stroke-[1.6]"
      />
      <span class="text-xs leading-none">{{ half.label }}</span>
    </button>
  </div>
</template>

<script setup lang="ts">
import { BoltIcon, HandRaisedIcon } from '@heroicons/vue/24/outline'
import { computed } from 'vue'
import { useI18n } from 'vue-i18n'

defineProps<{
  active: 'auto' | 'manual'
  switchable: boolean
}>()

const emit = defineEmits<{
  toggle: []
}>()

const { t } = useI18n()

const HALVES = computed(() => [
  {
    key: 'auto' as const,
    icon: BoltIcon,
    label: t('matrixAuto'),
    clip: 'polygon(0 0, 100% 0, 0 100%)',
    align: 'items-start justify-start',
  },
  {
    key: 'manual' as const,
    icon: HandRaisedIcon,
    label: t('matrixManual'),
    clip: 'polygon(100% 0, 100% 100%, 0 100%)',
    align: 'flex-col-reverse items-end justify-start',
  },
])
</script>

<style scoped>
.mode-glow {
  filter: drop-shadow(0 0 1.5px currentColor) drop-shadow(0 0 4px currentColor);
}
</style>

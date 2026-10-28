<template>
  <div
    class="proxy-matrix bg-base-100 relative rounded-lg p-3 max-md:p-2"
    @keydown.esc="closePopover"
  >
    <!-- as many columns as fit, cards dealt into them left to right. Each column
         stacks on its own, so an opened card only pushes down its own column. -->
    <div
      ref="containerRef"
      class="flex gap-x-6"
    >
      <div
        v-for="(column, index) in cardColumns"
        :key="index"
        class="flex min-w-0 flex-1 flex-col"
      >
        <div
          v-for="row in column"
          :key="row.name"
          data-matrix-card
          :class="[
            'border-base-content/10 relative isolate flex flex-col gap-3 overflow-hidden border-b px-1 py-3.5',
            !row.isDirect && 'cursor-pointer',
          ]"
          @click="toggleExpand(row)"
        >
          <!-- the service's logo as a faint backdrop in the card's open middle. Its
               drawn shape (not the image box, whose padding varies) is sized and
               centred the same way on every card -->
          <div
            v-if="proxyMap[row.name]?.icon"
            class="pointer-events-none absolute inset-0 -z-10 saturate-[.75]"
            aria-hidden="true"
          >
            <img
              class="absolute max-w-none"
              :src="backdropSrc(proxyMap[row.name].icon)"
              :style="backdropStyle(proxyMap[row.name].icon)"
              alt=""
            />
          </div>
          <!-- frosted glass over the lower part of the backdrop, always there: its top
               edge fades out upwards, and an open drawer simply extends it -->
          <div
            class="bg-base-100/40 pointer-events-none absolute inset-x-0 bottom-0 -z-[5] [mask-image:linear-gradient(to_bottom,transparent,black_48px)] backdrop-blur-md"
            :style="{ top: `${frostTop}px` }"
            aria-hidden="true"
          />
          <!-- latency of the route pinned to the card corner; a click re-tests the
               group as on the policy cards -->
          <LatencyTag
            v-if="!row.isDirect"
            class="bg-base-200/50 hover:bg-base-200 absolute top-1 right-0 cursor-pointer"
            :loading="latencyTesting.has(row.name)"
            :name="row.now"
            :group-name="row.name"
            @click.stop="handlerLatencyTest(row.name)"
          />
          <div class="relative flex flex-col gap-2.5">
            <!-- identity: the group's name, the logo now lies behind the card -->
            <div class="flex h-7 items-center">
              <span
                class="text-base-content min-w-0 flex-1 truncate pr-12 text-xl leading-none font-semibold max-md:text-lg"
                >{{ row.name }}</span
              >
            </div>
            <!-- routing, flush with the card's left edge: the chosen exit beside the
                 node actually in use and the groups it was reached through, the other
                 exits under it; on the right the direct / reject mark above the node's
                 live traffic. Every line always renders so the card never changes height -->
            <div class="flex gap-3">
              <div class="flex min-w-0 flex-1 flex-col gap-2.5">
                <div
                  class="flex min-w-0 items-center gap-2"
                  :style="{ height: `${slotSizes[0]}px` }"
                >
                  <div
                    class="shrink-0"
                    :style="{ width: `${slotSizes[0]}px`, height: `${slotSizes[0]}px` }"
                  />
                  <div class="flex min-w-0 flex-col gap-0.5 leading-tight">
                    <span
                      class="text-base-content truncate text-base font-medium max-md:text-[15px]"
                      >{{ row.leaf }}</span
                    >
                    <span class="text-base-content/40 h-4 truncate text-xs">{{
                      row.via.join(' › ')
                    }}</span>
                  </div>
                </div>
                <!-- room for the row of other exits -->
                <div :style="{ height: `${slotSizes[1]}px` }" />
              </div>
              <div class="flex shrink-0 flex-col items-end justify-between">
                <!-- direct / reject are not exits to browse but a verdict on the whole
                     service: one mark whose halves are direct and reject, always in
                     colour; a tap on a half picks it, a tap on the lit half goes back -->
                <div class="flex h-[54px] w-[54px] justify-center max-md:h-[42px] max-md:w-[42px]">
                  <VerdictGlyph
                    v-if="row.terminals.direct || row.terminals.reject"
                    class="h-full w-full"
                    :active="row.verdict"
                    :available="row.terminals"
                    interactive
                    @pick="(kind) => handlerVerdict(row, kind)"
                  />
                </div>
                <!-- live traffic of the node, on the exits' baseline: upload faint and
                     small above, download stronger and larger below. Every reading has
                     the same shape (four characters and a k / M / G unit), so arrows,
                     digits and units hold their places as it moves -->
                <div
                  class="grid shrink-0 grid-cols-[auto_auto_auto] items-baseline gap-x-1.5 font-mono leading-tight tabular-nums"
                >
                  <template
                    v-for="line in trafficLines(row.leaf)"
                    :key="line.arrow"
                  >
                    <span :class="line.class">{{ line.arrow }}</span>
                    <span :class="[line.class, 'justify-self-end whitespace-pre']">{{
                      line.value
                    }}</span>
                    <span :class="line.class">{{ line.unit }}</span>
                  </template>
                </div>
              </div>
            </div>
            <!-- the exits: the chosen one leading the routing block, the largest; the
                 rest in a row under the text, stepping down in size on one baseline.
                 All are placed by position, so a new choice glides up from the row
                 into its slot while the others close up behind it. -->
            <button
              v-for="column in exitOrder(row)"
              :key="column"
              type="button"
              :class="[
                'group/cell absolute flex cursor-pointer items-center justify-center rounded-full transition-[left,top,width,height,box-shadow] duration-300 ease-out',
                row.nowExit === column && 'ring-primary ring-offset-base-100 ring-2 ring-offset-2',
              ]"
              :style="exitStyle(row, column)"
              :aria-label="`${row.name} → ${column}`"
              :aria-pressed="row.nowExit === column"
              :aria-expanded="row.nowExit === column ? expandedRow === row.name : undefined"
              @click.stop="handlerCellClick(row, column)"
            >
              <!-- drawn at the large size and scaled down, so a size change animates -->
              <span
                :class="[
                  'absolute flex items-center justify-center transition-[opacity,filter,scale] duration-300',
                  row.nowExit === column
                    ? 'opacity-100'
                    : 'brightness-(--fade) saturate-(--sat) group-hover/cell:brightness-100 group-hover/cell:saturate-100',
                ]"
                :style="{
                  '--fade': fadeOf(row, column),
                  '--sat': saturationOf(row, column),
                  width: `${slotSizes[0]}px`,
                  height: `${slotSizes[0]}px`,
                  scale: slotOf(row, column) / slotSizes[0],
                }"
                :title="exitTitle(column)"
              >
                <VerdictBadge
                  v-if="verdictOf(column)"
                  :kind="verdictOf(column)!"
                  :size="badgeSize"
                />
                <ProxyIcon
                  v-else-if="proxyMap[column]?.icon"
                  :icon="proxyMap[column].icon"
                  :size="badgeSize"
                  :margin="0"
                />
                <ProxyRegionGlyph
                  v-else-if="columnGlyphs[column]"
                  :glyph="columnGlyphs[column]"
                  :size="badgeSize"
                />
                <span
                  v-else-if="columnFlags[column] && columnFlags[column] !== MIXED"
                  class="text-[26px] leading-none"
                  >{{ columnFlags[column] }}</span
                >
                <GlobeAltIcon
                  v-else-if="columnFlags[column] === MIXED"
                  class="text-base-content h-7 w-7"
                />
                <span
                  v-else
                  class="text-base-content max-w-14 truncate text-xs"
                >
                  {{ column }}
                </span>
              </span>
            </button>
          </div>
          <!-- drill-down, opened by tapping the chosen exit again: it sits inside the
               card and pushes the cards below down, its notch under the chosen exit,
               which always leads the row. Mode on the left, the exit's nodes on the right. -->
          <!-- the drawer slides open: its height grows from nothing while it fades
               in. The wrapper takes back the card's row gap so nothing jumps. -->
          <Transition
            :css="false"
            @enter="drawerEnter"
            @after-enter="drawerSettle"
            @leave="drawerLeave"
          >
            <div
              v-if="expandedRow === row.name && !row.isDirect"
              class="-mt-3 overflow-hidden"
            >
              <div
                class="border-base-content/15 relative mt-4 flex flex-col gap-2.5 rounded-lg border p-3"
                @click.stop
              >
                <span
                  class="bg-base-100 border-base-content/15 absolute -top-1.5 left-[21px] h-3 w-3 -translate-x-1/2 rotate-45 border-t border-l max-md:left-[18px]"
                />
                <!-- two faces of the drawer: the nodes behind the exit, or the domains
                     the group's rules send through it -->
                <div class="flex items-baseline gap-4 text-sm">
                  <button
                    v-for="tab in DRAWER_TABS"
                    :key="tab"
                    type="button"
                    :class="[
                      'cursor-pointer transition-colors',
                      drawerTab === tab
                        ? 'text-base-content'
                        : 'text-base-content/40 hover:text-base-content/70',
                    ]"
                    @click="drawerTab = tab"
                  >
                    {{ tab === 'nodes' ? t('nodeGroup') : t('domainPenetration') }}
                  </button>
                </div>
                <!-- switching faces slides the new one in from the side it lives on
                     (domains to the right of nodes) while the box eases to its height -->
                <div class="overflow-hidden transition-[height] duration-300 ease-out">
                  <Transition
                    mode="out-in"
                    enter-active-class="transition duration-200 ease-out"
                    leave-active-class="transition duration-150 ease-in"
                    :enter-from-class="faceEnterFrom"
                    :leave-to-class="faceLeaveTo"
                    @before-leave="lockFaceHeight"
                    @enter="growFaceHeight"
                    @after-enter="releaseFaceHeight"
                  >
                    <div
                      v-if="drawerTab === 'nodes'"
                      key="nodes"
                      class="flex gap-3"
                    >
                      <div class="w-24 shrink-0 self-center max-md:w-20">
                        <MatrixModeSwitch
                          :active="popoverMode.active"
                          :switchable="popoverMode.switchable"
                          @toggle="popoverRow && handlerModeToggle(popoverRow)"
                        />
                      </div>
                      <div class="bg-base-content/10 w-px shrink-0 self-stretch" />
                      <div
                        class="grid max-h-64 min-w-0 flex-1 auto-rows-max grid-cols-2 content-start gap-2 overflow-y-auto pr-0.5"
                      >
                        <button
                          v-for="node in popoverNodes.list"
                          :key="node.name"
                          type="button"
                          :disabled="!popoverNodes.selectable"
                          :title="node.name"
                          :class="[
                            'relative flex min-w-0 flex-col gap-1 overflow-hidden rounded-md border px-2 pt-1.5 pb-2 text-left',
                            node.current ? 'border-primary' : 'border-base-content/10',
                            popoverNodes.selectable && !node.current
                              ? 'hover:border-base-content/30 cursor-pointer'
                              : 'cursor-default',
                            !node.alive && 'opacity-40',
                          ]"
                          @click="handlerNodeSelect(node.name)"
                        >
                          <span
                            :class="[
                              'truncate pr-3 text-xs',
                              node.current ? 'text-base-content' : 'text-base-content/70',
                            ]"
                            >{{ node.name }}</span
                          >
                          <!-- latency history, framed to its own min~max so the shape stays readable -->
                          <svg
                            :class="['h-5 w-full', node.color]"
                            viewBox="0 0 100 20"
                            preserveAspectRatio="none"
                            aria-hidden="true"
                          >
                            <polyline
                              v-if="node.spark"
                              :points="node.spark"
                              fill="none"
                              stroke="currentColor"
                              stroke-width="1.5"
                              stroke-linejoin="round"
                              vector-effect="non-scaling-stroke"
                            />
                          </svg>
                          <span
                            class="text-base-content/50 font-mono text-[10px] leading-none tabular-nums"
                          >
                            {{ node.alive ? node.metric || '—' : t('matrixUnavailable') }}
                          </span>
                          <!-- folded corner: the current node's is filled -->
                          <span
                            :class="[
                              'absolute right-0 bottom-0 h-2.5 w-2.5 [clip-path:polygon(100%_0,100%_100%,0_100%)]',
                              node.current ? 'bg-primary' : 'bg-base-content/20',
                            ]"
                          />
                        </button>
                      </div>
                    </div>
                    <MatrixDomainList
                      v-else
                      key="domains"
                      :group-name="row.name"
                    />
                  </Transition>
                </div>
              </div>
            </div>
          </Transition>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { policyGroups } from '@/composables/proxies'
import { GLOBAL, NOT_CONNECTED, PROXY_TYPE } from '@/constant'
import { getColorForLatency } from '@/helper'
import { showNotification } from '@/helper/notification'
import { isMiddleScreen, prettyBytesHelper } from '@/helper/utils'
import { activeConnections } from '@/store/connections'
import {
  getDescendantProxyNames,
  getHistoryByName,
  getLatencyByName,
  getNowProxyNodeName,
  getProxyGroupChains,
  handlerProxySelect,
  proxyGroupLatencyTest,
  proxyMap,
} from '@/store/proxies'
import { GlobeAltIcon } from '@heroicons/vue/24/outline'
import { useElementSize, useStorage } from '@vueuse/core'
import { computed, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import LatencyTag from './LatencyTag.vue'
import MatrixDomainList from './MatrixDomainList.vue'
import MatrixModeSwitch from './MatrixModeSwitch.vue'
import ProxyIcon from './ProxyIcon.vue'
import ProxyRegionGlyph from './ProxyRegionGlyph.vue'
import VerdictBadge from './VerdictBadge.vue'
import VerdictGlyph from './VerdictGlyph.vue'

type ModePair = {
  auto: string
  manual: string
  isAuto: boolean
  // the service lists both halves itself, so switching touches only this service;
  // otherwise the halves sit under a shared exit group and the switch is shared too
  flat: boolean
}

type MatrixRow = {
  name: string
  now: string
  options: string[]
  // the options as exits: both halves of an auto / manual pair become one exit,
  // named by its automatic half
  exits: string[]
  nowExit: string
  isDirect: boolean
  leaf: string
  leafGroup: string
  via: string[]
  terminals: Partial<Record<Verdict, string>>
  verdict: Verdict | null
  mode: ModePair | null
  speedText: string
  latencyText: string
  latencyClass: string
}

const { t } = useI18n()

// One column of business blocks on phones, two side by side on wider screens;
// icons shrink on phones so a full row of exits still fits.
// Exits step down in size: the chosen one beside the node name, then along the row
// large, medium and small for the rest, all standing on one baseline. Badges are drawn at the large size.
const slotSizes = computed(() => (isMiddleScreen.value ? [36, 30, 26, 22] : [42, 36, 30, 24]))
const badgeSize = computed(() => slotSizes.value[0])

const AUTO_GROUP_TYPES = new Set<string>([
  PROXY_TYPE.URLTest,
  PROXY_TYPE.Smart,
  PROXY_TYPE.Fallback,
  PROXY_TYPE.LoadBalance,
])
const TERMINAL_TYPES = new Set<string>([
  PROXY_TYPE.Direct,
  PROXY_TYPE.Reject,
  PROXY_TYPE.RejectDrop,
  PROXY_TYPE.Pass,
])

const typeOf = (name: string) => proxyMap.value[name]?.type?.toLowerCase() ?? ''

type Verdict = 'direct' | 'reject'

const verdictOf = (name: string): Verdict | null => {
  const type = typeOf(name)

  if (type === PROXY_TYPE.Direct) return 'direct'
  if (type === PROXY_TYPE.Reject || type === PROXY_TYPE.RejectDrop) return 'reject'
  return null
}

const getTerminals = (options: string[]) => {
  const result: Partial<Record<Verdict, string>> = {}

  options.forEach((option) => {
    const verdict = verdictOf(option)

    if (verdict && !result[verdict]) {
      result[verdict] = option
    }
  })

  return result
}

// An exit group is switchable when it is a selector choosing between exactly
// one automatic group and one manual selector, e.g. 香港 → [香港-速率, 香港-手动].
const getModePair = (exit: string): ModePair | null => {
  const group = proxyMap.value[exit]

  if (!group || typeOf(exit) !== PROXY_TYPE.Selector || group.all?.length !== 2) {
    return null
  }

  const auto = group.all.find((name) => AUTO_GROUP_TYPES.has(typeOf(name)))
  const manual = group.all.find((name) => typeOf(name) === PROXY_TYPE.Selector)

  if (!auto || !manual) {
    return null
  }

  return { auto, manual, isAuto: group.now === auto, flat: false }
}

const sameMembers = (a: string, b: string) => {
  const left = proxyMap.value[a]?.all ?? []
  const right = new Set(proxyMap.value[b]?.all ?? [])

  return left.length > 0 && left.length === right.size && left.every((name) => right.has(name))
}

// A service may list an exit's two halves itself, e.g. 流媒体 → [日本-速率, 日本-手动]:
// an automatic group and a selector over the same nodes. Each such option maps to its pair.
const flatPairs = computed(() => {
  const result: Record<string, { auto: string; manual: string }> = {}

  policyGroups.value.forEach((groupName) => {
    const options = proxyMap.value[groupName]?.all ?? []
    const manuals = options.filter((name) => typeOf(name) === PROXY_TYPE.Selector)

    options
      .filter((name) => AUTO_GROUP_TYPES.has(typeOf(name)) && !result[name])
      .forEach((auto) => {
        const manual = manuals.find((name) => !result[name] && sameMembers(auto, name))

        if (manual) {
          result[auto] = result[manual] = { auto, manual }
        }
      })
  })

  return result
})

const exitOf = (option: string) => flatPairs.value[option]?.auto ?? option

const getRowMode = (groupName: string, now: string): ModePair | null => {
  const pair = flatPairs.value[now]
  const own = proxyMap.value[groupName]?.all ?? []

  if (pair && own.includes(pair.auto) && own.includes(pair.manual)) {
    return { ...pair, isAuto: now === pair.auto, flat: true }
  }

  return getModePair(now)
}

// The option behind an exit for this row: a paired exit keeps the row's current
// mode, so moving from 日本-手动 to 香港 lands on 香港-手动.
const optionOf = (row: MatrixRow, exit: string) => {
  const pair = flatPairs.value[exit]

  if (!pair || !row.options.includes(pair.manual)) {
    return exit
  }

  return row.mode?.flat && !row.mode.isAuto ? pair.manual : pair.auto
}

const exitTitle = (exit: string) => {
  const pair = flatPairs.value[exit]

  return pair ? `${pair.auto} / ${pair.manual}` : exit
}

// Throughput results: per-group `speeds` (bytes/s) from the url-test throughput
// strategy, or per-proxy `speed[url]` from the proxy level speed test.
const getSpeed = (groupName: string, nodeName: string): number | undefined => {
  const group = proxyMap.value[groupName] as unknown as {
    speeds?: Record<string, number>
    throughputUrl?: string
  }
  const groupSpeed = group?.speeds?.[nodeName]

  if (typeof groupSpeed === 'number' && groupSpeed > 0) {
    return groupSpeed
  }

  const url = group?.throughputUrl
  const node = proxyMap.value[nodeName] as unknown as {
    speed?: Record<string, { speed: number }>
  }
  const nodeSpeed = url ? node?.speed?.[url]?.speed : undefined

  return nodeSpeed && nodeSpeed > 0 ? nodeSpeed : undefined
}

const formatSpeed = (bytesPerSecond: number) => `${prettyBytesHelper(bytesPerSecond)}/s`

const isAlive = (name: string) => {
  return (proxyMap.value[name] as unknown as { alive?: boolean })?.alive !== false
}

const rows = computed<MatrixRow[]>(() => {
  return policyGroups.value
    .filter((name) => name !== GLOBAL && proxyMap.value[name]?.all?.length)
    .map((name) => {
      const group = proxyMap.value[name]
      const chain = getProxyGroupChains(name)
      const leafGroup = chain[chain.length - 1]
      const leaf = getNowProxyNodeName(name)
      const isDirect = TERMINAL_TYPES.has(typeOf(leaf))
      const speed = isDirect ? undefined : getSpeed(leafGroup, leaf)
      const latency = isDirect ? NOT_CONNECTED : getLatencyByName(leaf, leafGroup)
      const speedText = speed ? formatSpeed(speed) : ''
      const latencyText = latency > NOT_CONNECTED ? `${latency}ms` : ''

      const options = group.all ?? []

      return {
        name,
        now: group.now,
        options,
        exits: [...new Set(options.map(exitOf))],
        nowExit: exitOf(group.now),
        isDirect,
        leaf,
        leafGroup,
        via: chain.filter((hop) => hop !== name && hop !== leaf),
        terminals: getTerminals(group.all ?? []),
        verdict: verdictOf(group.now),
        mode: getRowMode(name, group.now),
        speedText,
        latencyText,
        latencyClass: getColorForLatency(latency),
      }
    })
})

// Rows keep the configured order: switching a row's exit must never move it.
const orderedRows = rows

// Each card: title over a faint logo backdrop, the node in use and its route, live traffic on the right,
// one row of exits below. Cards need about 24rem; as many columns as fit.
const CARD_MIN_WIDTH = 384
const COLUMN_GAP = 24
const containerRef = ref<HTMLElement>()
const { width: containerWidth } = useElementSize(containerRef)

const cardColumns = computed(() => {
  const count = isMiddleScreen.value
    ? 1
    : Math.max(1, Math.floor((containerWidth.value + COLUMN_GAP) / (CARD_MIN_WIDTH + COLUMN_GAP)))
  const result: MatrixRow[][] = Array.from({ length: count }, () => [])

  orderedRows.value.forEach((row, index) => result[index % count].push(row))

  return result
})

// A traffic reading in a fixed shape: three significant digits padded to four
// characters, always with a k / M / G prefix (0 reads "0.00 kB/s").
const SPEED_UNITS = ['kB/s', 'MB/s', 'GB/s']

const splitSpeed = (bytesPerSecond: number) => {
  let value = bytesPerSecond / 1000
  let index = 0

  while (value >= 999.5 && index < SPEED_UNITS.length - 1) {
    value /= 1000
    index++
  }

  const digits =
    value >= 99.95 ? value.toFixed(0) : value >= 9.995 ? value.toFixed(1) : value.toFixed(2)

  return { value: digits.padStart(4, ' '), unit: SPEED_UNITS[index] }
}

const trafficLines = (name: string) => [
  {
    arrow: '↑',
    class: 'text-base-content/40 text-[11px]',
    ...splitSpeed(traffic.value[name]?.up ?? 0),
  },
  {
    arrow: '↓',
    class: 'text-base-content/80 text-sm',
    ...splitSpeed(traffic.value[name]?.down ?? 0),
  },
]

// Where the frosted glass over the backdrop starts: 36px above the bottom of
// the card's three rows (title, chosen exit, other exits), so its 48px fade-in is
// complete just where an open drawer would begin.
const frostTop = computed(() => 14 + 28 + 10 + slotSizes.value[0] + 10 + slotSizes.value[1] - 36)

// Backdrop logos: each icon's drawn area is measured once (alpha bounding box,
// as fractions of the image), so every logo's shape gets the same size and centre
// regardless of the padding inside its image.
// salience: how loudly the drawn shape reads on the dark card once normalised in
// size: its filled share times its contrast with the background plus its colour.
type InkBox = { x: number; y: number; w: number; h: number; salience: number }

const SVG_TEXT_PREFIX = 'data:image/svg+xml,'
const inkBoxes = ref<Record<string, InkBox>>({})
const inkPending = new Set<string>()

const backdropSrc = (icon: string) =>
  icon.startsWith(SVG_TEXT_PREFIX)
    ? SVG_TEXT_PREFIX + encodeURIComponent(icon.slice(SVG_TEXT_PREFIX.length))
    : icon

const measureInk = (icon: string) => {
  if (inkPending.has(icon)) return
  inkPending.add(icon)

  const image = new Image()

  image.onload = () => {
    const size = 128
    const canvas = document.createElement('canvas')

    canvas.width = size
    canvas.height = size

    const context = canvas.getContext('2d')

    if (!context) return
    context.drawImage(image, 0, 0, size, size)

    const { data } = context.getImageData(0, 0, size, size)
    let minX = size
    let minY = size
    let maxX = -1
    let maxY = -1

    for (let y = 0; y < size; y++) {
      for (let x = 0; x < size; x++) {
        if (data[(y * size + x) * 4 + 3] > 16) {
          minX = Math.min(minX, x)
          maxX = Math.max(maxX, x)
          minY = Math.min(minY, y)
          maxY = Math.max(maxY, y)
        }
      }
    }

    if (maxX < 0) return

    const boxW = maxX - minX + 1
    const boxH = maxY - minY + 1
    const side = Math.max(boxW, boxH)
    const BACKGROUND_LUMA = 0.13
    let weight = 0
    let contrast = 0
    let chroma = 0

    for (let y = minY; y <= maxY; y++) {
      for (let x = minX; x <= maxX; x++) {
        const i = (y * size + x) * 4
        const a = data[i + 3] / 255

        if (a === 0) continue

        const r = data[i] / 255
        const g = data[i + 1] / 255
        const b = data[i + 2] / 255

        weight += a
        contrast += a * Math.abs(0.2126 * r + 0.7152 * g + 0.0722 * b - BACKGROUND_LUMA)
        chroma += a * (Math.max(r, g, b) - Math.min(r, g, b))
      }
    }

    const filled = weight / (side * side)
    const salience = weight ? filled * (contrast / weight + 1.2 * (chroma / weight)) : 0

    inkBoxes.value = {
      ...inkBoxes.value,
      [icon]: {
        x: minX / size,
        y: minY / size,
        w: boxW / size,
        h: boxH / size,
        salience,
      },
    }
  }
  image.src = backdropSrc(icon)
}

const backdropStyle = (icon: string) => {
  const mobile = isMiddleScreen.value
  // the shape's longer side, and where its centre sits: in the open space between
  // the routing block and the right-hand column
  const shape = mobile ? 120 : 180
  const centreX = mobile ? 56 : 64
  const centreY = mobile ? 104 : 124
  const ink = inkBoxes.value[icon]

  if (!ink) {
    measureInk(icon)
  }

  const box = ink ?? { x: 0, y: 0, w: 1, h: 1, salience: 0 }
  const image = shape / Math.max(box.w, box.h)

  return {
    width: `${image}px`,
    height: `${image}px`,
    left: `calc(${centreX}% - ${(box.x + box.w / 2) * image}px)`,
    top: `${centreY - (box.y + box.h / 2) * image}px`,
    // every logo carries the same visual weight: quiet ones (thin, grey) show
    // stronger, loud ones (large, saturated) fainter; the square root keeps the
    // correction from overshooting
    opacity: ink
      ? Math.min(0.32, Math.max(0.1, 0.2 * Math.sqrt(0.44 / Math.max(ink.salience, 0.01))))
      : 0,
    // tilted 30° anticlockwise about the shape's own centre
    transform: 'rotate(-30deg)',
    transformOrigin: `${(box.x + box.w / 2) * 100}% ${(box.y + box.h / 2) * 100}%`,
  }
}

// Live traffic per proxy or group name: the speeds of the open connections whose
// chain passes through it. Cards show it for the node they currently end at.
const traffic = computed(() => {
  const result: Record<string, { up: number; down: number }> = {}

  activeConnections.value.forEach((connection) => {
    connection.chains.forEach((hop) => {
      const entry = (result[hop] ??= { up: 0, down: 0 })

      entry.up += connection.uploadSpeed ?? 0
      entry.down += connection.downloadSpeed ?? 0
    })
  })

  return result
})

const columns = computed(() => {
  const seen = new Set<string>()
  const result: string[] = []

  // Every exit any row can pick, in one shared order that lays out every row's
  // slots and resolves each exit's icon once.
  const byOptionCount = [...rows.value].sort((a, b) => b.exits.length - a.exits.length)

  byOptionCount.forEach((row) => {
    row.exits.forEach((exit) => {
      if (!seen.has(exit)) {
        seen.add(exit)
        result.push(exit)
      }
    })
  })

  return result
})

// The keypad only lists real exits; direct / reject have their own mark.
const exitColumns = computed(() => columns.value.filter((column) => !verdictOf(column)))

// A row's exits as shown: the chosen one first, the rest in the global order.
// A row on direct / reject leads with that verdict's badge, and the exit it would
// go back to opens the row below.
const exitOrder = (row: MatrixRow) => {
  const own = exitColumns.value.filter((column) => row.exits.includes(column))

  if (row.verdict) {
    const previous = lastExits.value[row.name] && exitOf(lastExits.value[row.name])
    const back = previous && own.includes(previous) ? previous : own[0]
    const rest = back ? [back, ...own.filter((column) => column !== back)] : own

    return [row.nowExit, ...rest]
  }

  const lead = own.includes(row.nowExit) ? row.nowExit : own[0]

  return lead ? [lead, ...own.filter((column) => column !== lead)] : own
}

// Exit groups without a configured icon fall back to a flag: first by the
// region named in the group name, then by the flag most of its nodes carry.
// Groups spanning several regions get a globe.
const MIXED = 'mixed'
const FLAG_PATTERN = /[\u{1F1E6}-\u{1F1FF}]{2}/u
const FLAG_ALIASES: Record<string, string> = { '🇺🇲': '🇺🇸' }
const REGION_FLAGS: [RegExp, string][] = [
  [/香港|港|\bHK\b|Hong ?Kong/i, '🇭🇰'],
  [/台湾|台灣|\bTW\b|Taiwan/i, '🇹🇼'],
  [/日本|\bJP\b|Japan/i, '🇯🇵'],
  [/新加坡|狮城|\bSG\b|Singapore/i, '🇸🇬'],
  [/美国|美國|\bUS\b|United States|America/i, '🇺🇸'],
  [/韩国|韓國|\bKR\b|Korea/i, '🇰🇷'],
  [/英国|英國|\bUK\b|\bGB\b|United Kingdom/i, '🇬🇧'],
  [/德国|德國|\bDE\b|Germany/i, '🇩🇪'],
]

const getRegionFlag = (name: string) => REGION_FLAGS.find(([pattern]) => pattern.test(name))?.[1]

// Grayscale flags are hard to tell apart at icon size, so the common regions
// are drawn as their single most distinctive feature instead.
const FLAG_GLYPHS: Record<string, string> = {
  '🇭🇰': 'hk',
  '🇹🇼': 'tw',
  '🇯🇵': 'jp',
  '🇸🇬': 'sg',
  '🇺🇸': 'us',
}
const DEDICATED_LINE = /专线|專線|IPLC|IEPL|dedicated/i
// A globe reads as "abroad", not "every node": groups over all nodes say ALL.
const EVERY_NODE = /^(所有|全部)|\bALL\b/i

const columnGlyphs = computed(() => {
  const result: Record<string, string> = {}

  columns.value.forEach((column) => {
    const type = typeOf(column)
    const flag = columnFlags.value[column]

    if (type === PROXY_TYPE.Direct) {
      result[column] = 'direct'
    } else if (type === PROXY_TYPE.Reject || type === PROXY_TYPE.RejectDrop) {
      result[column] = 'reject'
    } else if (DEDICATED_LINE.test(column)) {
      result[column] = 'line'
    } else if (flag === MIXED) {
      result[column] = EVERY_NODE.test(column) ? 'all' : 'globe'
    } else if (flag && FLAG_GLYPHS[flag]) {
      result[column] = FLAG_GLYPHS[flag]
    }
  })

  return result
})

const columnFlags = computed(() => {
  const result: Record<string, string> = {}

  columns.value.forEach((column) => {
    if (!proxyMap.value[column]?.all?.length) {
      return
    }

    const named = getRegionFlag(column)

    if (named) {
      result[column] = named
      return
    }

    const nodes = getDescendantProxyNames(column).filter(
      (name) => !proxyMap.value[name]?.all?.length && !TERMINAL_TYPES.has(typeOf(name)),
    )
    const counts = new Map<string, number>()

    nodes.forEach((name) => {
      const raw = name.match(FLAG_PATTERN)?.[0]
      const flag = raw ? (FLAG_ALIASES[raw] ?? raw) : undefined

      if (flag) {
        counts.set(flag, (counts.get(flag) ?? 0) + 1)
      }
    })

    const [top] = [...counts.entries()].sort((a, b) => b[1] - a[1])

    if (top && top[1] / nodes.length >= 0.6) {
      result[column] = top[0]
    } else if (counts.size > 1) {
      result[column] = MIXED
    }
  })

  return result
})

// Unselected exits keep a tint of their colours but dim with their distance from the chosen
// one, so the eye lands on the choice first and reads the rest as weaker options.
// Brightness rather than opacity, so the backdrop never shows through them.
const FADE_NEAREST = 0.7
const FADE_STEP = 0.07
const FADE_MIN = 0.4
// Brightness alone cannot tame a saturated badge (a red one still jumps out), so
// colour drains with distance too, leaving only a tint on the far ones.
const SATURATION_NEAREST = 0.65
const SATURATION_STEP = 0.1
const SATURATION_MIN = 0.25

// Position counted from the lead exit (the chosen one, or the one to go back to).
const rankOf = (row: MatrixRow, column: string) => {
  return exitOrder(row).indexOf(column)
}

const slotOf = (row: MatrixRow, column: string) =>
  slotSizes.value[Math.min(rankOf(row, column), slotSizes.value.length - 1)]

// Where each exit sits inside the card body, flush left. Matches the template:
// the title row, the chosen exit's row, then the row of other exits, 10px apart.
const exitStyle = (row: MatrixRow, column: string) => {
  const mobile = isMiddleScreen.value
  const titleRow = 28
  const gap = mobile ? 4 : 6
  const rowGap = 10
  const x0 = 0
  const chosenTop = titleRow + rowGap
  const baseline = chosenTop + slotSizes.value[0] + rowGap + slotSizes.value[1]
  const rank = rankOf(row, column)
  const size = slotOf(row, column)
  let left = x0
  let top = chosenTop

  if (rank > 0) {
    for (let k = 1; k < rank; k++) {
      left += slotSizes.value[Math.min(k, slotSizes.value.length - 1)] + gap
    }
    top = baseline - size
  }

  return { left: `${left}px`, top: `${top}px`, width: `${size}px`, height: `${size}px` }
}

const saturationOf = (row: MatrixRow, column: string) => {
  const distance = rankOf(row, column)

  return String(Math.max(SATURATION_MIN, SATURATION_NEAREST - SATURATION_STEP * (distance - 1)))
}

const fadeOf = (row: MatrixRow, column: string) => {
  const distance = rankOf(row, column)

  return String(Math.max(FADE_MIN, FADE_NEAREST - FADE_STEP * (distance - 1)))
}

// The card whose drill-down panel is open, and which face of it is showing.
const expandedRow = ref('')
const DRAWER_TABS = ['nodes', 'domains'] as const
const drawerTab = ref<(typeof DRAWER_TABS)[number]>('nodes')

watch(expandedRow, () => {
  drawerTab.value = 'nodes'
})

// Domains sit to the right of nodes: going there slides in from the right and
// out to the left, coming back the other way round.
const faceEnterFrom = computed(() =>
  drawerTab.value === 'domains' ? 'opacity-0 translate-x-6' : 'opacity-0 -translate-x-6',
)
const faceLeaveTo = computed(() =>
  drawerTab.value === 'domains' ? 'opacity-0 -translate-x-6' : 'opacity-0 translate-x-6',
)

// The box holding the faces keeps the old height until the new face is in, then
// eases to the new one, so the card below never jumps.
const lockFaceHeight = (el: Element) => {
  const box = el.parentElement

  if (box) box.style.height = `${box.offsetHeight}px`
}

const growFaceHeight = (el: Element) => {
  const box = el.parentElement

  if (box) {
    requestAnimationFrame(() => {
      box.style.height = `${(el as HTMLElement).offsetHeight}px`
    })
  }
}

const releaseFaceHeight = (el: Element) => {
  const box = el.parentElement

  if (box) {
    window.setTimeout(() => {
      box.style.height = ''
    }, 300)
  }
}

const popoverRow = computed(() => rows.value.find((row) => row.name === expandedRow.value))

const popoverNodes = computed(() => {
  const row = popoverRow.value

  if (!row) {
    return { selectable: false, list: [] }
  }

  const group = proxyMap.value[row.leafGroup]
  const selectable = typeOf(row.leafGroup) === PROXY_TYPE.Selector
  const list = (group?.all ?? []).map((name) => {
    const speed = getSpeed(row.leafGroup, name)
    const latency = getLatencyByName(name, row.leafGroup)

    return {
      name,
      current: group.now === name,
      alive: isAlive(name),
      speed: speed ?? 0,
      spark: sparkOf(name, row.leafGroup),
      color: getColorForLatency(latency),
      metric: speed ? formatSpeed(speed) : latency > NOT_CONNECTED ? `${latency}ms` : '',
    }
  })

  // An automatic group explains its choice best when sorted by the measured speed.
  if (!selectable) {
    list.sort((a, b) => Number(b.alive) - Number(a.alive) || b.speed - a.speed)
  }

  return { selectable, list }
})

// Sparkline points of a node's recent latency tests, scaled to their own range.
const sparkOf = (name: string, groupName: string) => {
  const delays = (getHistoryByName(name, groupName) ?? [])
    .slice(-12)
    .map((item) => item.delay)
    .filter((delay) => delay > 0)

  if (delays.length < 2) {
    return ''
  }

  const min = Math.min(...delays)
  const span = Math.max(...delays) - min || 1

  return delays
    .map((delay, index) => {
      const x = (index / (delays.length - 1)) * 100
      const y = 18 - ((delay - min) / span) * 16

      return `${x.toFixed(1)},${y.toFixed(1)}`
    })
    .join(' ')
}

// Without an auto / manual pair the switch only shows what the group is.
const popoverMode = computed(() => {
  const row = popoverRow.value

  if (row?.mode) {
    return { active: row.mode.isAuto ? 'auto' : 'manual', switchable: true } as const
  }

  const isAuto = !!row && AUTO_GROUP_TYPES.has(typeOf(row.leafGroup))

  return { active: isAuto ? 'auto' : 'manual', switchable: false } as const
})

const closePopover = () => {
  expandedRow.value = ''
}

// Drawer animation: height from 0 to its natural height and back.
const DRAWER_EASE = 'cubic-bezier(0.2, 0.8, 0.2, 1)'

const drawerAnimate = (el: Element, from: number, to: number, done: () => void) => {
  const box = el as HTMLElement

  box.style.height = `${from}px`
  box.style.opacity = from ? '1' : '0'
  void box.offsetHeight
  box.style.transition = `height 0.3s ${DRAWER_EASE}, opacity 0.22s ease`
  box.style.height = `${to}px`
  box.style.opacity = to ? '1' : '0'

  let finished = false
  const finish = (event?: TransitionEvent) => {
    if (finished || (event && (event.target !== box || event.propertyName !== 'height'))) return
    finished = true
    box.removeEventListener('transitionend', finish)
    done()
  }

  box.addEventListener('transitionend', finish)
  // fallback for when no transitionend arrives (e.g. the height did not change)
  window.setTimeout(finish, 400)
}

const drawerEnter = (el: Element, done: () => void) =>
  drawerAnimate(el, 0, (el as HTMLElement).scrollHeight, done)

const drawerLeave = (el: Element, done: () => void) =>
  drawerAnimate(el, (el as HTMLElement).offsetHeight, 0, done)

const drawerSettle = (el: Element) => {
  const box = el as HTMLElement

  box.style.height = ''
  box.style.opacity = ''
  box.style.transition = ''
}

// A tap on the card's blank space or its chosen exit opens or closes the drill-down.
const toggleExpand = (row: MatrixRow) => {
  if (row.isDirect) return

  expandedRow.value = expandedRow.value === row.name ? '' : row.name
}

// The chosen exit toggles the drill-down; any other exit is picked.
const handlerCellClick = async (row: MatrixRow, column: string) => {
  // the lit verdict badge goes back, like its half of the verdict mark
  if (row.verdict && row.nowExit === column) {
    await handlerVerdict(row, row.verdict)
    return
  }

  if (row.nowExit === column) {
    toggleExpand(row)
    return
  }

  closePopover()
  await handlerProxySelect(row.name, optionOf(row, column))
}

// Leaving a verdict returns the row to the exit it used before, or its first exit.
const lastExits = useStorage<Record<string, string>>('config/matrix-last-exits', {})
const handlerVerdict = async (row: MatrixRow, kind: Verdict) => {
  closePopover()

  if (row.verdict === kind) {
    const fallback = row.options.find((option) => !verdictOf(option))
    const previous = lastExits.value[row.name]
    const target = previous && row.options.includes(previous) ? previous : fallback

    if (target) {
      await handlerProxySelect(row.name, target)
    }
    return
  }

  if (!row.verdict) {
    lastExits.value = { ...lastExits.value, [row.name]: row.now }
  }

  await handlerProxySelect(row.name, row.terminals[kind]!)
}

const latencyTesting = ref(new Set<string>())

const handlerLatencyTest = async (name: string) => {
  if (latencyTesting.value.has(name)) return

  latencyTesting.value = new Set(latencyTesting.value).add(name)
  try {
    await proxyGroupLatencyTest(name)
  } finally {
    const next = new Set(latencyTesting.value)

    next.delete(name)
    latencyTesting.value = next
  }
}

const handlerModeToggle = async (row: MatrixRow) => {
  const mode = row.mode

  if (!mode) {
    return
  }

  const target = mode.isAuto ? mode.manual : mode.auto

  // the service picks the other half itself: no other service follows
  if (mode.flat) {
    await handlerProxySelect(row.name, target)
    return
  }

  const linked = rows.value
    .filter((other) => other.name !== row.name && other.now === row.now)
    .map((other) => other.name)

  await handlerProxySelect(row.now, target)

  if (linked.length) {
    showNotification({
      content: 'matrixLinkedHint',
      params: {
        exit: escapeHtml(row.now),
        groups: linked.map(escapeHtml).join('、'),
      },
      type: 'alert-info',
    })
  }
}

const handlerNodeSelect = async (node: string) => {
  const row = popoverRow.value

  if (!row || !popoverNodes.value.selectable) {
    return
  }

  await handlerProxySelect(row.leafGroup, node)
}

const escapeHtml = (value: string) =>
  value.replace(/[&<>"']/g, (char) => `&#${char.charCodeAt(0)};`)
</script>

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
            'border-base-content/10 relative flex flex-col gap-3 border-b px-1 py-3.5',
            !row.isDirect && 'cursor-pointer',
          ]"
          @click="toggleExpand(row)"
        >
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
          <div class="flex items-center gap-3">
            <!-- direct / reject are not exits to browse but a verdict on the whole
                 service, so they live on its logo: a blurred cover with one mark whose
                 halves are direct and reject. A tap on the logo opens it, a second tap
                 on a half picks; an active verdict only shows until opened -->
            <div
              :class="[
                'group/logo relative flex h-14 w-14 shrink-0 items-center justify-center max-md:h-12 max-md:w-12',
                (row.terminals.direct || row.terminals.reject) && 'cursor-pointer',
              ]"
              :title="row.name"
              @click.stop="toggleLogo(row)"
            >
              <ProxyIcon
                v-if="proxyMap[row.name]?.icon"
                :icon="proxyMap[row.name].icon"
                :size="logoSize"
                :margin="0"
              />
              <span
                v-else
                class="truncate text-xs font-medium"
                >{{ row.name }}</span
              >
              <div
                v-if="row.terminals.direct || row.terminals.reject"
                :class="[
                  'bg-base-100/40 absolute -inset-1 flex items-center justify-center rounded-lg backdrop-blur-[3px] transition-opacity duration-150',
                  row.verdict || openLogo === row.name
                    ? 'opacity-100'
                    : 'pointer-events-none opacity-0',
                ]"
              >
                <VerdictGlyph
                  class="aspect-square w-full"
                  :active="row.verdict"
                  :available="row.terminals"
                  :interactive="openLogo === row.name"
                  @pick="(kind) => handlerVerdict(row, kind)"
                />
              </div>
            </div>
            <!-- the group's name as the card title, then the node actually in use and
                 the groups it was reached through;
                 both lines always render so the card height never changes -->
            <div class="flex min-w-0 flex-1 flex-col gap-1 leading-tight">
              <span class="text-base-content truncate pr-12 text-xl font-semibold max-md:text-lg">{{
                row.name
              }}</span>
              <span class="text-base-content truncate text-base font-medium max-md:text-[15px]">{{
                row.leaf
              }}</span>
              <span class="text-base-content/40 h-4 truncate text-xs">{{
                row.via.join(' › ')
              }}</span>
            </div>
            <!-- upload faint and small above, download stronger and larger below.
                 Every reading has the same shape, four characters and a k / M / G
                 unit, so arrows, digits and units hold their places as it moves -->
            <div
              class="grid shrink-0 grid-cols-[auto_auto_auto] items-baseline gap-x-1.5 self-end font-mono leading-tight tabular-nums"
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
          <!-- one row of exits lined up under the text, the chosen one always first
               and grown; the others keep the global exit order behind it. Switching
               slides the new choice to the front. -->
          <TransitionGroup
            tag="div"
            name="exit"
            class="flex items-end gap-2 pl-17 max-md:gap-1.5 max-md:pl-15"
          >
            <button
              v-for="column in exitOrder(row)"
              :key="column"
              type="button"
              :class="[
                'group/cell relative flex shrink-0 cursor-pointer items-center justify-center rounded-full transition-[width,height,box-shadow] duration-300',
                row.now === column && 'ring-primary ring-offset-base-100 ring-2 ring-offset-2',
              ]"
              :style="{ width: `${slotOf(row, column)}px`, height: `${slotOf(row, column)}px` }"
              :aria-label="`${row.name} → ${column}`"
              :aria-pressed="row.now === column"
              :aria-expanded="row.now === column ? expandedRow === row.name : undefined"
              @click.stop="handlerCellClick(row, column)"
            >
              <!-- drawn at the large size and scaled down, so a size change animates -->
              <span
                :class="[
                  'absolute flex items-center justify-center transition-[opacity,filter,scale] duration-300',
                  row.now === column
                    ? 'opacity-100'
                    : 'opacity-(--fade) brightness-75 contrast-75 grayscale group-hover/cell:opacity-80 group-hover/cell:brightness-100',
                ]"
                :style="{
                  '--fade': fadeOf(row, column),
                  width: `${slotSizes[0]}px`,
                  height: `${slotSizes[0]}px`,
                  scale: slotOf(row, column) / slotSizes[0],
                }"
                :title="column"
              >
                <ProxyRegionGlyph
                  v-if="columnGlyphs[column]"
                  :glyph="columnGlyphs[column]"
                  :size="badgeSize"
                />
                <ProxyIcon
                  v-else-if="proxyMap[column]?.icon"
                  :icon="proxyMap[column].icon"
                  :size="exitSize"
                  :margin="0"
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
          </TransitionGroup>
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
                class="border-base-content/15 relative mt-4 flex gap-3 rounded-lg border p-3"
                @click.stop
              >
                <span
                  class="bg-base-100 border-base-content/15 absolute -top-1.5 left-[90px] h-3 w-3 -translate-x-1/2 rotate-45 border-t border-l max-md:left-[80px]"
                />
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
                    :disabled="!popoverNodes.selectable || !node.alive"
                    :title="node.name"
                    :class="[
                      'relative flex min-w-0 flex-col gap-1 overflow-hidden rounded-md border px-2 pt-1.5 pb-2 text-left',
                      node.current ? 'border-primary' : 'border-base-content/10',
                      popoverNodes.selectable && node.alive && !node.current
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
import { useElementSize, useEventListener, useStorage } from '@vueuse/core'
import { computed, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import LatencyTag from './LatencyTag.vue'
import MatrixModeSwitch from './MatrixModeSwitch.vue'
import ProxyIcon from './ProxyIcon.vue'
import ProxyRegionGlyph from './ProxyRegionGlyph.vue'
import VerdictGlyph from './VerdictGlyph.vue'

type ModePair = {
  auto: string
  manual: string
  isAuto: boolean
}

type MatrixRow = {
  name: string
  now: string
  options: string[]
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
const logoSize = computed(() => (isMiddleScreen.value ? 48 : 56))
// Exits step down in size along the row, all standing on one baseline: the chosen
// one large, the next medium, the rest small. Badges are drawn at the large size.
const slotSizes = computed(() => (isMiddleScreen.value ? [40, 32, 24] : [44, 34, 26]))
const badgeSize = computed(() => slotSizes.value[0])
const exitSize = computed(() => Math.round(slotSizes.value[0] * 0.58))

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

  return { auto, manual, isAuto: group.now === auto }
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

      return {
        name,
        now: group.now,
        options: group.all ?? [],
        isDirect,
        leaf,
        leafGroup,
        via: chain.filter((hop) => hop !== name && hop !== leaf),
        terminals: getTerminals(group.all ?? []),
        verdict: verdictOf(group.now),
        mode: getModePair(group.now),
        speedText,
        latencyText,
        latencyClass: getColorForLatency(latency),
      }
    })
})

// Rows keep the configured order: switching a row's exit must never move it.
const orderedRows = rows

// Each card: logo, the node in use and the route to it, live traffic on the right,
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
  const byOptionCount = [...rows.value].sort(
    (a, b) => (proxyMap.value[b.name].all?.length ?? 0) - (proxyMap.value[a.name].all?.length ?? 0),
  )

  byOptionCount.forEach((row) => {
    proxyMap.value[row.name].all?.forEach((option) => {
      if (!seen.has(option)) {
        seen.add(option)
        result.push(option)
      }
    })
  })

  return result
})

// The keypad only lists real exits; direct / reject are picked on the logo.
const exitColumns = computed(() => columns.value.filter((column) => !verdictOf(column)))

// A row's exits as shown: the chosen one first, the rest in the global order.
const exitOrder = (row: MatrixRow) => {
  const rest = exitColumns.value.filter(
    (column) => column !== row.now && row.options.includes(column),
  )

  return exitColumns.value.includes(row.now) ? [row.now, ...rest] : rest
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
      result[column] = 'globe'
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

// Unselected exits fade with their distance from the selected one, so the eye
// lands on the choice first and reads its neighbours as weaker alternatives.
const FADE_NEAREST = 0.6
const FADE_STEP = 0.06
const FADE_MIN = 0.22

// Position counted from the chosen exit; rows on direct / reject have none in front.
const rankOf = (row: MatrixRow, column: string) => {
  const order = exitOrder(row)
  const index = order.indexOf(column)

  return order[0] === row.now ? index : index + 1
}

const slotOf = (row: MatrixRow, column: string) =>
  slotSizes.value[Math.min(rankOf(row, column), slotSizes.value.length - 1)]

const fadeOf = (row: MatrixRow, column: string) => {
  const distance = rankOf(row, column)

  return String(Math.max(FADE_MIN, FADE_NEAREST - FADE_STEP * (distance - 1)))
}

// The card whose drill-down panel is open.
const expandedRow = ref('')

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
  if (row.now === column) {
    toggleExpand(row)
    return
  }

  closePopover()
  await handlerProxySelect(row.name, column)
}

// Leaving a verdict returns the row to the exit it used before, or its first exit.
const lastExits = useStorage<Record<string, string>>('config/matrix-last-exits', {})
const openLogo = ref('')

// a tap anywhere else puts an opened logo cover away again
useEventListener(document, 'click', () => {
  openLogo.value = ''
})

const toggleLogo = (row: MatrixRow) => {
  openLogo.value = openLogo.value === row.name ? '' : row.name
}

const handlerVerdict = async (row: MatrixRow, kind: Verdict) => {
  openLogo.value = ''
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

<style scoped>
.exit-move {
  transition: transform 0.36s cubic-bezier(0.2, 0.8, 0.2, 1);
}
</style>

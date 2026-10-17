<template>
  <!-- Domain penetration for one group, shown inside the matrix drawer instead of
       a dialog: what the group's rules send through it, one line per rule. -->
  <div class="flex min-w-0 flex-1 flex-col gap-2">
    <div class="flex min-w-0 items-center gap-3">
      <div class="flex shrink-0 items-baseline gap-3 text-xs">
        <button
          v-for="family in families"
          :key="family.value"
          type="button"
          :class="[
            'flex cursor-pointer items-baseline gap-1 transition-colors',
            proxyGroupRulePenetrationDialogTab === family.value
              ? 'text-base-content'
              : 'text-base-content/40 hover:text-base-content/70',
          ]"
          @click="proxyGroupRulePenetrationDialogTab = family.value"
        >
          {{ family.label }}
          <span class="font-mono text-[10px] tabular-nums opacity-70">{{ family.count }}</span>
        </button>
      </div>
      <input
        v-model="search"
        type="search"
        class="border-base-content/10 focus:border-base-content/30 placeholder:text-base-content/30 ml-auto h-7 w-40 min-w-0 rounded-md border bg-transparent px-2 text-xs outline-none max-md:w-28"
        :placeholder="$t('domainPenetrationSearchPlaceholder')"
      />
    </div>

    <!-- fixed height, so switching filters or loading never moves the card -->
    <div
      ref="listRef"
      class="border-base-content/10 h-64 overflow-y-auto overscroll-contain rounded-md border"
      @scroll.passive="handleScroll"
    >
      <div
        v-if="state"
        class="text-base-content/40 flex h-full items-center justify-center px-4 text-center text-xs"
      >
        {{ state }}
      </div>
      <template v-else>
        <button
          v-for="(item, index) in proxyGroupRulePenetrationDialogEntries"
          :key="`${item.raw}-${index}`"
          type="button"
          :title="item.raw"
          class="hover:bg-base-content/5 flex h-8 w-full min-w-0 cursor-copy items-center gap-3 px-2.5 text-left"
          @click="copy(item.raw)"
        >
          <span class="text-base-content/90 min-w-0 flex-1 truncate font-mono text-xs">
            {{ item.content || item.raw }}
          </span>
          <span class="text-base-content/35 shrink-0 text-[11px]">{{ typeLabel(item.type) }}</span>
        </button>
        <div
          v-if="proxyGroupRulePenetrationDialogLoadingMore"
          class="text-base-content/40 py-2 text-center text-xs"
        >
          {{ $t('domainPenetrationLoading') }}
        </div>
      </template>
    </div>
  </div>
</template>

<script setup lang="ts">
import { showNotification } from '@/helper/notification'
import {
  isProxyGroupRuleCacheEmpty,
  loadProxyGroupRulePenetration,
  proxyGroupRulePenetrationDialogCounts,
  proxyGroupRulePenetrationDialogEntries,
  proxyGroupRulePenetrationDialogError,
  proxyGroupRulePenetrationDialogGroupName,
  proxyGroupRulePenetrationDialogHasMore,
  proxyGroupRulePenetrationDialogLoading,
  proxyGroupRulePenetrationDialogLoadingMore,
  proxyGroupRulePenetrationDialogMode,
  proxyGroupRulePenetrationDialogSearch,
  proxyGroupRulePenetrationDialogSortDirection,
  proxyGroupRulePenetrationDialogSortKey,
  proxyGroupRulePenetrationDialogTab,
} from '@/store/proxyGroupRulePenetration'
import { computed, onBeforeUnmount, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'

const props = defineProps<{
  groupName: string
}>()

const { t } = useI18n()
const listRef = ref<HTMLElement>()
const search = ref('')
let searchTimer = 0

// Only families that have rules are offered; "all" is always there.
const families = computed(() => {
  const counts = proxyGroupRulePenetrationDialogCounts.value

  return [
    { value: 'all' as const, label: t('all'), count: counts.all },
    { value: 'domain' as const, label: t('domain'), count: counts.domain },
    { value: 'ip' as const, label: t('ip'), count: counts.ip },
    { value: 'port' as const, label: t('port'), count: counts.port },
  ].filter((family) => family.value === 'all' || family.count > 0)
})

const TYPE_LABEL_KEYS: Record<string, string> = {
  DOMAIN: 'ruleTypeDomain',
  'DOMAIN-SUFFIX': 'ruleTypeDomainSuffix',
  'DOMAIN-KEYWORD': 'ruleTypeDomainKeyword',
  'IP-CIDR': 'ruleTypeDestinationIP',
  'IP-CIDR6': 'ruleTypeDestinationIP',
  'SRC-IP': 'ruleTypeSourceIP',
  'SRC-IP-CIDR': 'ruleTypeSourceIP',
  'SRC-IP-CIDR6': 'ruleTypeSourceIP',
  'DST-PORT': 'ruleTypeDestinationPort',
  'SRC-PORT': 'ruleTypeSourcePort',
  'IN-PORT': 'ruleTypeInboundPort',
  GEOIP: 'ruleTypeGeoIP',
  MATCH: 'ruleTypeMatch',
  FINAL: 'ruleTypeFinal',
}

const typeLabel = (type: string) => t(TYPE_LABEL_KEYS[type] || 'ruleTypeOther')

// One message in place of the list while there is nothing to list.
const state = computed(() => {
  if (proxyGroupRulePenetrationDialogLoading.value) return t('domainPenetrationLoading')
  if (proxyGroupRulePenetrationDialogError.value) return proxyGroupRulePenetrationDialogError.value
  if (proxyGroupRulePenetrationDialogEntries.value.length) return ''
  if (isProxyGroupRuleCacheEmpty()) return t('domainPenetrationCacheEmptyHint')
  return t('domainPenetrationEmpty')
})

const reload = () => {
  listRef.value?.scrollTo({ top: 0 })
  void loadProxyGroupRulePenetration(props.groupName)
}

// The rule penetration store is shared with the policy cards' dialog; only one
// matrix drawer is open at a time, so it is pointed at this group while shown.
watch(
  () => props.groupName,
  (name) => {
    proxyGroupRulePenetrationDialogMode.value = 'proxy-group'
    proxyGroupRulePenetrationDialogGroupName.value = name
    proxyGroupRulePenetrationDialogTab.value = 'all'
    proxyGroupRulePenetrationDialogSortKey.value = null
    proxyGroupRulePenetrationDialogSortDirection.value = 'asc'
    proxyGroupRulePenetrationDialogSearch.value = ''
    search.value = ''
    reload()
  },
  { immediate: true },
)

watch(search, (value) => {
  window.clearTimeout(searchTimer)
  searchTimer = window.setTimeout(() => {
    proxyGroupRulePenetrationDialogSearch.value = value
    reload()
  }, 180)
})

watch(proxyGroupRulePenetrationDialogTab, reload)

const handleScroll = () => {
  const list = listRef.value

  if (
    !list ||
    !proxyGroupRulePenetrationDialogHasMore.value ||
    proxyGroupRulePenetrationDialogLoading.value ||
    proxyGroupRulePenetrationDialogLoadingMore.value
  ) {
    return
  }

  if (list.scrollTop + list.clientHeight >= list.scrollHeight - 120) {
    void loadProxyGroupRulePenetration(props.groupName, { append: true })
  }
}

const copy = async (text: string) => {
  try {
    await navigator.clipboard.writeText(text)
    showNotification({ content: 'copySuccess', type: 'alert-success' })
  } catch {
    // clipboard unavailable (insecure context): the full rule stays in the tooltip
  }
}

onBeforeUnmount(() => window.clearTimeout(searchTimer))
</script>

<script setup lang="ts">
import { computed, ref, watchEffect } from 'vue'
import metricsIndex from './data/index.json'

type MetricBase = {
  Category: string
  SubCategory?: string | null
  MetricTitle: string
  MetricDescription: string
  ReportPeriod?: string | null
  Target?: string | null
  Comment?: string | null
  Contributor?: string | null
  Source?: string | null
}

type Metric = MetricBase & { id: string }

type CategoryMeta = {
  category: string
  slug: string
  file: string
  count: number
}

type CategoryEntry = CategoryMeta & { items: Metric[] }

const modules = import.meta.glob<{ default: MetricBase[] }>('./data/metrics/*.json', {
  eager: true
})

const hydrateMetric = (metric: MetricBase, slug: string, index: number): Metric => ({
  id: `${slug}-${index}`,
  Category: metric.Category ?? '',
  SubCategory: metric.SubCategory ?? null,
  MetricTitle: metric.MetricTitle ?? '',
  MetricDescription: metric.MetricDescription ?? '',
  ReportPeriod: metric.ReportPeriod ?? '',
  Target: metric.Target ?? '',
  Comment: metric.Comment ?? '',
  Contributor: metric.Contributor ?? '',
  Source: metric.Source ?? ''
})

const initialCategories = (metricsIndex as CategoryMeta[]).map((meta) => {
  const moduleKey = `./data/metrics/${meta.slug}.json`
  const module = modules[moduleKey]
  const rawItems = module?.default ?? []
  const items = rawItems.map((item, itemIndex) => hydrateMetric(item, meta.slug, itemIndex))
  return {
    ...meta,
    items,
    count: items.length
  }
})

const categories = ref<CategoryEntry[]>(initialCategories)
const searchQuery = ref('')
const selectedCategory = ref(categories.value[0]?.category ?? '')
const isSidebarOpen = ref(false)

const searchableFields: Array<keyof Metric> = [
  'MetricTitle',
  'MetricDescription',
  'Category',
  'SubCategory',
  'ReportPeriod',
  'Target',
  'Comment',
  'Contributor',
  'Source'
]

const toComparable = (input: unknown) =>
  typeof input === 'string' ? input.toLowerCase() : input != null ? String(input).toLowerCase() : ''

const matchesSearch = (metric: Metric, query: string) => {
  if (!query) return true
  const normalizedQuery = query.toLowerCase()
  return searchableFields.some((field) => toComparable(metric[field]).includes(normalizedQuery))
}

const allMetrics = computed(() => categories.value.flatMap((category) => category.items))

const filteredCategories = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  return categories.value
    .map((category) => {
      const matchCount = query
        ? category.items.filter((metric) => matchesSearch(metric, query)).length
        : category.count
      return {
        ...category,
        matchCount
      }
    })
    .filter((category) => (query ? category.matchCount > 0 : category.count > 0))
})

watchEffect(() => {
  if (!filteredCategories.value.some((category) => category.category === selectedCategory.value)) {
    selectedCategory.value = filteredCategories.value[0]?.category ?? ''
  }
})

const displayMetrics = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  if (query) {
    return allMetrics.value.filter((metric) => matchesSearch(metric, query))
  }
  return (
    categories.value.find((category) => category.category === selectedCategory.value)?.items ?? []
  )
})

const activeCategorySummary = computed(() =>
  !searchQuery.value.trim()
    ? filteredCategories.value.find((category) => category.category === selectedCategory.value)
    : null
)

const activeCategoryCount = computed(() => activeCategorySummary.value?.count ?? 0)

const iconFieldMap: Record<string, string> = {
  ReportPeriod: 'repeat',
  Target: 'bullseye',
  Comment: 'comment',
  Source: 'arrow-up-right-from-square',
  Contributor: 'user'
}

const detailFields: ReadonlyArray<keyof Metric> = [
  'ReportPeriod',
  'Target',
  'Comment',
  'Contributor',
  'Source'
]

type MetricDetail = {
  field: string
  value: string
  icon: string | null
}

const metricDetails = (metric: Metric): MetricDetail[] =>
  detailFields.reduce<MetricDetail[]>((details, field) => {
    const raw = metric[field]
    const value = typeof raw === 'string' ? raw.trim() : raw != null ? String(raw) : ''
    if (!value || value === '-' || value.toLowerCase() === 'null') return details
    details.push({
      field: field.toString(),
      value,
      icon: iconFieldMap[field as string] ?? null
    })
    return details
  }, [])

const clearSearch = () => {
  searchQuery.value = ''
}

const toggleSidebar = () => {
  isSidebarOpen.value = !isSidebarOpen.value
}

const closeSidebar = () => {
  isSidebarOpen.value = false
}

const selectCategory = (category: string) => {
  selectedCategory.value = category
  closeSidebar()
}
</script>

<template>
  <div class="min-h-screen bg-slate-950 text-slate-100 flex">
    <transition name="fade">
      <div
        v-if="isSidebarOpen"
        class="fixed inset-0 z-30 bg-slate-900/70 backdrop-blur-sm lg:hidden"
        @click="closeSidebar"
      />
    </transition>

    <aside
      class="fixed inset-y-0 left-0 z-40 w-72 transform border-r border-slate-800 bg-slate-900/60 backdrop-blur transition-transform duration-300 lg:static lg:translate-x-0"
      :class="isSidebarOpen ? 'translate-x-0' : '-translate-x-full lg:translate-x-0'"
    >
      <div class="flex h-full flex-col overflow-y-auto">
        <div class="px-6 py-6">
          <h2 class="text-lg font-semibold tracking-tight text-white">KPI Matrix</h2>
          <p class="mt-2 text-sm text-slate-400">Curated cybersecurity metrics by focus area.</p>
          <a
            class="mt-4 inline-flex gap-2 text-sm font-semibold text-white hover:text-emerald-300"
            href="https://github.com/lavenix-com/sec-kpi-metrics"
          >
            <font-awesome-icon :icon="['fab', 'github']" class="text-lg" />
            View on GitHub
          </a>
        </div>
        <nav class="flex-1 space-y-1 px-4 pb-6">
          <button
            v-for="category in filteredCategories"
            :key="category.slug"
            class="flex w-full items-center justify-between rounded-xl px-4 py-3 text-left text-sm font-medium transition"
            :class="
              category.category === selectedCategory && !searchQuery
                ? 'bg-emerald-500/20 text-emerald-300 ring-1 ring-emerald-400/60'
                : 'text-slate-300 hover:bg-white/5 hover:text-white'
            "
            @click="selectCategory(category.category)"
          >
            <span class="truncate">{{ category.category }}</span>
            <span
              class="ml-3 inline-flex min-w-[2rem] items-center justify-center rounded-full bg-slate-800 px-2 py-0.5 text-xs text-slate-300"
            >
              {{ searchQuery ? category.matchCount : category.count }}
            </span>
          </button>
        </nav>
        <div class="border-t border-slate-800 px-6 py-5">
          <p class="mt-2 text-sm text-slate-400 opacity-50 mb-10">
            An open-source catalog of metrics and insights curated by Lavenix to help teams track
            and communicate cyber performance.
          </p>
          <p class="text-xs uppercase tracking-[0.2em] text-slate-500">Powered by</p>
          <a
            class="mt-2 inline-flex items-center gap-2 text-sm font-semibold text-white hover:text-emerald-300"
            href="https://lavenix.com"
            target="_blank"
            rel="noopener"
          >
            <font-awesome-icon :icon="['fas', 'arrow-up-right-from-square']" class="text-xs" />
            Lavenix
          </a>
        </div>
      </div>
    </aside>

    <main class="relative flex min-h-screen flex-1 flex-col">
      <header class="relative overflow-hidden px-6 pb-12 pt-10 sm:px-10 lg:px-14">
        <div
          class="absolute inset-0 -z-10 bg-gradient-to-br from-emerald-500/20 via-slate-900/40 to-slate-950"
        />
        <div class="flex items-start justify-between gap-4">
          <div>
            <button
              class="inline-flex items-center gap-2 rounded-full bg-white/10 px-4 py-2 text-xs font-medium uppercase tracking-[0.2em] text-emerald-200 ring-1 ring-inset ring-white/20 hover:bg-white/15 lg:hidden"
              @click="toggleSidebar"
            >
              <font-awesome-icon :icon="['fas', 'layer-group']" />
              Browse Categories
            </button>
          </div>
        </div>

        <div class="flex flex-col gap-4 sm:flex-row sm:items-center">
          <div class="relative w-full sm:max-w-lg">
            <font-awesome-icon
              :icon="['fas', 'magnifying-glass']"
              class="pointer-events-none absolute left-4 top-1/2 -translate-y-1/2 text-slate-400"
            />
            <input
              v-model.trim="searchQuery"
              type="search"
              placeholder="Search metrics by keyword, goal, or source…"
              class="w-full rounded-full border border-white/10 bg-slate-900/70 py-3 pl-11 pr-14 text-sm text-white placeholder:text-slate-400 focus:border-emerald-400 focus:outline-none focus:ring-2 focus:ring-emerald-400/40"
            />
            <button
              v-if="searchQuery"
              class="absolute right-3 top-1/2 -translate-y-1/2 rounded-full bg-white/10 px-3 py-1 text-xs font-medium text-slate-200 hover:bg-white/20"
              @click="clearSearch"
            >
              Clear
            </button>
          </div>

          <div class="w-full sm:w-auto lg:hidden">
            <label class="block text-xs font-semibold uppercase tracking-[0.2em] text-slate-400">
              Focus Area
            </label>
            <select
              v-model="selectedCategory"
              class="mt-2 w-full rounded-xl border border-white/10 bg-slate-900/70 px-4 py-3 text-sm text-white focus:border-emerald-400 focus:outline-none focus:ring-2 focus:ring-emerald-400/40"
            >
              <option
                v-for="category in filteredCategories"
                :key="category.slug"
                :value="category.category"
              >
                {{ category.category }} ({{ searchQuery ? category.matchCount : category.count }})
              </option>
            </select>
          </div>
        </div>
      </header>

      <section class="flex-1 px-6 pb-16 sm:px-10 lg:px-14">
        <div class="mb-8 flex flex-wrap items-center justify-between gap-3 text-sm text-slate-300">
          <div v-if="searchQuery">
            <span class="font-semibold text-emerald-200">{{ displayMetrics.length }}</span>
            results for
            <span class="font-semibold text-white">“{{ searchQuery }}”</span>
          </div>
          <div v-else class="flex items-center gap-2">
            <span
              class="rounded-full bg-emerald-500/20 px-3 py-1 text-xs font-semibold uppercase tracking-[0.2em] text-emerald-200"
            >
              {{ selectedCategory }}
            </span>
            <span class="text-slate-400"> {{ activeCategoryCount }} metrics available </span>
          </div>
          <button
            v-if="!searchQuery"
            class="ml-auto inline-flex items-center gap-2 text-xs font-semibold uppercase tracking-[0.2em] text-slate-400 hover:text-emerald-200 lg:hidden"
            @click="toggleSidebar"
          >
            <font-awesome-icon :icon="['fas', 'layer-group']" />
            Categories
          </button>
        </div>

        <div
          v-if="!displayMetrics.length"
          class="rounded-2xl border border-dashed border-slate-700 bg-slate-900/70 p-12 text-center text-slate-400"
        >
          <font-awesome-icon :icon="['fas', 'circle-info']" class="text-3xl text-slate-500" />
          <p class="mt-4 text-lg font-semibold text-white">No metrics found</p>
          <p class="mt-2 text-sm">
            Try a different search term or switch focus areas to explore more metrics.
          </p>
        </div>

        <div v-else class="grid grid-cols-1 gap-6 md:grid-cols-2 xl:grid-cols-3">
          <article
            v-for="metric in displayMetrics"
            :key="metric.id"
            class="group relative overflow-hidden rounded-2xl border border-slate-800 bg-slate-900/70 p-6 shadow-lg transition hover:-translate-y-1 hover:border-emerald-500/60 hover:shadow-emerald-500/20"
          >
            <div class="flex items-start justify-between gap-4">
              <div class="flex flex-col">
                <span class="text-xs font-semibold uppercase tracking-[0.2em] text-emerald-200">
                  {{ metric.Category }}
                </span>
                <h2 class="mt-3 text-xl font-semibold text-white">
                  {{ metric.MetricTitle }}
                </h2>
                <span
                  v-if="metric.SubCategory"
                  class="mt-2 inline-flex w-fit items-center gap-2 rounded-full bg-white/10 px-3 py-1 text-xs font-medium text-slate-200"
                >
                  <font-awesome-icon :icon="['fas', 'tag']" class="text-slate-300" />
                  {{ metric.SubCategory }}
                </span>
              </div>
              <font-awesome-icon
                :icon="['fas', 'chart-simple']"
                class="text-lg text-emerald-300/80"
              />
            </div>
            <p class="mt-4 text-sm leading-relaxed text-slate-200">
              {{ metric.MetricDescription }}
            </p>
            <ul class="mt-5 space-y-2 text-sm text-slate-300">
              <li
                v-for="detail in metricDetails(metric)"
                :key="detail.field"
                class="flex gap-3 rounded-xl bg-white/5 px-3 py-2"
              >
                <font-awesome-icon
                  v-if="detail.icon"
                  :icon="['fas', detail.icon]"
                  class="mt-0.5 text-emerald-300/80"
                />
                <div>
                  <p class="text-xs font-semibold uppercase tracking-[0.2em] text-slate-400">
                    {{ detail.field }}
                  </p>
                  <p class="mt-1 whitespace-pre-line text-slate-200">
                    {{ detail.value }}
                  </p>
                </div>
              </li>
            </ul>
          </article>
        </div>
      </section>
    </main>
  </div>
</template>

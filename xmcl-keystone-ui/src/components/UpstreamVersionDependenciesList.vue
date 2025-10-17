<template>
  <div class="flex flex-col gap-3">
    <div class="font-semibold text-gray-700 dark:text-gray-300">
      {{ t('modrinth.dependencies') || 'Dependencies' }}
    </div>
    <div
      ref="scrollElement"
      class="overflow-auto max-h-96"
    >
      <v-list
        ref="containerRef"
        class="overflow-auto"
        color="transparent"
        :style="{
          height: `${totalHeight}px`,
          position: 'relative',
          width: '100%',
          marginTop: `${-offsetTop}px`,
        }"
      >
        <div
          v-for="row of virtualRows"
          :key="getKey(row.index)"
          :ref="measureElement"
          :data-index="row.index"
          :style="{
            position: 'absolute',
            top: 0,
            left: 0,
            width: '100%',
            transform: `translateY(${row.start}px)`
          }"
        >
          <v-list-item
            v-if="dependencies[row.index]"
            :href="dependencies[row.index].href"
            :target="dependencies[row.index].href ? '_blank' : undefined"
          >
            <v-list-item-avatar
              v-if="dependencies[row.index].icon"
              tile
            >
              <img
                :src="dependencies[row.index].icon"
                :alt="dependencies[row.index].name"
              >
            </v-list-item-avatar>
            <v-list-item-content>
              <v-list-item-title>
                {{ dependencies[row.index].name }}
              </v-list-item-title>
              <v-list-item-subtitle>
                {{ dependencies[row.index].type }}
              </v-list-item-subtitle>
            </v-list-item-content>
            <v-list-item-action>
              <v-chip
                :color="getDependencyColor(dependencies[row.index].type)"
                text-color="white"
                small
              >
                {{ dependencies[row.index].type }}
              </v-chip>
            </v-list-item-action>
          </v-list-item>
        </div>
      </v-list>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { useVirtualizer, VirtualItem, VirtualizerOptions } from '@tanstack/vue-virtual'
import { VueInstance } from '@vueuse/core'
import { computed, nextTick, ref, watch } from 'vue'
import { getEl } from '@/util/el'

export interface DependencyItem {
  name: string
  type: string
  icon?: string
  href?: string
}

const props = defineProps<{
  dependencies: DependencyItem[]
}>()

const { t } = useI18n()

function getDependencyColor(type: string): string {
  switch (type.toLowerCase()) {
    case 'required':
      return '#ef5350'
    case 'optional':
      return '#42a5f5'
    case 'incompatible':
      return '#ab47bc'
    case 'embedded':
      return '#29b6f6'
    default:
      return '#90caf9'
  }
}

function getKey(index: number): string {
  if (!props.dependencies[index]) return index.toString()
  return `${props.dependencies[index].name}-${props.dependencies[index].type}`
}

// Virtual scroll
const offsetTop = ref(0)
const containerRef = ref<HTMLElement | VueInstance | null>(null)
const scrollElement = ref<VueInstance | HTMLElement | null>(null)

watch(containerRef, container => {
  if (container) {
    nextTick().then(() => { offsetTop.value = getEl(container)?.offsetTop || 0 })
  }
})

const virtualizerOptions = computed(() => ({
  count: props.dependencies.length,
  getScrollElement: () => getEl(scrollElement.value) as any,
  estimateSize: () => 62,
  overscan: 10,
  paddingStart: offsetTop.value,
} satisfies Partial<VirtualizerOptions<HTMLElement, HTMLElement>>))

const virtualizer = useVirtualizer(virtualizerOptions)
const totalHeight = computed(() => virtualizer.value.getTotalSize())
const virtualRows = computed(() => virtualizer.value.getVirtualItems() as (Omit<VirtualItem, 'key'> & { key: number })[])

const measureElement = (el: any) => {
  if (!el) return
  if ('$el' in el) {
    el = el.$el
  }
  virtualizer.value.measureElement(el)
}

watch(() => props.dependencies, () => {
  nextTick().then(() => {
    const el = getEl(containerRef.value)
    if (el) {
      virtualizer.value.scrollToIndex(0)
      offsetTop.value = 0
    }
  })
}, { deep: true })
</script>

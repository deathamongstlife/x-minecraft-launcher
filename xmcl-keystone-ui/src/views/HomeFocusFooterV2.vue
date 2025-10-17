<template>
  <div class="w-full">
    <div class="relative mx-6 flex-grow flex gap-6 items-end">
      <v-card flat class="rounded-lg overflow-hidden tabs-card" @mouseenter="onMouseEnter" @mouseleave="onMouseLeaved">
        <HomeScreenshotCard class="h-full" :height="screenshotHeight" :instance="instance" />
        <HomeModsCard @pin="onPin" @expand="onViewDashboard" />
      </v-card>
      <div
        key="launch-button-group"
        class="flex flex-wrap justify-end items-center gap-y-6 gap-x-2"
      >
        <HomeHeaderInstallStatus
          v-if="status === 1 || status === 3"
          class="mr-2"
          :name="taskName"
          :total="total"
          :progress="progress"
        />
        <HomeLaunchButtonStatus
          class="mr-4 ml-2"
          v-else
          :active="active"
        />
        <HomeLaunchButton
          class="ml-4"
          :status="status"
          @pause="pause"
          @resume="resume"
          @mouseenter="active = true"
          @mouseleave="active = false"
        />
      </div>
    </div>
  </div>
</template>
<script lang="ts" setup>
import { getCurseforgeProjectModel, useCurseforgeUpstreamHeader } from '@/composables/curseforge'
import { kInstance } from '@/composables/instance'
import { kLaunchTask } from '@/composables/launchTask'
import { useModrinthHeaderData } from '@/composables/modrinth'
import { getModrinthProjectModel } from '@/composables/modrinthProject'
import { useQueryNumber } from '@/composables/query'
import { useSWRVModel } from '@/composables/swrv'
import { kTheme } from '@/composables/theme'
import { useInFocusMode } from '@/composables/uiLayout'
import { injection } from '@/util/inject'
import { useLocalStorage, useWindowSize } from '@vueuse/core'
import HomeHeaderInstallStatus from './HomeHeaderInstallStatus.vue'
import HomeLaunchButton from './HomeLaunchButton.vue'
import HomeLaunchButtonStatus from './HomeLaunchButtonStatus.vue'
import HomeModsCard from './HomeModsCard.vue'
import HomeSavesCard from './HomeSavesCard.vue'
import HomeScreenshotCard from './HomeScreenshotCard.vue'
import HomeUpstreamHeader from './HomeUpstreamHeader.vue'

const active = ref(false)
const { path, refreshing, instance } = injection(kInstance)
const { total, progress, status, name: taskName, pause, resume } = injection(kLaunchTask)
const tab = useQueryNumber('homeTab', 0)
const { t } = useI18n()
const tabItems = ref(null as null | Vue)
const counter = ref(0)
const visible = ref(false)
const { blurCard, cardColor } = injection(kTheme)

const { data: project } = useSWRVModel(getCurseforgeProjectModel(computed(() => instance.value.upstream?.type === 'curseforge-modpack' ? Number(instance.value.upstream.modId) : undefined)))
const curseforgeHeaderData = useCurseforgeUpstreamHeader(project)
const { data: modrinthProject } = useSWRVModel(getModrinthProjectModel(computed(() => instance.value.upstream?.type === 'modrinth-modpack' ? instance.value.upstream.projectId : undefined)))
const modrinthHeaderData = useModrinthHeaderData(modrinthProject)
const headerData = computed(() => {
  const val = instance.value.upstream
  if (!val) return undefined
  if (val.type === 'curseforge-modpack') {
    return curseforgeHeaderData.value
  }
  if (val.type === 'modrinth-modpack') {
    return modrinthHeaderData.value
  }
  return undefined
})


const isFoucs = useInFocusMode()
const { height } = useWindowSize()

const screenshotHeight = computed(() => {
  return 240
})
const rowCount = computed(() => {
  const el = tabItems.value?.$el
  // oxlint-disable-next-line
  const windowHeight = height.value // keep the effect
  if (!el) {
    return 52
  }

  const rect = el.getBoundingClientRect()
  const rows = Math.floor((rect.height - 64 - 52 - 22 - 16) / 30)
  const cols = Math.floor((rect.width - 32) / 30)

  return rows * cols
})

const pinned = useLocalStorage('home-pinned', true)
function onPin() {
  pinned.value = !pinned.value
}

function onViewDashboard() {
  isFoucs.value = false
}

function onMouseLeaved() {
  counter.value -= 1
  if (counter.value <= 0) {
    counter.value = 0
    setTimeout(() => {
      visible.value = false
    }, 2000)
  }
}

function onMouseEnter() {
  counter.value += 1
}
</script>

<style scoped>
.tabs-card {
  max-width: 450px;
  min-width: 450px;
}

.tabs-card>.icons, .tabs-card>.tabs-items {
transition: all 0.2s ease-in-out;
  opacity: 0;
}

.dark .tabs-card>.icons .v-icon {
  color: var(--icon-color);
}

.dark .tabs-card>.icons .v-icon:hover {
  color: var(--icon-color-hovered);
}

.visibled {
  opacity: 1 !important;
}

.tabs {
  @apply rounded-md!;
  margin-bottom: 0.125rem;
}

.tabs-items, .tabs-items .v-window-item {
  min-height: 8rem;
  max-height: 8rem;
  height: 8rem;
}

@media (min-width: 1350px) {
  .tabs-card {
    /* max-width: max(450px, 33%);
    min-width: max(450px, 33%); */
  }
}

/* when height > 830px */
@media (min-height: 830px) {
  .tabs-items, .tabs-items .v-window-item {
    /* min-height: max(280px, 33vh);
    max-height: max(280px, 33vh);
    height: max(280px, 33vh); */
  }
}

/* when height < 600px */
@media (max-height: 600px) {
  .tabs-items, .tabs-items .v-window-item {
    /* min-height: min(280px, 46vh);
    max-height: min(280px, 46vh);
    height: min(280px, 46vh); */
  }
}

/* when height < 490px */
</style>
<style>
.tabs>div[role="tablist"] {
  background: var(--color-sidebar-bg) !important;
  backdrop-filter: blur(var(--blur-card));
}
</style>
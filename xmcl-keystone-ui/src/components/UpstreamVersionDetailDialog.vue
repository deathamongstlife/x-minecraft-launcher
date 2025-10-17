<template>
  <v-dialog
    v-model="isShown"
    :width="800"
    scrollable
  >
    <v-card class="flex flex-col overflow-hidden" style="height: 100%; max-height: 90vh">
      <v-card-title class="select-none flex-shrink-0">
        {{ versionName }}
      </v-card-title>

      <v-divider />

      <v-card-text class="flex flex-col gap-4 px-6 py-4 flex-grow overflow-y-auto min-h-0">
        <!-- Version Metadata -->
        <div class="flex flex-col gap-3">
          <!-- <div class="font-semibold text-gray-700 dark:text-gray-300">
            {{ t('modrinth.versionMetadata') || 'Version Information' }}
          </div> -->
          <div class="grid grid-cols-2 gap-4 rounded-lg bg-gray-100 p-4 dark:bg-gray-800">
            <!-- Version Number -->
            <div>
              <div class="text-sm text-gray-600 dark:text-gray-400">
                {{ t('modrinth.versionNumber') || 'Version' }}
              </div>
              <div class="font-semibold text-gray-900 dark:text-gray-100">
                {{ versionNumber }}
              </div>
            </div>

            <!-- Release Type -->
            <div>
              <div class="text-sm text-gray-600 dark:text-gray-400">
                {{ t('versionType.name') || 'Release Type' }}
              </div>
              <div class="flex items-center gap-2">
                <span
                  :style="{ color: typeColor, borderColor: typeColor }"
                  class="border-l-[3px] pl-2 font-semibold"
                >
                  {{ t(`versionType.${versionType}`) }}
                </span>
              </div>
            </div>

            <!-- Loaders -->
            <div>
              <div class="text-sm text-gray-600 dark:text-gray-400">
                {{ t('modrinth.modLoaders.name') }}
              </div>
              <div class="flex flex-wrap gap-1">
                <v-chip
                  v-for="loader of loaders"
                  :key="loader"
                  small
                  outlined
                >
                  {{ loader }}
                </v-chip>
              </div>
            </div>

            <!-- Minecraft Versions -->
            <div>
              <div class="text-sm text-gray-600 dark:text-gray-400">
                {{ t('modrinth.gameVersions') }}
              </div>
              <div class="flex flex-wrap gap-1">
                <v-chip
                  v-for="gameVersion of gameVersions"
                  :key="gameVersion"
                  small
                  outlined
                >
                  {{ gameVersion }}
                </v-chip>
              </div>
            </div>

            <!-- Download Count -->
            <div>
              <div class="text-sm text-gray-600 dark:text-gray-400">
                {{ t('modrinth.downloads') }}
              </div>
              <div class="font-semibold text-gray-900 dark:text-gray-100">
                {{ formatNumber(downloads) }}
              </div>
            </div>

            <!-- Published Date -->
            <div>
              <div class="text-sm text-gray-600 dark:text-gray-400">
                {{ t('modrinth.createAt') }}
              </div>
              <div class="font-semibold text-gray-900 dark:text-gray-100">
                {{ getDateString(publishedDate, { dateStyle: 'long' }) }}
              </div>
            </div>
          </div>
        </div>

        <!-- Changelog -->
        <div v-if="changelog" class="flex flex-col gap-3">
          <div class="font-semibold text-gray-700 dark:text-gray-300">
            {{ t('FeedTheBeastProject.changelog') }}
          </div>
          <div
            :style="{ borderColor: typeColor }"
            class="border-l-[3px] pl-3"
          >
            <div
              class="markdown-body hover:(bg-[rgba(0,0,0,0.05)]) dark:hover:(bg-[rgba(0,0,0,0.3)]) max-h-96 overflow-auto rounded-lg bg-[rgba(0,0,0,0.07)] py-2 pl-2 text-gray-500 transition-colors hover:text-black dark:bg-[rgba(0,0,0,0.4)] dark:hover:text-gray-300"
              v-html="changelog"
            />
          </div>
        </div>

        <!-- Dependencies -->
        <UpstreamVersionDependenciesList
          v-if="dependenciesData && dependenciesData.length > 0"
          :dependencies="dependenciesData"
        />

        <!-- Files -->
        <UpstreamVersionFilesList
          v-if="filesData && filesData.length > 0"
          :files="filesData"
        />
      </v-card-text>

      <v-divider />
      <v-card-actions class="flex-shrink-0">
        <v-btn
          text
          @click="isShown = false"
        >
          {{ t('delete.no') }}
        </v-btn>
        <v-spacer />
        <v-btn
          color="primary"
          text
          @click="emit('duplicate', version)"
        >
          {{ t('instances.add') }}
          <v-icon right>
            add
          </v-icon>
        </v-btn>
        <v-btn
          color="primary"
          text
          @click="emit('update', version)"
        >
          {{ t('upstream.update') }}
          <v-icon right>
            upgrade
          </v-icon>
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts" setup>
import { useDateString } from '@/composables/date'
import { useMarkdown } from '@/composables/markdown'
import { getColorForReleaseType } from '@/util/color'
import { useVuetifyColor } from '@/composables/vuetify'
import { ProjectVersion } from '@xmcl/modrinth'
import { File as CurseforgeFile } from '@xmcl/curseforge'
import { Ref, computed, ref } from 'vue'
import UpstreamVersionDependenciesList from './UpstreamVersionDependenciesList.vue'
import UpstreamVersionFilesList from './UpstreamVersionFilesList.vue'

export interface VersionDetail {
  name?: string
  displayName?: string
  version_number?: string
  version_type?: string
  loaders?: string[]
  game_versions?: string[]
  downloads?: number
  date_published?: string
  changelog?: string
  dependencies?: Array<{
    version_id?: string | null
    project_id?: string
    dependency_type?: string
  }>
  files?: Array<{
    name: string
    size?: number
    url?: string
    filename?: string
    primary?: boolean
  }>
}

const props = defineProps<{
  shown: boolean
  version: VersionDetail | ProjectVersion | CurseforgeFile | null
  type?: 'modrinth' | 'curseforge'
}>()

const emit = defineEmits<{
  (e: 'update:shown', value: boolean): void
  (e: 'update', version: any): void
  (e: 'duplicate', version: any): void
}>()

const { t } = useI18n()
const { render } = useMarkdown()
const { getColorCode } = useVuetifyColor()
const { getDateString } = useDateString()

const isShown = computed({
  get: () => props.shown,
  set: (value) => emit('update:shown', value),
})

// Extract version data based on source
const version = computed(() => props.version)

const versionName = computed(() => {
  const v = version.value as any
  return v?.name || v?.displayName || 'Unknown Version'
})

const versionNumber = computed(() => {
  const v = version.value as any
  return v?.version_number || v?.displayName || 'N/A'
})

const versionType = computed(() => {
  const v = version.value as any
  return v?.version_type || v?.releaseType || 'release'
})

const loaders = computed(() => {
  const v = version.value as any
  return v?.loaders || v?.modLoaders || []
})

const gameVersions = computed(() => {
  const v = version.value as any
  return v?.game_versions || v?.gameVersions || []
})

const downloads = computed(() => {
  const v = version.value as any
  return v?.downloads || v?.downloadCount || 0
})

const publishedDate = computed(() => {
  const v = version.value as any
  return v?.date_published || v?.releaseTime || new Date().toISOString()
})

const changelog = computed(() => {
  const v = version.value as any
  const raw = v?.changelog || v?.changeLog || ''
  if (!raw) return ''
  try {
    return render(raw)
  } catch {
    return raw
  }
})

const dependencies = computed(() => {
  const v = version.value as any
  const deps = v?.dependencies || []
  return deps.map((dep: any) => ({
    projectId: dep.project_id || dep.dependency_id || '',
    name: dep.name || dep.project_id || dep.dependency_id || 'Unknown Dependency',
    type: dep.dependency_type || dep.type || 'unknown',
    icon: dep.icon || undefined,
    href: dep.href || (dep.project_id ? `https://modrinth.com/mod/${dep.project_id}` : undefined),
  }))
})

const dependenciesData = computed(() => {
  return dependencies.value.map((dep: any) => ({
    name: dep.name,
    type: dep.type,
    icon: dep.icon,
    href: dep.href,
  }))
})

const files = computed(() => {
  const v = version.value as any
  const fileList = v?.files || []
  return fileList.map((f: any) => ({
    name: f.filename || f.name || 'Unknown',
    size: f.size || f.fileLength || 0,
    url: f.url || f.downloadUrl || '',
    primary: f.primary || false,
  }))
})

const filesData = computed(() => {
  return files.value.map((f: any) => ({
    name: f.name,
    size: f.size,
    url: f.url,
    primary: f.primary,
  }))
})

const typeColor = computed(() => {
  const v = version.value as any
  const type = v?.version_type || v?.releaseType || 'release'
  return getColorCode(getColorForReleaseType(type))
})

function formatNumber(num: number): string {
  if (num >= 1_000_000) {
    return `${(num / 1_000_000).toFixed(1)}M`
  }
  if (num >= 1_000) {
    return `${(num / 1_000).toFixed(1)}K`
  }
  return num.toString()
}

function formatSize(bytes: number): string {
  if (!bytes) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}

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

function openLink(url: string) {
  if (url) {
    window.open(url, '_blank')
  }
}
</script>

<style scoped>
:deep(.markdown-body) {
  background: transparent;
  font-size: 0.9rem;
  line-height: 1.6;
}

:deep(.markdown-body h1),
:deep(.markdown-body h2),
:deep(.markdown-body h3) {
  margin-top: 0.5rem;
  margin-bottom: 0.5rem;
  font-weight: 600;
}

:deep(.markdown-body code) {
  background: rgba(0, 0, 0, 0.05);
  border-radius: 3px;
  padding: 0.2rem 0.4rem;
}

:deep(.dark .markdown-body code) {
  background: rgba(255, 255, 255, 0.05);
}

:deep(.markdown-body pre) {
  background: rgba(0, 0, 0, 0.05);
  border-radius: 4px;
  padding: 0.5rem;
  overflow-x: auto;
}

:deep(.dark .markdown-body pre) {
  background: rgba(255, 255, 255, 0.05);
}
</style>

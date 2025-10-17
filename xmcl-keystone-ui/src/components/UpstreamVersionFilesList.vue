<template>
  <div class="flex flex-col gap-3">
    <div class="font-semibold text-gray-700 dark:text-gray-300">
      {{ t('modrinth.files') || 'Files' }}
    </div>
    <v-list
      class="rounded-lg"
      color="transparent"
    >
      <v-list-item
        v-for="file of files"
        :key="file.name"
        :href="file.url"
        :target="file.url ? '_blank' : undefined"
      >
        <v-list-item-avatar
          v-if="file.primary"
          class="mr-2"
        >
          <v-icon
            class="material-icons-outlined"
            color="primary"
          >
            file_download
          </v-icon>
        </v-list-item-avatar>
        <v-list-item-avatar v-else class="mr-2">
          <v-icon class="material-icons-outlined">
            attachment
          </v-icon>
        </v-list-item-avatar>
        <v-list-item-content>
          <v-list-item-title v-text="file.name" />
          <v-list-item-subtitle>
            <div class="flex items-center gap-2">
              <span>{{ formatSize(file.size || 0) }}</span>
              <v-chip
                v-if="file.primary"
                x-small
                color="primary"
                text-color="white"
              >
                {{ t('modrinth.primary') || 'Primary' }}
              </v-chip>
            </div>
          </v-list-item-subtitle>
        </v-list-item-content>
      </v-list-item>
    </v-list>
  </div>
</template>

<script lang="ts" setup>
export interface FileItem {
  name: string
  size?: number
  url?: string
  primary?: boolean
}

const props = defineProps<{
  files: FileItem[]
}>()

const { t } = useI18n()

function formatSize(bytes: number): string {
  if (!bytes) return '0 B'
  const k = 1024
  const sizes = ['B', 'KB', 'MB', 'GB']
  const i = Math.floor(Math.log(bytes) / Math.log(k))
  return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
}
</script>

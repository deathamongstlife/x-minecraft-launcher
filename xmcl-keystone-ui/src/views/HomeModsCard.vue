<template>
  <div
    class="flex h-full flex-col transition-all duration-500 home-card rounded-lg"
    style="box-sizing: border-box"
    ref="root"
  >
    <v-card-text class="flex flex-col gap-2 p-3 pt-2">
      <div class="flex gap-4 items-center">
      <div class="tabs flex gap-4 items-center" :style="{ '--underline-left': underlineLeft + 'px', '--underline-width': underlineWidth + 'px' }">
        <span ref="contentRef" @click="onSelectContent" class="cursor-pointer" :class="{ 'selected': selected === 0 }">Content</span>
        <span ref="newsRef" @click="onSelectNews" class="cursor-pointer" :class="{ 'selected': selected === 1 }">Updates</span>
      </div>
        <div class="flex-grow" />
        <div class="controls" style="margin-right: 0.18rem">
          <v-btn icon small @click="emit('expand')">
            <v-icon size="20" class="rotate-[45deg]">
              unfold_more
            </v-icon>
          </v-btn>
        </div>
      </div>
      <div class="flex flex-col gap-2 transition-all duration-400" :style="(active || dragover) ? { 'max-height': '192px', overflow: selected === 1 ? 'auto' : 'unset' } : { 'max-height': '4rem' }">
        <HomeCardListItem v-for="item in items" :key="item.icon + item.text" :icon="item.icon" :tooltip="item.tooltip" :text="item.text" :highlighted="item.highlighted" @install="item.install" @setting="item.setting" @drop="item.drop" />
      </div>
    </v-card-text>
    <UpstreamVersionDetailDialog
      :shown="showVersionDialog"
      :version="selectedVersion"
      :type="upstream?.type === 'modrinth-modpack' ? 'modrinth' : 'curseforge'"
      @update:shown="showVersionDialog = $event"
      @update="onVersionUpdate"
      @duplicate="onVersionDuplicate"
    />
  </div>
</template>
<script lang="ts" setup>
import HomeCardListItem from '@/components/HomeCardListItem.vue'
import UpstreamVersionDetailDialog from '@/components/UpstreamVersionDetailDialog.vue'
import { useService } from '@/composables'
import { kDropHandler } from '@/composables/dropHandler'
import { kInstance } from '@/composables/instance'
import { kInstanceModsContext } from '@/composables/instanceMods'
import { kInstanceResourcePacks } from '@/composables/instanceResourcePack'
import { kInstanceSave } from '@/composables/instanceSave'
import { kInstanceShaderPacks } from '@/composables/instanceShaderPack'
import { injection } from '@/util/inject'
import { useElementHover } from '@vueuse/core'
import { getModrinthVersionModel } from '@/composables/modrinthVersions'
import { getCurseforgeProjectFilesModel } from '@/composables/curseforge'
import { useSWRVModel } from '@/composables/swrv'
import { InstanceModsServiceKey, InstanceResourcePacksServiceKey, InstanceSavesServiceKey, InstanceShaderPacksServiceKey } from '@xmcl/runtime-api'
import type { ProjectVersion } from '@xmcl/modrinth'
import type { File as CurseforgeFile } from '@xmcl/curseforge'

const emit = defineEmits<{
  (e: 'pin'): void
  (e: 'expand'): void
}>()

const { enabledMods } = injection(kInstanceModsContext)
const { enabled: enabledResourcePacks } = injection(kInstanceResourcePacks)
const { shaderPack } = injection(kInstanceShaderPacks)

const { push } = useRouter()
const { t } = useI18n()

const { dragover } = injection(kDropHandler)
const { path, instance } = injection(kInstance)

const upstream = computed(() => instance.value?.upstream)
const modrinthProjectId = ref<string>()
const curseforgeProjectId = ref<number>()
const minecraftVersion = computed(() => instance.value?.runtime.minecraft)
const modLoaderType = computed(() => {
  const runtime = instance.value?.runtime
  if (!runtime) return undefined
  if (runtime.forge) return 'forge'
  if (runtime.fabricLoader) return 'fabric'
  if (runtime.neoForged) return 'neoforge'
  if (runtime.quiltLoader) return 'quilt'
  return undefined
})

watch(upstream, (u) => {
  if (u?.type === 'modrinth-modpack') {
    modrinthProjectId.value = u.projectId
  } else {
    modrinthProjectId.value = undefined
  }
  if (u?.type === 'curseforge-modpack') {
    curseforgeProjectId.value = (u as any).id || (u as any).projectId
  } else {
    curseforgeProjectId.value = undefined
  }
})
const modrinthVersions = useSWRVModel(getModrinthVersionModel(modrinthProjectId as any, undefined, modLoaderType as any, computed(() => minecraftVersion.value ? [minecraftVersion.value] : undefined)))
const curseforgeModLoaderTypeRef = computed(() => {
  const type = modLoaderType.value
  if (type === 'forge') return 1
  if (type === 'fabric') return 4
  if (type === 'neoforge') return 6
  if (type === 'quilt') return 5
  return undefined
})
const curseforgeFiles = useSWRVModel(getCurseforgeProjectFilesModel(curseforgeProjectId as any, minecraftVersion as any, curseforgeModLoaderTypeRef as any) as any)

const showVersionDialog = ref(false)
const selectedVersion = ref<ProjectVersion | CurseforgeFile | null>(null)

const latestVersions = computed(() => {
  if (upstream.value?.type === 'modrinth-modpack') {
    return modrinthVersions.data.value || []
  } else if (upstream.value?.type === 'curseforge-modpack') {
    return (curseforgeFiles.data.value as any)?.data || []
  }
  return []
})

const root = ref<HTMLElement | null>(null)
const active = useElementHover(root)

const contentRef = ref<HTMLElement>()
const newsRef = ref<HTMLElement>()

const underlineLeft = computed(() => {
  if (selected.value === 0) return 0
  const contentWidth = contentRef.value?.offsetWidth || 0
  return contentWidth + 16
})

const underlineWidth = computed(() => {
  if (selected.value === 0) return contentRef.value?.offsetWidth || 0
  return newsRef.value?.offsetWidth || 0
})

const { install: installMod } = useService(InstanceModsServiceKey)
function onDropMod(e: DragEvent) {
  if (e.dataTransfer) {
    const filePaths = Array.from(e.dataTransfer.files).map(f => f.path)
    installMod({
      path: path.value,
      files: filePaths,
    })
    e.preventDefault()
  }
}

const { install: installResourcePack } = useService(InstanceResourcePacksServiceKey)
function onDropResourcePack(e: DragEvent) {
  if (e.dataTransfer) {
    const filePaths = Array.from(e.dataTransfer.files).map(f => f.path)
    installResourcePack({ path: path.value, files: filePaths })
    e.preventDefault()
  }
}

const { install: installShaderPack } = useService(InstanceShaderPacksServiceKey)
function onDropShaderPack(e: DragEvent) {
  if (e.dataTransfer) {
    const filePaths = Array.from(e.dataTransfer.files).map(f => f.path)
    installShaderPack({ path: path.value, files: filePaths })
    e.preventDefault()
  }
}

const { saves } = injection(kInstanceSave)
const savesLength = computed(() => saves.value.length)
const { importSave } = useService(InstanceSavesServiceKey)
function onDropSave(e: DragEvent) {
  if (e.dataTransfer) {
    const filePaths = Array.from(e.dataTransfer.files).map(f => f.path)
    importSave({ instancePath: path.value, path: filePaths[0] })
    e.preventDefault()
  }
}

const items = computed(() => {
  if (selected.value === 0) {
    return [
      {
        icon: 'extension',
        tooltip: t('mod.name'),
        text: dragover.value ? t('mod.dropHint') : t('mod.enabled', { count: enabledMods.value.length }),
        highlighted: false,
        install: () => push('/mods?source=remote'),
        setting: () => push('/mods'),
        drop: onDropMod
      },
      {
        icon: 'palette',
        tooltip: t('resourcepack.name'),
        text: dragover.value ? t('resourcepack.dropHint') : t('resourcepack.enable', { count: enabledResourcePacks.value.length }),
        highlighted: false,
        install: () => push('/resourcepacks?source=remote'),
        setting: () => push('/resourcepacks'),
        drop: onDropResourcePack
      },
      {
        icon: 'gradient',
        tooltip: t('shaderPack.name'),
        text: dragover.value ? t('shaderPack.dropHint') : !shaderPack.value ? t('shaderPack.empty') : shaderPack.value,
        highlighted: false,
        install: () => push('/shaderpacks?source=remote'),
        setting: () => push('/shaderpacks'),
        drop: onDropShaderPack
      },
      {
        icon: 'map',
        tooltip: t('save.name'),
        text: dragover.value ? t('save.dropHint') : t('save.createdWorlds', { count: savesLength.value }),
        highlighted: false,
        install: () => push('/save?source=remote'),
        setting: () => push('/save'),
        drop: onDropSave
      }
    ]
  } else if (selected.value === 1) {
    return latestVersions.value.map((v: any) => ({
      icon: 'update',
      tooltip: '',
      text: (v as any).name || (v as any).displayName,
      highlighted: upstream.value?.type === 'modrinth-modpack' && upstream.value.versionId === v.id || upstream.value?.type === 'curseforge-modpack' && upstream.value.fileId === v.id,
      install: () => {},
      setting: () => {
        selectedVersion.value = v
        showVersionDialog.value = true
      },
      drop: () => {}
    }))
  } else {
    return [{
      icon: 'extension',
      tooltip: t('mod.name'),
      text: dragover.value ? t('mod.dropHint') : t('mod.enabled', { count: enabledMods.value.length }),
      highlighted: false,
      install: () => push('/mods?source=remote'),
      setting: () => push('/mods'),
      drop: onDropMod
    }]
  }
})

const selected = ref(0)
function onSelectContent() {
  selected.value = 0
}

function onSelectNews() {
  selected.value = 1
}

function onVersionUpdate(version: any) {
  // Emit update event to parent or handle update
  // This would typically trigger an update of the modpack
}

function onVersionDuplicate(version: any) {
  // Emit duplicate event to create new instance
  // This would typically open a dialog to create a new instance with this version
}
</script>
<style scoped>
.tabs-card>.icons, .tabs-card>.tabs-items {
transition: all 0.2s ease-in-out;
  opacity: 0;
}

.dark .controls .v-icon {
  color: var(--icon-color);
}

.dark .controls .v-icon:hover {
  color: var(--icon-color-hovered);
}

.tabs {
  position: relative;
}

.tabs::after {
  content: '';
  position: absolute;
  bottom: -2px;
  height: 3px;
  background: var(--highlight-color);
  border-radius: 2px;
  left: var(--underline-left);
  width: var(--underline-width);
  transition: left 0.3s ease, width 0.3s ease;
}
</style>
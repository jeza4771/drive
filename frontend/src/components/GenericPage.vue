<template>
  <Navbar
    v-if="!verify?.error && !getEntities.error"
    :column-headers="$route.name === 'Recents' ? null : columnHeaders"
    :selections="selectedEntitities"
    :action-items="actionItems"
  />
  <FolderContentsError
    v-if="verify?.error || getEntities.error"
    :error="verify?.error || getEntities.error"
  />
  <NoFilesSection
    v-else-if="rows?.length === 0"
    :icon="icon"
    :primary-message="activeFilters.length ? 'Nothing found.' : primaryMessage"
    :secondary-message="
      activeFilters.length ? 'Try changing filters, maybe?' : secondaryMessage
    "
  />
  <template v-else>
    <ListView
      v-if="$store.state.view === 'list'"
      v-model="selections"
      :folder-contents="rows && grouper(rows)"
      :action-items="actionItems"
      :user-data="userData"
    />
    <GridView
      v-else
      v-model="selections"
      :folder-contents="rows"
      :action-items="actionItems"
      :user-data="userData"
    />
    <InfoPopup :entities="infoEntities" />
  </template>

  <Dialogs
    v-model="dialog"
    :selections="activeEntity ? [activeEntity] : selectedEntitities"
    :get-entities="getEntities"
  />
  <FileUploader v-if="$store.state.user.id" @success="getEntities.fetch()" />
</template>
<script setup>
import ListView from "@/components/ListView.vue"
import GridView from "@/components/GridView.vue"
import Navbar from "@/components/Navbar.vue"
import NoFilesSection from "@/components/NoFilesSection.vue"
import Dialogs from "@/components/Dialogs.vue"
import FolderContentsError from "@/components/FolderContentsError.vue"
import InfoPopup from "@/components/InfoPopup.vue"
import { getLink } from "@/utils/getLink"
import { toggleFav, clearRecent } from "@/resources/files"
import { allUsers } from "@/resources/permissions"
import { entitiesDownload } from "@/utils/download"
import { RotateCcw } from "lucide-vue-next"
import FileUploader from "@/components/FileUploader.vue"
import Team from "./EspressoIcons/Organization.vue"
import Share from "./EspressoIcons/Share.vue"
import Download from "./EspressoIcons/Download.vue"
import Link from "./EspressoIcons/Link.vue"
import Rename from "./EspressoIcons/Rename.vue"
import Move from "./EspressoIcons/Move.vue"
import Info from "./EspressoIcons/Info.vue"
import Preview from "./EspressoIcons/Preview.vue"
import Trash from "./EspressoIcons/Trash.vue"
import { ref, computed, watch } from "vue"
import { useRoute } from "vue-router"
import { useStore } from "vuex"
import { openEntity } from "@/utils/files"
import { togglePersonal } from "@/resources/files"
import { toast } from "@/utils/toasts"

const props = defineProps({
  grouper: { type: Function, default: (d) => d },
  showSort: { type: Boolean, default: true },
  verify: { Object, default: null },
  icon: Object,
  primaryMessage: String,
  secondaryMessage: { type: String, default: "" },
  getEntities: Object,
})
const route = useRoute()
const store = useStore()

const dialog = ref(null)
const infoEntities = ref([])
const team = route.params.team
const sortOrder = computed(() => store.state.sortOrder)
const activeFilters = computed(() => store.state.activeFilters)
const activeEntity = computed(() => store.state.activeEntity)
const rows = computed(() => props.getEntities.data)

// We do client side sorting for immediate UI updates
// Can't check for entity data updates as we are updating - so check for loading
watch(
  [sortOrder, () => props.getEntities.loading],
  ([val, loading]) => {
    if (!props.getEntities.data || loading) return
    const field = val.field
    const order = val.ascending ? 1 : -1
    const sorted = props.getEntities.data.toSorted((a, b) => {
      return a[field] == b[field] ? 0 : a[field] < b[field] ? order : -order
    })
    props.getEntities.setData(sorted)
    store.commit("setCurrentFolder", {
      name: sorted[0]?.parent_entity || "",
      entities: sorted.filter?.((k) => k.title[0] !== "."),
    })
  },
  { immediate: true }
)

const selections = ref(new Set())
const selectedEntitities = computed(
  () =>
    props.getEntities.data?.filter?.(({ name }) =>
      selections.value.has(name)
    ) || []
)

const verifyAccess = computed(() => props.verify?.data || !props.verify)
watch(
  verifyAccess,
  async (data) => {
    if (data)
      await props.getEntities.fetch({
        team,
      })
  },
  { immediate: true }
)

watch(activeFilters.value, async (val) => {
  props.getEntities.fetch({
    team,
    file_kinds: JSON.stringify(val.map((k) => k.label)),
  })
})

allUsers.fetch({ team })

// Action Items
const actionItems = computed(() => {
  if (route.name === "Trash") {
    return [
      {
        label: "Restore",
        icon: RotateCcw,
        onClick: () => (dialog.value = "restore"),
        multi: true,
        important: true,
      },
      {
        label: "Delete forever",
        icon: Trash,
        onClick: () => (dialog.value = "d"),
        isEnabled: () => route.name === "Trash",
        multi: true,
        important: true,
      },
    ].filter((a) => !a.isEnabled || a.isEnabled())
  } else {
    return [
      {
        label: "Preview",
        icon: Preview,
        onClick: ([entity]) => openEntity(team, entity),
        isEnabled: (e) => !e.is_link,
      },
      {
        label: "Open",
        icon: "external-link",
        onClick: ([entity]) => openEntity(team, entity),
        isEnabled: (e) => e.is_link,
      },
      {
        label: "Download",
        icon: Download,
        isEnabled: (e) => !e.is_link,
        onClick: (entities) => entitiesDownload(team, entities),
        multi: true,
        important: true,
      },
      {
        label: "Share",
        icon: Share,
        onClick: () => (dialog.value = "s"),
        isEnabled: (e) => e.share,
        important: true,
      },
      {
        label: "Get Link",
        icon: Link,
        onClick: ([entity]) => getLink(entity),
        important: true,
      },
      {
        label: "Rename",
        icon: Rename,
        onClick: () => (dialog.value = "rn"),
        isEnabled: (e) => e.write,
      },
      {
        label: "Move",
        icon: Move,
        onClick: () => (dialog.value = "m"),
        isEnabled: (e) => e.write,
        multi: true,
        important: true,
      },
      {
        label: "Move to Team",
        icon: Team,
        onClick: (entities) =>
          confirm(
            `Are you sure you want to move ${entities.length} ${
              entities.length === 1 ? "item" : "items"
            } to the team?`
          ) &&
          entities.map((e) =>
            togglePersonal.submit({ entity_name: e.name, new_value: 0 })
          ),
        isEnabled: () => route.name == "Home",
        multi: true,
        important: true,
      },
      {
        label: "Show Info",
        icon: Info,
        onClick: () => infoEntities.value.push(store.state.activeEntity),
        isEnabled: () => !store.state.activeEntity || !store.state.showInfo,
      },
      {
        label: "Hide Info",
        icon: Info,
        onClick: () => (dialog.value = "info"),
        isEnabled: () => store.state.activeEntity && store.state.showInfo,
      },
      {
        label: "Favourite",
        icon: "star",
        onClick: (entities) => {
          entities.forEach((e) => (e.is_favourite = true))
          // Hack to cache
          props.getEntities.setData(props.getEntities.data)
          toggleFav.submit({ entities })
        },
        isEnabled: (e) => !e.is_favourite,
        important: true,
        multi: true,
      },
      {
        label: "Unfavourite",
        icon: "star",
        class: "stroke-amber-500 fill-amber-500",
        onClick: (entities) => {
          entities.forEach((e) => (e.is_favourite = false))
          props.getEntities.setData(props.getEntities.data)
          toggleFav.submit({ entities })
        },
        isEnabled: (e) => e.is_favourite,
        important: true,
        multi: true,
      },
      {
        label: "Remove from Recents",
        icon: "clock",
        onClick: (entities) => {
          clearRecent.submit({
            entities,
          })
        },
        isEnabled: () => route.name == "Recents",
        important: true,
        multi: true,
      },
      {
        label: "Unshare",
        danger: true,
        icon: "trash-2",
        onClick: () => (dialog.value = "unshare"),
        isEnabled: (e) =>
          e.owner != "You" && e.user_doctype === "User" && e.everyone !== 1,
      },
      {
        label: "Move to Trash",
        icon: Trash,
        onClick: () => (dialog.value = "remove"),
        isEnabled: (e) => e.write,
        important: true,
        multi: true,
        danger: true,
      },
    ]
  }
})

const columnHeaders = [
  {
    label: "Name",
    field: "title",
  },
  {
    label: "Owner",
    field: "owner",
  },
  {
    label: "Modified",
    field: "modified",
  },
  {
    label: "Size",
    field: "file_size",
  },
  {
    label: "Type",
    field: "mime_type",
  },
]
const userData = computed(() =>
  allUsers.data ? Object.fromEntries(allUsers.data.map((k) => [k.name, k])) : {}
)

async function newLink() {
  if (!document.hasFocus()) return
  try {
    const text = await navigator.clipboard.readText()
    if (localStorage.getItem("prevClip") === text) return
    new URL(text)
    localStorage.setItem("prevClip", text)
    toast({
      title: "Link detected",
      text,
      buttons: [
        {
          label: "Add",
          action: () => {
            dialog.value = "l"
          },
        },
      ],
    })
  } catch (_) {}
}

// Hacky but performant way to track links - when the user loads page, copies on page, or comes to page
// JS doesn't allow direct reading of clipboard
newLink()
addEventListener("copy", () => document.getElementById("popovers").click())
document.getElementById("popovers").addEventListener("click", newLink)
document.addEventListener("visibilitychange", () => {
  window.focus()
  !document.hidden && setTimeout(newLink, 100)
})
</script>

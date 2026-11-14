<template>
  <div class="w-full h-full py-6 px-4">
    <div class="flex items-center mb-2">
      <p class="text-base font-semibold">{{ $strings.LabelFolder }}: {{ folderName }}</p>
      <div class="flex-grow" />

      <button v-if="isSelectionMode" class="text-sm text-fg-muted px-2 py-1" :disabled="deletingItems" @click="clearSelection">{{ $strings.ButtonCancel }}</button>
      <span v-if="dialogItems.length" class="material-symbols text-2xl" @click="showDialog = true">more_vert</span>
    </div>

    <p class="text-sm mb-4 text-fg-muted">{{ $strings.LabelMediaType }}: {{ mediaType }}</p>

    <p class="mb-2 text-base text-fg">{{ $strings.HeaderLocalLibraryItems }} ({{ localLibraryItems.length }})</p>

    <div class="w-full media-item-container overflow-y-auto" :class="{ 'selection-mode': isSelectionMode }">
      <template v-for="localLibraryItem in localLibraryItems">
        <div
          :key="localLibraryItem.id"
          class="flex my-1 rounded selected-media-item"
          :class="{ selected: selectedLocalLibraryItemIds.includes(localLibraryItem.id) }"
          role="link"
          tabindex="0"
          @click="clickLocalLibraryItem(localLibraryItem)"
          @keydown.enter="clickLocalLibraryItem(localLibraryItem)"
          @pointerdown="startLongPress($event, localLibraryItem)"
          @pointermove="handlePointerMove"
          @pointerup="cancelLongPress"
          @pointercancel="cancelLongPress"
          @contextmenu.prevent
        >
          <div class="w-12 h-12 min-w-12 min-h-12 bg-primary">
            <img v-if="localLibraryItem.coverPathSrc" :src="localLibraryItem.coverPathSrc" class="w-full h-full object-contain" />
          </div>
          <div class="flex-grow px-2">
            <p class="text-sm">{{ localLibraryItem.media.metadata.title }}</p>
            <p class="text-xs text-fg-muted">{{ getLocalLibraryItemSubText(localLibraryItem) }}</p>
          </div>
          <div v-if="isSelectionMode" class="w-12 h-12 flex items-center justify-center">
            <span v-if="selectedLocalLibraryItemIds.includes(localLibraryItem.id)" class="material-symbols fill text-xl text-success">check_circle</span>
            <span v-else class="material-symbols text-xl text-fg-muted">radio_button_unchecked</span>
          </div>
          <div v-else class="w-12 h-12 flex items-center justify-center">
            <span class="material-symbols text-xl text-fg-muted">arrow_right</span>
          </div>
        </div>
      </template>
    </div>

    <div v-if="isSelectionMode" class="fixed left-0 w-full py-4 px-4 bg-bg box-shadow-book flex items-center" :style="{ bottom: isPlayerOpen ? '120px' : '0px' }">
      <p class="text-sm text-fg-muted">{{ $getString('LabelSelectedCount', [$formatNumber(selectedLocalLibraryItemIds.length)]) }}</p>
      <div class="flex-grow" />
      <ui-btn small color="error" :disabled="deletingItems" @click="deleteSelectedItems">{{ deletingItems ? $strings.MessageDeleting : $strings.ButtonDeleteSelected }}</ui-btn>
    </div>

    <modals-dialog v-model="showDialog" :items="dialogItems" @action="dialogAction" />
  </div>
</template>

<script>
import { Capacitor } from '@capacitor/core'
import { Dialog } from '@capacitor/dialog'
import { AbsFileSystem } from '@/plugins/capacitor'

const LONG_PRESS_DELAY_MS = 500
const LONG_PRESS_MOVE_TOLERANCE_PX = 10

export default {
  asyncData({ params, query }) {
    return {
      folderId: params.id
    }
  },
  data() {
    return {
      localLibraryItems: [],
      folder: null,
      removingFolder: false,
      showDialog: false,
      selectedLocalLibraryItemIds: [],
      longPressTimeout: null,
      longPressStartPosition: null,
      suppressNextClick: false,
      deletingItems: false
    }
  },
  computed: {
    isPlayerOpen() {
      return this.$store.getters['getIsPlayerOpen']
    },
    folderName() {
      return this.folder?.name || null
    },
    mediaType() {
      return this.folder?.mediaType
    },
    isInternalStorage() {
      return this.folder?.id.startsWith('internal-')
    },
    isSelectionMode() {
      return this.selectedLocalLibraryItemIds.length > 0
    },
    selectedLocalLibraryItems() {
      return this.localLibraryItems.filter((item) => this.selectedLocalLibraryItemIds.includes(item.id))
    },
    dialogItems() {
      if (this.isInternalStorage) return []
      const items = []
      items.push({
        text: this.$strings.ButtonRemove,
        value: 'remove'
      })
      return items
    }
  },
  methods: {
    startLongPress(event, localLibraryItem) {
      if (!event.isPrimary || event.button !== 0 || this.isSelectionMode) return

      this.cancelLongPress()
      this.suppressNextClick = false
      this.longPressStartPosition = { x: event.clientX, y: event.clientY }
      this.longPressTimeout = setTimeout(() => {
        this.longPressTimeout = null
        this.longPressStartPosition = null
        this.suppressNextClick = true
        this.selectLocalLibraryItem(localLibraryItem)
      }, LONG_PRESS_DELAY_MS)
    },
    handlePointerMove(event) {
      if (!this.longPressStartPosition) return

      const horizontalDistance = Math.abs(event.clientX - this.longPressStartPosition.x)
      const verticalDistance = Math.abs(event.clientY - this.longPressStartPosition.y)
      if (horizontalDistance > LONG_PRESS_MOVE_TOLERANCE_PX || verticalDistance > LONG_PRESS_MOVE_TOLERANCE_PX) {
        this.cancelLongPress()
      }
    },
    cancelLongPress() {
      if (this.longPressTimeout) {
        clearTimeout(this.longPressTimeout)
        this.longPressTimeout = null
      }
      this.longPressStartPosition = null
    },
    selectLocalLibraryItem(localLibraryItem) {
      this.cancelLongPress()
      this.toggleSelectedLocalLibraryItem(localLibraryItem)
      this.$hapticsImpact()
    },
    clickLocalLibraryItem(localLibraryItem) {
      if (this.deletingItems) return

      if (this.suppressNextClick) {
        this.suppressNextClick = false
        return
      }
      if (this.isSelectionMode) {
        this.toggleSelectedLocalLibraryItem(localLibraryItem)
      } else {
        this.$router.push(`/localMedia/item/${localLibraryItem.id}`)
      }
    },
    toggleSelectedLocalLibraryItem(localLibraryItem) {
      if (this.deletingItems) return

      const index = this.selectedLocalLibraryItemIds.indexOf(localLibraryItem.id)
      if (index >= 0) {
        this.selectedLocalLibraryItemIds.splice(index, 1)
      } else {
        this.selectedLocalLibraryItemIds.push(localLibraryItem.id)
      }
    },
    clearSelection() {
      if (this.deletingItems) return
      this.selectedLocalLibraryItemIds = []
    },
    getLocalLibraryItemSubText(localLibraryItem) {
      if (!localLibraryItem) return ''
      if (localLibraryItem.mediaType == 'book') {
        const txts = []
        if (localLibraryItem.media.ebookFile) {
          txts.push(`${localLibraryItem.media.ebookFile.ebookFormat} ${this.$strings.LabelEbook}`)
        }
        if (localLibraryItem.media.tracks?.length) {
          txts.push(`${localLibraryItem.media.tracks.length} ${this.$strings.LabelTracks}`)
        }
        return txts.join(' • ')
      } else {
        return `${localLibraryItem.media.episodes?.length || 0} ${this.$strings.HeaderEpisodes}`
      }
    },
    dialogAction(action) {
      console.log('Dialog action', action)
      if (action == 'remove') {
        this.removeFolder()
      }
      this.showDialog = false
    },
    async removeFolder() {
      var deleteMessage = 'Are you sure you want to remove this folder? (does not delete anything in your file system)'
      if (this.localLibraryItems.length) {
        deleteMessage = `Are you sure you want to remove this folder and ${this.localLibraryItems.length} items? (does not delete anything in your file system)`
      }
      const { value } = await Dialog.confirm({
        title: this.$strings.HeaderConfirm,
        message: deleteMessage
      })
      if (value) {
        this.removingFolder = true
        await AbsFileSystem.removeFolder({ folderId: this.folderId })
        this.removingFolder = false
        this.$router.replace('/localMedia/folders')
      }
    },
    async deleteSelectedItems() {
      if (!this.selectedLocalLibraryItems.length || this.deletingItems) return

      const selectedItems = this.selectedLocalLibraryItems.slice()
      const count = selectedItems.length
      const { value } = await Dialog.confirm({
        title: this.$strings.HeaderConfirm,
        message: count === 1 ? this.$strings.MessageConfirmDeleteSelectedLocalFile : this.$getString('MessageConfirmDeleteSelectedLocalFiles', [this.$formatNumber(count)])
      })
      if (!value) return

      this.deletingItems = true
      const deletedIds = []

      try {
        for (const item of selectedItems) {
          try {
            const res = await AbsFileSystem.deleteItem(item)
            if (res?.success) deletedIds.push(item.id)
          } catch (error) {
            console.error(`Failed to delete local library item ${item.id}`, error)
          }
        }
      } finally {
        this.localLibraryItems = this.localLibraryItems.filter((item) => !deletedIds.includes(item.id))
        this.selectedLocalLibraryItemIds = this.selectedLocalLibraryItemIds.filter((id) => !deletedIds.includes(id))
        this.deletingItems = false
      }

      if (deletedIds.length === count) {
        this.$toast.success('Deleted Successfully')
      } else {
        this.$toast.error(this.$getString('MessageDeletedSelectedLocalItems', [this.$formatNumber(deletedIds.length), this.$formatNumber(count)]))
      }
    },
    async init() {
      var folder = await this.$db.getLocalFolder(this.folderId)
      this.folder = folder

      var items = (await this.$db.getLocalLibraryItemsInFolder(this.folderId)) || []
      console.log('Init folder', this.folderId, items)
      this.localLibraryItems = items.map((lmi) => {
        console.log('Local library item', JSON.stringify(lmi))
        return {
          ...lmi,
          coverPathSrc: lmi.coverContentUrl ? Capacitor.convertFileSrc(lmi.coverContentUrl) : null
        }
      })
    },
    newLocalLibraryItem(item) {
      if (item.folderId == this.folderId) {
        console.log('New local library item', item.id)
        if (this.localLibraryItems.find((li) => li.id == item.id)) {
          console.warn('Item already added', item.id)
          return
        }

        var _item = {
          ...item,
          coverPathSrc: item.coverContentUrl ? Capacitor.convertFileSrc(item.coverContentUrl) : null
        }
        this.localLibraryItems.push(_item)
      }
    }
  },
  mounted() {
    this.$eventBus.$on('new-local-library-item', this.newLocalLibraryItem)
    this.init()
  },
  beforeDestroy() {
    this.cancelLongPress()
    this.$eventBus.$off('new-local-library-item', this.newLocalLibraryItem)
  }
}
</script>

<style scoped>
.media-item-container {
  height: calc(100vh - 210px);
  max-height: calc(100vh - 210px);
}
.media-item-container.selection-mode {
  height: calc(100vh - 290px);
  max-height: calc(100vh - 290px);
}
.selected-media-item.selected {
  background-color: rgba(63, 131, 248, 0.12);
}
.playerOpen .media-item-container {
  height: calc(100vh - 310px);
  max-height: calc(100vh - 310px);
}
.playerOpen .media-item-container.selection-mode {
  height: calc(100vh - 390px);
  max-height: calc(100vh - 390px);
}
</style>

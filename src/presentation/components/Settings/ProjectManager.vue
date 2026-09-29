<template>
  <div>
    <h4 class="mb-2">
      Project
      <v-tooltip text="Save project" location="bottom">
        <template v-slot:activator="{ props }">
          <v-btn
            v-bind="props"
            @click="saveProject"
            size="x-small"
            class="ml-2"
            :loading="saving"
            ><v-icon>mdi-content-save</v-icon></v-btn
          >
        </template>
      </v-tooltip>

      <v-tooltip text="Load project" location="bottom">
        <template v-slot:activator="{ props }">
          <v-btn
            v-bind="props"
            @click="triggerLoadProject"
            size="x-small"
            class="ml-2"
            :loading="loading"
            ><v-icon>mdi-folder-open</v-icon></v-btn
          >
        </template>
      </v-tooltip>

      <input
        ref="fileInput"
        type="file"
        accept=".zip"
        style="display: none"
        @change="onFileSelected"
      />
    </h4>

    <v-alert v-if="errorMessage" type="error" class="mt-2" density="compact">
      {{ errorMessage }}
    </v-alert>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import {
  canvasHandler,
  projectService,
} from '@/instanceStore/applicationServiceInstances'
import {
  datasetRepository,
  axisSetRepository,
} from '@/instanceStore/repositoryInatances'
import { POINT_MODE } from '@/constants'

export default defineComponent({
  data() {
    return {
      canvasHandler,
      projectService,
      datasetRepository,
      axisSetRepository,
      saving: false,
      loading: false,
      errorMessage: '',
      messageHandler: null as ((event: MessageEvent) => void) | null,
    }
  },
  mounted() {
    // Cross-window bridge for iframe embeds. The parent page can post one of:
    //   { type: 'load-project', zip: <Blob> }
    //   { type: 'load-project', zipBase64: '<base64>' }
    // Base64 is preferred because Blobs sometimes fail to survive nested
    // cross-origin iframe hops (e.g. Streamlit → component iframe → us).
    this.messageHandler = async (event: MessageEvent) => {
      const data = event.data
      if (!data || data.type !== 'load-project') return

      let blob: Blob | null = null
      if (data.zip instanceof Blob) {
        blob = data.zip
      } else if (typeof data.zipBase64 === 'string') {
        try {
          const bin = atob(data.zipBase64)
          const bytes = new Uint8Array(bin.length)
          for (let i = 0; i < bin.length; i++) bytes[i] = bin.charCodeAt(i)
          blob = new Blob([bytes], { type: 'application/zip' })
        } catch (e) {
          console.error('base64 decode failed', e)
        }
      }
      if (blob) await this.loadProjectFromBlob(blob)
    }
    window.addEventListener('message', this.messageHandler)
    // Announce readiness so the parent knows when it's safe to send the ZIP.
    // Retry a few times to defeat a race where the parent's listener isn't
    // attached yet when the iframe first mounts.
    const announce = () => {
      try {
        window.parent?.postMessage(
          { type: 'starrydigitizer-ready' },
          '*',
        )
      } catch (_) {
        // No parent window — direct visit, nothing to do.
      }
    }
    announce()
    setTimeout(announce, 300)
    setTimeout(announce, 1000)
    setTimeout(announce, 3000)
  },
  beforeUnmount() {
    if (this.messageHandler) {
      window.removeEventListener('message', this.messageHandler)
      this.messageHandler = null
    }
  },
  methods: {
    async saveProject() {
      this.saving = true
      this.errorMessage = ''

      try {
        // Export project using ProjectService
        const zipBlob = await this.projectService.exportProject()

        // Download ZIP file
        this.projectService.downloadZip(zipBlob)
      } catch (error) {
        console.error('Error saving project:', error)
        this.errorMessage = `Error saving project: ${(error as Error).message}`
      } finally {
        this.saving = false
      }
    },

    triggerLoadProject() {
      const fileInput = this.$refs.fileInput as HTMLInputElement
      fileInput.click()
    },

    async onFileSelected(event: Event) {
      const target = event.target as HTMLInputElement
      const file = target.files?.[0]
      if (!file) {
        this.errorMessage = 'No file selected'
        return
      }
      await this.loadProjectFromBlob(file)
      // Reset file input so re-selecting the same file re-fires @change.
      target.value = ''
    },

    async loadProjectFromBlob(blob: Blob) {
      this.loading = true
      this.errorMessage = ''

      try {
        // projectService.loadProject accepts anything with .arrayBuffer(),
        // so a Blob works even though its signature says File.
        const imageData = await this.projectService.loadProject(blob as File)

        // Initialize canvas with loaded image
        await this.canvasHandler.initializeImageElement(imageData)
        this.canvasHandler.drawFitSizeImage()
        this.canvasHandler.setUploadImageUrl(imageData)

        // Remove empty "dataset 1" if it was created during initialization
        const emptyDataset1 = this.datasetRepository.datasets.find(
          (d) => d.id === 1 && d.name === 'dataset 1' && d.points.length === 0,
        )
        if (emptyDataset1 && this.datasetRepository.datasets.length > 1) {
          this.datasetRepository.datasets =
            this.datasetRepository.datasets.filter((d) => d.id !== 1)
        }

        // Enable "View All Datasets" mode after loading project
        this.datasetRepository.setActiveDataset(0)

        // Set all axis sets to 4 points mode
        this.axisSetRepository.axisSets.forEach((axisSet) => {
          axisSet.pointMode = POINT_MODE.FOUR_POINTS
        })

        try {
          window.parent?.postMessage(
            { type: 'starrydigitizer-loaded' },
            '*',
          )
        } catch (_) {
          // Ignore — no parent.
        }
      } catch (error) {
        console.error('Error loading project:', error)
        this.errorMessage = `Error loading project: ${(error as Error).message}`
        try {
          window.parent?.postMessage(
            { type: 'starrydigitizer-error', message: this.errorMessage },
            '*',
          )
        } catch (_) {
          // Ignore.
        }
      } finally {
        this.loading = false
      }
    },
  },
})
</script>

<style scoped>
.gap-2 {
  gap: 0.5rem;
}
</style>

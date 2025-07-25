<!-- components/VstScanner.vue -->
<template>
  <div class="flex flex-col gap-4">
    <div class="flex w-full flex-wrap items-center justify-between gap-2">
      <div class="flex flex-wrap items-center gap-2">
        <button @click="scanPlugins" :disabled="isScanning" class="c-button c-button--mint w-fit">
          <div v-if="isScanning" class="animate-spin">
            <IconLoader2 class="size-4 animate-spin" />
          </div>
          {{ isScanning ? "Scanning" : "Scan Plugins" }}
        </button>
        <button @click="deletePlugins" class="c-button c-button--red w-fit">Delete Plugins</button>
      </div>
      <button class="c-button c-button--clear w-fit" @click="exportPlugins">Export Database</button>
    </div>

    <Transition name="fade" mode="out-in">
      <div v-if="results" class="bg-jet/50 relative mt-4 rounded-lg p-6">
        <button @click="closeResults" class="text-powder/50 hover:text-powder/80 absolute top-2 right-2 cursor-pointer transition-colors" title="Close results">
          <IconX class="size-5" />
        </button>
        <h3 class="text-powder/90 text-lg font-bold">Scan Results</h3>
        <p class="text-powder/70">Total plugins: {{ results.totalPlugins }}</p>
        <p class="text-powder/70">Valid plugins: {{ results.validPlugins }}</p>
        <p class="text-powder/70">Invalid plugins: {{ results.totalPlugins - results.validPlugins }}</p>

        <!-- Invalid Plugins Section -->
        <div v-if="invalidPlugins.length > 0" class="mt-6">
          <h4 class="text-powder/80 text-md mb-3 font-semibold">Invalid Plugins ({{ invalidPlugins.length }})</h4>
          <div class="space-y-3">
            <div v-for="plugin in invalidPlugins" :key="plugin.path" class="rounded-md border border-red-400/20 bg-red-400/5 p-4">
              <div class="flex items-start justify-between">
                <div class="flex-1">
                  <div class="text-powder/90 font-medium">{{ getPluginName(plugin) }}</div>
                  <div class="text-powder/60 mt-1 text-sm">{{ plugin.path }}</div>
                  <div class="mt-2 text-sm font-medium text-red-400">Error: {{ plugin.error }}</div>
                </div>
                <div class="ml-4 flex-shrink-0">
                  <span class="bg-red/20 text-red border-red rounded-md border px-2 py-1 text-xs">Invalid</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Transition>

    <!-- Error Modal -->
    <ModalWindow v-model="showErrorModal" title="Error">
      <p class="text-powder/70 mt-2 text-sm">
        {{ errorMessage }}
      </p>

      <template #footer>
        <button @click="showErrorModal = false" class="c-button c-button--clear">OK</button>
      </template>
    </ModalWindow>
  </div>
</template>

<script setup>
import { IconLoader2 } from "@tabler/icons-vue";
import { IconX } from "@tabler/icons-vue";
const emit = defineEmits(["scan-complete"]);

const isScanning = ref(false);
const results = ref(null);
const showErrorModal = ref(false);
const errorMessage = ref("");

const { success, error: showError } = useToast();

// Computed property to get invalid plugins
const invalidPlugins = computed(() => {
  if (!results.value || !results.value.plugins) return [];
  return results.value.plugins.filter((plugin) => !plugin.isValid);
});

// Helper function to get plugin name from path or name field
const getPluginName = (plugin) => {
  if (plugin.name) return plugin.name;

  // Extract name from path if no name field
  const pathParts = plugin.path.split(/[\\\/]/);
  const fileName = pathParts[pathParts.length - 1];
  return fileName.replace(/\.vst3?$/i, "") || "Unknown Plugin";
};

async function scanPlugins() {
  isScanning.value = true;
  results.value = null;

  try {
    const response = await $fetch("/api/vst/scan", {
      method: "POST",
    });

    if (response.success) {
      // Get the current database state after scan
      const dbResponse = await $fetch("/api/plugins");
      if (dbResponse.success && dbResponse.data) {
        const totalPlugins = dbResponse.data.length;
        const validPlugins = dbResponse.data.filter((plugin) => plugin.isValid).length;
        const invalidPlugins = totalPlugins - validPlugins;

        // Create results object from database state
        results.value = {
          totalPlugins,
          validPlugins,
          plugins: dbResponse.data,
        };
      } else {
        // Fallback to scan results if database fetch fails
        results.value = response.results;
      }
      success("Plugins scanned successfully!");
      emit("scan-complete", results.value);
    } else {
      errorMessage.value = response.error || "Scan failed";
      showErrorModal.value = true;
    }
  } catch (error) {
    console.error("Scan failed:", error);
    errorMessage.value = "Failed to connect to scanner service";
    showErrorModal.value = true;
  } finally {
    isScanning.value = false;
  }
}

async function deletePlugins() {
  const response = await $fetch("/api/vst/delete");
  if (response.success) {
    success("Plugins deleted successfully!");
    emit("scan-complete", []);
  } else {
    errorMessage.value = response.error || "Failed to delete plugins";
    showErrorModal.value = true;
  }
}

async function exportPlugins() {
  try {
    const response = await $fetch("/api/vst/export");
    if (response.success) {
      success("Plugins exported successfully!");
      await downloadFile(response.filePath);
    } else {
      errorMessage.value = response.error || "Failed to export plugins";
      showErrorModal.value = true;
    }
  } catch (error) {
    console.error("Export failed:", error);
    errorMessage.value = "Failed to export plugins";
    showErrorModal.value = true;
  }
}

async function downloadFile(filePath) {
  try {
    // Fetch the file content from the server
    const fileResponse = await $fetch(`/api/vst/download?filePath=${encodeURIComponent(filePath)}`, {
      method: "GET",
    });

    if (fileResponse.success && fileResponse.data) {
      // Create a blob from the JSON data
      const blob = new Blob([JSON.stringify(fileResponse.data, null, 2)], {
        type: "application/json",
      });

      // Create download link
      const url = window.URL.createObjectURL(blob);
      const link = document.createElement("a");
      link.href = url;
      link.download = "exported-plugins.json";

      // Trigger download
      document.body.appendChild(link);
      link.click();

      // Cleanup
      document.body.removeChild(link);
      window.URL.revokeObjectURL(url);
    } else {
      throw new Error("Failed to fetch file data");
    }
  } catch (error) {
    console.error("Download failed:", error);
    errorMessage.value = "Failed to download file";
    showErrorModal.value = true;
  }
}

function closeResults() {
  results.value = null;
}
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>

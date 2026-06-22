<template>
  <div
    v-if="isIntegratedSelected"
    :class="borderClass"
    class="flex flex-col sm:flex-row border p-2 rounded-lg h-full gap-2 sm:items-center transition-all duration-300"
  >
    <label class="text-sm sm:w-36 shrink-0" for="integratedGuiderCamSelect">
      {{ $t('components.guider.integrated.guideCamera') }}:
    </label>
    <div class="flex gap-2 items-center w-full">
      <select
        id="integratedGuiderCamSelect"
        class="w-full default-select min-w-0"
        v-model="selectedCamId"
        @change="setCamera"
        :disabled="store.guiderInfo.Connected"
      >
        <option value="" disabled>{{ $t('common.select') }}</option>
        <option v-for="cam in cameras" :key="cam.id" :value="cam.id">
          {{ cam.name }}
        </option>
      </select>
      <div class="flex shrink-0 gap-1">
        <button
          @click="loadCameras"
          :disabled="isLoading || store.guiderInfo.Connected"
          class="flex justify-center items-center w-10 h-10 border border-cyan-500/20 bg-gray-700 text-white rounded-lg hover:bg-gray-600 disabled:opacity-70"
        >
          <ArrowPathIcon
            class="w-6 h-6"
            :class="{ 'text-green-500 spin': isLoading, 'text-white': !isLoading }"
          />
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue';
import { apiStore } from '@/store/store';
import { useGuiderStore } from '@/store/guiderStore';
import apiService from '@/services/apiService';
import { ArrowPathIcon } from '@heroicons/vue/24/outline';

const props = defineProps({
  selectedGuiderDevice: { type: String, default: '' },
});

const store = apiStore();
const guiderStore = useGuiderStore();

// The equipment connect page identifies the guider by DisplayName ("PHD2 (Integrated)")
// or by Id ("PHD2_Integrated"); accept either.
const isIntegratedSelected = computed(
  () =>
    props.selectedGuiderDevice === 'PHD2_Integrated' ||
    props.selectedGuiderDevice === 'PHD2 (Integrated)'
);

const cameras = ref([]);
const selectedCamId = ref('');
const isLoading = ref(false);
const borderClass = ref('border-gray-500');

function updateBorder() {
  // The connect handler gates on guiderStore.guidecamOk in PINS mode, so keep it in
  // sync with whether a valid guide camera is selected.
  const ok = isIntegratedSelected.value && !!selectedCamId.value;
  guiderStore.guidecamOk = ok;
  borderClass.value = ok
    ? store.guiderInfo?.Connected
      ? 'border-green-500 connected-glow'
      : 'border-gray-500'
    : 'border-red-500 error-glow';
}

async function loadCurrent() {
  try {
    const res = await apiService.getIntegratedGuiderSelectedCamera();
    if (res?.Success && res.Response) {
      selectedCamId.value = res.Response.Id || '';
    }
  } catch (error) {
    console.error('Error reading selected guide camera:', error);
  } finally {
    updateBorder();
  }
}

async function loadCameras() {
  isLoading.value = true;
  try {
    const res = await apiService.getIntegratedGuiderCameras();
    if (res?.Success && Array.isArray(res.Response)) {
      cameras.value = res.Response.map((d) => ({
        id: d.Id,
        name: d.DisplayName || d.Name || d.Id,
      }));
    }
    updateBorder();
  } catch (error) {
    console.error('Error loading integrated guide cameras:', error);
    borderClass.value = 'border-red-500 error-glow';
  } finally {
    isLoading.value = false;
  }
}

async function setCamera() {
  if (!selectedCamId.value) return;
  try {
    await apiService.selectIntegratedGuiderCamera(selectedCamId.value);
    updateBorder();
  } catch (error) {
    console.error('Error selecting integrated guide camera:', error);
    borderClass.value = 'border-red-500 error-glow';
  }
}

watch(
  () => store.guiderInfo?.Connected,
  () => updateBorder()
);

watch(isIntegratedSelected, async (selected) => {
  if (selected) {
    await loadCurrent();
    await loadCameras();
  }
});

onMounted(async () => {
  if (isIntegratedSelected.value) {
    await loadCurrent();
    await loadCameras();
  }
});
</script>

<style scoped>
@keyframes error-glow {
  0% {
    box-shadow: 0 0 10px rgba(255, 0, 0, 0.5);
  }
  50% {
    box-shadow: 0 0 20px rgba(255, 0, 0, 0.8);
  }
  100% {
    box-shadow: 0 0 10px rgba(255, 0, 0, 0.5);
  }
}

.error-glow {
  animation: error-glow 1.5s infinite alternate;
}

.connected-glow {
  box-shadow: 0 0 6px rgba(34, 197, 94, 0.6);
}

.spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}
</style>

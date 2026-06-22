<template>
  <div class="container max-w-3xl mx-auto p-4">
    <h5 class="text-xl text-center font-bold text-white mb-4">
      {{ $t('components.guider.integrated.title') }}
    </h5>

    <div
      v-if="!store.guiderInfo.Connected"
      class="p-4 bg-red-500/10 border border-red-500/30 rounded-lg"
    >
      <p class="text-red-400 font-medium text-center">
        {{ $t('components.guider.notConnected') }}
      </p>
    </div>

    <div v-else class="space-y-4">
      <!-- Controls -->
      <div
        class="flex flex-col md:flex-row gap-1 md:space-x-4 border border-gray-700 rounded-lg bg-gradient-to-br from-gray-800 to-gray-900 shadow-lg p-5"
      >
        <ControlGuider />
        <button
          class="default-button-cyan"
          :disabled="store.guiderInfo.State !== 'Guiding' || isDithering"
          @click="dither"
        >
          {{ $t('components.guider.integrated.dither') }}
        </button>
      </div>

      <!-- Live guide image with lock + star overlay -->
      <div class="border border-gray-700 rounded-lg bg-black/60 overflow-hidden">
        <div v-if="imageUrl" class="relative inline-block w-full">
          <img :src="imageUrl" alt="Guide frame" class="block w-full select-none" />
          <!-- Lock position box -->
          <div
            v-if="hasFrame && igState.IsCalibrated"
            class="absolute border border-green-400 pointer-events-none"
            :style="lockStyle"
          ></div>
          <!-- Current star marker -->
          <div
            v-if="hasFrame && igState.StarFound"
            class="absolute border border-yellow-400 rounded-full pointer-events-none"
            :style="starStyle"
          ></div>
        </div>
        <p v-else class="text-gray-400 text-center text-sm p-6">
          {{ $t('components.guider.integrated.noImage') }}
        </p>
      </div>

      <!-- Status / calibration readout -->
      <div
        class="border border-gray-700 rounded-lg bg-gray-800/50 p-4 grid grid-cols-2 md:grid-cols-3 gap-x-6 gap-y-2 text-sm"
      >
        <div>
          <span class="text-gray-400">{{ $t('components.guider.statusLabel') }}:</span>
          <span class="text-white font-medium ml-1">{{ igState.State || '—' }}</span>
        </div>
        <div>
          <span class="text-gray-400">{{ $t('components.guider.integrated.error') }}:</span>
          <span class="text-white ml-1">{{ fmt(igState.ErrorPx) }} px</span>
        </div>
        <div>
          <span class="text-gray-400">{{ $t('components.guider.integrated.snr') }}:</span>
          <span class="text-white ml-1">{{ fmt(igState.Snr) }}</span>
        </div>
        <div>
          <span class="text-gray-400">{{ $t('components.guider.integrated.hfd') }}:</span>
          <span class="text-white ml-1">{{ fmt(igState.Hfd) }}</span>
        </div>
        <div>
          <span class="text-gray-400">{{ $t('components.guider.integrated.calibrated') }}:</span>
          <span class="text-white ml-1">{{ igState.IsCalibrated ? '✓' : '—' }}</span>
        </div>
        <div>
          <span class="text-gray-400">{{ $t('components.guider.integrated.pixelScale') }}:</span>
          <span class="text-white ml-1">{{ fmt(igState.PixelScale) }} "/px</span>
        </div>

        <template v-if="igState.IsCalibrated">
          <div>
            <span class="text-gray-400">{{ $t('components.guider.integrated.raRate') }}:</span>
            <span class="text-white ml-1">{{ fmt(igState.CalXRate, 4) }} px/ms</span>
          </div>
          <div>
            <span class="text-gray-400">{{ $t('components.guider.integrated.decRate') }}:</span>
            <span class="text-white ml-1">{{ fmt(igState.CalYRate, 4) }} px/ms</span>
          </div>
          <div>
            <span class="text-gray-400">{{ $t('components.guider.integrated.declination') }}:</span>
            <span class="text-white ml-1">{{ declinationDeg }}</span>
          </div>
        </template>
      </div>

      <!-- Guide graph -->
      <GuiderGraph />
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue';
import { apiStore } from '@/store/store';
import apiService from '@/services/apiService';
import ControlGuider from '@/components/guider/ControlGuider.vue';
import GuiderGraph from '@/components/guider/GuiderGraph.vue';

const store = apiStore();

const igState = reactive({
  State: '',
  IsCalibrated: false,
  PixelScale: 0,
  StarFound: false,
  StarX: 0,
  StarY: 0,
  LockX: 0,
  LockY: 0,
  ErrorPx: 0,
  Snr: 0,
  Hfd: 0,
  Mass: 0,
  FrameWidth: 0,
  FrameHeight: 0,
  CalXRate: 0,
  CalYRate: 0,
  CalDeclination: 997,
});

const imageUrl = ref(null);
const isDithering = ref(false);
let pollTimer = null;

const hasFrame = computed(() => igState.FrameWidth > 0 && igState.FrameHeight > 0);

const LOCK_BOX_PX = 24;
const STAR_BOX_PX = 16;

function overlayBox(x, y, sizePx) {
  if (!hasFrame.value) return {};
  return {
    left: `${(x / igState.FrameWidth) * 100}%`,
    top: `${(y / igState.FrameHeight) * 100}%`,
    width: `${sizePx}px`,
    height: `${sizePx}px`,
    transform: 'translate(-50%, -50%)',
  };
}

const lockStyle = computed(() => overlayBox(igState.LockX, igState.LockY, LOCK_BOX_PX));
const starStyle = computed(() => overlayBox(igState.StarX, igState.StarY, STAR_BOX_PX));

const declinationDeg = computed(() => {
  if (igState.CalDeclination >= 997) return '—';
  return `${((igState.CalDeclination * 180) / Math.PI).toFixed(1)}°`;
});

function fmt(v, digits = 2) {
  return typeof v === 'number' ? v.toFixed(digits) : '—';
}

async function dither() {
  isDithering.value = true;
  try {
    await apiService.guiderAction('dither');
  } catch (error) {
    console.error('Integrated guider dither failed:', error);
  } finally {
    isDithering.value = false;
  }
}

async function refresh() {
  if (!store.guiderInfo.Connected) return;
  try {
    const res = await apiService.getIntegratedGuiderState();
    if (res?.Success && res.Response) {
      Object.assign(igState, res.Response);
    }
    const url = await apiService.getIntegratedGuiderImage(1, 85);
    if (url) imageUrl.value = url;
  } catch (error) {
    // Endpoints 409 until a frame exists; ignore transient errors.
  }
}

onMounted(() => {
  refresh();
  pollTimer = setInterval(refresh, 1500);
});

onUnmounted(() => {
  if (pollTimer) clearInterval(pollTimer);
});
</script>

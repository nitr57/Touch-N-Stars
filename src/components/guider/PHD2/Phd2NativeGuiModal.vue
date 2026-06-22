<template>
  <teleport to="body">
    <div v-if="show" class="fixed inset-0 z-[60] flex flex-col bg-black/90 backdrop-blur-sm">
      <!-- Header bar -->
      <div
        class="flex items-center justify-between gap-2 px-4 py-2 bg-gray-900 border-b border-gray-700"
      >
        <h2 class="text-sm font-semibold text-gray-200 truncate">
          {{ $t('components.guider.phd2.nativeGui.title') }}
        </h2>
        <div class="flex items-center gap-2">
          <!-- Port input -->
          <div class="hidden sm:flex items-center gap-1">
            <label class="text-xs text-gray-400">
              {{ $t('components.guider.phd2.nativeGui.port') }}
            </label>
            <input
              v-model.number="settingsStore.guider.phd2NativeGuiPort"
              type="number"
              min="1"
              max="65535"
              class="w-20 px-2 py-1 text-xs bg-gray-800 border border-gray-600 rounded text-gray-200"
              @change="settingsStore.saveGuiderSettings()"
            />
          </div>

          <!-- Reload -->
          <button
            @click="reload"
            class="default-button-gray flex items-center justify-center w-9 h-9"
            :title="$t('components.guider.phd2.nativeGui.reload')"
          >
            <ArrowPathIcon class="w-5 h-5" />
          </button>

          <!-- Open in new tab -->
          <a
            :href="guiUrl"
            target="_blank"
            rel="noopener"
            class="default-button-gray flex items-center justify-center w-9 h-9"
            :title="$t('components.guider.phd2.nativeGui.openInTab')"
          >
            <ArrowTopRightOnSquareIcon class="w-5 h-5" />
          </a>

          <!-- Close -->
          <button
            @click="$emit('close')"
            class="default-button-red flex items-center justify-center w-9 h-9"
          >
            <XMarkIcon class="w-5 h-5" />
          </button>
        </div>
      </div>

      <!-- Iframe area -->
      <div class="relative flex-1 bg-black">
        <!-- Loading overlay -->
        <div
          v-if="loading && !error"
          class="absolute inset-0 z-10 flex flex-col items-center justify-center gap-3 text-gray-300 pointer-events-none"
        >
          <ArrowPathIcon class="w-8 h-8 animate-spin" />
          <span class="text-sm">{{
            starting
              ? $t('components.guider.phd2.nativeGui.starting')
              : $t('components.guider.phd2.nativeGui.loading')
          }}</span>
        </div>

        <!-- Error / unavailable overlay -->
        <div
          v-if="error"
          class="absolute inset-0 z-20 flex flex-col items-center justify-center gap-4 px-6 text-center"
        >
          <ExclamationTriangleIcon class="w-10 h-10 text-yellow-400" />
          <p class="text-sm text-gray-200 max-w-md">{{ error }}</p>
          <div class="flex flex-col items-stretch gap-1 w-full max-w-md">
            <label class="text-xs text-gray-400 text-left">
              {{ $t('components.guider.phd2.nativeGui.urlOverride') }}
            </label>
            <input
              v-model="settingsStore.guider.phd2NativeGuiUrlOverride"
              type="text"
              :placeholder="$t('components.guider.phd2.nativeGui.urlOverridePlaceholder')"
              class="w-full px-2 py-1 text-sm bg-gray-800 border border-gray-600 rounded text-gray-200"
              @change="settingsStore.saveGuiderSettings()"
            />
          </div>
          <button @click="init" class="default-button-cyan px-4 py-2 rounded-lg text-sm">
            {{ $t('components.guider.phd2.nativeGui.retry') }}
          </button>
        </div>

        <iframe
          v-if="show && showFrame"
          ref="frame"
          :key="reloadKey"
          :src="guiUrl"
          class="w-full h-full border-0"
          allow="clipboard-read; clipboard-write; fullscreen"
          @load="loading = false"
        ></iframe>

        <!-- Hint footer -->
        <div
          class="absolute bottom-0 left-0 right-0 px-4 py-1 text-[10px] text-gray-500 bg-gray-900/70 pointer-events-none"
        >
          {{ $t('components.guider.phd2.nativeGui.hint') }}
        </div>
      </div>
    </div>
  </teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue';
import { useSettingsStore } from '@/store/settingsStore';
import { useI18n } from 'vue-i18n';
import apiService from '@/services/apiService';
import {
  ArrowPathIcon,
  ArrowTopRightOnSquareIcon,
  XMarkIcon,
  ExclamationTriangleIcon,
} from '@heroicons/vue/24/outline';

const props = defineProps({
  show: { type: Boolean, default: false },
});
defineEmits(['close']);

const settingsStore = useSettingsStore();
const { t: $t } = useI18n();

const loading = ref(true);
const starting = ref(false);
const error = ref('');
const showFrame = ref(false);
const reloadKey = ref(0);
const frame = ref(null);

const hasOverride = computed(() => !!settingsStore.guider.phd2NativeGuiUrlOverride?.trim());

// Resolve the xpra HTML5 endpoint. An explicit override wins; otherwise build
// it from the same host the rest of the frontend talks to plus the configured
// xpra port. Mirrors the host/protocol resolution in apiService.
const guiUrl = computed(() => {
  const override = settingsStore.guider.phd2NativeGuiUrlOverride?.trim();
  if (override) return override;

  const protocol = settingsStore.backendProtocol || 'http';
  const host = settingsStore.connection.ip || window.location.hostname;
  const port = settingsStore.guider.phd2NativeGuiPort || 14500;
  return `${protocol}://${host}:${port}/`;
});

// Ask the backend to make sure an xpra/PHD2 session is up, then show the iframe.
// Falls back to a best-effort direct embed when the backend has no xpra support
// (older plugin, or an externally managed session reached via the URL override).
async function init() {
  loading.value = true;
  starting.value = false;
  error.value = '';
  showFrame.value = false;

  try {
    const status = await apiService.getPhd2GuiStatus();
    const info = status?.Response ?? {};

    if (!info.Available) {
      // Backend can't manage xpra (not Linux / not installed). Only embed if the
      // user pointed us at an externally managed session.
      if (hasOverride.value) {
        revealFrame();
      } else {
        loading.value = false;
        error.value = $t('components.guider.phd2.nativeGui.notAvailable');
      }
      return;
    }

    if (!info.Running) {
      starting.value = true;
      const res = await apiService.startPhd2Gui({
        port: settingsStore.guider.phd2NativeGuiPort,
      });
      starting.value = false;
      if (!res?.Success) {
        loading.value = false;
        error.value =
          res?.Error || $t('components.guider.phd2.nativeGui.notReachable', { url: guiUrl.value });
        return;
      }
    }
    revealFrame();
  } catch (e) {
    // No phd2-gui endpoint (older backend) — embed the configured URL directly.
    console.warn('[PHD2 GUI] status/start unavailable, embedding directly:', e?.message || e);
    revealFrame();
  }
}

function revealFrame() {
  error.value = '';
  loading.value = true;
  showFrame.value = true;
  reloadKey.value += 1;
}

function reload() {
  init();
}

watch(
  () => props.show,
  (open) => {
    if (open) {
      init();
    } else {
      showFrame.value = false;
    }
  }
);
</script>

<template>
  <div class="flex flex-col space-y-4">
    <button
      @click="$emit('capture')"
      :disabled="isCapturing"
      class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-6 rounded-full text-lg shadow-lg transition-all duration-300 transform hover:scale-105 disabled:opacity-50 disabled:cursor-not-allowed"
    >
      {{ isCapturing ? 'Mengambil...' : 'Ambil Foto' }}
    </button>
    <button
      @click="$emit('download')"
      :disabled="!canDownload"
      class="bg-green-600 hover:bg-green-700 text-white font-bold py-3 px-6 rounded-full text-lg shadow-lg transition-all duration-300 transform hover:scale-105 disabled:opacity-50 disabled:cursor-not-allowed"
    >
      Unduh Hasil
    </button>
    <button
      @click="$emit('reset')"
      v-if="canDownload"
      class="bg-red-600 hover:bg-red-700 text-white font-bold py-3 px-6 rounded-full text-lg shadow-lg transition-all duration-300 transform hover:scale-105"
    >
      Mulai Ulang
    </button>
  </div>
</template>

<script setup>

const props = defineProps({
  isCapturing: {
    type: Boolean,
    default: false
  },
  canDownload: {
    type: Boolean,
    default: false
  }
});

const emit = defineEmits(['capture', 'download', 'reset']);
</script>
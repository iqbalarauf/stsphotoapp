<template>
  <div class="relative w-full h-full bg-gray-900 rounded-lg overflow-hidden">
    <video ref="videoPlayer" autoplay class="w-full h-full object-cover"></video>
    <div v-if="!cameraReady" class="absolute inset-0 flex items-center justify-center bg-gray-800 bg-opacity-75 text-white text-lg">
      Mengakses kamera...
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const videoPlayer = ref(null);
const cameraReady = ref(false);

onMounted(async () => {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ video: true });
    videoPlayer.value.srcObject = stream;
    videoPlayer.value.onloadedmetadata = () => {
      cameraReady.value = true;
      videoPlayer.value.play();
    };
  } catch (error) {
    console.error("Gagal mengakses kamera:", error);
    alert("Gagal mengakses kamera. Pastikan Anda memberikan izin.");
  }
});

onUnmounted(() => {
  if (videoPlayer.value && videoPlayer.value.srcObject) {
    const stream = videoPlayer.value.srcObject;
    const tracks = stream.getTracks();
    tracks.forEach(track => track.stop());
  }
});

defineExpose({
  videoPlayer
});
</script>
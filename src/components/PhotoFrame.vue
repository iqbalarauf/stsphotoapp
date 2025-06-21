<template>
  <div class="relative w-full aspect-square overflow-hidden rounded-xl shadow-lg">
    <div class="absolute inset-0 z-10" v-if="flashActive" :class="['bg-white bg-opacity-80 animate-flash']"></div>

    <!-- Show captured image if available -->
    <img v-if="capturedImage" :src="capturedImage" class="object-cover w-full h-full" alt="captured" />
    <!-- Else show live video -->
    <video v-else ref="videoRef" autoplay playsinline muted class="object-cover w-full h-full"></video>

    <img :src="frameSrc"
      class="absolute top-0 left-0 w-full h-full object-none pointer-events-none" ref="frameImageRef" />
  </div>
</template>

<script setup>
import { ref, onMounted, defineExpose } from 'vue'

const props = defineProps({ frameSrc: String })
const videoRef = ref(null)
const frameImageRef = ref(null)
const capturedImage = ref(null)
const flashActive = ref(false)

onMounted(async () => {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ video: true })
    if (videoRef.value) {
      videoRef.value.srcObject = stream
      // Optional: Add a check to ensure video is loaded
      videoRef.value.onloadedmetadata = () => {
        console.log('Video stream loaded successfully.');
      };
    }
  } catch (err) {
    console.error('Error accessing camera:', err);
    // Tampilkan pesan ke user bahwa kamera tidak bisa diakses
    // Misalnya: capturedImage.value = '/path/to/error_image.png'
    // Atau tampilkan div error di template
  }
})

function setCapturedImage(dataURL) {
  capturedImage.value = dataURL
}

function triggerFlash() {
  flashActive.value = true
  setTimeout(() => flashActive.value = false, 300)
}

defineExpose({
  getVideo: () => videoRef.value,
  getFrame: () => frameImageRef.value,
  setCapturedImage,
  triggerFlash,
})
</script>

<style scoped>
@keyframes flash {
  0% {
    opacity: 0;
  }

  50% {
    opacity: 1;
  }

  100% {
    opacity: 0;
  }
}

.animate-flash {
  animation: flash 0.3s ease-in-out;
}
</style>

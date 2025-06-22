<template>
  <div class="bg-contain bg-center bg-fixed" :style="{ backgroundImage: 'url(' + wallpaperUrl + ')' }">
    <div class="max-w-screen flex flex-col items-center justify-center p-4">
      <h1 class="bg-black/75 rounded-lg p-4 text-3xl sm:text-4xl font-bold text-white mb-8 text-center">T.E.L.E. Party Booth!</h1>

      <div class="relative w-full max-w-sm sm:max-w-md lg:max-w-lg xl:max-w-xl rounded-lg shadow-xl overflow-hidden mb-8">
        <video v-if="photos.length < maxPhotos || shootingInProgress" ref="videoElement" class="w-full h-auto object-cover border-4 border-transparent video-border-animation" autoplay></video>
        <canvas ref="canvasElement" class="hidden"></canvas>
        <canvas ref="gridCanvasElement" class="hidden"></canvas>

        <div v-if="flashActive" class="absolute inset-0 bg-white opacity-0 animate-flash"></div>

        <div v-if="countdown > 0" class="absolute inset-0 flex items-center justify-center">
          <span class="text-black text-6xl sm:text-9xl font-bold animate-pulse" :style="{ 'text-shadow': '2px 2px 4px rgba(255, 255, 255, 0.7)' }">{{ countdown }}</span>
        </div>

        <div v-if="shootingInProgress" class="absolute top-4 right-4 bg-gray-800 text-white text-base sm:text-lg font-bold py-1 px-3 sm:py-2 sm:px-4 rounded-full">
          Foto ke-{{ photos.length + 1 }}
        </div>
      </div>

      <div v-if="photos.length < maxPhotos && !shootingInProgress" class="flex flex-col sm:flex-row space-y-4 sm:space-y-0 sm:space-x-4 mb-8 w-full max-w-sm sm:max-w-md xl:max-w-xl">
        <button
          @click="startPhotoSequence"
          :disabled="!cameraActive || shootingInProgress"
          class="bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out w-full"
        >
          Ambil 4 Foto Otomatis
        </button>
      </div>

      <div v-if="photos.length === maxPhotos" class="w-full max-w-sm sm:max-w-md lg:max-w-xl bg-gray-700 rounded-lg shadow-lg p-4">
        <h2 class="text-xl sm:text-2xl font-semibold text-white mb-4 text-center">Foto Grid Anda:</h2>
        <img :src="gridPhotoUrl" alt="Foto Grid" class="w-full h-auto rounded-md mb-4 object-contain" v-if="gridPhotoUrl" />
        <div class="flex flex-col sm:flex-row space-y-4 sm:space-y-0 sm:space-x-4 justify-center">
          <button
            @click="downloadGridPhoto"
            :disabled="!gridPhotoUrl"
            class="flex-1 bg-indigo-500 hover:bg-indigo-600 text-white font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out"
          >
            Unduh Foto Grid
          </button>
          <button
            @click="resetPhotos"
            class="flex-1 bg-yellow-500 hover:bg-yellow-600 text-white font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out"
          >
            Reset Foto
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';

// Import wallpaper
import wallpaper from './assets/wallpaper.jpg';
const wallpaperUrl = ref(wallpaper);

const frameImports = import.meta.glob('./assets/frame*.png', { eager: true, as: 'url' });
const frameUrls = Object.values(frameImports);

const videoElement = ref(null);
const canvasElement = ref(null);
const gridCanvasElement = ref(null);
const photos = ref([]);
const gridPhotoUrl = ref('');
const maxPhotos = 4;
const cameraActive = ref(false);
const countdown = ref(0);
const shootingInProgress = ref(false);
const flashActive = ref(false);

let mediaStream = null;
let countdownInterval = null;

const targetPhotoSize = 480;

const startCamera = async () => {
  try {
    mediaStream = await navigator.mediaDevices.getUserMedia({
      video: {
        width: { ideal: targetPhotoSize },
        height: { ideal: targetPhotoSize },
        facingMode: 'user'
      }
    });
    videoElement.value.srcObject = mediaStream;
    cameraActive.value = true;
    photos.value = [];
    gridPhotoUrl.value = '';
  } catch (error) {
    console.error('Error accessing camera:', error);
    alert('Tidak dapat mengakses kamera. Pastikan Anda memberikan izin dan kamera tidak sedang digunakan.');
  }
};

const triggerFlash = () => {
  flashActive.value = true;
  setTimeout(() => {
    flashActive.value = false;
  }, 200);
};

const takePhoto = async () => {
  if (!videoElement.value || !canvasElement.value) {
    return;
  }

  triggerFlash();

  const video = videoElement.value;
  const canvas = canvasElement.value;
  const context = canvas.getContext('2d');

  canvas.width = targetPhotoSize;
  canvas.height = targetPhotoSize;

  context.drawImage(video, 0, 0, canvas.width, canvas.height);

  const currentPhotoIndex = photos.value.length;
  if (currentPhotoIndex < frameUrls.length) {
    const frameImage = new Image();
    frameImage.src = frameUrls[currentPhotoIndex];
    
    await new Promise(resolve => {
      frameImage.onload = () => {
        context.drawImage(frameImage, 0, 0, canvas.width, canvas.height);
        resolve();
      };
      frameImage.onerror = (e) => {
        console.error('Error loading frame image:', frameImage.src, e);
        resolve();
      };
    });
  } else {
    console.warn(`Tidak ada bingkai untuk foto ke-${currentPhotoIndex + 1}. Pastikan ada ${maxPhotos} bingkai.`);
  }

  const newPhotoUrl = canvas.toDataURL('image/png');
  photos.value.push(newPhotoUrl);
};

const startPhotoSequence = async () => {
  if (!cameraActive.value || photos.value.length === maxPhotos || shootingInProgress.value) {
    return;
  }

  shootingInProgress.value = true;
  photos.value = [];
  gridPhotoUrl.value = '';

  for (let i = 0; i < maxPhotos; i++) {
    countdown.value = 3;
    await new Promise(resolve => {
      countdownInterval = setInterval(() => {
        countdown.value--;
        if (countdown.value === 0) {
          clearInterval(countdownInterval);
          resolve();
        }
      }, 1000);
    });

    await takePhoto();
    await new Promise(resolve => setTimeout(resolve, 500));
  }
  shootingInProgress.value = false;
  countdown.value = 0;
};

const combinePhotosIntoGrid = async () => {
  if (photos.value.length !== maxPhotos) return;

  const gridCanvas = gridCanvasElement.value;
  const ctx = gridCanvas.getContext('2d');

  const photoWidth = targetPhotoSize;
  const photoHeight = targetPhotoSize;

  gridCanvas.width = photoWidth * 2;
  gridCanvas.height = photoHeight * 2;

  const imagesToLoad = photos.value.map(src => {
    const img = new Image();
    img.src = src;
    return new Promise(resolve => img.onload = () => resolve(img));
  });

  const loadedImages = await Promise.all(imagesToLoad);

  const positions = [
    { x: 0, y: 0 },
    { x: photoWidth, y: 0 },
    { x: 0, y: photoHeight },
    { x: photoWidth, y: photoHeight }
  ];

  loadedImages.forEach((img, index) => {
    const pos = positions[index];
    ctx.drawImage(img, pos.x, pos.y, photoWidth, photoHeight);
  });

  gridPhotoUrl.value = gridCanvas.toDataURL('image/png');
};

const downloadGridPhoto = () => {
  if (gridPhotoUrl.value) {
    const link = document.createElement('a');
    link.href = gridPhotoUrl.value;
    link.download = 'photobooth_grid_image.png';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }
};

const resetPhotos = () => {
  photos.value = [];
  gridPhotoUrl.value = '';
  shootingInProgress.value = false;
  countdown.value = 0;
  clearInterval(countdownInterval);
  flashActive.value = false;
  startCamera();
};

watch(photos, (newPhotos) => {
  if (newPhotos.length === maxPhotos) {
    combinePhotosIntoGrid();
  }
}, { deep: true });

onMounted(() => {
  startCamera();
});

onBeforeUnmount(() => {
  if (mediaStream) {
    mediaStream.getTracks().forEach(track => track.stop());
    videoElement.value.srcObject = null;
    cameraActive.value = false;
  }
  clearInterval(countdownInterval);
});
</script>

<style>
/* Keyframes untuk animasi flash */
@keyframes flash {
  0% { opacity: 0; }
  10% { opacity: 1; }
  100% { opacity: 0; }
}

.animate-flash {
  animation: flash 0.2s ease-out forwards;
}

/* Animasi border warna-warni */
@keyframes border-glow {
  0% { border-color: #ef4444; } /* Red 500 */
  25% { border-color: #3b82f6; } /* Blue 500 */
  50% { border-color: #22c55e; } /* Green 500 */
  75% { border-color: #eab308; } /* Yellow 500 */
  100% { border-color: #ef4444; } /* Kembali ke Red 500 */
}

.video-border-animation {
  animation: border-glow 5s infinite linear;
}

.object-cover {
  object-fit: cover;
}
</style>
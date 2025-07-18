<template>
  <div class="max-h-screen relative overflow-auto">
    <div class="absolute top-0 bottom-0 z-0 bg-contain bg-center min-h-screen mx-auto left-0 right-0"
      :style="{ backgroundImage: 'url(' + wallpaperUrl + ')', aspectRatio: '16 / 9', height: 'auto' }"></div>
    <div class="relative">
      <div v-if="showWelcomePopup" class="fixed inset-0 flex items-center justify-center z-50 p-4">
        <div class="bg-gray-300 rounded-lg shadow-xl p-6 max-w-sm w-full text-center">
          <h2 class="text-xl font-bold text-gray-900 mb-4">Informasi Mengenai Photobooth Let's Party On!</h2>
          <p class="text-gray-700 mb-6 text-justify">
            1. Photobooth ini akan mengambil sebanyak 6 foto untuk dua frame. Hasil foto dapat kamu unduh setelah
            selesai.
          </p>
          <p class="text-gray-700 mb-6 text-justify">
            2. Data foto kamu <b>tidak akan disimpan di server Onielity</b>. Semua proses dan penyimpanan hanya terjadi
            di perangkat
            kamu.
          </p>
          <p class="text-gray-700 mb-6 text-justify">
            3. Kami merekomendasikan kamu untuk menggunakan Google Chrome (baik versi mobile ataupun desktop)
            kamu.
          </p>
          <button @click="dismissWelcomePopup"
            class="bg-black dark:bg-indigo-600 dark:text-white text-black font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out">
            OK
          </button>
        </div>
      </div>
    </div>
    <div class="flex flex-col items-center p-4 relative z-10 max-w-[600px] mx-auto min-h-screen">
      <img :src="logoUrl" alt="Vue Photobooth Logo" class="h-24 max-w-full object-contain" />

      <div
        class="relative w-full max-w-sm sm:max-w-md lg:max-w-lg xl:max-w-xl bg-gray-800 rounded-lg shadow-xl overflow-hidden mb-8 mt-4">
        <video v-if="photos.length < maxPhotos || shootingInProgress" ref="videoElement"
          class="w-full h-auto object-cover border-4 border-transparent video-border-animation" autoplay></video>
        <canvas ref="canvasElement" class="hidden"></canvas>
        <canvas ref="gridCanvasElement" class="hidden"></canvas>

        <div v-if="flashActive" class="absolute inset-0 bg-white opacity-0 animate-flash"></div>

        <div v-if="countdown > 0" class="absolute inset-0 flex items-center justify-center">
          <span class="text-black text-6xl sm:text-9xl font-bold animate-pulse"
            :style="{ 'text-shadow': '2px 2px 4px rgba(255, 255, 255, 0.7)' }">{{ countdown }}</span>
        </div>

        <div v-if="shootingInProgress"
          class="absolute top-4 right-4 bg-gray-800 text-white text-base sm:text-lg font-bold py-1 px-3 sm:py-2 sm:px-4 rounded-full">
          Foto ke-{{ photos.length + 1 }}
        </div>
      </div>

      <div v-if="photos.length < maxPhotos && !shootingInProgress"
        class="flex flex-col sm:flex-row space-y-4 sm:space-y-0 sm:space-x-4 mb-8 w-full max-w-sm sm:max-w-md xl:max-w-xl">
        <button @click="startPhotoSequence" :disabled="!cameraActive || shootingInProgress"
          class="bg-green-500 hover:bg-green-600 dark:text-white text-black font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out w-full">
          Ambil Foto
        </button>
      </div>

      <div v-if="photos.length === maxPhotos"
        class="w-full max-w-sm sm:max-w-md lg:max-w-xl bg-gray-300 rounded-lg shadow-lg p-4">
        <h2 class="text-xl sm:text-2xl font-semibold text-black mb-4 text-center">Foto Anda</h2>
        <img :src="gridPhotoUrl" alt="Foto Grid" class="w-full h-auto rounded-md mb-4 object-contain"
          v-if="gridPhotoUrl" />
        <div class="flex flex-col sm:flex-row space-y-4 sm:space-y-0 sm:space-x-4 justify-center">
          <button @click="downloadGridPhoto" :disabled="!gridPhotoUrl"
            class="flex-1 bg-indigo-500 hover:bg-indigo-600 dark:text-white text-black font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out">
            Unduh Foto
          </button>
          <button @click="resetPhotos"
            class="flex-1 bg-yellow-500 hover:bg-yellow-600 dark:text-white text-black font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out">
            Retake
          </button>
        </div>
      </div>
    </div>

    <footer class="fixed bottom-0 left-0 right-0 py-4 text-center text-white z-20">
      <p class="text-xs">&copy; {{ currentYear }}, <a href="https://corsyava.com" target="__blank">Onielity Official</a>
      </p>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';

import wallpaper from './assets/background.png';
const wallpaperUrl = ref(wallpaper);

import logo from './assets/logo.png';
const logoUrl = ref(logo);
import flogo from './assets/footer-logo.png';
const furllogo = ref(flogo);

const currentYear = new Date().getFullYear()

// Impor frame akhir untuk grid
import finalGridFrame from './assets/framegrid.png';
const finalGridFrameUrl = ref(finalGridFrame);

const videoElement = ref(null);
const canvasElement = ref(null);
const gridCanvasElement = ref(null);
const photos = ref([]);
const maxPhotos = 6;
const gridPhotoUrl = ref('');
const cameraActive = ref(false);
const countdown = ref(0);
const shootingInProgress = ref(false);
const flashActive = ref(false);

// State baru untuk mengontrol tampilan popup
const showWelcomePopup = ref(false);

let mediaStream = null;
let countdownInterval = null;

const targetPhotoSize = 1080; // Tetap 1080 untuk resolusi pengambilan foto individual

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

  // 1. Muat gambar frame akhir untuk mendapatkan dimensinya
  const finalFrameImage = new Image();
  finalFrameImage.src = finalGridFrameUrl.value;

  await new Promise(resolve => {
    finalFrameImage.onload = () => resolve();
    finalFrameImage.onerror = (e) => {
      console.error('Error loading final frame image:', finalFrameImage.src, e);
      // Lanjutkan proses meskipun frame gagal dimuat, tetapi grid mungkin tidak sesuai
      resolve();
    };
  });

  // Pastikan frame sudah dimuat dan memiliki dimensi
  if (!finalFrameImage.width || !finalFrameImage.height) {
    console.error("Gambar frame akhir gagal dimuat atau tidak memiliki dimensi. Tidak dapat membuat grid.");
    // Tetapkan resolusi fallback jika frame tidak dimuat, agar tidak crash
    gridCanvas.width = targetPhotoSize * 2; // Default 2 kolom * 1080
    gridCanvas.height = targetPhotoSize * 3; // Default 3 baris * 1080
  } else {
    // 2. Atur dimensi canvas grid sesuai dengan dimensi asli frame
    gridCanvas.width = finalFrameImage.width;
    gridCanvas.height = finalFrameImage.height;
  }

  // --- BAGIAN PENTING: DEFINISIKAN SLOT FOTO ANDA DI SINI ---
  // Anda HARUS mengganti nilai-nilai ini dengan koordinat dan dimensi asli
  // dari area transparan di 'framegrid.png' Anda.
  // Urutan array ini harus sesuai dengan urutan pengambilan foto (0-5).
  const slotDefinitions = [
    // Contoh dummy data: GANTI INI DENGAN NILAI NYATA!
    // { x: 100, y: 50, width: 900, height: 900 },  // Slot untuk Foto 1 (indeks 0)
    // { x: 1150, y: 50, width: 900, height: 900 }, // Slot untuk Foto 2 (indeks 1)
    // { x: 100, y: 1000, width: 900, height: 900 },// Slot untuk Foto 3 (indeks 2)
    // { x: 1150, y: 1000, width: 900, height: 900 },// Slot untuk Foto 4 (indeks 3)
    // { x: 100, y: 1950, width: 900, height: 900 },// Slot untuk Foto 5 (indeks 4)
    // { x: 1150, y: 1950, width: 900, height: 900 },// Slot untuk Foto 6 (indeks 5)

    // Contoh nilai berdasarkan asumsi 2160x3240 frame dan 6 slot 1080x1080 yang pas
    // Jika frame Anda memiliki margin, Anda harus sesuaikan nilai x, y, width, height ini.
    { x: 45, y: 260, width: 500, height: 500 },
    { x: (gridCanvas.width / 2) + 45, y: 260, width: 500, height: 500 },
    { x: 45, y: 670, width: 500, height: 500 },
    { x: (gridCanvas.width / 2) + 45, y: 670, width: 500, height: 500 },
    { x: 45, y: 1120, width: 500, height: 500 },
    { x: (gridCanvas.width / 2) + 45, y: 1120, width: 500, height: 500 },
  ];
  // --- AKHIR BAGIAN DEFINISI SLOT FOTO ---


  const imagesToLoad = photos.value.map(src => {
    const img = new Image();
    img.src = src;
    return new Promise(resolve => img.onload = () => resolve(img));
  });

  const loadedImages = await Promise.all(imagesToLoad);

  loadedImages.forEach((img, index) => {
    if (slotDefinitions[index]) {
      const slot = slotDefinitions[index];
      // Gambar setiap foto (yang awalnya 1080x1080) ke dalam slot yang ditentukan.
      // Ini akan menskalakan/meregangkan foto individual agar sesuai dengan slot frame.
      ctx.drawImage(img, slot.x, slot.y, slot.width, slot.height);
    } else {
      console.warn(`Definisi slot tidak ditemukan untuk foto ke-${index + 1}.`);
    }
  });

  // Gambar frame akhir di atas foto yang sudah digabung
  // Ini akan menutupi seluruh canvas grid sesuai dengan ukuran asli frame
  if (finalFrameImage.width && finalFrameImage.height) {
    ctx.drawImage(finalFrameImage, 0, 0, gridCanvas.width, gridCanvas.height);
  }

  // Perbarui nama file unduhan agar sesuai dengan resolusi aktual dari canvas grid
  gridPhotoUrl.value = gridCanvas.toDataURL('image/png', 1.0);
};

const downloadGridPhoto = () => {
  if (gridPhotoUrl.value) {
    const link = document.createElement('a');
    link.href = gridPhotoUrl.value;
    const canvas = gridCanvasElement.value;
    link.download = `photobooth_grid_image_${canvas.width}x${canvas.height}.png`;
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

// Fungsi untuk menutup popup
const dismissWelcomePopup = () => {
  showWelcomePopup.value = false;
  startCamera(); // Memulai kamera setelah popup ditutup
};

watch(photos, (newPhotos) => {
  if (newPhotos.length === maxPhotos) {
    combinePhotosIntoGrid();
  }
}, { deep: true });

onMounted(() => {
  showWelcomePopup.value = true;
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
  0% {
    opacity: 0;
  }

  10% {
    opacity: 1;
  }

  100% {
    opacity: 0;
  }
}

.animate-flash {
  animation: flash 0.2s ease-out forwards;
}

/* Animasi border warna-warni */
@keyframes border-glow {
  0% {
    border-color: #ef4444;
  }

  /* Red 500 */
  25% {
    border-color: #3b82f6;
  }

  /* Blue 500 */
  50% {
    border-color: #22c55e;
  }

  /* Green 500 */
  75% {
    border-color: #eab308;
  }

  /* Yellow 500 */
  100% {
    border-color: #ef4444;
  }

  /* Kembali ke Red 500 */
}

.video-border-animation {
  animation: border-glow 5s infinite linear;
}

/* Pastikan elemen root dan body tidak memiliki scrolling */
html,
body {
  overflow: auto;
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
}

#app {
  width: 100%;
  height: 100%;
  overflow: auto;
}
</style>
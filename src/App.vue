<template>
  <div class="max-h-screen relative overflow-auto">
    <div class="fixed inset-0 z-0 bg-cover bg-no-repeat min-h-screen"
      :style="{ backgroundImage: 'url(' + wallpaperUrl + ')', backgroundPosition: 'center center' }"></div>

    <div v-if="waitingRoomActive" class="relative flex flex-col items-center justify-center min-h-screen p-4">
      <img :src="displayLogo" alt="Logo" class="h-48 max-w-full object-contain mb-4" />
      <p class="text-white text-lg sm:text-xl font-bold text-center">
        {{ message }}
      </p>
    </div>

    <div v-else>
      <div v-if="showWelcomePopup" class="fixed inset-0 flex items-center justify-center z-50 p-4">
        <div class="bg-gray-300 rounded-lg shadow-xl p-6 max-w-sm w-full text-center">
          <h2 class="text-xl font-bold text-gray-900 mb-4">Informasi Mengenai Photobooth #24thJourNiel</h2>
          <p class="text-gray-700 mb-6 text-justify">
            1. Photobooth ini akan mengambil sebanyak 6 foto untuk dua frame. Hasil foto dapat kamu unduh setelah
            selesai.
          </p>
          <p class="text-gray-700 mb-6 text-justify">
            2. Data foto kamu <b>tidak akan disimpan di server Onielity</b>. Semua proses dan penyimpanan hanya terjadi
            di perangkat
            kamu.
          </p>
          <button @click="dismissWelcomePopup"
            class="bg-black dark:bg-indigo-600 dark:text-white text-black font-bold py-3 px-6 rounded-lg shadow-md transition duration-300 ease-in-out">
            OK
          </button>
        </div>
      </div>
      <div class="flex flex-col items-center p-4 pb-24 relative max-w-[360px] mx-auto min-h-screen">
        <img :src="logoUrl" alt="Vue Photobooth Logo" class="h-24 max-w-full object-contain" />

        <div
          class="relative w-full max-w-sm sm:max-w-md lg:max-w-lg xl:max-w-xl bg-gray-800 rounded-lg shadow-xl overflow-hidden mb-8 mt-4">
          <video v-if="photos.length < maxPhotos || shootingInProgress" ref="videoElement"
            class="w-full h-auto object-cover border-4 border-transparent video-border-animation" autoplay playsinline
            muted></video>
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
          <h2 class="text-xl sm:text-2xl font-semibold text-black mb-4 text-center">Preview Foto</h2>
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
        <footer class="fixed bottom-0 left-0 right-0 z-20 py-4 text-center text-white">
          <img :src="furllogo" alt="Footer Logo" class="h-16 mx-auto object-contain" />
          <p class="text-xs">&copy; {{ currentYear }}, <a href="https://corsyava.id" target="__blank">Onielity
              Official</a>
          </p>
        </footer>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch, nextTick } from 'vue';

import wallpaper from './assets/background.jpeg';
const wallpaperUrl = ref(wallpaper);

import logo from './assets/logo.png';
const logoUrl = ref(logo);
import flogo from './assets/footer-logo.png';
const furllogo = ref(flogo);

// Import logo kedua (pastikan Anda memiliki file ini)
import secondLogo from './assets/footer-logo.png'; // Ganti dengan path logo kedua Anda
const secondLogoUrl = ref(secondLogo);

const currentYear = new Date().getFullYear();

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

// State baru untuk waiting room
const waitingRoomActive = ref(false);
const message = ref('');
const displayLogo = ref('');

let mediaStream = null; // Deklarasikan sebagai `let` agar bisa di-reassign ke `null`
let countdownInterval = null;

const targetPhotoSize = 1080; // Tetap 1080 untuk resolusi pengambilan foto individual

const getBrowserDiagnostics = () => {
  const ua = navigator.userAgent || '';
  const isSafari = /safari/i.test(ua) && !/chrome|chromium|android/i.test(ua);
  const isIOS = /iPhone|iPad|iPod/i.test(ua);

  return {
    ua,
    isSafari,
    isIOS,
    hasMediaDevices: !!navigator.mediaDevices,
    hasGetUserMedia: !!navigator.mediaDevices?.getUserMedia,
    isSecureContext: window.isSecureContext,
  };
};

const resolveCameraErrorMessage = (error, diagnostics) => {
  const errorName = error?.name || 'UnknownError';

  if (!diagnostics.isSecureContext) {
    return 'Kamera hanya bisa diakses di koneksi HTTPS atau localhost.';
  }

  if (!diagnostics.hasGetUserMedia) {
    return 'Browser ini belum mendukung akses kamera (getUserMedia).';
  }

  if (errorName === 'NotAllowedError' || errorName === 'SecurityError') {
    return 'Akses kamera ditolak. Buka pengaturan browser dan izinkan kamera untuk situs ini.';
  }

  if (errorName === 'NotReadableError' || errorName === 'TrackStartError') {
    return 'Kamera sedang dipakai aplikasi lain. Tutup aplikasi lain lalu coba lagi.';
  }

  if (errorName === 'OverconstrainedError' || errorName === 'ConstraintNotSatisfiedError') {
    return 'Konfigurasi kamera tidak didukung perangkat ini. Sistem akan mencoba mode kompatibel.';
  }

  if (errorName === 'NotFoundError' || errorName === 'DevicesNotFoundError') {
    return 'Tidak ada kamera yang terdeteksi pada perangkat ini.';
  }

  return 'Tidak dapat mengakses kamera. Pastikan izin kamera aktif dan kamera tidak dipakai aplikasi lain.';
};

const requestCameraStream = async (diagnostics) => {
  const safariFriendlyConstraints = [
    { video: { facingMode: { ideal: 'user' } }, audio: false },
    { video: true, audio: false },
  ];

  const defaultConstraints = {
    video: {
      width: { ideal: targetPhotoSize },
      height: { ideal: targetPhotoSize },
      facingMode: 'user'
    },
    audio: false,
  };

  const constraintCandidates = diagnostics.isSafari
    ? [
      ...safariFriendlyConstraints,
      defaultConstraints,
    ]
    : [defaultConstraints, ...safariFriendlyConstraints];

  let lastError = null;

  for (const constraints of constraintCandidates) {
    try {
      return await navigator.mediaDevices.getUserMedia(constraints);
    } catch (error) {
      lastError = error;
      console.warn('getUserMedia gagal dengan constraints:', constraints, error);
    }
  }

  throw lastError || new Error('Semua percobaan akses kamera gagal.');
};

const startCamera = async () => {
  try {
    const diagnostics = getBrowserDiagnostics();

    if (!diagnostics.hasGetUserMedia) {
      throw new Error('getUserMedia tidak tersedia pada browser ini.');
    }

    await nextTick();

    if (!videoElement.value) {
      throw new Error('Video element belum siap.');
    }

    // Jika stream lama masih ada, cukup pasang ulang ke elemen video.
    if (mediaStream && mediaStream.getTracks().some(track => track.readyState === 'live')) {
      videoElement.value.srcObject = mediaStream;
      cameraActive.value = true;
      photos.value = [];
      gridPhotoUrl.value = '';
      return;
    }

    // Hentikan dan bersihkan stream yang ada sebelum meminta yang baru
    if (mediaStream) {
      mediaStream.getTracks().forEach(track => track.stop());
      videoElement.value.srcObject = null;
      mediaStream = null; // Penting: reset referensi di sini
      cameraActive.value = false;
    }

    mediaStream = await requestCameraStream(diagnostics);

    videoElement.value.srcObject = mediaStream;
    await videoElement.value.play().catch((playError) => {
      console.warn('Pemutaran video kamera gagal otomatis:', playError);
    });
    cameraActive.value = true;
    photos.value = [];
    gridPhotoUrl.value = '';

    // Tunggu video untuk memuat metadata sebelum menandainya siap
    // Ini penting untuk Safari yang mungkin membutuhkan waktu lebih lama untuk inisialisasi
    await new Promise(resolve => {
      videoElement.value.onloadedmetadata = () => {
        resolve();
      };
      // Timeout jika onloadedmetadata tidak pernah terpanggil (misal ada error lain)
      setTimeout(() => {
        console.warn("Video metadata did not load in time.");
        resolve();
      }, 3000); // Batas waktu 3 detik
    });

  } catch (error) {
    const diagnostics = getBrowserDiagnostics();
    const errorName = error?.name || 'UnknownError';
    const userMessage = resolveCameraErrorMessage(error, diagnostics);

    console.error('Error accessing camera:', {
      errorName,
      error,
      diagnostics,
    });

    alert(`${userMessage}\n\nKode error: ${errorName}\nBrowser: ${diagnostics.ua}`);
    cameraActive.value = false; // Setel ke false jika gagal
  }
};

const triggerFlash = () => {
  flashActive.value = true;
  setTimeout(() => {
    flashActive.value = false;
  }, 200);
};

const takePhoto = async () => {
  if (!videoElement.value || !canvasElement.value || !cameraActive.value) {
    console.warn("Cannot take photo: video element or canvas not ready, or camera not active.");
    return;
  }

  triggerFlash();

  const video = videoElement.value;
  const canvas = canvasElement.value;
  const context = canvas.getContext('2d');

  canvas.width = targetPhotoSize;
  canvas.height = targetPhotoSize;

  // Pastikan video sedang memutar frame yang valid
  if (video.readyState < video.HAVE_CURRENT_DATA) {
    console.warn("Video is not ready to draw frame.");
    // Anda mungkin ingin menambahkan penundaan atau mencoba lagi
    await new Promise(resolve => setTimeout(resolve, 100)); // Coba lagi setelah sedikit penundaan
    if (video.readyState < video.HAVE_CURRENT_DATA) {
        console.error("Video still not ready after retry, skipping photo.");
        return;
    }
  }

  context.drawImage(video, 0, 0, canvas.width, canvas.height);

  const newPhotoUrl = canvas.toDataURL('image/png');
  photos.value.push(newPhotoUrl);
};

const startPhotoSequence = async () => {
  if (!cameraActive.value || photos.value.length === maxPhotos || shootingInProgress.value) {
    console.warn("Cannot start photo sequence: camera not active, max photos reached, or shooting already in progress.");
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
    await new Promise(resolve => setTimeout(resolve, 500)); // Jeda antar foto
  }
  shootingInProgress.value = false;
  countdown.value = 0;
};

const combinePhotosIntoGrid = async () => {
  if (photos.value.length !== maxPhotos) return;

  const gridCanvas = gridCanvasElement.value;
  const ctx = gridCanvas.getContext('2d');

  const finalFrameImage = new Image();
  finalFrameImage.src = finalGridFrameUrl.value;

  await new Promise(resolve => {
    finalFrameImage.onload = () => resolve();
    finalFrameImage.onerror = (e) => {
      console.error('Error loading final frame image:', finalFrameImage.src, e);
      // Lanjutkan proses meskipun frame gagal dimuat
      resolve();
    };
  });

  if (!finalFrameImage.width || !finalFrameImage.height) {
    console.error("Gambar frame akhir gagal dimuat atau tidak memiliki dimensi. Tidak dapat membuat grid. Menggunakan fallback.");
    gridCanvas.width = targetPhotoSize * 2;
    gridCanvas.height = targetPhotoSize * 3;
  } else {
    gridCanvas.width = finalFrameImage.width;
    gridCanvas.height = finalFrameImage.height;
  }

  // --- BAGIAN PENTING: DEFINISIKAN SLOT FOTO ANDA DI SINI ---
  // Anda HARUS mengganti nilai-nilai ini dengan koordinat dan dimensi asli
  // dari area transparan di 'framegrid.png' Anda.
  // Urutan array ini harus sesuai dengan urutan pengambilan foto (0-5).
  const slotDefinitions = [
    // Contoh nilai berdasarkan asumsi 2160x3240 frame dan 6 slot 1080x1080 yang pas
    // Jika frame Anda memiliki margin, Anda harus sesuaikan nilai x, y, width, height ini.
    { x: 45, y: 200, width: 500, height: 500 },
    { x: (gridCanvas.width / 2) + 45, y: 200, width: 500, height: 500 },
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
      ctx.drawImage(img, slot.x, slot.y, slot.width, slot.height);
    } else {
      console.warn(`Definisi slot tidak ditemukan untuk foto ke-${index + 1}.`);
    }
  });

  if (finalFrameImage.width && finalFrameImage.height) {
    ctx.drawImage(finalFrameImage, 0, 0, gridCanvas.width, gridCanvas.height);
  }

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

  nextTick(() => {
    // Pastikan elemen video sudah terpasang ulang sebelum mengakses kamera.
    startCamera();
  });
};

const dismissWelcomePopup = () => {
  showWelcomePopup.value = false;
  startCamera();
};

watch(photos, (newPhotos) => {
  if (newPhotos.length === maxPhotos) {
    combinePhotosIntoGrid();
  }
}, { deep: true });

onMounted(() => {
  const jakartaOffset = 7 * 60; // UTC+7 for Jakarta in minutes
  const now = new Date();
  const utc = now.getTime() + (now.getTimezoneOffset() * 60000); // Current UTC time in milliseconds
  const jakartaTime = new Date(utc + (jakartaOffset * 60000)); // Current Jakarta time in milliseconds

  // Define the target dates in Jakarta time (UTC+7)
  const comingSoonDate = new Date('2026-07-01T00:00:00+07:00'); // July 26, 2026, 00:00 Jakarta time
  const thankYouDate = new Date('2026-08-26T23:59:59+07:00'); // August 26, 2026, 23:59 Jakarta time

  if (jakartaTime < comingSoonDate) {
    waitingRoomActive.value = true;
    message.value = 'Coming Soon. July 1st, 2026';
    displayLogo.value = logoUrl.value;
  } else if (jakartaTime > thankYouDate) {
    waitingRoomActive.value = true;
    message.value = 'Thank you, see you at next event!';
    displayLogo.value = secondLogoUrl.value;
  } else {
    showWelcomePopup.value = true;
    // Kamera akan dimulai oleh dismissWelcomePopup setelah user klik OK
  }
});

onBeforeUnmount(() => {
  if (mediaStream) {
    mediaStream.getTracks().forEach(track => track.stop());
    videoElement.value.srcObject = null;
    mediaStream = null; // Pastikan mediaStream di-null-kan di sini
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

/* Drop shadow for text */
.drop-shadow-lg {
  text-shadow: 0 4px 6px rgba(0, 0, 0, 0.4);
}
</style>
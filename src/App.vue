<template>
  <div class="min-h-screen bg-gray-900 text-white flex flex-col items-center justify-center gap-6 p-4">
    <!-- Semua frame hanya tampil jika belum final preview -->
    <div v-if="!finalImage">
      <!-- Frame Live -->
      <PhotoFrame v-if="currentCaptureIndex < 4" :key="currentCaptureIndex" :ref="`frame${currentCaptureIndex + 1}`"
        :frameSrc="framesSrc[currentCaptureIndex]" mode="live" />
      <!-- Captured -->
      <div v-if="currentCaptureIndex > 0" class="grid grid-cols-2 gap-4 mt-6">
        <PhotoFrame v-for="(src, i) in framesSrc" v-if="i < currentCaptureIndex" :key="i" :ref="`frame${i + 1}`"
          :frameSrc="src" mode="captured" />
      </div>
    </div>

    <!-- Tombol -->
    <div v-if="!finalImage" class="flex flex-col items-center gap-3">
      <button @click="startCapture" :disabled="isCapturing || currentCaptureIndex >= 4"
        class="bg-green-500 hover:bg-green-600 disabled:opacity-50 px-6 py-2 rounded-lg text-lg font-bold shadow-md transition">
        📸 Capture Frame {{ currentCaptureIndex + 1 }}
      </button>
      <div v-if="countdown > 0" class="text-5xl font-bold animate-pulse">{{ countdown }}</div>
    </div>

    <!-- Preview -->
    <div v-if="finalImage" class="flex flex-col items-center gap-4">
      <img :src="finalImage" alt="Preview" class="rounded-lg shadow-lg max-w-full w-[600px]" />
      <div class="flex gap-4">
        <button @click="downloadImage" class="bg-blue-500 hover:bg-blue-600 px-6 py-2 rounded-lg font-bold">
          💾 Download
        </button>
        <button @click="resetCapture" class="bg-red-500 hover:bg-red-600 px-6 py-2 rounded-lg font-bold">
          🔁 Retake
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import PhotoFrame from './components/PhotoFrame.vue'

const framesSrc = [
  '/src/assets/1.png',
  '/src/assets/2.png',
  '/src/assets/3.png',
  '/src/assets/4.png',
]

const frame1 = ref(null)
const frame2 = ref(null)
const frame3 = ref(null)
const frame4 = ref(null)

const frames = [frame1, frame2, frame3, frame4]

const currentCaptureIndex = ref(0)
const countdown = ref(5)
const isCapturing = ref(false)
const finalImage = ref(null)

const capturedCanvases = [null, null, null, null]

function startCapture() {
  if (isCapturing.value || currentCaptureIndex.value >= 4) return

  isCapturing.value = true
  countdown.value = 5

  const countdownInterval = setInterval(() => {
    countdown.value--
    if (countdown.value <= 0) {
      clearInterval(countdownInterval)
      captureSingle(currentCaptureIndex.value) // ⬅️ Otomatis ambil gambar
    }
  }, 1000)
}

async function captureSingle(index) {
  const frameRef = frames[index].value
  const video = frameRef.getVideo()
  const overlay = frameRef.getFrame()
  const size = 500

  frameRef.triggerFlash()

  const canvas = document.createElement('canvas')
  canvas.width = size
  canvas.height = size
  const ctx = canvas.getContext('2d')

  // Ambil sisi tengah video (agar square, tidak stretch)
  const videoWidth = video.videoWidth
  const videoHeight = video.videoHeight
  const side = Math.min(videoWidth, videoHeight)
  const sx = (videoWidth - side) / 2
  const sy = (videoHeight - side) / 2

  // Gambar video (ter-crop)
  ctx.drawImage(video, sx, sy, side, side, 0, 0, size, size)

  // ✅ Gambar overlay frame di atasnya
  await new Promise((resolve) => {
    const img = new Image()
    img.src = overlay.src
    img.onload = () => {
      ctx.drawImage(img, 0, 0, size, size) // ✅ ini akan tertanam di hasil capture
      resolve()
    }
  })

  // Simpan hasil ke canvas dan tampilkan preview
  capturedCanvases[index] = canvas
  const dataUrl = canvas.toDataURL('image/png')
  frameRef.setCapturedImage(dataUrl)

  currentCaptureIndex.value++
  isCapturing.value = false

  if (currentCaptureIndex.value === 4) {
    composeFinalImage()
  }
}


function composeFinalImage() {
  const size = 500
  const canvas = document.createElement('canvas')
  canvas.width = size * 2
  canvas.height = size * 2
  const ctx = canvas.getContext('2d')

  for (let i = 0; i < 4; i++) {
    const col = i % 2
    const row = Math.floor(i / 2)
    const x = col * size
    const y = row * size

    // ⛔️ Jangan tambah frame PNG lagi di sini
    ctx.drawImage(capturedCanvases[i], x, y, size, size)
  }

  finalImage.value = canvas.toDataURL('image/png')
}

function downloadImage() {
  const link = document.createElement('a')
  link.download = `photobooth-${Date.now()}.png`
  link.href = finalImage.value
  link.click()
}

function resetCapture() {
  finalImage.value = null
  currentCaptureIndex.value = 0
  countdown.value = 0
  isCapturing.value = false
  capturedCanvases.fill(null)
  // Reset image in frames
  [frame1, frame2, frame3, frame4].forEach(f => f.value?.setCapturedImage(null))
}
</script>

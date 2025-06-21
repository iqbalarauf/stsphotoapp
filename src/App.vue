<template>
  <div class="min-h-screen bg-gray-900 text-white flex flex-col items-center justify-center gap-6 p-4">
    <!-- Semua frame hanya tampil jika belum final preview -->
    <div v-if="!finalImage">
      <!-- Frame Live (aktif satu per satu) -->
      <PhotoFrame v-if="currentCaptureIndex < 4" :key="currentCaptureIndex" :ref="`frame${currentCaptureIndex + 1}`"
        :frameSrc="framesSrc[currentCaptureIndex]" mode="live" />

      <!-- Captured -->
      <div v-if="currentCaptureIndex > 0" class="grid grid-cols-2 gap-4 mt-6">
        <PhotoFrame v-for="(src, i) in framesSrc" v-if="i < currentCaptureIndex" :key="i" :ref="`frame${i + 1}`"
          :frameSrc="src" mode="captured" />
      </div>
    </div>

    <!-- Tombol dan countdown -->
    <div v-if="!finalImage" class="flex flex-col items-center gap-3">
      <button @click="startCapture" :disabled="isCapturing || currentCaptureIndex >= 4"
        class="bg-green-500 hover:bg-green-600 disabled:opacity-50 px-6 py-2 rounded-lg text-lg font-bold shadow-md transition">
        📸 Start Capture
      </button>
      <div v-if="countdown > 0" class="text-5xl font-bold animate-pulse">{{ countdown }}</div>
    </div>

    <!-- Preview akhir gabungan -->
    <div v-if="finalImage" class="flex flex-col items-center gap-4 transition-all duration-700 ease-out scale-100 opacity-100">
      <img :src="finalImage" alt="Preview" class="rounded-lg shadow-lg max-w-full w-[600px]" />
      <div class="flex gap-4">
        <button @click="downloadImage"
          class="bg-blue-500 hover:bg-blue-600 px-6 py-2 rounded-lg font-bold transition">💾 Download</button>
        <button @click="resetCapture" class="bg-red-500 hover:bg-red-600 px-6 py-2 rounded-lg font-bold transition">🔁
          Retake</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import PhotoFrame from './components/PhotoFrame.vue'

// Frame image imports
import framea from './assets/1.png'
import frameb from './assets/2.png'
import framec from './assets/3.png'
import framed from './assets/4.png'

// Frame refs
const framesSrc = [framea, frameb, framec, framed]

const frame1 = ref(null)
const frame2 = ref(null)
const frame3 = ref(null)
const frame4 = ref(null)
const frames = [frame1, frame2, frame3, frame4]

// State
const currentCaptureIndex = ref(0)
const countdown = ref(0)
const isCapturing = ref(false)
const finalImage = ref(null)
const capturedCanvases = [null, null, null, null]

async function waitForFrameMount(index, maxWait = 3000) {
  const start = Date.now()
  while (true) {
    const ref = frames[index]?.value
    const video = ref?.getVideo?.()
    if (video?.videoWidth > 0 && video?.videoHeight > 0) {
      console.log(`✅ Frame siap: ${index}`)
      return
    }

    await new Promise(r => setTimeout(r, 50))
    if (Date.now() - start > maxWait) {
      console.warn('❗ Timeout: frame belum muncul:', index)
      return
    }
  }
}

// Otomatis ambil semua frame
async function startCapture() {
  if (isCapturing.value || currentCaptureIndex.value >= 4) return

  isCapturing.value = true

  for (let i = currentCaptureIndex.value; i < 4; i++) {
    currentCaptureIndex.value = i
    countdown.value = 5

    // ⏳ Countdown
    await new Promise((resolve) => {
      const interval = setInterval(() => {
        countdown.value--
        if (countdown.value <= 0) {
          clearInterval(interval)
          resolve()
        }
      }, 1000)
    })

    // ⏱️ Tunggu Vue render PhotoFrame
    await nextTick()
    await waitForFrameMount(i) // tunggu sampai frame[i] benar-benar ada

    console.log(`📸 Mencoba ambil frame ke-${i+1}`)
    await captureSingle(i)
  }

  isCapturing.value = false
  composeFinalImage()
}



// Tangkap satu frame
async function captureSingle(index) {
  const frameRef = frames[index]?.value
  if (!frameRef) {
    console.warn('❗ Frame belum siap:', index)
    return
  }

  const video = frameRef.getVideo()
  const size = 500

  frameRef.triggerFlash()

  const canvas = document.createElement('canvas')
  canvas.width = size
  canvas.height = size
  const ctx = canvas.getContext('2d')

  const videoWidth = video.videoWidth
  const videoHeight = video.videoHeight
  const side = Math.min(videoWidth, videoHeight)
  const sx = (videoWidth - side) / 2
  const sy = (videoHeight - side) / 2

  if (!video || video.videoWidth === 0 || video.videoHeight === 0) {
  console.warn('❗ Video belum siap ditangkap:', video);
  return;
}

  ctx.drawImage(video, sx, sy, side, side, 0, 0, size, size)

  // Tambah overlay dari public/
  const frameImage = new Image()
  frameImage.crossOrigin = 'anonymous'
  frameImage.src = framesSrc[index]

  await new Promise((resolve, reject) => {
    frameImage.onload = resolve
    frameImage.onerror = reject
  })

  ctx.drawImage(frameImage, 0, 0, size, size)

  const dataUrl = canvas.toDataURL('image/png')
  capturedCanvases[index] = canvas
  frameRef.setCapturedImage(dataUrl)

  console.log('🔍 Cek frame:', index, frames[index]?.value)
console.log('🔍 Video element:', frames[index]?.value?.getVideo())
  console.log(`✅ Frame ${index+1} selesai`)
}

// Gabung semua ke PNG akhir
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

    ctx.drawImage(capturedCanvases[i], x, y, size, size)
  }

  finalImage.value = canvas.toDataURL('image/png')
  console.log('✅ Final image composed!')
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

  // Kosongkan semua ref frame
  frames.forEach(f => f.value?.setCapturedImage(null))
}
</script>

<template>
  <div class="container">
    <h1>OCR 文字识别</h1>

    <!-- 摄像头控制 -->
    <div class="toolbar">
      <button v-if="!cameraActive" @click="startCamera" :disabled="loading">启动摄像头</button>
      <button v-else @click="stopCamera" :disabled="loading">关闭摄像头</button>
      <button @click="captureAndRecognize" :disabled="!cameraActive || loading">
        {{ loading ? '识别中...' : '拍照识别' }}
      </button>
    </div>

    <!-- 本地图片控制 -->
    <div class="toolbar">
      <input ref="fileInputRef" type="file" accept="image/*" hidden @change="onFileSelected" />
      <button @click="selectImage" :disabled="loading">选择本地图片</button>
      <button @click="recognizeLocalImage" :disabled="!localImageLoaded || loading">
        {{ loading ? '识别中...' : '识别图片' }}
      </button>
      <button v-if="capturedImage" class="btn-clear" @click="clearPreview" :disabled="loading">清除</button>
    </div>

    <!-- 预览区域 -->
    <div class="preview-area">
      <video v-show="cameraActive && !capturedImage" ref="videoRef" autoplay playsinline></video>
      <canvas v-show="capturedImage" ref="canvasRef"></canvas>
      <div v-if="!cameraActive && !capturedImage" class="placeholder">
        请启动摄像头或选择本地图片
      </div>
    </div>

    <div v-if="snapshotTaken" class="snapshot-info">已拍照，正在识别...</div>

    <div v-if="error" class="error">{{ error }}</div>

    <div v-if="result" class="result">
      <h3>识别结果：</h3>
      <pre>{{ result }}</pre>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import Tesseract from 'tesseract.js'

const videoRef = ref(null)
const canvasRef = ref(null)
const fileInputRef = ref(null)
const cameraActive = ref(false)
const loading = ref(false)
const result = ref('')
const error = ref('')
const capturedImage = ref(false)
const localImageLoaded = ref(false)
const snapshotTaken = ref(false)
let stream = null

async function startCamera() {
  error.value = ''
  result.value = ''
  capturedImage.value = false
  localImageLoaded.value = false
  snapshotTaken.value = false
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: 'environment' },
      audio: false,
    })
    videoRef.value.srcObject = stream
    cameraActive.value = true
  } catch (e) {
    error.value = '无法访问摄像头：' + (e.message || '未知错误')
  }
}

function stopCamera() {
  if (stream) {
    stream.getTracks().forEach(t => t.stop())
    stream = null
  }
  cameraActive.value = false
}

function selectImage() {
  fileInputRef.value.click()
}

function onFileSelected(e) {
  const file = e.target.files[0]
  if (!file) return
  error.value = ''
  result.value = ''

  const reader = new FileReader()
  reader.onload = (event) => {
    const img = new Image()
    img.onload = () => {
      const canvas = canvasRef.value
      canvas.width = img.width
      canvas.height = img.height
      const ctx = canvas.getContext('2d')
      ctx.drawImage(img, 0, 0)
      capturedImage.value = true
      localImageLoaded.value = true
    }
    img.src = event.target.result
  }
  reader.readAsDataURL(file)
  e.target.value = ''
}

function clearPreview() {
  capturedImage.value = false
  localImageLoaded.value = false
  result.value = ''
  error.value = ''
}

async function captureAndRecognize() {
  if (!cameraActive.value) return
  loading.value = true
  error.value = ''
  result.value = ''
  snapshotTaken.value = false

  await nextTick()
  const video = videoRef.value
  const canvas = canvasRef.value
  canvas.width = video.videoWidth
  canvas.height = video.videoHeight
  const ctx = canvas.getContext('2d')
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height)
  capturedImage.value = true
  snapshotTaken.value = true

  try {
    const { data } = await Tesseract.recognize(canvas, 'chi_sim+eng', {
      logger: (m) => {
        if (m.status === 'recognizing text') {
          snapshotTaken.value = false
        }
      },
    })
    result.value = data.text
    if (!result.value.trim()) {
      result.value = '未识别到文字'
    }
  } catch (e) {
    error.value = 'OCR 识别失败：' + (e.message || '未知错误')
  } finally {
    loading.value = false
    snapshotTaken.value = false
  }
}

async function recognizeLocalImage() {
  if (!localImageLoaded.value) return
  loading.value = true
  error.value = ''
  result.value = ''

  try {
    const { data } = await Tesseract.recognize(canvasRef.value, 'chi_sim+eng', {})
    result.value = data.text
    if (!result.value.trim()) {
      result.value = '未识别到文字'
    }
  } catch (e) {
    error.value = 'OCR 识别失败：' + (e.message || '未知错误')
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 24px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}
h1 {
  text-align: center;
  color: #333;
  margin-bottom: 24px;
}
.toolbar {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 14px;
  flex-wrap: wrap;
}
button {
  padding: 10px 22px;
  font-size: 15px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  background: #4a90d9;
  color: #fff;
  transition: background 0.2s;
  white-space: nowrap;
}
button:hover:not(:disabled) {
  background: #357abd;
}
button:disabled {
  background: #aaa;
  cursor: not-allowed;
}
.btn-clear {
  background: #888;
}
.btn-clear:hover:not(:disabled) {
  background: #666;
}
.preview-area {
  display: flex;
  justify-content: center;
  margin-bottom: 14px;
  background: #f0f0f0;
  border-radius: 8px;
  min-height: 240px;
  align-items: center;
  overflow: hidden;
}
video, canvas {
  max-width: 100%;
  max-height: 480px;
}
.placeholder {
  color: #999;
  font-size: 16px;
}
.snapshot-info {
  text-align: center;
  color: #666;
  margin-bottom: 12px;
}
.error {
  color: #d32f2f;
  text-align: center;
  padding: 12px;
  background: #fce4e4;
  border-radius: 6px;
  margin-bottom: 12px;
}
.result {
  background: #f5f5f5;
  border-radius: 8px;
  padding: 16px 20px;
}
.result h3 {
  margin: 0 0 8px;
  color: #333;
}
.result pre {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-all;
  font-size: 15px;
  line-height: 1.6;
  color: #222;
}
</style>

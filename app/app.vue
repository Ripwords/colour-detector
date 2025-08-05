<script setup lang="ts">
const videoRef = ref<HTMLVideoElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)
const isCameraActive = ref(false)
const isPaused = ref(false)
const errorMessage = ref("")
const rgbColor = ref("--")
const hexColor = ref("#000000")
const targetColor = ref("#ff0000")
const similarityPercentage = ref(0)
const similarityClass = ref("")
let stream: MediaStream | null = null
let animationId: number | null = null
let capturedImageData: ImageData | null = null

// Computed properties for target color
const targetRgb = computed(() => {
  const hex = targetColor.value.replace("#", "")
  const r = parseInt(hex.substr(0, 2), 16)
  const g = parseInt(hex.substr(2, 2), 16)
  const b = parseInt(hex.substr(4, 2), 16)
  return `${r}, ${g}, ${b}`
})

// Color similarity calculation using Euclidean distance
const calculateSimilarity = (color1: number[], color2: number[]): number => {
  const maxDistance = Math.sqrt(255 * 255 + 255 * 255 + 255 * 255) // Maximum possible distance
  const distance = Math.sqrt(
    Math.pow((color1?.[0] ?? 0) - (color2?.[0] ?? 0), 2) +
      Math.pow((color1?.[1] ?? 0) - (color2?.[1] ?? 0), 2) +
      Math.pow((color1?.[2] ?? 0) - (color2?.[2] ?? 0), 2)
  )
  return Math.max(0, 100 - (distance / maxDistance) * 100)
}

// Update similarity class based on percentage
const updateSimilarityClass = (percentage: number) => {
  if (percentage >= 90) {
    similarityClass.value = "excellent"
  } else if (percentage >= 75) {
    similarityClass.value = "good"
  } else if (percentage >= 50) {
    similarityClass.value = "fair"
  } else {
    similarityClass.value = "poor"
  }
}

const startCamera = async () => {
  try {
    errorMessage.value = ""
    stream = await navigator.mediaDevices.getUserMedia({
      video: {
        width: { ideal: 1280 },
        height: { ideal: 720 },
      },
    })

    if (videoRef.value) {
      videoRef.value.srcObject = stream
      isCameraActive.value = true
      isPaused.value = false
      capturedImageData = null
      startColorDetection()
    }
  } catch (error) {
    errorMessage.value =
      "Camera access denied or unavailable. Please check your browser permissions."
    console.error("Camera error:", error)
  }
}

const togglePause = () => {
  if (isPaused.value) {
    // Resume
    isPaused.value = false
    capturedImageData = null
    if (videoRef.value) {
      videoRef.value.play()
    }
    startColorDetection()
  } else {
    // Pause and capture
    isPaused.value = true
    if (animationId) {
      cancelAnimationFrame(animationId)
      animationId = null
    }
    if (videoRef.value) {
      videoRef.value.pause()
    }
    captureCurrentFrame()
  }
}

const captureCurrentFrame = () => {
  if (!videoRef.value || !canvasRef.value) return

  const video = videoRef.value
  const canvas = canvasRef.value
  const ctx = canvas.getContext("2d")

  if (!ctx) return

  // Set canvas size to match video
  canvas.width = video.videoWidth
  canvas.height = video.videoHeight

  // Draw current video frame to canvas
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height)

  // Capture the entire frame
  capturedImageData = ctx.getImageData(0, 0, canvas.width, canvas.height)

  // Analyze the captured frame
  analyzeCapturedFrame()
}

const analyzeCapturedFrame = () => {
  if (!capturedImageData) return

  const canvas = canvasRef.value
  if (!canvas) return

  // Get center box coordinates (50x50 pixels)
  const centerX = Math.floor(canvas.width / 2)
  const centerY = Math.floor(canvas.height / 2)
  const boxSize = 25 // Half of 50px box

  // Calculate the area to analyze in the captured image data
  const startX = Math.max(0, centerX - boxSize)
  const startY = Math.max(0, centerY - boxSize)
  const width = Math.min(boxSize * 2, capturedImageData.width - startX)
  const height = Math.min(boxSize * 2, capturedImageData.height - startY)

  // Extract pixel data from the center box area
  const data = capturedImageData.data
  let r = 0,
    g = 0,
    b = 0,
    count = 0

  for (let y = startY; y < startY + height; y++) {
    for (let x = startX; x < startX + width; x++) {
      const index = (y * capturedImageData.width + x) * 4
      r += data?.[index] ?? 0
      g += data?.[index + 1] ?? 0
      b += data?.[index + 2] ?? 0
      count++
    }
  }

  if (count > 0) {
    r = Math.round(r / count)
    g = Math.round(g / count)
    b = Math.round(b / count)

    // Update color values
    rgbColor.value = `${r}, ${g}, ${b}`
    hexColor.value = rgbToHex(r, g, b)

    // Calculate similarity with target color
    const targetHex = targetColor.value.replace("#", "")
    const targetR = parseInt(targetHex.substr(0, 2), 16)
    const targetG = parseInt(targetHex.substr(2, 2), 16)
    const targetB = parseInt(targetHex.substr(4, 2), 16)

    const similarity = calculateSimilarity(
      [r, g, b],
      [targetR, targetG, targetB]
    )
    similarityPercentage.value = Math.round(similarity)
    updateSimilarityClass(similarity)
  }
}

const startColorDetection = () => {
  const analyzeColor = () => {
    if (
      !videoRef.value ||
      !canvasRef.value ||
      !isCameraActive.value ||
      isPaused.value
    )
      return

    const video = videoRef.value
    const canvas = canvasRef.value
    const ctx = canvas.getContext("2d")

    if (!ctx) return

    // Set canvas size to match video
    canvas.width = video.videoWidth
    canvas.height = video.videoHeight

    // Draw current video frame to canvas
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height)

    // Get center box coordinates (50x50 pixels)
    const centerX = Math.floor(canvas.width / 2)
    const centerY = Math.floor(canvas.height / 2)
    const boxSize = 25 // Half of 50px box

    // Get pixel data from center box
    const imageData = ctx.getImageData(
      centerX - boxSize,
      centerY - boxSize,
      boxSize * 2,
      boxSize * 2
    )
    const data = imageData.data

    // Calculate average RGB values
    let r = 0,
      g = 0,
      b = 0,
      count = 0

    for (let i = 0; i < (data?.length ?? 0); i += 4) {
      r += data?.[i] ?? 0
      g += data?.[i + 1] ?? 0
      b += data?.[i + 2] ?? 0
      count++
    }

    if (count > 0) {
      r = Math.round(r / count)
      g = Math.round(g / count)
      b = Math.round(b / count)

      // Update color values
      rgbColor.value = `${r}, ${g}, ${b}`
      hexColor.value = rgbToHex(r, g, b)

      // Calculate similarity with target color
      const targetHex = targetColor.value.replace("#", "")
      const targetR = parseInt(targetHex.substr(0, 2), 16)
      const targetG = parseInt(targetHex.substr(2, 2), 16)
      const targetB = parseInt(targetHex.substr(4, 2), 16)

      const similarity = calculateSimilarity(
        [r, g, b],
        [targetR, targetG, targetB]
      )
      similarityPercentage.value = Math.round(similarity)
      updateSimilarityClass(similarity)
    }

    // Continue animation loop (10+ times per second)
    animationId = requestAnimationFrame(analyzeColor)
  }

  analyzeColor()
}

const rgbToHex = (r: number, g: number, b: number) => {
  return (
    "#" +
    [r, g, b]
      .map((x) => {
        const hex = x.toString(16)
        return hex.length === 1 ? "0" + hex : hex
      })
      .join("")
  )
}

const stopCamera = () => {
  if (stream) {
    stream.getTracks().forEach((track) => track.stop())
    stream = null
  }
  if (animationId) {
    cancelAnimationFrame(animationId)
    animationId = null
  }
  isCameraActive.value = false
  isPaused.value = false
  capturedImageData = null
}

// Cleanup on component unmount
onUnmounted(() => {
  stopCamera()
})
</script>

<template>
  <div class="container">
    <h1>Camera Color Detector</h1>

    <div class="target-color-section">
      <label for="targetColor">Target Color:</label>
      <input
        id="targetColor"
        type="color"
        v-model="targetColor"
        class="color-picker"
      />
      <div
        class="target-preview"
        :style="{ backgroundColor: targetColor }"
      ></div>
      <div class="target-values">
        RGB: <span>{{ targetRgb }}</span> | HEX: <span>{{ targetColor }}</span>
      </div>
    </div>

    <div class="camera-container">
      <video
        ref="videoRef"
        autoplay
        playsinline
        muted
      ></video>
      <canvas
        ref="canvasRef"
        style="display: none"
      ></canvas>
      <div class="overlay-box"></div>
      <div
        v-if="isPaused"
        class="paused-overlay"
      >
        <div class="paused-text">PAUSED</div>
      </div>
    </div>

    <div class="color-display">
      <div
        class="color-preview"
        :style="{ backgroundColor: hexColor }"
      ></div>
      <div class="color-values">
        <div class="rgb-value">
          RGB: <span>{{ rgbColor }}</span>
        </div>
        <div class="hex-value">
          HEX: <span>{{ hexColor }}</span>
        </div>
        <div
          class="similarity-value"
          :class="similarityClass"
        >
          Similarity: <span>{{ similarityPercentage }}%</span>
        </div>
      </div>
    </div>

    <div class="controls">
      <button
        @click="startCamera"
        :disabled="isCameraActive"
        class="control-button start-button"
      >
        {{ isCameraActive ? "Camera Active" : "Start Camera" }}
      </button>

      <button
        v-if="isCameraActive"
        @click="togglePause"
        class="control-button pause-button"
        :class="{ paused: isPaused }"
      >
        {{ isPaused ? "Resume" : "Pause & Capture" }}
      </button>
    </div>

    <div
      class="error-message"
      v-if="errorMessage"
    >
      {{ errorMessage }}
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
  font-family: Arial, sans-serif;
}

h1 {
  color: #333;
  margin-bottom: 30px;
}

.target-color-section {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 15px;
  margin-bottom: 30px;
  padding: 20px;
  background: #f8f9fa;
  border-radius: 8px;
  border: 1px solid #e9ecef;
}

.target-color-section label {
  font-weight: bold;
  color: #333;
}

.color-picker {
  width: 50px;
  height: 50px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  background: none;
}

.target-preview {
  width: 60px;
  height: 60px;
  border-radius: 8px;
  border: 2px solid #ddd;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.target-values {
  font-family: "Courier New", monospace;
  font-size: 14px;
  color: #666;
}

.camera-container {
  position: relative;
  display: inline-block;
  margin-bottom: 20px;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

video {
  width: 100%;
  max-width: 640px;
  height: auto;
  display: block;
}

.overlay-box {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 50px;
  height: 50px;
  border: 2px solid #fff;
  border-radius: 4px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  pointer-events: none;
}

.paused-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;
}

.paused-text {
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 10px 20px;
  border-radius: 6px;
  font-weight: bold;
  font-size: 18px;
}

.color-display {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 20px;
  margin-bottom: 20px;
  padding: 20px;
  background: #f5f5f5;
  border-radius: 8px;
}

.color-preview {
  width: 60px;
  height: 60px;
  border-radius: 8px;
  border: 2px solid #ddd;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.color-values {
  text-align: left;
  font-family: "Courier New", monospace;
  font-size: 16px;
}

.rgb-value,
.hex-value,
.similarity-value {
  margin: 5px 0;
  color: #333;
}

.similarity-value {
  font-weight: bold;
}

.similarity-value.excellent {
  color: #2e7d32;
}

.similarity-value.good {
  color: #1976d2;
}

.similarity-value.fair {
  color: #f57c00;
}

.similarity-value.poor {
  color: #d32f2f;
}

.controls {
  display: flex;
  gap: 15px;
  justify-content: center;
  margin-bottom: 20px;
}

.control-button {
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: bold;
}

.start-button {
  background: #2196f3;
  color: white;
}

.start-button:hover:not(:disabled) {
  background: #1976d2;
}

.start-button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.pause-button {
  background: #ff9800;
  color: white;
}

.pause-button:hover {
  background: #f57c00;
}

.pause-button.paused {
  background: #4caf50;
}

.pause-button.paused:hover {
  background: #388e3c;
}

.error-message {
  color: #d32f2f;
  background: #ffebee;
  padding: 15px;
  border-radius: 4px;
  margin: 20px 0;
  border: 1px solid #ffcdd2;
}

@media (max-width: 768px) {
  .container {
    padding: 10px;
  }

  .target-color-section {
    flex-direction: column;
    gap: 10px;
  }

  .color-display {
    flex-direction: column;
    gap: 15px;
  }

  .color-values {
    text-align: center;
  }

  .controls {
    flex-direction: column;
    align-items: center;
  }

  .control-button {
    width: 200px;
  }
}
</style>

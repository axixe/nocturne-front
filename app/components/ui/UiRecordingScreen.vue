<template>
  <section
    ref="screenRef"
    class="recording-screen"
    :class="{ 'recording-screen--recording': isRecording }"
  >
    <div class="recording-screen__noise" aria-hidden="true" />
    <div ref="bgGlowRef" class="recording-screen__active-glow" aria-hidden="true" />

    <header class="recording-screen__header">
      <button class="recording-screen__close" type="button" aria-label="Close" @click="handleClose">
        <span />
        <span />
      </button>
      <span class="recording-screen__label">RECORD</span>
      <span class="recording-screen__spacer" aria-hidden="true" />
    </header>

    <main class="recording-screen__center">
      <p class="recording-screen__timer">{{ formattedTime }}</p>
      <p class="recording-screen__subtitle">{{ subtitle }}</p>
    </main>

    <footer class="recording-screen__footer">
      <button class="record-button" type="button" :aria-label="isRecording ? 'Stop recording' : 'Start recording'" @click="toggleRecording">
        <span class="record-button__waves" aria-hidden="true">
          <span class="record-button__wave" />
          <span class="record-button__wave" />
          <span class="record-button__wave" />
        </span>
        <span ref="ringRef" class="record-button__ring">
          <span ref="innerRef" class="record-button__inner">
            <span v-if="isRecording" class="record-button__stop" aria-hidden="true" />
            <BaseMicIcon v-else class="record-button__icon" />
          </span>
        </span>
      </button>
    </footer>
  </section>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, ref } from 'vue'

const isRecording = ref(false)
const elapsedSeconds = ref(0)

const screenRef = ref<HTMLElement | null>(null)
const ringRef = ref<HTMLElement | null>(null)
const innerRef = ref<HTMLElement | null>(null)
const bgGlowRef = ref<HTMLElement | null>(null)

let mediaRecorder: MediaRecorder | null = null
let mediaStream: MediaStream | null = null
let audioContext: AudioContext | null = null
let analyser: AnalyserNode | null = null
let dataArray: Uint8Array | null = null
let animationFrame = 0
let timerInterval: ReturnType<typeof setInterval> | null = null
let recorderStartedAt = 0
let smoothedLevel = 0

const formattedTime = computed(() => {
  const minutes = Math.floor(elapsedSeconds.value / 60)
  const seconds = elapsedSeconds.value % 60

  return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
})

const subtitle = computed(() => (isRecording.value ? 'Recording...' : 'Tap to record'))

const setAudioLevel = (nextLevel: number) => {
  const level = Math.max(0, Math.min(nextLevel, 1))

  if (screenRef.value) {
    screenRef.value.style.setProperty('--audio-level', level.toFixed(3))
  }

  if (ringRef.value) {
    ringRef.value.style.transform = `scale(${1 + level * 0.1})`
  }

  if (innerRef.value) {
    const nearGlow = 20 + level * 18
    const farGlow = 40 + level * 34
    const glowOpacity = 0.34 + level * 0.45

    innerRef.value.style.boxShadow = `
      0 0 ${nearGlow}px rgba(0, 230, 138, ${glowOpacity}),
      0 0 ${farGlow}px rgba(0, 230, 138, ${0.18 + level * 0.26})
    `
  }

  if (bgGlowRef.value) {
    bgGlowRef.value.style.opacity = `${0.18 + level * 0.45}`
    bgGlowRef.value.style.transform = `translate(-50%, -50%) scale(${1 + level * 0.28})`
  }
}

const tickTimer = () => {
  elapsedSeconds.value = Math.floor((Date.now() - recorderStartedAt) / 1000)
}

const cleanupMedia = () => {
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }

  if (animationFrame) {
    cancelAnimationFrame(animationFrame)
    animationFrame = 0
  }

  if (mediaRecorder && mediaRecorder.state !== 'inactive') {
    mediaRecorder.stop()
  }

  mediaRecorder = null

  if (mediaStream) {
    mediaStream.getTracks().forEach((track) => track.stop())
    mediaStream = null
  }

  if (audioContext) {
    void audioContext.close()
    audioContext = null
  }

  analyser = null
  dataArray = null
  smoothedLevel = 0
  setAudioLevel(0)
}

const updateVolumeFrame = () => {
  if (!analyser || !dataArray || !isRecording.value) {
    return
  }

  analyser.getByteFrequencyData(dataArray)

  let total = 0
  for (let index = 0; index < dataArray.length; index += 1) {
    total += dataArray[index]
  }

  const average = total / dataArray.length
  const normalized = Math.min(1, average / 80)
  smoothedLevel = smoothedLevel * 0.8 + normalized * 0.2

  setAudioLevel(smoothedLevel)
  animationFrame = requestAnimationFrame(updateVolumeFrame)
}

const startRecording = async () => {
  try {
    mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true })
    audioContext = new AudioContext()

    const source = audioContext.createMediaStreamSource(mediaStream)
    analyser = audioContext.createAnalyser()
    analyser.fftSize = 256

    source.connect(analyser)
    dataArray = new Uint8Array(analyser.frequencyBinCount)

    mediaRecorder = new MediaRecorder(mediaStream)
    mediaRecorder.start()

    recorderStartedAt = Date.now()
    elapsedSeconds.value = 0
    tickTimer()
    timerInterval = setInterval(tickTimer, 1000)

    isRecording.value = true
    updateVolumeFrame()
  } catch (error) {
    console.error('Microphone permission failed', error)
    cleanupMedia()
    isRecording.value = false
  }
}

const stopRecording = () => {
  isRecording.value = false
  cleanupMedia()
}

const toggleRecording = () => {
  if (isRecording.value) {
    stopRecording()
    return
  }

  void startRecording()
}

const handleClose = () => {
  stopRecording()
  elapsedSeconds.value = 0
}

onBeforeUnmount(() => {
  cleanupMedia()
})
</script>

<style scoped lang="scss">
.recording-screen {
  --audio-level: 0;
  position: relative;
  isolation: isolate;
  width: 100%;
  min-height: 100dvh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
  padding: 28px 18px 70px;
  background:
    radial-gradient(circle at 50% 40%, rgba(0, 230, 138, 0.15), transparent 60%),
    linear-gradient(135deg, #0A0F0D 0%, #0B1F1A 40%, #0A0F0D 100%);
  overflow: hidden;
}

.recording-screen__noise {
  position: absolute;
  inset: 0;
  pointer-events: none;
  opacity: 0.02;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='180' height='180' viewBox='0 0 180 180'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='1.05' numOctaves='2' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='180' height='180' filter='url(%23n)'/%3E%3C/svg%3E");
  z-index: -2;
}

.recording-screen__active-glow {
  position: absolute;
  top: 58%;
  left: 50%;
  width: min(74vw, 420px);
  aspect-ratio: 1;
  border-radius: 50%;
  transform: translate(-50%, -50%);
  background: radial-gradient(circle, rgba(0, 230, 138, 0.26) 0%, rgba(0, 230, 138, 0) 70%);
  opacity: 0.18;
  filter: blur(20px);
  z-index: -1;
  transition: transform 0.16s ease-out, opacity 0.18s ease-out;
}

.recording-screen__header {
  width: 100%;
  display: grid;
  grid-template-columns: 28px 1fr 28px;
  align-items: center;
}

.recording-screen__close {
  position: relative;
  width: 20px;
  height: 20px;
  border: 0;
  background: transparent;
  opacity: 0.8;

  span {
    position: absolute;
    top: 9px;
    left: 1px;
    width: 18px;
    height: 1.5px;
    border-radius: 999px;
    background: rgba(232, 255, 246, 0.56);
  }

  span:first-child {
    transform: rotate(45deg);
  }

  span:last-child {
    transform: rotate(-45deg);
  }
}

.recording-screen__label {
  justify-self: center;
  color: rgba(232, 255, 246, 0.5);
  font-size: 13px;
  letter-spacing: 0.08em;
}

.recording-screen__spacer {
  width: 20px;
  height: 20px;
}

.recording-screen__center {
  margin-top: auto;
  text-align: center;
}

.recording-screen__timer {
  margin: 0;
  font-size: clamp(52px, 12vw, 64px);
  font-weight: 400;
  line-height: 1;
  letter-spacing: 0.03em;
  font-variant-numeric: tabular-nums;
  color: #e8fff6;
}

.recording-screen__subtitle {
  margin-top: 12px;
  font-size: 26px;
  color: rgba(232, 255, 246, 0.66);
  transition: color 0.3s ease, opacity 0.3s ease;
}

.recording-screen--recording .recording-screen__subtitle {
  color: rgba(232, 255, 246, 0.84);
}

.recording-screen__footer {
  margin-top: auto;
  display: flex;
  justify-content: center;
  width: 100%;
}

.record-button {
  position: relative;
  display: grid;
  place-items: center;
  width: 190px;
  aspect-ratio: 1;
  border: 0;
  background: transparent;
}

.record-button__waves {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  pointer-events: none;
}

.record-button__wave {
  position: absolute;
  width: 100%;
  aspect-ratio: 1;
  border-radius: 50%;
  border: 2px solid rgba(0, 230, 138, calc(0.26 + var(--audio-level) * 0.18));
  transform: scale(1);
  opacity: 0;
  animation: wave-pulse calc(1.7s - var(--audio-level) * 0.55s) ease-out infinite;
  animation-play-state: paused;
}

.record-button__wave:nth-child(2) {
  animation-delay: 0.5s;
}

.record-button__wave:nth-child(3) {
  animation-delay: 1s;
}

.recording-screen--recording .record-button__wave {
  animation-play-state: running;
}

.record-button__ring {
  position: relative;
  display: grid;
  place-items: center;
  width: 118px;
  aspect-ratio: 1;
  border-radius: 50%;
  border: 5px solid #00e68a;
  box-shadow:
    0 0 20px rgba(0, 230, 138, 0.4),
    0 0 40px rgba(0, 230, 138, 0.2);
  transition: transform 0.12s cubic-bezier(0.2, 0.8, 0.3, 1);
  will-change: transform;
}

.record-button__inner {
  display: grid;
  place-items: center;
  width: 74px;
  aspect-ratio: 1;
  border-radius: 50%;
  background: #00e68a;
  color: #022f1f;
  box-shadow:
    0 0 20px rgba(0, 230, 138, 0.34),
    0 0 40px rgba(0, 230, 138, 0.18);
  transition: box-shadow 0.16s ease-out, transform 0.16s ease-out, background 0.3s ease;
  will-change: transform, box-shadow;
}

.record-button__icon {
  width: 28px;
  height: 28px;
}

.record-button__stop {
  width: 24px;
  height: 24px;
  border-radius: 6px;
  background: #082217;
}

@keyframes wave-pulse {
  0% {
    transform: scale(calc(1 + var(--audio-level) * 0.05));
    opacity: calc(0.18 + var(--audio-level) * 0.46);
  }

  100% {
    transform: scale(calc(1.6 + var(--audio-level) * 0.14));
    opacity: 0;
  }
}
</style>

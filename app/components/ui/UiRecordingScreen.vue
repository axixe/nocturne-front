<template>
  <section
    ref="screenRef"
    class="recording-screen"
    :class="{
      'recording-screen--recording': isRecording,
      'recording-screen--recorded': isRecorded,
      'recording-screen--playing': isPlaying
    }"
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

      <div v-if="isRecorded" class="recording-screen__playback-progress" aria-hidden="true">
        <span class="recording-screen__playback-progress-fill" :style="{ transform: `scaleX(${playbackProgress})` }" />
      </div>
    </main>

    <footer class="recording-screen__footer">
      <button
        class="record-button"
        type="button"
        :aria-label="primaryButtonLabel"
        @click="handlePrimaryAction"
      >
        <span class="record-button__waves" aria-hidden="true">
          <span class="record-button__wave" />
          <span class="record-button__wave" />
          <span class="record-button__wave" />
        </span>
        <span ref="ringRef" class="record-button__ring">
          <span ref="innerRef" class="record-button__inner">
            <template v-if="isRecording">
              <span class="record-button__stop" aria-hidden="true" />
            </template>
            <template v-else-if="isRecorded">
              <span v-if="isPlaying" class="record-button__pause" aria-hidden="true">
                <span />
                <span />
              </span>
              <span v-else class="record-button__play" aria-hidden="true" />
            </template>
            <BaseMicIcon v-else class="record-button__icon" />
          </span>
        </span>
      </button>

      <div v-if="isRecorded" class="recording-actions">
        <button class="recording-actions__button recording-actions__button--outline" type="button" @click="handleRerecord">
          Re-record
        </button>
        <button class="recording-actions__button recording-actions__button--submit" type="button" @click="handleSubmit">
          Submit
        </button>
      </div>
    </footer>
  </section>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, ref } from 'vue'

type RecorderMode = 'idle' | 'recording' | 'recorded'

const mode = ref<RecorderMode>('idle')
const elapsedSeconds = ref(0)
const isPlaying = ref(false)
const playbackProgress = ref(0)

const recordedBlob = ref<Blob | null>(null)
const recordedAudioUrl = ref<string | null>(null)

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
let recordedChunks: Blob[] = []
let audioPlayer: HTMLAudioElement | null = null

const isRecording = computed(() => mode.value === 'recording')
const isRecorded = computed(() => mode.value === 'recorded')

const formattedTime = computed(() => {
  const minutes = Math.floor(elapsedSeconds.value / 60)
  const seconds = elapsedSeconds.value % 60

  return `${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`
})

const subtitle = computed(() => {
  if (mode.value === 'recording') {
    return 'Recording...'
  }

  if (mode.value === 'recorded') {
    return 'Ready to send'
  }

  return 'Tap to record'
})

const primaryButtonLabel = computed(() => {
  if (mode.value === 'recording') {
    return 'Stop recording'
  }

  if (mode.value === 'recorded') {
    return isPlaying.value ? 'Pause recording preview' : 'Play recording preview'
  }

  return 'Start recording'
})

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

const stopMonitoring = () => {
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }

  if (animationFrame) {
    cancelAnimationFrame(animationFrame)
    animationFrame = 0
  }

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

const cleanupRecorder = () => {
  if (mediaRecorder && mediaRecorder.state !== 'inactive') {
    mediaRecorder.stop()
  }

  mediaRecorder = null
  recordedChunks = []
  stopMonitoring()
}

const clearRecordedAudio = () => {
  if (audioPlayer) {
    audioPlayer.pause()
    audioPlayer.currentTime = 0
    audioPlayer.onended = null
    audioPlayer.ontimeupdate = null
    audioPlayer = null
  }

  isPlaying.value = false
  playbackProgress.value = 0

  if (recordedAudioUrl.value) {
    URL.revokeObjectURL(recordedAudioUrl.value)
    recordedAudioUrl.value = null
  }

  recordedBlob.value = null
}

const updateVolumeFrame = () => {
  if (!analyser || !dataArray || mode.value !== 'recording') {
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
  clearRecordedAudio()

  try {
    mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true })
    audioContext = new AudioContext()

    const source = audioContext.createMediaStreamSource(mediaStream)
    analyser = audioContext.createAnalyser()
    analyser.fftSize = 256

    source.connect(analyser)
    dataArray = new Uint8Array(analyser.frequencyBinCount)

    recordedChunks = []
    mediaRecorder = new MediaRecorder(mediaStream)
    mediaRecorder.ondataavailable = (event) => {
      if (event.data.size > 0) {
        recordedChunks.push(event.data)
      }
    }

    mediaRecorder.onstop = () => {
      const blob = new Blob(recordedChunks, { type: mediaRecorder?.mimeType || 'audio/webm' })
      recordedChunks = []

      if (blob.size === 0) {
        mode.value = 'idle'
        return
      }

      recordedBlob.value = blob
      recordedAudioUrl.value = URL.createObjectURL(blob)
      audioPlayer = new Audio(recordedAudioUrl.value)
      audioPlayer.onended = () => {
        isPlaying.value = false
        playbackProgress.value = 0
        if (audioPlayer) {
          audioPlayer.currentTime = 0
        }
      }
      audioPlayer.ontimeupdate = () => {
        if (!audioPlayer || !audioPlayer.duration) {
          playbackProgress.value = 0
          return
        }

        playbackProgress.value = Math.min(1, audioPlayer.currentTime / audioPlayer.duration)
      }

      mode.value = 'recorded'
    }

    mediaRecorder.start()

    recorderStartedAt = Date.now()
    elapsedSeconds.value = 0
    tickTimer()
    timerInterval = setInterval(tickTimer, 1000)

    mode.value = 'recording'
    updateVolumeFrame()
  } catch (error) {
    console.error('Microphone permission failed', error)
    cleanupRecorder()
    mode.value = 'idle'
  }
}

const stopRecording = () => {
  if (!mediaRecorder || mediaRecorder.state === 'inactive') {
    mode.value = 'idle'
    stopMonitoring()
    return
  }

  mode.value = 'idle'
  stopMonitoring()
  mediaRecorder.stop()
}

const togglePlayback = async () => {
  if (!audioPlayer) {
    return
  }

  if (isPlaying.value) {
    audioPlayer.pause()
    isPlaying.value = false
    return
  }

  try {
    await audioPlayer.play()
    isPlaying.value = true
  } catch (error) {
    console.error('Unable to play recording preview', error)
  }
}

const handlePrimaryAction = () => {
  if (mode.value === 'idle') {
    void startRecording()
    return
  }

  if (mode.value === 'recording') {
    stopRecording()
    return
  }

  void togglePlayback()
}

const handleRerecord = () => {
  cleanupRecorder()
  clearRecordedAudio()
  elapsedSeconds.value = 0
  mode.value = 'idle'
}

const handleSubmit = () => {
  if (!recordedBlob.value) {
    return
  }

  console.log('Submit recorded audio blob', {
    size: recordedBlob.value.size,
    type: recordedBlob.value.type,
    durationSeconds: elapsedSeconds.value
  })
}

const handleClose = () => {
  handleRerecord()
}

onBeforeUnmount(() => {
  cleanupRecorder()
  clearRecordedAudio()
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
  padding: 28px 18px calc(32px + env(safe-area-inset-bottom));
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

.recording-screen--recorded .recording-screen__active-glow {
  opacity: 0.14;
  transform: translate(-50%, -50%) scale(1);
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
  width: min(340px, 100%);
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

.recording-screen--recording .recording-screen__subtitle,
.recording-screen--recorded .recording-screen__subtitle {
  color: rgba(232, 255, 246, 0.84);
}

.recording-screen__playback-progress {
  margin: 18px auto 0;
  width: 180px;
  height: 5px;
  border-radius: 999px;
  overflow: hidden;
  background: rgba(232, 255, 246, 0.12);
}

.recording-screen__playback-progress-fill {
  display: block;
  width: 100%;
  height: 100%;
  transform-origin: left;
  transform: scaleX(0);
  background: linear-gradient(90deg, #00c97a 0%, #00e68a 100%);
  box-shadow: 0 0 18px rgba(0, 230, 138, 0.45);
  transition: transform 0.09s linear;
}

.recording-screen__footer {
  margin-top: auto;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 28px;
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

.recording-screen--recorded .record-button__wave {
  display: none;
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
  transition: transform 0.12s cubic-bezier(0.2, 0.8, 0.3, 1), box-shadow 0.2s ease;
  will-change: transform;
}

.recording-screen--playing .record-button__ring {
  transform: scale(1.04);
  box-shadow:
    0 0 26px rgba(0, 230, 138, 0.48),
    0 0 58px rgba(0, 230, 138, 0.3);
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

.record-button__play {
  width: 0;
  height: 0;
  margin-left: 4px;
  border-top: 13px solid transparent;
  border-bottom: 13px solid transparent;
  border-left: 19px solid #082217;
}

.record-button__pause {
  display: flex;
  gap: 6px;

  span {
    width: 7px;
    height: 24px;
    border-radius: 3px;
    background: #082217;
  }
}

.recording-actions {
  width: min(440px, 100%);
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 14px;
  padding: 0 4px;
}

.recording-actions__button {
  min-height: 52px;
  border-radius: 999px;
  font-size: 21px;
  font-weight: 500;
  transition: transform 0.2s ease, box-shadow 0.2s ease, border-color 0.2s ease, background 0.2s ease;

  &:active {
    transform: scale(0.985);
  }
}

.recording-actions__button--outline {
  border: 1px solid rgba(0, 230, 138, 0.3);
  background: rgba(0, 230, 138, 0.04);
  color: rgba(155, 235, 197, 0.88);

  &:hover {
    border-color: rgba(0, 230, 138, 0.5);
    box-shadow: 0 0 24px rgba(0, 230, 138, 0.12);
  }
}

.recording-actions__button--submit {
  border: 0;
  color: #001f14;
  background: linear-gradient(135deg, #00E68A, #00C97A);
  box-shadow:
    0 0 20px rgba(0, 230, 138, 0.4),
    0 0 40px rgba(0, 230, 138, 0.2);

  &:hover {
    transform: scale(1.015);
    box-shadow:
      0 0 24px rgba(0, 230, 138, 0.52),
      0 0 52px rgba(0, 230, 138, 0.3);
  }
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

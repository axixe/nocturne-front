<template>
  <header class="voice-header">
    <span class="voice-header__title">VOICE</span>
    <span class="voice-header__count">{{ progress }}</span>
  </header>
</template>

<script setup lang="ts">
defineProps<{ progress: string }>()
</script>

<style scoped lang="scss">
.voice-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 68px;
}

.voice-header__title,
.voice-header__count {
  color: rgba(232, 255, 246, 0.8);
  font-size: 16px;
  letter-spacing: 0.06em;
}
</style>

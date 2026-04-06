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
  margin-bottom: 84px;
  padding: 0 2px;
}

.voice-header__title,
.voice-header__count {
  color: rgba(232, 255, 246, 0.5);
  font-size: 13px;
  letter-spacing: 0.08em;
}
</style>

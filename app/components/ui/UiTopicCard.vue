<template>
  <article class="topic-card">
    <p class="topic-card__label">• TOPIC OF THE DAY</p>
    <p class="topic-card__text">“{{ text }}”</p>
  </article>
</template>

<script setup lang="ts">
defineProps<{ text: string }>()
</script>

<style scoped lang="scss">
.topic-card {
  position: relative;
  overflow: hidden;
  margin-top: 62px;
  padding: 19px 18px 20px;
  border-radius: 18px;
  border: 1px solid rgba(0, 230, 138, 0.2);
  background: rgba(0, 230, 138, 0.05);
  backdrop-filter: blur(10px);
  transition:
    border-color 0.25s ease,
    box-shadow 0.25s ease;

  &:hover {
    border-color: rgba(0, 230, 138, 0.42);
    box-shadow: 0 0 30px rgba(0, 230, 138, 0.12);
  }
}

.topic-card::before {
  content: '';
  position: absolute;
  inset: -30% 20% auto;
  height: 120px;
  pointer-events: none;
  background: radial-gradient(circle, rgba(0, 230, 138, 0.22), transparent 72%);
  filter: blur(28px);
}

.topic-card__label {
  margin-bottom: 12px;
  color: $color-primary;
  font-size: 15px;
  font-weight: 600;
  letter-spacing: 0.03em;
}

.topic-card__text {
  color: rgba(232, 255, 246, 0.92);
  font-size: 26px;
  line-height: 1.35;
}
</style>

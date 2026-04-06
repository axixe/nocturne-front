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
  margin-top: 62px;
  padding: 18px 18px 20px;
  border-radius: 16px;
  border: 1px solid rgba(0, 230, 138, 0.45);
  background: rgba(3, 26, 20, 0.6);
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

<template>
  <section class="hero-section">
    <h1 class="hero-section__title">
      <span>Send your voice.</span>
      <span>Hear others.</span>
    </h1>
    <p class="hero-section__subtitle">Record 1 message to unlock 5</p>

    <BaseButton>
      <template #icon>
        <BaseMicIcon />
      </template>
      Record voice
    </BaseButton>
  </section>
</template>

<style scoped lang="scss">
.hero-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.hero-section__title {
  display: grid;
  gap: 10px;
  margin-bottom: 20px;
  color: #E8FFF6;
  font-size: 52px;
  font-weight: 500;
  letter-spacing: 0.01em;
  line-height: 1.18;
}

.hero-section__subtitle {
  margin-bottom: 46px;
  color: rgba(232, 255, 246, 0.6);
  font-size: 18px;
  line-height: 1.4;
}
</style>

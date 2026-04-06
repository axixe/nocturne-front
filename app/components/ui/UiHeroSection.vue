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
  gap: 8px;
  margin-bottom: 18px;
  font-size: 44px;
  font-weight: 600;
  line-height: 1.15;
}

.hero-section__subtitle {
  margin-bottom: 34px;
  color: rgba(232, 255, 246, 0.76);
  font-size: 20px;
  line-height: 1.4;
}
</style>

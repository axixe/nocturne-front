<template>
  <div class="home-screen">
    <BaseScreenCard class="home-screen__card">
      <UiVoiceHeader progress="0/5" />
      <UiHeroSection />
      <UiTopicCard text="What moment made you laugh today?" />
      <UiBottomNav />
    </BaseScreenCard>
  </div>
</template>

<style scoped lang="scss">
.home-screen {
  min-height: 100dvh;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 28px 12px;
}

.home-screen__card {
  display: flex;
  flex-direction: column;
  width: 100%;
}
</style>

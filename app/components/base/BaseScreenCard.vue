<template>
  <section class="base-screen-card">
    <slot />
  </section>
</template>

<style scoped lang="scss">
.base-screen-card {
  position: relative;
  overflow: hidden;
  width: min(100%, 360px);
  min-height: 670px;
  border-radius: 26px;
  padding: 26px 20px 26px;
  color: $color-white;
  border: 1px solid rgba(0, 230, 138, 0.22);
  background:
    radial-gradient(circle at 50% 30%, rgba(0, 230, 138, 0.14), rgba(10, 15, 13, 0) 62%),
    linear-gradient(180deg, rgba(7, 21, 18, 0.92) 0%, rgba(5, 14, 12, 0.9) 100%);
  backdrop-filter: blur(7px);
  box-shadow:
    0 20px 60px rgba(0, 0, 0, 0.5),
    inset 0 0 70px rgba(0, 230, 138, 0.08);
}

.base-screen-card::before {
  content: '';
  position: absolute;
  inset: -28% -10% auto;
  height: 240px;
  pointer-events: none;
  background: radial-gradient(circle, rgba(0, 230, 138, 0.18) 0%, rgba(0, 230, 138, 0) 70%);
  filter: blur(38px);
}
</style>

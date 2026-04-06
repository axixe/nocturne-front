<template>
  <section class="base-screen-card">
    <slot />
  </section>
</template>

<style scoped lang="scss">
.base-screen-card {
  width: min(100%, 360px);
  min-height: 640px;
  border-radius: 24px;
  padding: 28px 18px 24px;
  color: $color-white;
  border: 1px solid rgba(0, 230, 138, 0.25);
  background:
    radial-gradient(circle at 50% 55%, rgba(0, 230, 138, 0.3), rgba(10, 15, 13, 0) 58%),
    linear-gradient(180deg, #041712 0%, #071110 100%);
  box-shadow:
    0 26px 50px rgba(0, 0, 0, 0.45),
    inset 0 0 80px rgba(0, 230, 138, 0.1);
}
</style>

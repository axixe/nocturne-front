<template>
  <nav class="bottom-nav">
    <button class="bottom-nav__item bottom-nav__item--active" type="button">Feed</button>
    <button class="bottom-nav__item" type="button">About</button>
  </nav>
</template>

<style scoped lang="scss">
.bottom-nav {
  margin-top: auto;
  display: flex;
  justify-content: center;
  gap: 44px;
  padding-bottom: 10px;
}

.bottom-nav__item {
  color: rgba(232, 255, 246, 0.45);
  font-size: 20px;
  transition: color 0.2s ease;

  &:hover,
  &--active {
    color: $color-primary;
  }
}
</style>

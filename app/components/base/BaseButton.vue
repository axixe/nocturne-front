<template>
  <button class="base-button" type="button">
    <span class="base-button__icon"><slot name="icon" /></span>
    <span class="base-button__label"><slot /></span>
  </button>
</template>

<style scoped lang="scss">
.base-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  width: 100%;
  max-width: 220px;
  min-height: 56px;
  padding: 0 20px;
  border-radius: 999px;
  background: linear-gradient(180deg, $color-primary 0%, $color-primary-medium 100%);
  color: #001f14;
  font-size: 26px;
  font-weight: 500;
  line-height: 1;
  transition:
    transform 0.2s ease,
    box-shadow 0.2s ease;

  &:hover {
    transform: translateY(-1px);
    box-shadow: 0 12px 24px rgba(0, 230, 138, 0.3);
  }

  &:active {
    transform: translateY(0);
  }
}

.base-button__icon,
.base-button__label {
  display: inline-flex;
  align-items: center;
}

.base-button__label {
  font-size: 26px;
  font-weight: 500;
}
</style>

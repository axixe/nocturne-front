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
  gap: 14px;
  width: 100%;
  max-width: 250px;
  min-height: 66px;
  padding: 0 26px;
  border-radius: 999px;
  background: linear-gradient(135deg, #00E68A, #00C97A);
  color: #001f14;
  font-size: 22px;
  font-weight: 500;
  line-height: 1;
  transition:
    transform 0.25s ease,
    box-shadow 0.25s ease,
    filter 0.25s ease;
  box-shadow:
    0 0 20px rgba(0, 230, 138, 0.4),
    0 0 40px rgba(0, 230, 138, 0.2);

  &:hover {
    transform: scale(1.05);
    filter: saturate(1.05);
    box-shadow:
      0 0 24px rgba(0, 230, 138, 0.5),
      0 0 52px rgba(0, 230, 138, 0.28);
  }

  &:active {
    transform: scale(1.015);
  }
}

.base-button__icon,
.base-button__label {
  display: inline-flex;
  align-items: center;
}

.base-button__label {
  font-size: 22px;
  font-weight: 500;
}
</style>

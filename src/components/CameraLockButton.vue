<template>
  <ControlButton
      :isOn="lockOn"
      onMessage="Unlock Camera"
      offMessage="Lock Camera"
      onIcon="/cam-lock.svg"
      offIcon="/cam-lock.svg"
      @toggle="emit('toggle-cam-lock')"
      class="cam-lock-button"
  />
</template>

<script setup lang="ts">
import ControlButton from './ControlButton.vue';

// props
defineProps<{
  lockOn: boolean;
}>();

// emits
const emit = defineEmits<{
  (e: 'toggle-cam-lock'): void
}>();
</script>

<style scoped>

</style>

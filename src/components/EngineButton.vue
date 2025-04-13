<template>
  <ControlButton
      :isOn="engineOn"
      onMessage="Stop Engine"
      offMessage="Start Engine"
      onIcon="/engine-on.svg"
      offIcon="/engine-off.svg"
      @toggle="emit('toggle-engine')"
      class="engine-button"
  />
</template>

<script setup lang="ts">
import ControlButton from './ControlButton.vue';

// props
defineProps<{
  engineOn: boolean;
}>();

// emits
const emit = defineEmits<{
  (e: 'toggle-engine'): void
}>();
</script>

<style scoped>
</style>

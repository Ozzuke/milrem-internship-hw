<template>
  <button @click="emit('toggle')">
    <img :src="isOn ? onIcon : offIcon" alt="Control Icon" class="control-icon"/>
    <span> {{ isOn ? onMessage : offMessage }} </span>
  </button>
</template>

<script setup lang="ts">
// props
defineProps<{
  isOn: boolean;
  onMessage: string;
  offMessage: string;
  onIcon: string;
  offIcon: string;
}>();

// emits
const emit = defineEmits<{
  (e: 'toggle'): void
}>();
</script>

<style scoped>
.control-icon {
  width: 24px;
  height: 24px;
  vertical-align: middle;
}

button {
  padding: 8px 15px;
  font-size: 1rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: left;
  background-color: rgba(128, 128, 128, 0.8);
  border: 2px solid black;
  color: white;
  border-radius: 5px;
  gap: 8px;
  transition: background-color 0.2s ease;
  min-width: 180px;
  text-align: center;
}

button:hover {
  background-color: rgba(100, 100, 100, 0.9);
}

button:focus {
  outline: 2px solid dodgerblue;
  outline-offset: 2px;
}

</style>

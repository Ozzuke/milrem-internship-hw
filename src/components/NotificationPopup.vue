<template>
  <div class="popup-overlay">
    <div class="popup-content">
      <p>
        <slot/>
      </p>
      <button @click="emit('close')">
        <img src="/close.svg" alt="Close" class="button-icon"/>
        <span>OK</span>
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import {onMounted, onUnmounted} from 'vue';

const emit = defineEmits<{
  (e: 'close'): void
}>();

const handleKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Escape' || event.key === ' ') {
    // prevent space from scrolling
    if (event.key === ' ') event.preventDefault();
    emit('close');
  }
};

onMounted(() => {
  window.addEventListener('keydown', handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
});
</script>

<style scoped>
.popup-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1001;
}

.popup-content {
  background-color: white;
  padding: 30px 35px;
  border-radius: 8px;
  text-align: center;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
  min-width: 250px;
  max-width: 90%;
}

p {
  margin-bottom: 20px;
  font-size: 1.1rem;
}

button {
  font-size: 1rem;
  margin-top: 15px;
  padding: 10px 25px;
  cursor: pointer;
  border: 1px solid #ccc;
  border-radius: 5px;
  background-color: #f0f0f0;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  transition: background-color 0.2s ease;
}

button:hover {
  background-color: #e0e0e0;
}

button:focus {
  outline: 2px solid dodgerblue;
  outline-offset: 2px;
}

.button-icon {
  width: 18px;
  height: 18px;
  vertical-align: middle;
}
</style>

<template>
  <div class="waypoint-list-container" v-if="waypoints.length > 0">
    <h4>Saved Waypoints</h4>
    <ul>
      <li
        v-for="wp in waypoints"
        :key="wp.id"
        @click="emit('select-waypoint', wp)"
      >
        {{ wp.name }}
      </li>
    </ul>
  </div>
</template>

<script setup lang="ts">
interface Waypoint {
  id: number;
  name: string;
  lat: number;
  lng: number;
}

// props
defineProps<{
  waypoints: Waypoint[];
}>();

// emits
const emit = defineEmits<{
  (e: 'select-waypoint', waypoint: Waypoint): void
}>();
</script>

<style scoped>
.waypoint-list-container {
  position: fixed;
  bottom: 15px;
  left: 15px;
  background-color: rgba(255, 255, 255, 0.9);
  border: 1px solid #ccc;
  border-radius: 5px;
  padding: 5px 15px;
  max-height: 200px;
  overflow-y: auto;
  z-index: 1000;
  max-width: 300px;
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

h4 {
  margin-top: 5px;
  margin-bottom: 10px;
  text-align: center;
  font-size: 0.9rem;
  color: #333;
}

ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

li {
  padding: 6px 8px;
  cursor: pointer;
  border-bottom: 1px solid #eee;
  font-size: 0.85rem;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

li:last-child {
  border-bottom: none;
}

li:hover {
  background-color: #e9e9e9;
}
</style>

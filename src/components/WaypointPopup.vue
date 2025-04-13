<template>
  <div class="popup-overlay" @click.self="handleOverlayClick">
    <div class="popup-content">
      <h4>{{ mode === 'new' ? 'New Waypoint' : 'Manage Waypoint' }}</h4>

      <!-- Display Coords or Name -->
      <div class="waypoint-info">
        <div v-if="waypointData?.coords">
          Coords: {{ waypointData.coords.lat.toFixed(5) }}, {{ waypointData.coords.lng.toFixed(5) }}
        </div>
        <div v-if="waypointData?.waypoint">
          Current Name: <strong>{{ waypointData.waypoint.name }}</strong>
          <br>
          <span class="coords-display">({{
              waypointData.waypoint.lat.toFixed(5)
            }}, {{ waypointData.waypoint.lng.toFixed(5) }})</span>
        </div>
      </div>


      <!-- Input for New/Rename -->
      <div v-if="mode === 'new' || isRenaming" class="input-group">
        <label for="wp-name">Name:</label>
        <input type="text" id="wp-name" v-model="waypointName" ref="nameInputRef" @keydown.enter="handleNameEnterKey"/>
      </div>

      <!-- Button Groups -->
      <div class="button-group">
        <!-- New Mode Buttons -->
        <template v-if="mode === 'new'">
          <button @click="handleSave" :disabled="!waypointName.trim()">
            <img src="/rename.svg" alt="Save" class="button-icon"/> Save
          </button>
          <button @click="handleDrive">
            <img src="/drive-to.svg" alt="Drive" class="button-icon"/> Drive Here
          </button>
          <button @click="emit('close')" class="discard">
            <img src="/close.svg" alt="Discard" class="button-icon"/> Discard
          </button>
        </template>

        <!-- Manage Mode Buttons -->
        <template v-if="mode === 'manage'">
          <!-- Show Rename or Save Name buttons -->
          <button v-if="!isRenaming" @click="startRename">
            <img src="/rename.svg" alt="Rename" class="button-icon"/> Rename
          </button>
          <button v-if="isRenaming" @click="handleRename" :disabled="!waypointName.trim()">
            <img src="/rename.svg" alt="Save" class="button-icon"/> Save Name
          </button>
          <!-- Show Cancel only when renaming -->
          <button v-if="isRenaming" @click="cancelRename">
            <img src="/close.svg" alt="Cancel" class="button-icon"/> Cancel
          </button>
          <!-- Show Drive and Delete only when not renaming -->
          <button v-if="!isRenaming" @click="handleDrive">
            <img src="/drive-to.svg" alt="Drive" class="button-icon"/> Drive Here
          </button>
          <button v-if="!isRenaming" @click="handleDelete" class="delete">
            <img src="/delete.svg" alt="Delete" class="button-icon"/> Delete
          </button>
        </template>
      </div>
      <button v-if="mode === 'manage' && !isRenaming" @click="emit('close')" class="close-manage">
        <img src="/close.svg" alt="Close" class="button-icon"/> Close
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import {ref, watch, nextTick, onMounted, onUnmounted} from 'vue';
import type {LatLng} from 'leaflet';

// types
interface Waypoint {
  id: number;
  name: string;
  lat: number;
  lng: number;
}

type WaypointPopupMode = 'new' | 'manage';

// props
const props = defineProps<{
  mode: WaypointPopupMode;
  waypointData: { waypoint?: Waypoint | null, coords?: LatLng | null };
}>();

// emits
const emit = defineEmits<{
  (e: 'save-waypoint', name: string): void
  (e: 'drive-to', target: Waypoint | LatLng): void
  (e: 'delete-waypoint', waypoint: Waypoint): void
  (e: 'rename-waypoint', waypoint: Waypoint, newName: string): void
  (e: 'close'): void
}>();

// refs
const waypointName = ref('');
const isRenaming = ref(false);
const nameInputRef = ref<HTMLInputElement | null>(null);

// watch for mode and waypointData changes
watch(() => [props.mode, props.waypointData], ([newMode, newData]) => {
  isRenaming.value = false;
  if (newMode === 'new' && newData?.coords) {
    // default name based on coords
    waypointName.value = `Wpt (${newData.coords.lat.toFixed(3)}, ${newData.coords.lng.toFixed(3)})`;
    nextTick(() => nameInputRef.value?.focus());
  } else if (newMode === 'manage' && newData?.waypoint) {
    waypointName.value = newData.waypoint.name;
  } else {
    waypointName.value = '';
  }
}, {immediate: true, deep: true});

// --- methods ---
const handleSave = () => {
  if (waypointName.value.trim() && props.mode === 'new') {
    emit('save-waypoint', waypointName.value.trim());
  }
};

const handleDrive = () => {
  const target = props.mode === 'new' ? props.waypointData.coords : props.waypointData.waypoint;
  if (target) {
    emit('drive-to', target);
  }
};

const handleDelete = () => {
  if (props.mode === 'manage' && props.waypointData.waypoint) {
    emit('delete-waypoint', props.waypointData.waypoint);
  }
};

const startRename = () => {
  isRenaming.value = true;
  nextTick(() => {
    nameInputRef.value?.focus();
    nameInputRef.value?.select();
  });
};

const cancelRename = () => {
  isRenaming.value = false;
  // reset name back to original
  if (props.waypointData.waypoint) {
    waypointName.value = props.waypointData.waypoint.name;
  }
};

const handleRename = () => {
  if (props.mode === 'manage' && props.waypointData.waypoint && waypointName.value.trim()) {
    // only emit if the name actually changed
    if (waypointName.value.trim() !== props.waypointData.waypoint.name) {
      emit('rename-waypoint', props.waypointData.waypoint, waypointName.value.trim());
    }
    isRenaming.value = false;
  }
};

// handle Enter key in name input
const handleNameEnterKey = () => {
  if (isRenaming.value) {
    handleRename();
  } else if (props.mode === 'new') {
    handleSave();
  }
};

// close popup on esc key
const handleKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Escape') {
    if (isRenaming.value) {
      cancelRename(); // cancel rename instead of closing
    } else {
      emit('close');
    }
  }
};

// close popup when clicking outside the content
const handleOverlayClick = () => {
  if (!isRenaming.value) { // don't close if renaming
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
  z-index: 1100;
}

.popup-content {
  background-color: white;
  padding: 20px 30px;
  border-radius: 8px;
  text-align: center;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
  min-width: 320px;
  max-width: 90%;
}

h4 {
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.2rem;
  color: #333;
}

.waypoint-info {
  margin-bottom: 15px;
  font-size: 0.9rem;
  color: #555;
}

.coords-display {
  font-size: 0.8rem;
  color: #777;
}


.input-group {
  margin: 15px 0;
  text-align: left;
}

.input-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
  font-size: 0.9rem;
}

.input-group input {
  width: calc(100% - 18px);
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  font-size: 0.95rem;
}

.input-group input:focus {
  outline: 2px solid dodgerblue;
  border-color: dodgerblue;
}


.button-group {
  margin-top: 20px;
  display: flex;
  justify-content: center;
  gap: 10px;
  flex-wrap: wrap;
}

button {
  padding: 8px 15px;
  cursor: pointer;
  border: 1px solid #ccc;
  border-radius: 4px;
  background-color: #f0f0f0;
  font-size: 0.9rem;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  transition: background-color 0.2s ease, border-color 0.2s ease;
}

button:hover {
  background-color: #e0e0e0;
  border-color: #bbb;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
  background-color: #f0f0f0;
}

button:focus {
  outline: 2px solid dodgerblue;
  outline-offset: 1px;
}


.button-icon {
  width: 16px;
  height: 16px;
}

button.discard, button.delete {
  background-color: #f8d7da;
  border-color: #f5c6cb;
  color: #721c24;
}

button.discard:hover, button.delete:hover {
  background-color: #f1b0b7;
  border-color: #ee9ca7;
}

.close-manage {
  margin-top: 10px;
  background-color: #e2e6ea;
  border-color: #dae0e5;
  color: #343a40;
}

.close-manage:hover {
  background-color: #d3d9df;
  border-color: #c6cdd4;
}
</style>

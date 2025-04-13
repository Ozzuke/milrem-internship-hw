<template>
  <div id="app-container">
    <!-- Map -->
    <MapComponent
        ref="mapComponentRef"
        :ugvPosition="ugvPosition"
        :ugvHeading="ugvHeading"
        :savedWaypoints="savedWaypoints"
        @long-press="handleMapLongPress"
        @select-waypoint-marker="handleSelectWaypointMarker"
    />

    <!-- Controls -->
    <div class="top-right-controls">
      <EngineButton :engineOn="engineOn" @toggle-engine="toggleEngine"/>
      <CameraLockButton :lockOn="cameraLocked" @toggle-cam-lock="toggleCameraLock"/>
    </div>

    <WaypointList
        :waypoints="savedWaypoints"
        @select-waypoint="handleSelectWaypointFromList"
    />

    <!-- Popups -->
    <NotificationPopup v-if="showEngineOffNotification" @close="showEngineOffNotification = false">
      Please start the engine to move the UGV.
    </NotificationPopup>
    <WaypointPopup
        v-if="showWaypointPopup"
        :mode="waypointPopupMode"
        :waypointData="waypointPopupData"
        @save-waypoint="saveWaypoint"
        @drive-to="driveToWaypoint"
        @delete-waypoint="deleteWaypoint"
        @rename-waypoint="renameWaypoint"
        @close="closeWaypointPopup"
    />
  </div>
</template>

<script setup lang="ts">
import {nextTick, onMounted, onUnmounted, reactive, ref} from 'vue';
import MapComponent from './components/MapComponent.vue';
import EngineButton from './components/EngineButton.vue';
import WaypointList from './components/WaypointList.vue';
import NotificationPopup from './components/NotificationPopup.vue';
import WaypointPopup from './components/WaypointPopup.vue';
import type {LatLng, Map as LeafletMap, Marker as LeafletMarker} from 'leaflet';
import CameraLockButton from "./components/CameraLockButton.vue";

// types
interface Waypoint {
  id: number;
  name: string;
  lat: number;
  lng: number;
}

type WaypointPopupMode = 'new' | 'manage';

// constants
const MAX_SPEED = 0.00002;
const ACCELERATION = 0.000001;
const DECELERATION = 0.000002;
const TURN_RATE = 5;
const UPDATE_INTERVAL = 50;

// state
const engineOn = ref(false);
const cameraLocked = ref(false);

const ugvPosition = reactive({lat: 58.375893, lng: 26.720945}); // Milrem Tartu
const ugvHeading = ref(0);
const currentSpeed = ref(0);
const keysPressed = reactive({ArrowUp: false, ArrowDown: false, ArrowLeft: false, ArrowRight: false});

const savedWaypoints = ref<Waypoint[]>([]);
let nextWaypointId = 1;

const showEngineOffNotification = ref(false);
const showWaypointPopup = ref(false);
const waypointPopupMode = ref<WaypointPopupMode>('new');
const waypointPopupData = reactive<{ waypoint?: Waypoint | null, coords?: LatLng | null }>({
  waypoint: null,
  coords: null
});

let gameLoopInterval: number | null = null;
const mapComponentRef = ref<any>(null);

// methods
const toggleEngine = () => {
  engineOn.value = !engineOn.value;
  if (!engineOn.value) {
    // Stop movement if engine is turned off
    keysPressed.ArrowUp = false;
    keysPressed.ArrowDown = false;
    keysPressed.ArrowLeft = false;
    keysPressed.ArrowRight = false;
  }
  console.log('Engine toggled:', engineOn.value);
};

const toggleCameraLock = () => {
  cameraLocked.value = !cameraLocked.value;
  if (cameraLocked.value) {
    panMapToUgv(); // pan when locking
  }
};

const panMapToUgv = () => {
  nextTick(() => {
    const mapCompInstance = mapComponentRef.value;
    if (mapCompInstance && mapCompInstance.mapInstance) {
      const leafletMap = mapCompInstance.mapInstance as LeafletMap;
      leafletMap.panTo([ugvPosition.lat, ugvPosition.lng]);
    } else {
      console.warn("Cannot get map instance to pan.");
    }
  });
};


const handleKeyDown = (event: KeyboardEvent) => {
  // prevent browser default actions for keys we handle
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'e', 'E', 'Enter', 'c', 'C'].includes(event.key)) {
      event.preventDefault();
  }

  // engine toggle
  if (event.key === 'e' || event.key === 'E' || event.key === 'Enter') {
      toggleEngine();
      return;
  }

  // cam lock toggle
  if (event.key === 'c' || event.key === 'C') {
      toggleCameraLock();
      return;
  }

  // movement keys
  if (['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'].includes(event.key)) {
    if (!engineOn.value) { // if engine is off show notification
      if (event.key === 'ArrowUp' || event.key === 'ArrowDown' || event.key === 'ArrowLeft' || event.key === 'ArrowRight') {
         showEngineOffNotification.value = true;
      }
      return;
    }
    if (event.key in keysPressed) {
      keysPressed[event.key as keyof typeof keysPressed] = true;
    }
  }
};

const handleKeyUp = (event: KeyboardEvent) => {
  if (event.key in keysPressed) {
    keysPressed[event.key as keyof typeof keysPressed] = false;
  }
};

const updateMovement = () => {
  if (!engineOn.value && currentSpeed.value === 0) return;

  // turning
  if (keysPressed.ArrowLeft) {
    ugvHeading.value = (ugvHeading.value - TURN_RATE + 360) % 360;
  }
  if (keysPressed.ArrowRight) {
    ugvHeading.value = (ugvHeading.value + TURN_RATE) % 360;
  }

  // accel / decel
  let targetSpeed = 0;
  if (engineOn.value) {
    if (keysPressed.ArrowUp) targetSpeed = MAX_SPEED;
    else if (keysPressed.ArrowDown) targetSpeed = -MAX_SPEED / 2; // Allow reversing
  }

  // apply accel / decel
  if (currentSpeed.value < targetSpeed) {
    currentSpeed.value = Math.min(targetSpeed, currentSpeed.value + ACCELERATION);
  } else if (currentSpeed.value > targetSpeed) {
    currentSpeed.value = Math.max(targetSpeed, currentSpeed.value - DECELERATION);
  }

  // stop if speed very low
  if (Math.abs(currentSpeed.value) < (DECELERATION / 2) && !keysPressed.ArrowUp && !keysPressed.ArrowDown && !engineOn.value) {
      currentSpeed.value = 0;
  }


  // position update
  let positionChanged = false;
  if (Math.abs(currentSpeed.value) > 1e-9) { // small threshold to check if moving
    const headingRad = ugvHeading.value * (Math.PI / 180);
    // latitude changes directly
    const deltaLat = Math.cos(headingRad) * currentSpeed.value;
    // longitude change depends on latitude (due to curvature)
    const latRad = ugvPosition.lat * (Math.PI / 180);
    const lngCorrectionFactor = Math.cos(latRad); // correction factor
    const deltaLng = Math.sin(headingRad) * currentSpeed.value / lngCorrectionFactor;


    if (Math.abs(deltaLat) > 1e-10 || Math.abs(deltaLng) > 1e-10) { // smaller threshold for update check
      ugvPosition.lat += deltaLat;
      ugvPosition.lng += deltaLng;
      positionChanged = true;
    }

  } else if (currentSpeed.value !== 0) {
     currentSpeed.value = 0;
  }

  if (positionChanged) {
    nextTick(() => {
      const mapCompInstance = mapComponentRef.value;
      if (mapCompInstance && mapCompInstance.ugvMarkerRef && mapCompInstance.ugvMarkerRef.leafletObject) {
        const leafletMarker = mapCompInstance.ugvMarkerRef.leafletObject as LeafletMarker;
        leafletMarker.setLatLng([ugvPosition.lat, ugvPosition.lng]);

        if (cameraLocked.value && mapCompInstance.mapInstance) {
          const leafletMap = mapCompInstance.mapInstance as LeafletMap;
          leafletMap.panTo([ugvPosition.lat, ugvPosition.lng]);
        }
      }
    });
  }
};


// --- waypoint stuff ---

const handleMapLongPress = (coords: LatLng) => {
  waypointPopupMode.value = 'new';
  waypointPopupData.coords = coords;
  waypointPopupData.waypoint = null;
  showWaypointPopup.value = true;
};

const handleSelectWaypointMarker = (waypointId: number) => {
  const waypoint = savedWaypoints.value.find(wp => wp.id === waypointId);
  if (waypoint) {
    waypointPopupMode.value = 'manage';
    waypointPopupData.waypoint = waypoint;
    waypointPopupData.coords = null;
    showWaypointPopup.value = true;
  }
};

const handleSelectWaypointFromList = (waypoint: Waypoint) => {
  waypointPopupMode.value = 'manage';
  waypointPopupData.waypoint = waypoint;
  waypointPopupData.coords = null;
  showWaypointPopup.value = true;
};

const saveWaypoint = (name: string) => {
  if (waypointPopupMode.value === 'new' && waypointPopupData.coords) {
    const newId = nextWaypointId++;
    savedWaypoints.value.push({
      id: newId,
      name: name || `Waypoint ${newId}`,
      lat: waypointPopupData.coords.lat,
      lng: waypointPopupData.coords.lng,
    });
    console.log('Waypoint saved:', savedWaypoints.value[savedWaypoints.value.length - 1]);
  }
  closeWaypointPopup();
};

const driveToWaypoint = (target: Waypoint | LatLng) => {
  const targetLat = 'lat' in target ? target.lat : target.lat;
  const targetLng = 'lng' in target ? target.lng : target.lng;

  // directly set position
  ugvPosition.lat = targetLat;
  ugvPosition.lng = targetLng;
  currentSpeed.value = 0;

  // force map update and pan
  nextTick(() => {
    const mapCompInstance = mapComponentRef.value;
    if (mapCompInstance && mapCompInstance.ugvMarkerRef && mapCompInstance.ugvMarkerRef.leafletObject) {
      const leafletMarker = mapCompInstance.ugvMarkerRef.leafletObject as LeafletMarker;
      leafletMarker.setLatLng([ugvPosition.lat, ugvPosition.lng]);
      // always pan on driving to waypoint (regardless of camera lock)
      panMapToUgv();
    } else {
      console.warn('Could not get marker/map instance to force update/pan');
    }
  });

  closeWaypointPopup();
};


const deleteWaypoint = (waypointToDelete: Waypoint) => {
  savedWaypoints.value = savedWaypoints.value.filter(wp => wp.id !== waypointToDelete.id);
  console.log('Waypoint deleted:', waypointToDelete.id);
  closeWaypointPopup();
};

const renameWaypoint = (waypointToRename: Waypoint, newName: string) => {
  const index = savedWaypoints.value.findIndex(wp => wp.id === waypointToRename.id);
  if (index !== -1 && newName.trim()) {
    savedWaypoints.value[index].name = newName.trim();
    console.log('Waypoint renamed:', waypointToRename.id, 'to', newName.trim());
  }
  closeWaypointPopup();
};

const closeWaypointPopup = () => {
  showWaypointPopup.value = false;
  waypointPopupData.waypoint = null;
  waypointPopupData.coords = null;
};


onMounted(() => {
  window.addEventListener('keydown', handleKeyDown);
  window.addEventListener('keyup', handleKeyUp);
  gameLoopInterval = window.setInterval(updateMovement, UPDATE_INTERVAL);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown);
  window.removeEventListener('keyup', handleKeyUp);
  if (gameLoopInterval) {
    clearInterval(gameLoopInterval);
  }
});

</script>

<style>
#app-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

body {
  margin: 0;
  font-family: sans-serif;
}

.top-right-controls {
    position: fixed;
    top: 15px;
    right: 15px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    z-index: 1000;
}
</style>

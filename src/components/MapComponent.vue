<template>
  <div style="height: 100%; width: 100%" @mousedown="handleMouseDown" @mouseup="handleMouseUp"
       @mouseleave="handleMouseUp" @mousemove="handleMouseMove">
    <l-map
        ref="mapRef"
        :zoom="mapZoom"
        :center="mapCenter"
        :options="{ zoomControl: false, keyboard: false }"
        style="height: 100%; width: 100%;"
        @ready="onMapReady"
    >
      <l-tile-layer
          url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
          layer-type="base"
          name="OpenStreetMap"
      />

      <!-- UGV Marker -->
      <l-marker
          ref="ugvMarkerRef"
          :lat-lng="ugvPosition"
          :options="{ rotationAngle: ugvHeading, rotationOrigin: 'center center' }"
      >
        <l-icon :icon-url="ugvIconUrl" :icon-size="[40, 40]" :icon-anchor="[20, 20]"/>
      </l-marker>

      <!-- Saved Waypoint Markers -->
      <l-marker
          v-for="wp in savedWaypoints"
          :key="wp.id"
          :lat-lng="[wp.lat, wp.lng]"
          @click="() => emit('select-waypoint-marker', wp.id)"
      >
        <l-icon :icon-url="waypointIconUrl" :icon-size="[32, 32]" :icon-anchor="[16, 32]"/>
        <l-tooltip>{{ wp.name }}</l-tooltip>
      </l-marker>

    </l-map>
  </div>
</template>

<script setup lang="ts">
import {ref, watch, nextTick} from 'vue';
import {LMap, LTileLayer, LIcon, LTooltip, LMarker} from "@vue-leaflet/vue-leaflet";
import type {LatLngExpression, Map as LeafletMap, LatLng, Marker as LeafletMarker} from 'leaflet';

import ugvIconUrl from '/ugv.png';
import waypointIconUrl from '/waypoint.svg';

// props
const props = defineProps<{
  ugvPosition: { lat: number; lng: number };
  ugvHeading: number;
  savedWaypoints: Array<{ id: number; name: string; lat: number; lng: number }>;
}>();

// emits
const emit = defineEmits<{
  (e: 'long-press', coords: LatLng): void
  (e: 'select-waypoint-marker', id: number): void
}>();

// refs
const mapRef = ref<any>(null);
const ugvMarkerRef = ref<any>(null);
const mapZoom = ref(15);
const mapCenter = ref<LatLngExpression>([props.ugvPosition.lat, props.ugvPosition.lng]);
const mapInstance = ref<LeafletMap | null>(null);

const onMapReady = () => {
  if (mapRef.value && mapRef.value.leafletObject) {
    mapInstance.value = mapRef.value.leafletObject;
    mapInstance.value?.setView([props.ugvPosition.lat, props.ugvPosition.lng], mapZoom.value);
  } else {
    console.error('leafletObject not found on mapRef');
  }
};

// rotation update watcher
watch(() => props.ugvHeading, (newAngle) => {
  nextTick(() => {
    const markerComponent = ugvMarkerRef.value;
    if (markerComponent && markerComponent.leafletObject) {
      const leafletMarker = markerComponent.leafletObject as LeafletMarker & {
        setRotationAngle?: (angle: number) => void
      };
      if (typeof leafletMarker.setRotationAngle === 'function') {
        leafletMarker.setRotationAngle(newAngle);
      } else {
        console.warn('setRotationAngle method not found');
      }
    }
  });
});


// long press logic
const longPressTimeout = ref<number | null>(null);
const LONG_PRESS_DURATION = 750;
const MAX_MOVE_THRESHOLD = 5; // pixels
const mouseDownPos = ref<{ x: number, y: number } | null>(null);
const isDraggingMap = ref(false);

const handleMouseDown = (event: MouseEvent) => {
  if (!mapInstance.value) return;
  const targetElement = event.target as HTMLElement;

  // ignore if on markers or controls
  if (targetElement.closest('.leaflet-marker-icon, .leaflet-control')) {
    return;
  }
  // ensure click is within map container
  if (!mapInstance.value.getContainer().contains(targetElement) && !mapInstance.value.getContainer().isSameNode(targetElement)) {
    return;
  }

  isDraggingMap.value = false;
  mouseDownPos.value = {x: event.clientX, y: event.clientY};
  clearTimeout(longPressTimeout.value!);

  longPressTimeout.value = window.setTimeout(() => {
    if (!isDraggingMap.value && mapInstance.value && mouseDownPos.value) {
      // check movement again at timeout time
      const currentMousePos = {x: event.clientX, y: event.clientY};
      const dx = currentMousePos.x - mouseDownPos.value.x;
      const dy = currentMousePos.y - mouseDownPos.value.y;

      if (Math.sqrt(dx * dx + dy * dy) <= MAX_MOVE_THRESHOLD) {
        const point = mapInstance.value.mouseEventToContainerPoint(event);
        const coords = mapInstance.value.containerPointToLatLng(point);
        console.log('Long press emitted for coords:', coords);
        emit('long-press', coords);
      }
    }
    mouseDownPos.value = null;
    longPressTimeout.value = null;
  }, LONG_PRESS_DURATION);

  mapInstance.value.once('movestart', () => {
    isDraggingMap.value = true;
    clearTimeout(longPressTimeout.value!);
    longPressTimeout.value = null;
    mouseDownPos.value = null;
  });
};

const handleMouseMove = (event: MouseEvent) => {
  if (longPressTimeout.value && mouseDownPos.value) {
    const dx = event.clientX - mouseDownPos.value.x;
    const dy = event.clientY - mouseDownPos.value.y;
    if (Math.sqrt(dx * dx + dy * dy) > MAX_MOVE_THRESHOLD) {
      clearTimeout(longPressTimeout.value);
      longPressTimeout.value = null;
      mouseDownPos.value = null;
    }
  }
};

const handleMouseUp = () => {
  clearTimeout(longPressTimeout.value!);
  longPressTimeout.value = null;
  mouseDownPos.value = null;
  isDraggingMap.value = false;
  if (mapInstance.value) {
    mapInstance.value.off('movestart'); // Clean up listener
  }
};

// expose marker ref and map instance
defineExpose({
  ugvMarkerRef,
  mapInstance
});

</script>

<style scoped>

</style>

<script setup>
import { onMounted, onUnmounted, watch, ref } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

const props = defineProps({
  suburbs: {
    type: Array,
    default: () => [],
  },
  selectedSuburb: {
    type: Object,
    default: null,
  },
  userLocation: {
    type: Object, // { lat, lng }
    default: null,
  },
})

const emit = defineEmits(['select'])

const mapContainer = ref(null)
let map = null
let geoJsonLayer = null
let locationMarker = null

const DEFAULT_CENTER = [-37.8136, 144.9631]
const DEFAULT_ZOOM = 12
const SELECTED_ZOOM = 14
let fitZoom = DEFAULT_ZOOM
let fitCenter = DEFAULT_CENTER

const riskColors = {
  high: { fill: '#c0392b', border: '#96281b' },
  moderate: { fill: '#e8903a', border: '#c97a2a' },
  low: { fill: '#4d9e5a', border: '#3a7d45' },
  default: { fill: '#c8bfb0', border: '#b0a898' },
}

function getStyle(feature) {
  const name = feature.properties.name
  const suburb = props.suburbs.find(
    (s) => s.suburb_name.toLowerCase().trim() === name.toLowerCase().trim(),
  )
  const level = suburb?.risk_level ?? 'default'
  const color = riskColors[level] ?? riskColors.default
  const isSelected =
    props.selectedSuburb?.suburb_name?.toLowerCase().trim() === name.toLowerCase().trim()

  return {
    fillColor: color.fill,
    fillOpacity: isSelected ? 0.75 : 0.50,
    color: color.border,
    weight: isSelected ? 2 : 0.8,
    opacity: 1,
  }
}

function onEachFeature(feature, layer) {
  layer.on({
    click: () => {
      const name = feature.properties.name
      const suburb = props.suburbs.find(
        (s) => s.suburb_name.toLowerCase().trim() === name.toLowerCase().trim(),
      )
      if (suburb) emit('select', suburb)
    },
    mouseover: (e) => {
      e.target.setStyle({ fillOpacity: 0.75, weight: 2 })
    },
    mouseout: (e) => {
      geoJsonLayer.resetStyle(e.target)
    },
  })

  const name = feature.properties.name
  const suburb = props.suburbs.find(
    (s) => s.suburb_name.toLowerCase().trim() === name.toLowerCase().trim(),
  )
  if (suburb) {
    layer.bindTooltip(
      `<strong>${suburb.suburb_name}</strong><br>⚠️ ${suburb.risk_level.charAt(0).toUpperCase() + suburb.risk_level.slice(1)} Risk<br>🌡️ Feels like ${Math.round(suburb.apparent_temperature ?? suburb.temperature)}°C`,
      { sticky: true, className: 'suburb-tooltip' },
    )
  }
}

async function initMap() {
  map = L.map(mapContainer.value, {
    center: DEFAULT_CENTER,
    zoom: DEFAULT_ZOOM,
    minZoom: 11, // prevent zooming out beyond Greater Melbourne
    maxZoom: 18,
    zoomControl: true,
    scrollWheelZoom: true,
    maxBounds: [
      [-39.5, 140.5], // south-west corner of Victoria
      [-33.5, 150.5], // north-east corner of Victoria
    ],
    maxBoundsViscosity: 1.0, // hard boundary, user cannot drag outside
  })

  // Background handled via CSS .leaflet-container override below

  L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager_only_labels/{z}/{x}/{y}{r}.png', {
    attribution: '© OpenStreetMap © CARTO',
    maxZoom: 18,
    minZoom: 11,
  }).addTo(map)

  try {
    const res = await fetch('/geojson/melbourne-suburbs.geojson')
    const geojson = await res.json()

    geoJsonLayer = L.geoJSON(geojson, {
      style: getStyle,
      onEachFeature,
    }).addTo(map)

    map.fitBounds(geoJsonLayer.getBounds(), { padding: [20, 20] })
    // Store the resulting zoom/center so deselect can fly back to exactly this view
    map.once('moveend', () => {
      fitZoom = map.getZoom()
      fitCenter = [map.getCenter().lat, map.getCenter().lng]
    })

    // 地图初始化完成后，如果 userLocation 已经有值（定位比地图快），补渲染 marker
    if (props.userLocation) {
      updateLocationMarker(props.userLocation)
    }

    // 地图初始化完成后，如果父组件已经传入了 selectedSuburb（从路由跳转），补一次 flyTo
    if (props.selectedSuburb) {
      refreshStyles()
      flyToSuburb(props.selectedSuburb)
    }
  } catch (e) {
    console.error('Failed to load GeoJSON:', e)
  }
}

function refreshStyles() {
  if (geoJsonLayer) {
    geoJsonLayer.setStyle(getStyle)
  }
}

// Fly to selected suburb centroid, or back to default when deselected
function flyToSuburb(suburb) {
  if (!map || !geoJsonLayer) return

  if (!suburb) {
    map.flyTo(fitCenter, fitZoom, { duration: 0.6 })
    return
  }

  // Find the matching GeoJSON layer and fly to its centre
  geoJsonLayer.eachLayer((layer) => {
    const name = layer.feature?.properties?.name
    if (name?.toLowerCase().trim() === suburb.suburb_name.toLowerCase().trim()) {
      const center = layer.getBounds().getCenter()
      map.flyTo(center, SELECTED_ZOOM, { duration: 0.8 })
    }
  })
}

function updateLocationMarker(location) {
  if (!map) return
  if (locationMarker) {
    locationMarker.remove()
    locationMarker = null
  }
  if (!location) return

  // 蓝色脉冲圆点
  const icon = L.divIcon({
    className: '',
    html: `<div class="user-location-marker"><div class="user-location-dot"></div><div class="user-location-pulse"></div></div>`,
    iconSize: [20, 20],
    iconAnchor: [10, 10],
  })

  locationMarker = L.marker([location.lat, location.lng], { icon, zIndexOffset: 1000 })
    .addTo(map)
    .bindTooltip('Your location', { permanent: false, className: 'suburb-tooltip' })
}

onMounted(() => {
  initMap()
})

onUnmounted(() => {
  if (map) {
    map.remove()
    map = null
  }
})

watch(() => props.suburbs, refreshStyles, { deep: true })
watch(
  () => props.selectedSuburb,
  (suburb) => {
    refreshStyles()
    flyToSuburb(suburb)
  },
)
watch(
  () => props.userLocation,
  (location) => {
    updateLocationMarker(location)
  },
)
</script>

<template>
  <div class="map-wrap">
    <div ref="mapContainer" class="map-container"></div>
  </div>
</template>

<style scoped>
.map-wrap {
  width: 100%;
  height: 100%;
  border-radius: 14px;
  overflow: hidden;
  position: relative;
  z-index: 0;
}

.map-container {
  width: 100%;
  height: 100%;
  min-height: 480px;
}
</style>

<style>
/* Match Leaflet background to page gradient */
.leaflet-container {
  background: #ffffff !important;
}

.suburb-tooltip {
  font-family: system-ui, sans-serif;
  font-size: 0.82rem;
  padding: 7px 11px;
  border-radius: 8px;
  border: 1px solid #d8eae6 !important;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1) !important;
  background: #ffffff !important;
  color: #1c2e2a !important;
}

.suburb-tooltip::before {
  display: none;
}

.user-location-marker {
  position: relative;
  width: 20px;
  height: 20px;
}

.user-location-dot {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 12px;
  height: 12px;
  background: #2d7a3a;
  border: 2.5px solid #ffffff;
  border-radius: 50%;
  box-shadow: 0 0 6px rgba(45,122,58,0.4);
  z-index: 2;
}

.user-location-pulse {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 28px;
  height: 28px;
  background: rgba(45, 122, 58, 0.18);
  border-radius: 50%;
  animation: location-pulse 1.8s ease-out infinite;
  z-index: 1;
}

@keyframes location-pulse {
  0% { transform: translate(-50%, -50%) scale(0.5); opacity: 1; }
  100% { transform: translate(-50%, -50%) scale(1.8); opacity: 0; }
}
</style>
<script setup>
import { ref, onMounted } from 'vue'

const props = defineProps({
  suburbs: {
    type: Array,
    required: true,
  },
  skipAutoLocate: {
    type: Boolean,
    default: false,
  },
})

const emit = defineEmits(['suburb-found', 'out-of-range'])

const status = ref('idle')
const foundSuburbName = ref('')

function haversineKm(lat1, lon1, lat2, lon2) {
  const R = 6371
  const dLat = ((lat2 - lat1) * Math.PI) / 180
  const dLon = ((lon2 - lon1) * Math.PI) / 180
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos((lat1 * Math.PI) / 180) * Math.cos((lat2 * Math.PI) / 180) * Math.sin(dLon / 2) ** 2
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
}

function findNearestSuburb(lat, lon) {
  let nearest = null
  let minDist = Infinity
  for (const s of props.suburbs) {
    const d = haversineKm(lat, lon, s.latitude, s.longitude)
    if (d < minDist) {
      minDist = d
      nearest = s
    }
  }
  return minDist <= 4 ? nearest : null
}

function locate() {
  if (!navigator.geolocation) {
    status.value = 'error'
    return
  }
  status.value = 'locating'
  navigator.geolocation.getCurrentPosition(
    (pos) => {
      const { latitude, longitude } = pos.coords
      const suburb = findNearestSuburb(latitude, longitude)
      if (suburb) {
        status.value = 'found'
        foundSuburbName.value = suburb.suburb_name
        emit('suburb-found', suburb, { lat: latitude, lng: longitude })
      } else {
        status.value = 'out-of-range'
        emit('out-of-range')
      }
    },
    (err) => {
      status.value = err.code === err.PERMISSION_DENIED ? 'denied' : 'error'
    },
    { timeout: 10000, maximumAge: 60000 },
  )
}

onMounted(() => {
  if (props.suburbs.length > 0 && !props.skipAutoLocate) locate()
})
</script>

<template>
  <div class="lf-wrap">
    <!-- Idle -->
    <button v-if="status === 'idle'" class="lf-btn lf-btn--idle" @click="locate">
      <svg class="lf-icon" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
        <path d="M21 10c0 7-9 13-9 13S3 17 3 10a9 9 0 0 1 18 0z"/><circle cx="12" cy="10" r="3"/>
      </svg>
      <span>My location</span>
    </button>

    <!-- Locating -->
    <div v-else-if="status === 'locating'" class="lf-btn lf-btn--locating">
      <span class="lf-spinner"></span>
      <span>Locating…</span>
    </div>

    <!-- Found -->
    <div v-else-if="status === 'found'" class="lf-btn lf-btn--found">
      <span class="lf-dot"></span>
      <span>{{ foundSuburbName }}</span>
      <button class="lf-retry" @click="locate" aria-label="Refresh location">
        <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"/>
        </svg>
      </button>
    </div>

    <!-- Out of range -->
    <div v-else-if="status === 'out-of-range'" class="lf-btn lf-btn--warn">
      <svg class="lf-icon" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
        <circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/>
      </svg>
      <span>Not in inner Melbourne</span>
      <button class="lf-retry" @click="locate" aria-label="Try again">
        <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"/>
        </svg>
      </button>
    </div>

    <!-- Denied -->
    <div v-else-if="status === 'denied'" class="lf-btn lf-btn--denied">
      <svg class="lf-icon" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
        <circle cx="12" cy="12" r="10"/><line x1="4.93" y1="4.93" x2="19.07" y2="19.07"/>
      </svg>
      <span>Access denied</span>
    </div>

    <!-- Error -->
    <div v-else-if="status === 'error'" class="lf-btn lf-btn--denied">
      <svg class="lf-icon" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
        <circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/>
      </svg>
      <span>Unavailable</span>
      <button class="lf-retry" @click="locate" aria-label="Try again">
        <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"/>
        </svg>
      </button>
    </div>
  </div>
</template>

<style scoped>
.lf-wrap {
  flex-shrink: 0;
}

.lf-btn {
  display: inline-flex;
  align-items: center;
  gap: 0.45rem;
  padding: 0.6rem 1.1rem;
  border-radius: 999px;
  font-size: 0.83rem;
  font-weight: 600;
  white-space: nowrap;
  border: none;
  cursor: default;
  transition: all 0.15s;
  line-height: 1;
}

/* Idle — solid green CTA */
.lf-btn--idle {
  background: #2d7a3a;
  color: #ffffff;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(45,122,58,0.35);
  border: none;
}
.lf-btn--idle:hover {
  background: #245f2e;
  box-shadow: 0 4px 14px rgba(45,122,58,0.45);
  transform: translateY(-1px);
}
.lf-btn--idle:active {
  transform: translateY(0);
}

/* Locating */
.lf-btn--locating {
  background: #f0f7ee;
  color: #2d7a3a;
  border: 1.5px solid #b8ddc0;
}

/* Found */
.lf-btn--found {
  background: #e6f4e8;
  color: #1e5c28;
  border: 1.5px solid #a3d4ac;
  padding-right: 0.5rem;
}

/* Warn */
.lf-btn--warn {
  background: #fef9ec;
  color: #7a5c10;
  border: 1.5px solid #f0d080;
  padding-right: 0.5rem;
}

/* Denied/error */
.lf-btn--denied {
  background: #fef2f2;
  color: #991b1b;
  border: 1.5px solid #fca5a5;
}

.lf-icon {
  flex-shrink: 0;
}

/* Pulsing green dot for found state */
.lf-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #2d7a3a;
  flex-shrink: 0;
  animation: lf-pulse 2s ease-out infinite;
}

@keyframes lf-pulse {
  0%   { box-shadow: 0 0 0 0 rgba(45,122,58,0.5); }
  70%  { box-shadow: 0 0 0 5px rgba(45,122,58,0); }
  100% { box-shadow: 0 0 0 0 rgba(45,122,58,0); }
}

/* Inline retry icon button */
.lf-retry {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  border: none;
  background: rgba(0,0,0,0.08);
  color: inherit;
  cursor: pointer;
  margin-left: 0.15rem;
  transition: background 0.15s;
  flex-shrink: 0;
}
.lf-retry:hover {
  background: rgba(0,0,0,0.15);
}

/* Spinner */
.lf-spinner {
  width: 13px;
  height: 13px;
  border-radius: 50%;
  border: 2px solid #b8ddc0;
  border-top-color: #2d7a3a;
  animation: lf-spin 0.7s linear infinite;
  flex-shrink: 0;
}

@keyframes lf-spin {
  to { transform: rotate(360deg); }
}
</style>
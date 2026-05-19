<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { useRoute } from 'vue-router'
import NavBar from '../components/NavBar.vue'
import SuburbSearch from '../components/SuburbSearch.vue'
import SuburbDetail from '../components/SuburbDetail.vue'
import SuburbMap from '../components/SuburbMap.vue'
import HeatLevelGuide from '../components/HeatLevelGuide.vue'
import LocationFinder from '../components/LocationFinder.vue'
import Footer from '../components/Footer.vue'

const API_BASE = 'https://qcbqul6ys2.execute-api.ap-southeast-2.amazonaws.com'

const INNER_MELBOURNE = new Set([
  'Aberfeldie',
  'Airport West',
  'Albert Park (Vic.)',
  'Armadale (Vic.)',
  'Ascot Vale',
  'Avondale Heights',
  'Balaclava (Vic.)',
  'Brunswick (Vic.)',
  'Brunswick East',
  'Brunswick West',
  'Burnley',
  'Carlton (Vic.)',
  'Carlton North',
  'Clifton Hill',
  'Coburg',
  'Coburg North',
  'Collingwood (Vic.)',
  'Cremorne (Vic.)',
  'Docklands',
  'East Melbourne',
  'Elwood',
  'Essendon',
  'Essendon Fields',
  'Essendon North',
  'Essendon West',
  'Fairfield (Vic.)',
  'Fawkner',
  'Fitzroy (Vic.)',
  'Fitzroy North',
  'Flemington',
  'Glenroy (Vic.)',
  'Gowanbrae',
  'Hadfield',
  'Keilor East',
  'Kensington (Vic.)',
  'Kooyong',
  'Malvern (Vic.)',
  'Malvern East',
  'Melbourne',
  'Middle Park (Vic.)',
  'Moonee Ponds',
  'Niddrie',
  'North Melbourne',
  'Oak Park',
  'Parkville (Vic.)',
  'Pascoe Vale',
  'Pascoe Vale South',
  'Port Melbourne',
  'Prahran',
  'Princes Hill',
  'Richmond (Vic.)',
  'Ripponlea',
  'South Melbourne',
  'South Wharf',
  'South Yarra',
  'Southbank',
  'St Kilda (Vic.)',
  'St Kilda West',
  'Strathmore (Vic.)',
  'Strathmore Heights',
  'Toorak',
  'Travancore',
  'West Melbourne',
  'Windsor (Vic.)',
  'Test Suburb', // 🧪 TEST ONLY — remove before production
])

const route = useRoute()

const allSuburbs = ref([])
const loading = ref(true)
const error = ref(null)
const selectedSuburb = ref(null)

// AC3.3.1 — High-risk warning popup
const showHighRiskPopup = ref(false)
const highRiskSuburbName = ref('')

// Out-of-range location popup
const showOutOfRangePopup = ref(false)

const userLocation = ref(null)

const innerSuburbs = computed(() =>
  allSuburbs.value.filter((s) => INNER_MELBOURNE.has(s.suburb_name)),
)

async function fetchSuburbs() {
  try {
    loading.value = true
    error.value = null
    const res = await fetch(`${API_BASE}/suburbs`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    allSuburbs.value = await res.json()

    // 🧪 TEST ONLY — mock high-risk suburb (remove before production)
    allSuburbs.value.push({
      suburb_id: 9999,
      suburb_name: 'Test Suburb',
      latitude: -33.8688,
      longitude: 151.2093,
      temperature: 38,
      apparent_temperature: 42,
      uv_index: 11,
      heat_level: 'higher',
      heat_score: 88,
      shade_score: 75,
      tree_canopy_percent: 8,
      risk_level: 'high',
      updated_at: new Date().toISOString(),
    })
  } catch (e) {
    error.value = 'Failed to load suburb data. Please try again.'
    console.error(e)
  } finally {
    loading.value = false
  }
}

function selectSuburb(suburb) {
  if (!suburb) {
    selectedSuburb.value = null
    return
  }
  const isSame = selectedSuburb.value?.suburb_name === suburb.suburb_name
  selectedSuburb.value = isSame ? null : suburb

  // AC3.3.1 — trigger popup when high-risk suburb selected (not deselected)
  if (!isSame && suburb.risk_level === 'high') {
    highRiskSuburbName.value = suburb.suburb_name
    showHighRiskPopup.value = true
  }
}

function clearSelection() {
  selectedSuburb.value = null
}

// LocationFinder handlers
function onSuburbFound(suburb, coords) {
  userLocation.value = coords
  selectSuburb(suburb)
}

function onOutOfRange() {
  showOutOfRangePopup.value = true
}

onMounted(async () => {
  await fetchSuburbs()
  // Auto-select suburb passed via route query (e.g. from Outfit Advisor or Trip Coach)
  if (route.query.suburbId) {
    const match = innerSuburbs.value.find(
      (s) => String(s.suburb_id) === String(route.query.suburbId),
    )
    if (match) selectSuburb(match)
  }
})
</script>

<template>
  <div class="page">
    <NavBar />

    <main class="content">
      <!-- Page header card -->
      <div class="page-header card">
        <div class="page-header-text">
          <h1 class="page-title">Heat <span class="page-title-accent">Safety</span> Map</h1>
          <p class="page-desc">
            See how hot it feels right now across inner Melbourne suburbs.
            <strong>Click any suburb</strong> on the map or use the search bar to view details.
          </p>
        </div>
      </div>

      <div v-if="loading" class="status-msg">Loading suburb data...</div>
      <div v-else-if="error" class="status-msg error">{{ error }}</div>

      <template v-else>
        <!-- Search bar -->
        <div class="search-card card">
          <SuburbSearch :suburbs="innerSuburbs" @select="selectSuburb" />
        </div>

        <!-- Location finder — separate row below search -->
        <div class="location-row">
          <span class="location-label">Or use your current location:</span>
          <LocationFinder
            :suburbs="innerSuburbs"
            :skipAutoLocate="!!route.query.suburbId"
            @suburb-found="onSuburbFound"
            @out-of-range="onOutOfRange"
          />
        </div>

        <!-- Map + right panel side by side -->
        <div class="map-panel-row">
          <!-- Map card -->
          <div class="map-col card">
            <SuburbMap
              :suburbs="innerSuburbs"
              :selectedSuburb="selectedSuburb"
              :userLocation="userLocation"
              @select="selectSuburb"
            />
          </div>

          <!-- Right panel: legend by default, detail when suburb selected -->
          <div class="right-col">
            <Transition name="panel-switch" mode="out-in">
              <SuburbDetail
                v-if="selectedSuburb"
                :key="selectedSuburb.suburb_name"
                :suburb="selectedSuburb"
                @close="clearSelection"
              />
              <HeatLevelGuide v-else />
            </Transition>
          </div>
        </div>
      </template>
    </main>
    <Footer />

    <!-- AC3.3.1 — High-risk suburb warning popup -->
    <Teleport to="body">
      <Transition name="popup-fade">
        <div
          v-if="showHighRiskPopup"
          class="popup-backdrop"
          @click.self="showHighRiskPopup = false"
        >
          <div
            class="popup"
            role="alertdialog"
            aria-modal="true"
            aria-label="High heat risk warning"
          >
            <button
              class="popup-close"
              @click="showHighRiskPopup = false"
              aria-label="Close warning"
            >
              <svg
                width="13"
                height="13"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2.5"
                stroke-linecap="round"
              >
                <line x1="18" y1="6" x2="6" y2="18" />
                <line x1="6" y1="6" x2="18" y2="18" />
              </svg>
            </button>

            <div class="popup-icon">🚨</div>
            <h3 class="popup-title">High Heat Risk Area</h3>
            <p class="popup-body">
              <strong>{{ highRiskSuburbName }}</strong> is currently rated high heat risk. Take
              precautions before spending time outdoors.
            </p>
            <button class="popup-dismiss" @click="showHighRiskPopup = false">Got it</button>
          </div>
        </div>
      </Transition>
    </Teleport>

    <!-- Out-of-range location popup -->
    <Teleport to="body">
      <Transition name="popup-fade">
        <div
          v-if="showOutOfRangePopup"
          class="popup-backdrop"
          @click.self="showOutOfRangePopup = false"
        >
          <div
            class="popup popup--outrange"
            role="dialog"
            aria-modal="true"
            aria-label="Outside coverage area"
          >
            <button class="popup-close" @click="showOutOfRangePopup = false" aria-label="Close">
              <svg
                width="13"
                height="13"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2.5"
                stroke-linecap="round"
              >
                <line x1="18" y1="6" x2="6" y2="18" />
                <line x1="6" y1="6" x2="18" y2="18" />
              </svg>
            </button>

            <div class="popup-icon">📍</div>
            <h3 class="popup-title popup-title--outrange">Outside Coverage Area</h3>
            <p class="popup-body">
              Your location is outside the inner Melbourne suburbs covered by MelbCool. Browse the
              map manually or use the search bar to find a suburb.
            </p>
            <button
              class="popup-dismiss popup-dismiss--outrange"
              @click="showOutOfRangePopup = false"
            >
              Got it
            </button>
          </div>
        </div>
      </Transition>
    </Teleport>
  </div>
</template>

<style scoped>
.page {
  min-height: 100vh;
  background: linear-gradient(160deg, #eaf4f4 0%, #f0f7ee 50%, #e8f4f0 100%);
}

.content {
  max-width: var(--max-width);
  margin: 0 auto;
  padding: 1.25rem 1.5rem 2rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

/* Shared card style */
.card {
  background: #ffffff;
  border-radius: 14px;
  box-shadow:
    0 2px 8px rgba(0, 0, 0, 0.06),
    0 8px 24px rgba(0, 0, 0, 0.06);
  border: 1px solid rgba(0, 0, 0, 0.05);
}

/* Header card */
.page-header {
  padding: 1.75rem 1.6rem;
  background: linear-gradient(135deg, rgba(13, 58, 143, 0.95), rgba(11, 127, 121, 0.88));
  border: none;
}

.page-header-text {
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.page-title {
  font-size: 1.95rem;
  font-weight: 700;
  color: #ffffff;
  margin: 0;
  letter-spacing: -0.02em;
}

.page-title-accent {
  color: #a3f77d;
}

.page-desc {
  font-size: 0.95rem;
  color: rgba(255, 255, 255, 0.8);
  line-height: 1.6;
  margin: 0;
}

.page-desc strong {
  color: #ffffff;
  font-weight: 600;
}

/* Search + location */
.search-card {
  flex: 1;
  padding: 0.5rem 0.75rem;
}

.location-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.location-label {
  font-size: 0.85rem;
  color: #1c2e2a;
  white-space: nowrap;
}

/* Map card */
.map-col {
  min-width: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 0;
}

/* Map + right panel */
.map-panel-row {
  display: grid;
  grid-template-columns: 1fr 400px;
  gap: 1.25rem;
  align-items: stretch;
}

.map-col {
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.right-col {
  position: sticky;
  top: 64px;
  max-height: calc(100vh - 76px);
  overflow-y: auto;
  scrollbar-width: thin;
  scrollbar-color: #c8bfb0 transparent;
}

@media (max-width: 960px) {
  .map-panel-row {
    grid-template-columns: 1fr;
  }
  .right-col {
    position: static;
    max-height: none;
  }
}

/* Panel switch transition */
.panel-switch-enter-active,
.panel-switch-leave-active {
  transition:
    opacity 0.2s ease,
    transform 0.2s ease;
}
.panel-switch-enter-from {
  opacity: 0;
  transform: translateX(10px);
}
.panel-switch-leave-to {
  opacity: 0;
  transform: translateX(-10px);
}

.status-msg {
  text-align: center;
  padding: 3rem 0;
  color: #6b6560;
  font-size: 1rem;
}

.status-msg.error {
  color: #dc2626;
}

/* Popups — shared base */
.popup-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.65);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1.5rem;
  backdrop-filter: blur(2px);
}

.popup {
  position: relative;
  background: #ffffff;
  border: 1.5px solid #fca5a5;
  border-radius: 16px;
  padding: 1.75rem 1.5rem 1.5rem;
  max-width: 360px;
  width: 100%;
  text-align: center;
  box-shadow: 0 16px 48px rgba(0, 0, 0, 0.12);
}

.popup--outrange {
  border-color: #4d9e5a;
  box-shadow: 0 16px 48px rgba(0, 0, 0, 0.1);
}

.popup-close {
  position: absolute;
  top: 0.9rem;
  right: 0.9rem;
  width: 26px;
  height: 26px;
  border-radius: 50%;
  border: 1px solid #e3ded5;
  background: #f5f2ec;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  color: #6b6560;
  transition: background 0.15s;
}
.popup-close:hover {
  background: rgba(255, 255, 255, 0.12);
  color: #fff;
}

.popup-icon {
  font-size: 2.2rem;
  margin-bottom: 0.6rem;
}

.popup-title {
  font-size: 1.15rem;
  font-weight: 700;
  color: #c0392b;
  margin: 0 0 0.6rem;
}

.popup-title--outrange {
  color: #2d7a3a;
}

.popup-body {
  font-size: 0.92rem;
  color: #1a1714;
  line-height: 1.6;
  margin: 0 0 1.25rem;
}

.popup-dismiss {
  background: linear-gradient(135deg, #dc2626, #b91c1c);
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 0.6rem 1.75rem;
  font-size: 0.92rem;
  font-weight: 600;
  cursor: pointer;
  transition: filter 0.15s;
}
.popup-dismiss:hover {
  filter: brightness(1.15);
}

.popup-dismiss--outrange {
  background: #2d7a3a;
}

/* Popup transition */
.popup-fade-enter-active,
.popup-fade-leave-active {
  transition: opacity 0.2s ease;
}
.popup-fade-enter-active .popup,
.popup-fade-leave-active .popup {
  transition:
    transform 0.2s ease,
    opacity 0.2s ease;
}
.popup-fade-enter-from {
  opacity: 0;
}
.popup-fade-enter-from .popup {
  transform: scale(0.95);
  opacity: 0;
}
.popup-fade-leave-to {
  opacity: 0;
}
</style>
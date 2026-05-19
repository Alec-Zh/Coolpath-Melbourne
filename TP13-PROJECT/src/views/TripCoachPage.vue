<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import NavBar from '../components/NavBar.vue'
import Footer from '../components/Footer.vue'
import TripPlannerForm from '../components/TripPlannerForm.vue'
import TripForecastBar from '../components/TripForecastBar.vue'
import TripExposureScore from '../components/TripExposureScore.vue'
import TripLegBreakdown from '../components/TripLegBreakdown.vue'
import TripAlternatives from '../components/TripAlternatives.vue'
import TripMethodology from '../components/TripMethodology.vue'

const API_BASE = 'https://qcbqul6ys2.execute-api.ap-southeast-2.amazonaws.com'
const HOURLY_API = 'https://qcbqul6ys2.execute-api.ap-southeast-2.amazonaws.com'

function minutesToLabel(mins) {
  const h = Math.floor(mins / 60)
  const m = mins % 60
  const period = h >= 12 ? 'pm' : 'am'
  const displayH = h > 12 ? h - 12 : h === 0 ? 12 : h
  return m === 0
    ? `${displayH}:00 ${period}`
    : `${displayH}:${String(m).padStart(2, '0')} ${period}`
}

const TRANSPORT_NAMES = {
  walk: 'walking',
  tram: 'taking the tram',
  bus: 'taking the bus',
  drive: 'driving',
}
const TRANSPORT_LABELS = { walk: 'Walk', tram: 'Tram', bus: 'Bus', drive: 'Drive' }
const TRANSPORT_ICONS = { walk: '🚶', tram: '🚃', bus: '🚌', drive: '🚗' }

const allSuburbs = ref([])
const loading = ref(true)
const error = ref(null)
const showResults = ref(false)
const showDetail = ref(false)

const router = useRouter()
const route = useRoute()

function goToOutfitAdvisor() {
  if (selectedSuburbId.value) {
    router.push({ path: '/outfit-advisor', query: { suburbId: selectedSuburbId.value } })
  } else {
    router.push('/outfit-advisor')
  }
}

onMounted(async () => {
  try {
    const res = await fetch(`${API_BASE}/suburbs`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    allSuburbs.value = (await res.json()).sort((a, b) => a.suburb_name.localeCompare(b.suburb_name))
    // Auto-select suburb passed via route query (e.g. from Outfit Advisor or Heat Map)
    if (route.query.suburbId) {
      const match = allSuburbs.value.find(
        (s) => String(s.suburb_id) === String(route.query.suburbId),
      )
      if (match) selectedSuburbId.value = String(match.suburb_id)
    }
  } catch (e) {
    error.value = 'Could not load suburb data. Please try again.'
    console.error(e)
  } finally {
    loading.value = false
  }
})

const selectedSuburbId = ref('')
const transportMode = ref('walk')
const durationMins = ref(30)
const departureMinutes = ref(540)

const selectedSuburb = computed(
  () => allSuburbs.value.find((s) => String(s.suburb_id) === String(selectedSuburbId.value)) ?? null,
)
const departureLabel = computed(() => minutesToLabel(departureMinutes.value))

const hourlyTemps = ref(null)

async function fetchHourly(suburbId) {
  try {
    const res = await fetch(`${HOURLY_API}/hourly/${suburbId}`)
    if (!res.ok) return
    const data = await res.json()
    hourlyTemps.value = data.hourly_apparent_temperature
  } catch (e) {
    console.warn('Hourly forecast unavailable, falling back to current data', e)
    hourlyTemps.value = null
  }
}

function getTempAtMinutes(mins) {
  const hour = Math.floor(mins / 60)
  if (hourlyTemps.value?.[hour] != null) return Math.round(hourlyTemps.value[hour] * 10) / 10
  return (
    Math.round(
      (selectedSuburb.value?.apparent_temperature ?? selectedSuburb.value?.temperature ?? 22) * 10,
    ) / 10
  )
}

const forecastTemp = computed(() => {
  if (!selectedSuburb.value) return null
  return getTempAtMinutes(departureMinutes.value)
})

// ── Exposure-based leg model ────────────────────────────────────────────────
// Outdoor exposure ratios based on:
//   Walk: 100% outdoor (full trip)
//   Tram/Bus: 25% outdoor — based on PTV average walk+wait share of total trip time
//     (Currie & Delbosc, 2011, "Modelling the social exclusion pedestrian catchment")
//   Drive: 10% outdoor — AustRoads parking walk estimate ~1–2 min each end
//
// Outdoor segment uses full apparent_temperature + shade_score heat model.
// Sheltered segment applies temperature reduction (vehicle air conditioning assumed).

const EXPOSURE = {
  walk: { outdoorRatio: 1.0, tempMod: 0, shadeMod: 0 },
  tram: { outdoorRatio: 0.25, tempMod: -5, shadeMod: -30 },
  bus: { outdoorRatio: 0.25, tempMod: -5, shadeMod: -30 },
  drive: { outdoorRatio: 0.1, tempMod: -8, shadeMod: -40 },
}

function legScore(temp, shadeScore) {
  let base
  if (temp >= 35) base = Math.min(Math.round((temp - 35) * 3 + 70), 100)
  else if (temp >= 28) base = Math.round((temp - 28) * 4.3 + 40)
  else base = Math.max(Math.round(temp * 1.4), 0)
  return Math.min(base + Math.round(shadeScore * 0.2), 100)
}

function riskBand(score) {
  if (score >= 75) return { label: 'Very High', color: '#e8735a', bg: '#fdecea' }
  if (score >= 50) return { label: 'High', color: '#e07b39', bg: '#fef3ec' }
  if (score >= 25) return { label: 'Moderate', color: '#8b8000', bg: '#fefbe6' }
  return { label: 'Low', color: '#4a7c59', bg: '#eef6f1' }
}

function weightedScore(legs) {
  const totalMins = legs.reduce((s, l) => s + l.minutes, 0)
  return Math.round(legs.reduce((s, l) => s + l.score * l.minutes, 0) / totalMins)
}

function buildLegs(mode, totalMins, temp, shade) {
  const exp = EXPOSURE[mode] ?? EXPOSURE.walk
  const outdoorMins = Math.max(Math.round(totalMins * exp.outdoorRatio), 1)
  const shelteredMins = totalMins - outdoorMins

  const outdoorScore = legScore(temp, shade)
  const legs = [
    {
      label: 'Time outdoors',
      minutes: outdoorMins,
      score: outdoorScore,
      color: '#e07b39',
      riskLabel: riskBand(outdoorScore).label,
      isOutdoor: true,
    },
  ]

  if (shelteredMins > 0) {
    const shelteredScore = legScore(
      Math.max(temp + exp.tempMod, 10),
      Math.max(shade + exp.shadeMod, 0),
    )
    legs.push({
      label: mode === 'drive' ? 'Time in car' : 'Time on vehicle',
      minutes: shelteredMins,
      score: shelteredScore,
      color: '#5b8dd9',
      riskLabel: riskBand(shelteredScore).label,
      isOutdoor: false,
    })
  }

  return legs
}

const tripLegs = computed(() => {
  if (!selectedSuburb.value || forecastTemp.value === null) return null
  return buildLegs(
    transportMode.value,
    durationMins.value,
    forecastTemp.value,
    selectedSuburb.value.shade_score ?? 50,
  )
})

const overallScore = computed(() => (tripLegs.value ? weightedScore(tripLegs.value) : null))
const overallBand = computed(() =>
  overallScore.value !== null ? riskBand(overallScore.value) : null,
)

function pctDiff(base, alt) {
  return Math.round(((alt - base) / base) * 100)
}

function buildAlt(mode, duration, departMins, title, explanation, action) {
  if (!selectedSuburb.value) return null
  const temp = getTempAtMinutes(departMins)
  const shade = selectedSuburb.value.shade_score ?? 50
  const legs = buildLegs(mode, duration, temp, shade)
  const score = weightedScore(legs)
  const band = riskBand(score)
  const pct = pctDiff(overallScore.value, score)
  return {
    title,
    explanation,
    action,
    score,
    legs,
    pctChange: pct,
    ...band,
    params: { mode, duration, departMins },
  }
}

const alternatives = computed(() => {
  if (!overallScore.value || !selectedSuburb.value) return []
  const base = overallScore.value
  const curMode = transportMode.value
  const curDur = durationMins.value
  const curDepart = departureMinutes.value
  const suburb = selectedSuburb.value.suburb_name
  const alts = []

  // Earlier departure
  const earlierDepart = Math.max(curDepart - 120, 540)
  if (earlierDepart !== curDepart) {
    const timeStr = minutesToLabel(earlierDepart)
    const altTemp = getTempAtMinutes(earlierDepart)
    const alt = buildAlt(
      curMode,
      curDur,
      earlierDepart,
      `Leave at ${timeStr}`,
      `At ${timeStr}, ${suburb} feels like ${altTemp}°C — cooler than your current departure time. Leaving earlier can make your trip more comfortable.`,
      `Leave at ${timeStr}`,
    )
    if (alt) alts.push(alt)
  }

  // After 5pm
  if (curDepart < 1020) {
    const altTemp = getTempAtMinutes(1020)
    const alt = buildAlt(
      curMode,
      curDur,
      1020,
      'Leave after 5:00 pm',
      `By 5:00 pm, ${suburb} feels like ${altTemp}°C. The sun is lower and UV levels drop — making ${TRANSPORT_NAMES[curMode]} noticeably safer.`,
      'Leave at 5:00 pm',
    )
    if (alt) alts.push(alt)
  }

  // Different transport mode (drive first as it minimises outdoor exposure)
  const modePriority = ['drive', 'tram', 'bus', 'walk'].filter((m) => m !== curMode)
  for (const mode of modePriority) {
    const alt = buildAlt(
      mode,
      curDur,
      curDepart,
      `Switch to ${TRANSPORT_LABELS[mode]}`,
      `${TRANSPORT_LABELS[mode]} means less time outdoors in ${forecastTemp.value}°C heat. You'll spend more of your trip in a cooler, sheltered space.`,
      `Switch to ${TRANSPORT_LABELS[mode]}`,
    )
    if (alt && alt.score < base) {
      alts.push(alt)
      break
    }
  }

  return alts
    .filter(Boolean)
    .sort((a, b) => a.score - b.score)
    .slice(0, 3)
})

async function handleSubmit() {
  if (!selectedSuburb.value) return
  await fetchHourly(selectedSuburb.value.suburb_id)
  showResults.value = true
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function handleReset() {
  showResults.value = false
  showDetail.value = false
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

function handleSelectAlt(params) {
  transportMode.value = params.mode
  durationMins.value = params.duration
  departureMinutes.value = params.departMins
  showResults.value = false
  showDetail.value = false
  window.scrollTo({ top: 0, behavior: 'smooth' })
}

const verdictMessage = computed(() => {
  if (!overallBand.value) return ''
  const label = overallBand.value.label
  const mode = transportMode.value
  const temp = forecastTemp.value
  if (label === 'Low')
    return `This trip looks fine. ${temp}°C is manageable and your exposure is low.`
  if (label === 'Moderate')
    return `Take water and stay in shade where you can. Heat exposure is moderate.`
  if (label === 'High') {
    if (mode === 'walk')
      return `Walking in this heat carries real risk. Consider driving or going later.`
    return `Heat exposure is high. Try leaving earlier or switching transport.`
  }
  return `This trip is very risky in current conditions. Strongly consider alternatives below.`
})
</script>

<template>
  <div class="page">
    <NavBar />

    <main class="content">
      <!-- Page header -->
      <div class="page-header card">
        <div class="page-header-text">
          <h1 class="page-title">Trip <span class="page-title-accent">Coach</span></h1>
          <p class="page-desc">
            Tell us about your trip and we'll show you
            <strong>how much heat you'll be exposed to</strong> — so you can decide the safest time
            and way to travel.
          </p>
        </div>
      </div>

      <div v-if="loading" class="status-msg">Loading suburb data…</div>
      <div v-else-if="error" class="status-msg error">{{ error }}</div>

      <template v-else>
        <!-- Page 1 — two column: form left, guide right -->
        <div v-if="!showResults" class="plan-wrap">
          <TripPlannerForm
            :suburbs="allSuburbs"
            v-model:selectedSuburbId="selectedSuburbId"
            v-model:transportMode="transportMode"
            v-model:durationMins="durationMins"
            v-model:departureMinutes="departureMinutes"
            :loading="loading"
            @submit="handleSubmit"
          />
          <div class="plan-right">
            <div class="guide-card card">
              <p class="guide-title">How it works</p>
              <ul class="guide-list">
                <li>
                  Choose your <strong>destination</strong> and
                  <strong>how you're travelling</strong>
                </li>
                <li>
                  Tell us <strong>how long</strong> the trip takes and <strong>when</strong> you're
                  leaving
                </li>
                <li>We'll show how much time you'll spend <strong>outdoors in the heat</strong></li>
                <li>We'll suggest <strong>safer options</strong> if your trip looks risky</li>
              </ul>
            </div>

            <div class="guide-card card">
              <p class="guide-title">Time spent outdoors by transport</p>
              <div class="transport-compare">
                <div class="tc-row">
                  <span class="tc-mode">🚶 Walk</span>
                  <div class="tc-bar-wrap">
                    <div class="tc-bar" style="width: 100%; background: #c0392b"></div>
                  </div>
                  <span class="tc-pct">100%</span>
                </div>
                <div class="tc-row">
                  <span class="tc-mode">🚃 Tram</span>
                  <div class="tc-bar-wrap">
                    <div class="tc-bar" style="width: 25%; background: #e8903a"></div>
                  </div>
                  <span class="tc-pct">~25%</span>
                </div>
                <div class="tc-row">
                  <span class="tc-mode">🚌 Bus</span>
                  <div class="tc-bar-wrap">
                    <div class="tc-bar" style="width: 25%; background: #e8903a"></div>
                  </div>
                  <span class="tc-pct">~25%</span>
                </div>
                <div class="tc-row">
                  <span class="tc-mode">🚗 Drive</span>
                  <div class="tc-bar-wrap">
                    <div class="tc-bar" style="width: 10%; background: #4d9e5a"></div>
                  </div>
                  <span class="tc-pct">~10%</span>
                </div>
              </div>
              <p class="tc-note">Less time outdoors = lower heat exposure</p>
            </div>
          </div>
        </div>

        <!-- Page 2 — results -->
        <div v-else class="results-wrap">
          <!-- Go back button -->
          <button class="go-back-btn" @click="handleReset">← Plan a different trip</button>

          <!-- ① Verdict card — first thing user sees -->
          <div
            class="verdict-card card"
            :style="{ borderTop: `5px solid ${overallBand.color}`, background: overallBand.bg }"
          >
            <div class="verdict-left">
              <div class="verdict-chips">
                <span class="verdict-chip chip-dest"
                  >{{ TRANSPORT_ICONS[transportMode] }} {{ selectedSuburb.suburb_name }}</span
                >
                <span class="verdict-chip chip-time"
                  >{{ departureLabel }} · {{ durationMins }} min</span
                >
                <span class="verdict-chip chip-temp">Feels like {{ forecastTemp }}°C</span>
              </div>
              <p class="verdict-advice">{{ verdictMessage }}</p>
            </div>
            <div class="verdict-right">
              <p class="verdict-score-label">Risk score</p>
              <p class="verdict-score" :style="{ color: overallBand.color }">
                {{ overallScore }}<span class="verdict-outof">/100</span>
              </p>
              <span class="verdict-badge" :style="{ background: overallBand.color }"
                >{{ overallBand.label }} Risk</span
              >
            </div>
          </div>

          <!-- ② Alternatives — compact pills, expand on click -->
          <TripAlternatives
            v-if="alternatives.length"
            :alternatives="alternatives"
            :baseScore="overallScore"
            @select-alt="handleSelectAlt"
          />

          <!-- ③ Detail toggle -->
          <button class="detail-toggle" @click="showDetail = !showDetail">
            {{ showDetail ? '▲ Hide breakdown' : '▼ See how this was calculated' }}
          </button>

          <template v-if="showDetail">
            <TripForecastBar
              :suburbName="selectedSuburb.suburb_name"
              :forecastTemp="forecastTemp"
              :treeCanopy="selectedSuburb.tree_canopy_percent"
              :departureLabel="departureLabel"
              :transportMode="transportMode"
              :durationMins="durationMins"
            />
            <TripLegBreakdown :legs="tripLegs" :transportMode="transportMode" />
            <TripMethodology />
          </template>

          <!-- Nav links at bottom -->
          <div class="trip-nav-links">
            <RouterLink :to="{ path: '/heatmap', query: { suburbId: selectedSuburbId } }" class="trip-nav-btn trip-nav-btn--green">
              <svg
                width="13"
                height="13"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2.5"
                stroke-linecap="round"
                stroke-linejoin="round"
                aria-hidden="true"
              >
                <circle cx="12" cy="12" r="10" />
                <line x1="2" y1="12" x2="22" y2="12" />
                <path
                  d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10z"
                />
              </svg>
              Heat map
            </RouterLink>
            <button class="trip-nav-btn trip-nav-btn--teal" @click="goToOutfitAdvisor">
              <svg
                width="13"
                height="13"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2.5"
                stroke-linecap="round"
                stroke-linejoin="round"
                aria-hidden="true"
              >
                <path
                  d="M20.38 3.46L16 2a4 4 0 0 1-8 0L3.62 3.46a2 2 0 0 0-1.34 2.23l.58 3.57a1 1 0 0 0 .99.84H6v10c0 1.1.9 2 2 2h8a2 2 0 0 0 2-2V10h2.15a1 1 0 0 0 .99-.84l.58-3.57a2 2 0 0 0-1.34-2.23z"
                />
              </svg>
              Outfit advisor
            </button>
          </div>
        </div>
      </template>
    </main>

    <Footer />
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
.page-title {
  font-size: 1.95rem;
  font-weight: 700;
  color: #ffffff;
  margin: 0 0 0.35rem;
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
  max-width: 580px;
}
.page-desc strong {
  color: #ffffff;
  font-weight: 600;
}

/* Guide card */
.guide-card {
  padding: 1.1rem 1.25rem;
}
.guide-title {
  font-size: 0.85rem;
  font-weight: 700;
  color: #2d7a3a;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  margin: 0 0 0.75rem;
}
.guide-list {
  list-style: none;
  padding-left: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 0.65rem;
}
.guide-list li {
  font-size: 1rem;
  color: #3a4e4a;
  line-height: 1.55;
  padding-left: 1.1rem;
  position: relative;
}
.guide-list li::before {
  content: '✓';
  position: absolute;
  left: 0;
  color: #2d7a3a;
  font-weight: 700;
}
.guide-list li strong {
  color: #1c2e2a;
  font-weight: 700;
}

/* Page 1 — two column */
.plan-wrap {
  display: grid;
  grid-template-columns: 1fr 300px;
  gap: 1.25rem;
  align-items: stretch;
}

.plan-right {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.plan-right .guide-card:last-child {
  flex: 1;
}

/* Transport comparison */
.transport-compare {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  margin-top: 0.25rem;
}
.tc-row {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}
.tc-mode {
  font-size: 0.95rem;
  font-weight: 600;
  color: #1c2e2a;
  width: 66px;
  flex-shrink: 0;
}
.tc-bar-wrap {
  flex: 1;
  height: 12px;
  background: #f0f5f4;
  border-radius: 6px;
  overflow: hidden;
}
.tc-bar {
  height: 100%;
  border-radius: 6px;
  transition: width 0.4s ease;
}
.tc-pct {
  font-size: 0.9rem;
  font-weight: 700;
  color: #5a6e6a;
  width: 38px;
  text-align: right;
  flex-shrink: 0;
}
.tc-note {
  font-size: 0.9rem;
  color: #5a6e6a;
  margin: 0.6rem 0 0;
  padding-top: 0.6rem;
  border-top: 1px solid #e8f0ee;
  font-style: italic;
}
/* Verdict card */
.verdict-card {
  padding: 1.5rem;
  border-radius: 14px;
  box-shadow:
    0 2px 8px rgba(0, 0, 0, 0.06),
    0 8px 24px rgba(0, 0, 0, 0.06);
  border: 1px solid rgba(0, 0, 0, 0.05);
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1.5rem;
  flex-wrap: wrap;
}
.verdict-left {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.65rem;
  min-width: 200px;
}
.verdict-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem;
}
.verdict-chip {
  font-size: 0.8rem;
  font-weight: 600;
  padding: 0.25rem 0.7rem;
  border-radius: 20px;
}
.chip-dest {
  background: #e8f0ee;
  color: #1c2e2a;
}
.chip-time {
  background: #eef0f8;
  color: #2a2e50;
}
.chip-temp {
  background: #fef3ec;
  color: #8b4000;
}

.verdict-advice {
  font-size: 1.05rem;
  font-weight: 600;
  color: #1c2e2a;
  line-height: 1.5;
  margin: 0;
}
.verdict-right {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.4rem;
  min-width: 100px;
}
.verdict-score-label {
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #7a9490;
  margin: 0;
}
.verdict-score {
  font-size: 3.8rem;
  font-weight: 800;
  line-height: 1;
  margin: 0;
}
.verdict-outof {
  font-size: 1.1rem;
  font-weight: 500;
  color: #aaa;
}
.verdict-badge {
  color: #fff;
  font-size: 0.85rem;
  font-weight: 700;
  padding: 0.35rem 0.9rem;
  border-radius: 20px;
  white-space: nowrap;
}

/* Results — single column, centred */
.results-wrap {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin: 0 auto;
  width: 100%;
}

/* Go back button */
.go-back-btn {
  align-self: flex-start;
  background: #fff;
  border: 2px solid #c8ddd9;
  border-radius: 10px;
  padding: 0.65rem 1.2rem;
  font-size: 1rem;
  font-weight: 700;
  color: #1c2e2a;
  cursor: pointer;
  font-family: inherit;
  transition:
    border-color 0.2s,
    background 0.2s;
}
.go-back-btn:hover {
  border-color: #2d7a3a;
  background: #f5fdf6;
  color: #2d7a3a;
}

/* Detail toggle */
.detail-toggle {
  background: none;
  border: 1.5px solid #c8ddd9;
  border-radius: 8px;
  padding: 0.55rem 1.1rem;
  font-size: 0.85rem;
  font-weight: 600;
  color: #5a6e6a;
  cursor: pointer;
  align-self: flex-start;
  font-family: inherit;
  transition:
    border-color 0.2s,
    color 0.2s;
}
.detail-toggle:hover {
  border-color: #2d7a3a;
  color: #2d7a3a;
}

/* Nav links */
.trip-nav-links {
  display: flex;
  gap: 8px;
  margin-top: 0.25rem;
}

.summary-card {
  display: none; /* no longer used */
}

@media (max-width: 600px) {
  .verdict-card {
    flex-direction: column-reverse;
    align-items: stretch;
  }
  .verdict-right {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
  }
  .verdict-score {
    font-size: 2.8rem;
  }
}

.trip-nav-btn {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 0.55rem 0.5rem;
  border-radius: 8px;
  font-size: 0.82rem;
  font-weight: 700;
  text-decoration: none;
  border: none;
  cursor: pointer;
  font-family: inherit;
  transition:
    filter 0.15s,
    transform 0.1s;
  letter-spacing: 0.01em;
}

.trip-nav-btn:active {
  transform: scale(0.98);
}

.trip-nav-btn--green {
  background: #2d7a3a;
  color: #ffffff;
}

.trip-nav-btn--green:hover {
  filter: brightness(1.12);
}

.trip-nav-btn--teal {
  background: #0f6e56;
  color: #ffffff;
}

.trip-nav-btn--teal:hover {
  filter: brightness(1.15);
}

.status-msg {
  text-align: center;
  padding: 3rem 0;
  color: #7a9490;
  font-size: 1rem;
}
.status-msg.error {
  color: #c0392b;
}

@media (max-width: 860px) {
  .plan-wrap {
    grid-template-columns: 1fr;
  }
  .plan-right {
    position: static;
  }
}
</style>
<script setup>
import NavBar from '@/components/NavBar.vue'
import Footer from '@/components/Footer.vue'
import LoadingCard from '@/components/LoadingCard.vue'
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

const DEMO_WEATHER_THRESHOLDS = {
  moderateFeels: 30,
  highFeels: 36,
  highUv: 8,
  sunProtectionUv: 3,
  highHumidity: 65,
  lowCanopy: 15,
}

const verifiedSources = [
  {
    id: 'vic-heat-illness',
    title: 'Better Health Channel - Heat-related health problems',
    label: 'Symptoms',
    url: 'https://www.betterhealth.vic.gov.au/health/healthyliving/heat-related-health-problems',
  },
  {
    id: 'vic-extreme-heat',
    title: 'Better Health Channel - Extreme heat',
    label: 'Stay cool',
    url: 'https://www.betterhealth.vic.gov.au/health/healthyliving/how-to-cope-and-stay-safe-in-extreme-heat',
  },
  {
    id: 'vic-warning',
    title: 'Victorian Department of Health - Heat health warnings',
    label: 'Vic warnings',
    url: 'https://www.health.vic.gov.au/environmental-health/heat-health-warning',
  },
  {
    id: 'uv',
    title: 'Cancer Council Australia - UV protection',
    label: 'UV advice',
    url: 'https://www.cancer.org.au/cancer-information/causes-and-prevention/sun-safety/uv-index',
  },
  {
    id: 'open-meteo',
    title: 'Open-Meteo Forecast API',
    label: 'Weather API',
    url: 'https://open-meteo.com/en/docs',
  },
]

const safetySuburbsRaw = [
  {
    id: 'melbourne',
    name: 'Melbourne',
    temp: 34,
    feels: 39,
    uv: 9,
    humidity: 34,
    canopy: 12,
    lat: -37.8136,
    lng: 144.9631,
    hourly: [
      28, 28, 27, 27, 28, 29, 31, 33, 35, 37, 39, 40, 41, 41, 40, 39, 37, 35, 33, 31, 30, 29, 29,
      28,
    ],
  },
  {
    id: 'carlton',
    name: 'Carlton',
    temp: 33,
    feels: 38,
    uv: 8,
    humidity: 39,
    canopy: 16,
    lat: -37.8001,
    lng: 144.9671,
    hourly: [
      27, 27, 27, 27, 28, 30, 31, 33, 35, 37, 38, 39, 40, 40, 39, 38, 36, 34, 32, 30, 29, 29, 28,
      28,
    ],
  },
  {
    id: 'richmond',
    name: 'Richmond',
    temp: 32,
    feels: 36,
    uv: 8,
    humidity: 42,
    canopy: 22,
    lat: -37.8198,
    lng: 145.0017,
    hourly: [
      26, 26, 26, 26, 27, 28, 30, 32, 34, 36, 37, 38, 38, 38, 37, 36, 34, 33, 31, 29, 28, 28, 27,
      27,
    ],
  },
  {
    id: 'st-kilda',
    name: 'St Kilda',
    temp: 30,
    feels: 34,
    uv: 7,
    humidity: 58,
    canopy: 25,
    lat: -37.8679,
    lng: 144.978,
    hourly: [
      25, 25, 25, 25, 26, 27, 28, 30, 31, 33, 34, 35, 35, 35, 34, 33, 32, 31, 29, 28, 27, 27, 26,
      26,
    ],
  },
  {
    id: 'brunswick',
    name: 'Brunswick',
    temp: 31,
    feels: 35,
    uv: 7,
    humidity: 48,
    canopy: 29,
    lat: -37.7667,
    lng: 144.9614,
    hourly: [
      25, 25, 25, 25, 26, 27, 29, 31, 33, 35, 36, 37, 37, 36, 35, 34, 32, 30, 29, 28, 27, 27, 26,
      26,
    ],
  },
  {
    id: 'docklands',
    name: 'Docklands',
    temp: 35,
    feels: 40,
    uv: 9,
    humidity: 36,
    canopy: 8,
    lat: -37.8152,
    lng: 144.9478,
    hourly: [
      28, 28, 28, 28, 29, 30, 32, 34, 36, 38, 40, 41, 42, 42, 41, 40, 38, 36, 34, 32, 30, 30, 29,
      29,
    ],
  },
]

const safetySuburbs = ref(safetySuburbsRaw.map((s) => ({ ...s, risk: deriveWeatherRisk(s).risk })))

const safetyRefuges = [
  {
    name: 'State Library Victoria',
    type: 'Library',
    address: '328 Swanston Street, Melbourne',
    hours: '10:00 am - 6:00 pm',
    lat: -37.8098,
    lng: 144.9652,
  },
  {
    name: 'Kathleen Syme Library',
    type: 'Library',
    address: '251 Faraday Street, Carlton',
    hours: '10:00 am - 6:00 pm',
    lat: -37.7981,
    lng: 144.9679,
  },
  {
    name: 'Richmond Library',
    type: 'Library',
    address: '415 Church Street, Richmond',
    hours: '10:00 am - 5:00 pm',
    lat: -37.8216,
    lng: 144.9997,
  },
  {
    name: 'St Kilda Library',
    type: 'Library',
    address: '150 Carlisle Street, St Kilda',
    hours: '10:00 am - 6:00 pm',
    lat: -37.867,
    lng: 144.9866,
  },
  {
    name: 'Brunswick Library',
    type: 'Library',
    address: '233 Sydney Road, Brunswick',
    hours: '10:00 am - 6:00 pm',
    lat: -37.771,
    lng: 144.9612,
  },
  {
    name: 'Library at The Dock',
    type: 'Library',
    address: '107 Victoria Harbour Promenade, Docklands',
    hours: '10:00 am - 6:00 pm',
    lat: -37.8197,
    lng: 144.9439,
  },
  {
    name: 'NGV International',
    type: 'Museum',
    address: '180 St Kilda Road, Southbank',
    hours: '10:00 am - 5:00 pm',
    lat: -37.8226,
    lng: 144.9689,
  },
]

const riskCopy = {
  Low: 'Keep cool and check again later.',
  Moderate: 'Use these steps before going out.',
  High: 'Stay in unless the trip is essential.',
  Urgent: 'Cool down now and get help.',
}

const labels = {
  suburb: Object.fromEntries(safetySuburbsRaw.map((s) => [s.id, s.name])),
  ageGroup: { under65: 'Under 65', '65-74': '65-74', '75-84': '75-84', '85plus': '85+' },
  support: {
    'with-someone': 'Someone nearby',
    'check-in': 'Check-in available',
    alone: 'Alone today',
  },
  coolingAccess: { aircon: 'Air conditioning', fan: 'Fan only', none: 'No reliable cooling' },
  homeHeat: { cool: 'Usually cool', warm: 'Gets warm', hot: 'Hard to cool' },
  outdoorActivity: {
    none: 'No trip',
    'essential-short': 'Short essential trip',
    'essential-long': 'Long appointment',
    'exercise-work': 'Outdoor activity',
  },
  plannedTime: {
    flexible: 'Flexible',
    morning: 'Before 10 am',
    midday: '10 am - 4 pm',
    evening: 'After 5 pm',
  },
  timeOutside: {
    none: 'Not outside',
    under30: 'Under 30 min',
    '30to60': '30-60 min',
    over60: 'Over 1 hour',
  },
  travelShade: {
    'private-cool': 'Door-to-door',
    'some-shade': 'Some shade',
    'public-walk': 'Transit walk',
    'no-shade': 'Little shade',
  },
  healthRisk: {
    none: 'No concern',
    chronic: 'Health condition',
    medication: 'Heat-sensitive medicine',
    both: 'Condition + medicine',
  },
  symptoms: { none: 'No symptoms', early: 'Early symptoms', emergency: 'Emergency signs' },
  fluidMobility: {
    ok: 'Independent',
    'fluid-limit': 'Fluid limit',
    limited: 'Limited mobility',
    'needs-help': 'Needs help',
  },
}

const questions = [
  {
    id: 'suburb',
    category: 'Location',
    visual: 'map',
    title: 'Where are you today?',
    hint: 'Pick the suburb for local heat and cool places.',
    options: () =>
      safetySuburbs.value.map((s) => ({
        value: s.id,
        label: s.name,
        detail: `${s.feels}\u00b0C feels-like`,
        tag: 'MAP',
        visual: s.id === 'docklands' ? 'building' : s.id === 'st-kilda' ? 'water' : 'map',
      })),
  },
  {
    id: 'ageGroup',
    category: 'Profile',
    visual: 'person',
    title: 'How old is the person?',
    hint: 'Choose the closest age group.',
    options: [
      { value: 'under65', label: 'Under 65', detail: 'Basic care', tag: 'AGE', visual: 'person' },
      { value: '65-74', label: '65-74', detail: 'Add check-ins', tag: '65+', visual: 'senior' },
      { value: '75-84', label: '75-84', detail: 'Take extra care', tag: '75+', visual: 'senior' },
      { value: '85plus', label: '85+', detail: 'Plan help early', tag: '85+', visual: 'support' },
    ],
  },
  {
    id: 'outdoorActivity',
    category: 'Trip',
    visual: 'walk',
    title: 'Do you need to go out?',
    hint: 'Choose the closest plan.',
    options: [
      { value: 'none', label: 'No trip', detail: 'Stay inside', tag: 'IN', visual: 'home' },
      {
        value: 'essential-short',
        label: 'Short errand',
        detail: 'Brief trip',
        tag: 'TRIP',
        visual: 'walk',
      },
      {
        value: 'essential-long',
        label: 'Long appointment',
        detail: 'More time out',
        tag: 'LONG',
        visual: 'bus',
      },
      {
        value: 'exercise-work',
        label: 'Outdoor work',
        detail: 'Exercise or garden',
        tag: 'WORK',
        visual: 'sun',
      },
    ],
  },
  {
    id: 'plannedTime',
    category: 'Timing',
    visual: 'clock',
    title: 'When would you go?',
    hint: 'Morning and evening are often safer.',
    options: [
      {
        value: 'flexible',
        label: 'Any time',
        detail: 'Pick coolest',
        tag: 'FLEX',
        visual: 'clock',
      },
      {
        value: 'morning',
        label: 'Before 10 am',
        detail: 'Usually cooler',
        tag: 'AM',
        visual: 'morning',
      },
      {
        value: 'midday',
        label: '10 am - 4 pm',
        detail: 'Usually hottest',
        tag: 'MID',
        visual: 'sun',
      },
      {
        value: 'evening',
        label: 'After 5 pm',
        detail: 'Later option',
        tag: 'PM',
        visual: 'evening',
      },
    ],
  },
  {
    id: 'timeOutside',
    category: 'Exposure',
    visual: 'timer',
    title: 'How long outside?',
    hint: 'Longer time means more heat stress.',
    options: [
      { value: 'none', label: 'Not outside', detail: 'No outing', tag: '0', visual: 'home' },
      {
        value: 'under30',
        label: 'Under 30 min',
        detail: 'Short time',
        tag: '<30',
        visual: 'timer',
      },
      { value: '30to60', label: '30-60 min', detail: 'Medium time', tag: '60', visual: 'clock' },
      {
        value: 'over60',
        label: 'Over 1 hour',
        detail: 'Needs breaks',
        tag: '1H+',
        visual: 'break',
      },
    ],
  },
  {
    id: 'fluidMobility',
    category: 'Mobility',
    visual: 'water',
    title: 'Can you drink and move?',
    hint: 'This helps set water and support advice.',
    options: [
      { value: 'ok', label: 'Yes', detail: 'Can drink and move', tag: 'OK', visual: 'water' },
      {
        value: 'fluid-limit',
        label: 'Fluid limit',
        detail: 'Follow medical advice',
        tag: 'FLUID',
        visual: 'medicine',
      },
      {
        value: 'limited',
        label: 'Moves slowly',
        detail: 'Needs extra time',
        tag: 'MOVE',
        visual: 'mobility',
      },
      {
        value: 'needs-help',
        label: 'Needs help',
        detail: 'Do not go alone',
        tag: 'HELP',
        visual: 'support',
      },
    ],
  },
]

const answers = ref({
  suburb: 'melbourne',
  ageGroup: 'under65',
  outdoorActivity: 'none',
  plannedTime: 'flexible',
  timeOutside: 'none',
  fluidMobility: 'ok',
})

const currentStep = ref(0)
let advanceTimer = null

const weatherCache = new Map()
const planGenerated = ref(false)
const loadingPlan = ref(false)

const currentQuestion = computed(() => questions[currentStep.value])
const stepNumber = computed(() => currentStep.value + 1)
const percent = computed(() => Math.round((stepNumber.value / questions.length) * 100))
const questionOptions = computed(() =>
  typeof currentQuestion.value.options === 'function'
    ? currentQuestion.value.options()
    : currentQuestion.value.options,
)

const selectedSuburb = computed(
  () => safetySuburbs.value.find((s) => s.id === answers.value.suburb) || safetySuburbs.value[0],
)
const weatherRisk = computed(() => deriveWeatherRisk(selectedSuburb.value))
const profileRisk = computed(() => riskScore(selectedSuburb.value, answers.value))
const riskBand = computed(() => riskBandCalc(profileRisk.value, answers.value))

const planData = ref(null)
const latestPlanText = ref('')
const planStatusText = ref('')
const usedFallbackWeather = ref(false)

async function fetchSuburbWeather(lat, lng) {
  const url =
    `https://api.open-meteo.com/v1/forecast` +
    `?latitude=${lat}&longitude=${lng}` +
    `&current=temperature_2m,apparent_temperature,uv_index,relative_humidity_2m` +
    `&hourly=apparent_temperature` +
    `&forecast_days=1` +
    `&timezone=Australia%2FMelbourne`
  const signal =
    typeof AbortSignal !== 'undefined' && AbortSignal.timeout
      ? AbortSignal.timeout(8000)
      : undefined
  const resp = await fetch(url, signal ? { signal } : {})
  if (!resp.ok) throw new Error('fetch failed')
  const data = await resp.json()
  return {
    temp: Math.round(data.current.temperature_2m),
    feels: Math.round(data.current.apparent_temperature),
    uv: Math.round(data.current.uv_index),
    humidity: Math.round(data.current.relative_humidity_2m),
    hourly: data.hourly.apparent_temperature.map(Math.round),
    weatherSource: 'Live Open-Meteo weather',
  }
}

const downloadPlan=()=>{
  location.reload();
}

function deriveWeatherRisk(suburb) {
  let score = 0
  const reasons = []
  if (suburb.feels >= DEMO_WEATHER_THRESHOLDS.highFeels) {
    score += 3
    reasons.push(`${suburb.feels}\u00b0C feels-like`)
  } else if (suburb.feels >= DEMO_WEATHER_THRESHOLDS.moderateFeels) {
    score += 2
    reasons.push(`${suburb.feels}\u00b0C feels-like`)
  } else {
    reasons.push(`${suburb.feels}\u00b0C feels-like`)
  }
  if (suburb.uv >= DEMO_WEATHER_THRESHOLDS.highUv) {
    score += 1
    reasons.push(`UV ${suburb.uv}`)
  }
  if (
    suburb.humidity >= DEMO_WEATHER_THRESHOLDS.highHumidity &&
    suburb.temp >= DEMO_WEATHER_THRESHOLDS.moderateFeels
  ) {
    score += 1
    reasons.push(`${suburb.humidity}% humidity`)
  }
  if (suburb.canopy <= DEMO_WEATHER_THRESHOLDS.lowCanopy) {
    score += 1
    reasons.push(`${suburb.canopy}% canopy`)
  }
  const hottest = Math.max(...suburb.hourly.filter(Number.isFinite))
  if (hottest >= DEMO_WEATHER_THRESHOLDS.highFeels) {
    score += 1
    reasons.push(`${hottest}\u00b0C forecast peak`)
  }
  return { risk: score >= 4 ? 'high' : score >= 2 ? 'moderate' : 'low', score, reasons }
}

function riskScore(suburb, profile) {
  const w = deriveWeatherRisk(suburb)
  let score = { low: 1, moderate: 2, high: 3 }[suburb.risk] || 1
  const factors = []
  const add = (cond, pts, lbl) => {
    if (cond) {
      factors.push(lbl)
      return pts
    }
    return 0
  }
  score += add(profile.ageGroup === '65-74', 1, 'Age 65+')
  score += add(profile.ageGroup === '75-84', 2, 'Age 75+')
  score += add(profile.ageGroup === '85plus', 3, 'Age 85+')
  score += add(profile.outdoorActivity === 'essential-short', 1, 'Short trip')
  score += add(profile.outdoorActivity === 'essential-long', 2, 'Long trip')
  score += add(profile.outdoorActivity === 'exercise-work', 3, 'Outdoor strain')
  score += add(profile.plannedTime === 'midday', 2, 'Midday timing')
  score += add(profile.timeOutside === '30to60', 1, '30-60 min outside')
  score += add(profile.timeOutside === 'over60', 2, '1h+ outside')
  score += add(profile.fluidMobility === 'fluid-limit', 1, 'Fluid limit')
  score += add(profile.fluidMobility === 'limited', 1, 'Limited mobility')
  score += add(profile.fluidMobility === 'needs-help', 3, 'Needs help')
  const going = profile.outdoorActivity !== 'none' && profile.timeOutside !== 'none'
  score += add(going && suburb.uv >= DEMO_WEATHER_THRESHOLDS.sunProtectionUv, 1, 'UV 3+')
  return { score, weather: w, factors }
}

function riskBandCalc(personalRisk, profile) {
  if (profile.symptoms === 'emergency') return 'Urgent'
  if (profile.symptoms === 'early' && personalRisk.score >= 5) return 'High'
  if (personalRisk.score <= 3) return 'Low'
  if (personalRisk.score <= 6) return 'Moderate'
  return 'High'
}

function selectAnswer(id, value) {
  answers.value[id] = value
  window.clearTimeout(advanceTimer)
  advanceTimer = window.setTimeout(() => goForward(), 260)
}

function goBack() {
  window.clearTimeout(advanceTimer)
  if (currentStep.value > 0) currentStep.value--
}

function goForward() {
  if (currentStep.value < questions.length - 1) {
    currentStep.value++
    return
  }
  generatePlan()
}

const nextBtnText = computed(() => {
  if (currentStep.value === questions.length - 1)
    return planGenerated.value ? 'Update plan' : 'Generate plan'
  return 'Next'
})

async function generatePlan(options = {}) {
  loadingPlan.value = true
  planGenerated.value = true
  const suburbBase = selectedSuburb.value

  let suburb = suburbBase
  usedFallbackWeather.value = false
  try {
    if (!weatherCache.has(suburbBase.id)) {
      const live = await fetchSuburbWeather(suburbBase.lat, suburbBase.lng)
      weatherCache.set(suburbBase.id, live)
    }
    suburb = { ...suburbBase, ...weatherCache.get(suburbBase.id) }
    suburb.risk = deriveWeatherRisk(suburb).risk
  } catch {
    usedFallbackWeather.value = true
    suburb = { ...suburbBase, risk: deriveWeatherRisk(suburbBase).risk }
  }

  const pr = riskScore(suburb, answers.value)
  const band = riskBandCalc(pr, answers.value)
  const timing = safestTiming(suburb, answers.value)
  const refuge = nearestRefuge(suburb)
  const route = buildActionRoute(suburb, answers.value, band, timing, refuge)
  const actions = buildVisualActions(suburb, answers.value, band, timing)
  const factors = buildFactorChips(suburb, answers.value, pr)

  const usefulHours = suburb.hourly
    .map((temp, hour) => ({ temp, hour }))
    .filter((e) => Number.isFinite(e.temp) && e.hour >= 6 && e.hour <= 20)
  const temps = usefulHours.map((e) => e.temp)
  const minTemp = Math.min(...temps)
  const maxTemp = Math.max(...temps)
  const coolest = [...usefulHours].sort((a, b) => a.temp - b.temp)[0]
  const peak = usefulHours.find((e) => e.temp === maxTemp)

  const heatBars = usefulHours.map((e) => ({
    ...e,
    height: 24 + ((e.temp - minTemp) / Math.max(maxTemp - minTemp, 1)) * 60,
    tone:
      e.temp >= DEMO_WEATHER_THRESHOLDS.highFeels
        ? 'hot'
        : e.temp >= DEMO_WEATHER_THRESHOLDS.moderateFeels
          ? 'warm'
          : 'cool',
    isPeak: e === peak,
    isCoolest: e === coolest,
  }))

  planData.value = {
    suburb,
    profileRisk: pr,
    band,
    timing,
    refuge,
    route,
    actions,
    factors,
    heatBars,
    primaryAction: primaryActionFor(band, answers.value, timing),
    scorePercent: band === 'Urgent' ? 100 : Math.min(Math.round((pr.score / 18) * 100), 100),
  }

  latestPlanText.value = buildPlanText(suburb, band, pr, timing, refuge, route, factors)
  planStatusText.value = usedFallbackWeather.value
    ? 'Plan generated with fallback demo weather because live weather was unavailable.'
    : 'Plan ready with live weather.'
  loadingPlan.value = false

  if (!options.silent) {
    focusGeneratedPlan()
  }
}

function safestTiming(suburb, profile) {
  if (profile.outdoorActivity === 'none' || profile.timeOutside === 'none')
    return { title: 'Stay in today', text: 'No trip selected.' }
  const daylight = suburb.hourly
    .map((temp, hour) => ({ temp, hour }))
    .filter((e) => Number.isFinite(e.temp) && e.hour >= 6 && e.hour <= 20)
  if (!daylight.length)
    return { title: 'Morning or after 5 pm', text: 'Avoid the middle of the day.' }
  const preferred = preferredWindow(profile.plannedTime, daylight)
  const bestPreferred = preferred.sort((a, b) => a.temp - b.temp)[0]
  const bestDaylight = [...daylight].sort((a, b) => a.temp - b.temp)[0]
  const best = bestPreferred || bestDaylight
  if (profile.plannedTime === 'midday' && bestDaylight.temp + 1 < best.temp)
    return {
      title: `Move trip to ${hourLabel(bestDaylight.hour)}`,
      text: `Coolest option is ${bestDaylight.temp}\u00b0C.`,
    }
  return {
    title: `Around ${hourLabel(best.hour)}`,
    text: `Best window is ${best.temp}\u00b0C feels-like.`,
  }
}

function preferredWindow(pt, entries) {
  if (pt === 'morning') return entries.filter((e) => e.hour >= 6 && e.hour <= 10)
  if (pt === 'midday') return entries.filter((e) => e.hour >= 10 && e.hour <= 16)
  if (pt === 'evening') return entries.filter((e) => e.hour >= 17 && e.hour <= 20)
  return entries
}

function nearestRefuge(suburb) {
  return safetyRefuges
    .map((r) => ({ ...r, distanceKm: haversine(suburb.lat, suburb.lng, r.lat, r.lng) }))
    .sort((a, b) => a.distanceKm - b.distanceKm)[0]
}

function buildVisualActions(suburb, profile, band, timing) {
  const actions = []
  if (band === 'Urgent' || band === 'High')
    actions.push({
      label: 'Stay cool indoors',
      detail: 'Use AC or find a cool refuge',
      tone: 'danger',
      visual: 'cool',
    })
  else
    actions.push({ label: 'Stay cool', detail: 'Keep hydrated', tone: 'default', visual: 'heart' })
  actions.push({
    label: 'Drink water',
    detail: 'Every 20 min outside',
    tone: 'default',
    visual: 'water',
  })
  if (suburb.uv >= 3)
    actions.push({
      label: 'Sun protection',
      detail: 'Hat, sunscreen, shade',
      tone: 'warm',
      visual: 'shade',
    })
  if (profile.fluidMobility === 'limited' || profile.fluidMobility === 'needs-help')
    actions.push({
      label: 'Mobility help',
      detail: 'Allow extra time',
      tone: 'warm',
      visual: 'mobility',
    })
  return actions.slice(0, 5)
}

function buildActionRoute(suburb, profile, band, timing, refuge) {
  const route = []
  route.push({
    stage: 'Now',
    title: timing.title,
    metric: timing.text,
    points: ['Check symptoms', 'Drink water'],
    visual: 'clock',
  })
  if (profile.outdoorActivity !== 'none') {
    route.push({
      stage: 'During trip',
      title: refuge.name,
      metric: `${refuge.type}, ${refuge.distanceKm.toFixed(1)} km`,
      points: ['Stay in shade', 'Take breaks'],
      visual: 'walk',
    })
  }
  route.push({
    stage: 'After',
    title: 'Cool down',
    metric: 'Return to cool space',
    points: ['Rest in AC', 'Monitor symptoms'],
    visual: 'home',
  })
  return route
}

function primaryActionFor(band, profile, timing) {
  if (band === 'Urgent') return 'Call 000 if emergency symptoms are present. Cool the person now.'
  if (band === 'High') return 'Avoid going out unless essential. Prepare support and a cool refuge.'
  if (profile.outdoorActivity !== 'none')
    return `Go ${timing.title.toLowerCase()} and keep the trip short.`
  return 'Stay cool indoors and check symptoms again later.'
}

function buildPlanText(suburb, band, pr, timing, refuge, route, factors) {
  return [
    'MelbCool Personal Heat Safety Plan',
    `Suburb: ${suburb.name}`,
    `Weather: ${suburb.feels}\u00b0C feels-like, UV ${suburb.uv}`,
    `Personal risk: ${band} (${pr.score})`,
    `Timing: ${timing.title}. ${timing.text}`,
    `Cool refuge: ${refuge.name}`,
    '',
    'Action route:',
    ...route.flatMap((i) => [`${i.stage}: ${i.title}`, ...i.points.map((p) => `- ${p}`)]),
    '',
    'Key drivers:',
    ...[...new Set(factors)].slice(0, 10).map((f) => `- ${f}`),
    '',
    'Emergency: Call 000 for confusion, fainting, collapse, seizure, or very hot dry skin.',
  ].join('\n')
}

async function copyPlan() {
  try {
    await navigator.clipboard.writeText(latestPlanText.value)
    planStatusText.value = 'Plan copied to clipboard.'
  } catch {
    planStatusText.value = 'Could not copy automatically.'
  }
}

async function sharePlan() {
  try {
    if (navigator.share) {
      await navigator.share({ title: 'MelbCool Safety Plan', text: latestPlanText.value })
      return
    }
    copyPlan()
  } catch {}
}

function savePlan() {
  localStorage.setItem(
    'melbcool_heat_safety_plan',
    JSON.stringify({ savedAt: new Date().toISOString(), planText: latestPlanText.value }),
  )
  planStatusText.value = 'Plan saved on this device.'
}

function printPlan() {
  document.body.classList.add('printing-safety-plan')
  window.print()
  setTimeout(() => document.body.classList.remove('printing-safety-plan'), 250)
}

function haversine(lat1, lon1, lat2, lon2) {
  const R = 6371
  const d = ((lat2 - lat1) * Math.PI) / 180
  const e = ((lon2 - lon1) * Math.PI) / 180
  const a =
    Math.sin(d / 2) ** 2 +
    Math.cos((lat1 * Math.PI) / 180) * Math.cos((lat2 * Math.PI) / 180) * Math.sin(e / 2) ** 2
  return R * (2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a)))
}

function hourLabel(h) {
  if (h === 0) return '12:00 am'
  if (h < 12) return `${h}:00 am`
  if (h === 12) return '12:00 pm'
  return `${h - 12}:00 pm`
}

function labelFor(group, value) {
  return labels[group]?.[value] || value || 'Not answered'
}

function buildFactorChips(suburb, profile, personalRisk) {
  const chips = [...personalRisk.factors]
  if (profile.outdoorActivity !== 'none')
    chips.unshift(labelFor('outdoorActivity', profile.outdoorActivity))
  if (suburb.uv >= 3) chips.unshift(`UV ${suburb.uv}`)
  chips.unshift(`${suburb.feels}\u00b0C feels-like`)
  return chips
}

function focusGeneratedPlan() {
  requestAnimationFrame(() => {
    const topbarHeight = document.querySelector('.topbar')?.offsetHeight || 0
    const planEl = document.querySelector('.care-plan')
    if (planEl) {
      const top = planEl.getBoundingClientRect().top + window.scrollY - topbarHeight - 16
      window.scrollTo({
        top: Math.max(0, top),
        behavior: window.matchMedia('(prefers-reduced-motion: reduce)').matches ? 'auto' : 'smooth',
      })
    }
  })
}

function editAnswer(index) {
  loadingPlan.value = false
  planGenerated.value = false
  currentStep.value = index
}

function pictureFor(type = 'map') {
  const c = 'viewBox="0 0 96 72" aria-hidden="true" focusable="false"',
    s =
      'stroke="currentColor" stroke-width="4" stroke-linecap="round" stroke-linejoin="round" fill="none"',
    f = 'fill="currentColor"'
  const scenes = {
    map: `<img src="/temperature.png" class="question-cover" />`,
    building: `<svg ${c}><path ${s} d="M22 58V18h24v40M50 58V26h24v32M16 58h64"/><path ${s} d="M31 28h6M31 40h6M59 36h6M59 48h6"/></svg>`,
    water: `<svg ${c}><path ${s} d="M48 12c12 16 20 27 20 38 0 10-8 16-20 16s-20-6-20-16c0-11 8-22 20-38Z"/></svg>`,
    person: `<svg ${c}><circle ${s} cx="48" cy="19" r="9"/><path ${s} d="M28 58c2-14 10-23 20-23s18 9 20 23"/></svg>`,
    senior: `<svg ${c}><circle ${s} cx="44" cy="18" r="8"/><path ${s} d="M26 57c2-13 9-21 18-21 7 0 13 5 16 14M66 34v24M66 58h8"/></svg>`,
    support: `<svg ${c}><circle ${s} cx="34" cy="22" r="8"/><circle ${s} cx="62" cy="22" r="8"/><path ${s} d="M17 58c2-13 9-21 17-21s15 8 17 21M45 58c2-13 9-21 17-21s15 8 17 21"/></svg>`,
    phone: `<svg ${c}><rect ${s} x="31" y="10" width="34" height="52" rx="7"/><path ${s} d="M43 52h10M39 20h18"/><circle ${f} cx="48" cy="42" r="4"/></svg>`,
    cool: `<svg ${c}><path ${s} d="M24 20h48v28H24zM31 56h34"/><path ${s} d="M35 34h26M40 26l-7 8 7 8M56 26l7 8-7 8"/></svg>`,
    fan: `<svg ${c}><circle ${s} cx="48" cy="31" r="7"/><path ${s} d="M48 24c7-14 20-8 13 2-4 5-9 5-13 5M55 35c14 7 8 20-2 13-5-4-5-9-5-13"/></svg>`,
    refuge: `<svg ${c}><path ${s} d="M18 58h60V30L48 12 18 30v28Z"/><path ${s} d="M39 58V40h18v18M34 30h28"/></svg>`,
    home: `<svg ${c}><path ${s} d="M18 58h60V32L48 14 18 32v26Z"/><path ${s} d="M40 58V42h16v16"/></svg>`,
    'sun-home': `<svg ${c}><circle ${s} cx="72" cy="18" r="8"/><path ${s} d="M18 58h48V36L42 20 18 36v22Z"/></svg>`,
    'hot-home': `<svg ${c}><circle ${f} cx="72" cy="18" r="9"/><path ${s} d="M18 58h48V36L42 20 18 36v22Z"/><path ${s} d="M30 46c5-5 10 5 15 0s10 5 15 0"/></svg>`,
    walk: `<svg ${c}><circle ${s} cx="45" cy="15" r="7"/><path ${s} d="M45 24l-8 16M45 24l10 13M37 40l-8 16M37 40l15 16M17 58h62"/></svg>`,
    bus: `<svg ${c}><rect ${s} x="19" y="18" width="58" height="34" rx="7"/><path ${s} d="M30 52v8M66 52v8M29 29h38"/></svg>`,
    sun: `<svg ${c}><circle ${s} cx="48" cy="34" r="14"/><path ${s} d="M48 8v8M48 52v8M22 34h8M66 34h8"/></svg>`,
    clock: `<svg ${c}><circle ${s} cx="48" cy="36" r="24"/><path ${s} d="M48 22v16l12 7"/></svg>`,
    morning: `<svg ${c}><path ${s} d="M18 54h60M28 45a20 20 0 0 1 40 0"/><path ${s} d="M48 13v10M27 24l7 7M69 24l-7 7"/></svg>`,
    evening: `<svg ${c}><path ${s} d="M18 54h60M28 45a20 20 0 0 1 40 0"/><path ${s} d="M66 14c-10 3-14 15-7 23"/></svg>`,
    timer: `<svg ${c}><circle ${s} cx="48" cy="39" r="22"/><path ${s} d="M39 9h18M48 17v9M48 39l10-8"/></svg>`,
    break: `<svg ${c}><path ${s} d="M22 54h52M34 54V28h24v26M31 28h30"/><path ${s} d="M66 54V38h9v16"/></svg>`,
    shade: `<svg ${c}><path ${s} d="M18 34c8-18 52-18 60 0H18Z"/><path ${s} d="M48 34v26M34 60h28"/><circle ${f} cx="73" cy="16" r="6"/></svg>`,
    car: `<svg ${c}><path ${s} d="M21 45h54l-7-16H28l-7 16Z"/><circle ${s} cx="33" cy="50" r="6"/><circle ${s} cx="63" cy="50" r="6"/></svg>`,
    medicine: `<svg ${c}><rect ${s} x="22" y="18" width="52" height="36" rx="16"/><path ${s} d="M48 18v36M33 36h30"/></svg>`,
    heart: `<svg ${c}><path ${s} d="M48 58S22 43 22 26c0-9 12-15 21-5l5 5 5-5c9-10 21-4 21 5 0 17-26 32-26 32Z"/></svg>`,
    health: `<svg ${c}><path ${s} d="M48 12v48M24 36h48"/><rect ${s} x="20" y="16" width="56" height="40" rx="12"/></svg>`,
    warning: `<svg ${c}><path ${s} d="M48 12 78 60H18L48 12Z"/><path ${s} d="M48 30v12M48 51h.1"/></svg>`,
    emergency: `<svg ${c}><path ${s} d="M48 12v48M24 36h48"/><circle ${s} cx="48" cy="36" r="26"/></svg>`,
    mobility: `<svg ${c}><circle ${s} cx="41" cy="15" r="7"/><path ${s} d="M41 24l-6 16 15 4 8 14M36 40l-9 18M61 30v28M61 58h8"/></svg>`,
  }
  return scenes[type] || scenes.map
}

onMounted(() => {})
</script>

<template>
  <div class="page">
    <NavBar :show-alert-button="false" />

    <main class="senior-plan-app">
      <section class="care-hero">
        <div class="care-hero-copy">
          <!-- <button class="back-btn" @click="router.back()">← Back</button> -->
          <h2>Make a heat plan in simple steps</h2>
          <p>Tap one answer. The next question opens by itself.</p>
        </div>
        <!-- <div class="care-hero-picture" aria-hidden="true">
          <div class="care-sun"></div>
          <div class="care-house"></div>
          <div class="care-tree"></div>
          <div class="care-path"></div>
        </div> -->
      </section>

      <section v-if="!planGenerated" class="gentle-wizard">
        <div class="gentle-progress">
          <button class="wizard-secondary-btn" @click="goBack" :disabled="currentStep === 0">
            Back
          </button>
          <div class="gentle-progress-copy">
            <span>Question {{ stepNumber }} of {{ questions.length }}</span>
            <strong>{{ percent }}%</strong>
          </div>
          <div class="wizard-progress-bar">
            <div :style="{ width: percent + '%' }"></div>
          </div>
          <div class="gentle-step-rail">
            <span
              v-for="(q, i) in questions"
              :key="i"
              class="step-dot"
              :class="{ active: i === currentStep, done: i < currentStep }"
            >
              <span>{{ String(i + 1).padStart(2, '0') }}</span>
            </span>
          </div>
        </div>

        <section class="question-stage">
          <div class="question-card gentle-question-card">
            <!-- {{ currentQuestion.visual }} -->
            <div class="question-visual">
              <img :src="'/question' + currentQuestion.visual + 'Cover.png'" class="question-cover"
            </div>
            <div class="question-kicker">
              <span>{{ String(stepNumber).padStart(2, '0') }}</span>
              <strong>{{ currentQuestion.category }}</strong>
            </div>
            <h2>{{ currentQuestion.title }}</h2>
            <p>{{ currentQuestion.hint }}</p>
            <p class="tap-note">Tap one answer to continue</p>
            <div class="answer-grid picture-answer-grid">
              <button
                v-for="(opt, i) in questionOptions"
                :key="opt.value"
                class="answer-option"
                :class="{ selected: answers[currentQuestion.id] === opt.value }"
                @click="selectAnswer(currentQuestion.id, opt.value)"
              >
              <!-- {{ '/question'+ currentQuestion.visual+ Number(i+1) + '.png' }} -->
                <img
                  class="option-picture"
                  :src="'/question' + currentQuestion.visual + Number(i+1)+'.png'"
                ></img>
                <span class="answer-copy">
                  <strong>{{ opt.label }}</strong>
                  <!-- <small>{{ opt.detail }}</small> -->
                </span>
              </button>
            </div>
            <button class="wizard-primary-btn hidden-next-btn" @click="goForward">
              {{ nextBtnText }}
            </button>
          </div>
        </section>
      </section>

      <LoadingCard
        v-if="loadingPlan"
        title="Generating your heat plan"
        :subtitle="'Fetching live weather data for ' + selectedSuburb.name + '…'"
      />

      <article v-if="planGenerated && planData" class="care-plan">
        <section class="care-plan-hero" :class="planData.band.toLowerCase()">
          <div class="care-plan-picture">
            <img :src="'/band' + planData.band + '.png'" class="band-picture"
          </div>
          <div>
            <!-- <button class="back-btn" @click="downloadPlan">
              Back
            </button> -->
            <p class="plan-small-label">Your heat plan</p>
            <h2>{{ planData.band }}</h2>
            <p>{{ riskCopy[planData.band] }}</p>
            <strong class="primary-plan-line">{{ planData.primaryAction }}</strong>
          </div>
          <div class="dashboard-score">
            <strong>{{ planData.band === 'Urgent' ? '!' : planData.profileRisk.score }}</strong>
            <span>score</span>
          </div>
        </section>

        <section class="do-first-panel">
          <div class="dashboard-card-heading">
            <span>Do first</span><strong>Simple actions</strong>
          </div>
          <div class="dashboard-action-grid">
            <div
              v-for="(action, i) in planData.actions"
              :key="i"
              class="dashboard-action"
              :class="action.tone"
            >
            <!-- {{ action.visual }} -->
            <div class="plan-card-picture">
                <img  :src="'/actions' + action.visual + '.png'"></img>
            </div>
              <strong>{{ action.label }}</strong>
              <small>{{ action.detail }}</small>
            </div>
          </div>
        </section>

        <section class="visual-plan-grid">
          <div class="visual-plan-card time-card">
            <div class="plan-card-picture">
                <img  :src="'/besttime.png'"></img>
            </div>
            <span>Best time</span>
            <strong>{{ planData.timing.title }}</strong>
            <p>{{ planData.timing.text }}</p>
            <div class="dashboard-heat-bars">
              <div
                v-for="(bar, i) in planData.heatBars"
                :key="i"
                class="dashboard-heat-bar"
                :class="[bar.tone, { peak: bar.isPeak, coolest: bar.isCoolest }]"
                :style="{ height: bar.height + '%' }"
              ></div>
            </div>
            <small>{{
              planData.heatBars.find((b) => b.isCoolest)
                ? `Coolest: ${hourLabel(planData.heatBars.find((b) => b.isCoolest).hour)} at ${planData.heatBars.find((b) => b.isCoolest).temp}\u00b0C`
                : ''
            }}</small>
          </div>

          <div class="visual-plan-card refuge-card">
            <div class="plan-card-picture">
                <img  :src="'/coolplace.png'"></img>
            </div>
            <span>Cool place</span>
            <strong>{{ planData.refuge.name }}</strong>
            <p>
              {{ planData.refuge.type }}, {{ planData.refuge.distanceKm.toFixed(1) }} km away. Open
              {{ planData.refuge.hours }}.
            </p>
          </div>

          <div class="visual-plan-card heat-card">
            <div class="plan-card-picture">
                <img  :src="'/todayoutside.png'"></img>
            </div>
            <span>Today outside</span>
            <strong>{{ planData.suburb.name }} - {{ planData.band }}</strong>
            <p>
              {{ planData.suburb.feels }}℃ feels-like, UV {{ planData.suburb.uv }}, humidity
              {{ planData.suburb.humidity }}%
            </p>
          </div>
        </section>

        <section class="picture-route">
          <div v-for="(item, i) in planData.route" :key="i" class="route-card">
            <!-- {{ item.visual }} -->
              <div  class="route-picture">
                  <img :src="'/route' + item.visual + '.png'"></img>
              </div>
            <span>{{ item.stage }}</span>
            <strong>{{ item.title }}</strong>
            <div>{{ item.metric }}</div>
            <p>{{ item.points.slice(0, 2).join(' | ') }}</p>
          </div>
        </section>

        <section class="factor-panel simple-factor-panel">
          <div class="dashboard-card-heading"><span>Why</span><strong>Main reasons</strong></div>
          <div class="factor-chip-grid">
            <span v-for="(f, i) in planData.factors" :key="i">{{ f }}</span>
          </div>
        </section>

        <section class="source-panel simple-source-panel">
          <div class="dashboard-card-heading">
            <span>Sources</span><strong>Verified advice</strong>
          </div>
          <div class="source-chip-grid">
            <a
              v-for="src in verifiedSources"
              :key="src.id"
              :href="src.url"
              target="_blank"
              rel="noreferrer"
              ><strong>{{ src.label }}</strong></a
            >
          </div>
        </section>

        <section class="emergency-strip simple-emergency-strip">
          <div>
            <strong>Emergency signs</strong>
            <p>Call 000 for confusion, fainting, collapse, seizure, or very hot dry skin.</p>
          </div>
          <div class="emergency-buttons">
            <a href="tel:000">Call 000</a>
            <a href="tel:1300606024">Nurse-on-Call</a>
          </div>
        </section>

        <!-- <div class="dashboard-actions simple-plan-actions">
          <button @click="printPlan">Print</button>
          <button @click="sharePlan">Share</button>
          <button @click="copyPlan">Copy</button>
          <button @click="savePlan">Save</button>
        </div> -->
        <p class="plan-status">{{ planStatusText }}</p>
      </article>
    </main>

    <Footer />
  </div>
</template>

<style scoped>
.band-picture{
  height: 120px;
  border-radius: 15%;
}
.question-cover{
  width: 100%;
  height: 150px;
  object-fit: cover;
}
.back-btn{
background: linear-gradient(135deg, rgba(13, 58, 143, 0.95), rgba(11, 127, 121, 0.88));
    padding: 1rem;
    border-radius: 16px;
    color: #fff;
    border: none;
    cursor: pointer;
    width: 110px;
    font-size: 20px;
}

.page {
  min-height: 100vh;
  background-color: #f5f5f5;
}

.senior-plan-app {
  width: min(1180px, calc(100% - 2rem));
  margin: 1rem auto 2.5rem;
}

.care-hero {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 310px;
  gap: 1.25rem;
  align-items: center;
  min-height: 230px;
  padding: 1.4rem 1.6rem;
  border-radius: 28px;
  background: linear-gradient(135deg, #1a348a 0%, #0d7070 100%);
  box-shadow: 0 8px 32px rgba(13, 58, 143, 0.25);
}

.care-hero .back-btn {
  background: rgba(255, 255, 255, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.4);
  color: #fff;
  padding: 0.5rem 1rem;
  border-radius: 20px;
  font-size: 0.9rem;
  cursor: pointer;
  margin-bottom: 0.5rem;
  transition: background 0.2s;
}

.care-hero .back-btn:hover {
  background: rgba(255, 255, 255, 0.3);
}

.care-hero h2 {
  max-width: 680px;
  margin: 0;
  color: #fff;
  font-family: Fraunces, Georgia, serif;
  font-size: 3.1rem;
  line-height: 1.02;
}

.care-hero p {
  max-width: 560px;
  margin: 0.7rem 0 0;
  color: rgba(255, 255, 255, 0.85);
  font-size: 1.2rem;
  line-height: 1.45;
}

.care-hero-picture {
  position: relative;
  min-height: 190px;
  border-radius: 26px;
  background: rgba(255, 255, 255, 0.12);
  overflow: hidden;
}

.care-sun {
  position: absolute;
  width: 68px;
  height: 68px;
  border-radius: 50%;
  background: #f3b45d;
  box-shadow: 0 0 0 12px rgba(243, 180, 93, 0.18);
  top: 22px;
  right: 26px;
  animation: float 5.8s ease-in-out infinite;
}

.care-house {
  position: absolute;
  left: 34px;
  bottom: 30px;
  width: 116px;
  height: 70px;
  border-radius: 14px 14px 10px 10px;
  background: #fff;
  box-shadow: 0 14px 28px rgba(18, 49, 75, 0.1);
}

.care-house::before {
  content: '';
  position: absolute;
  left: 14px;
  top: -32px;
  width: 88px;
  height: 88px;
  background: #7ab686;
  transform: rotate(45deg);
  border-radius: 10px;
  z-index: -1;
}

.care-house::after {
  content: '';
  position: absolute;
  left: 44px;
  bottom: 0;
  width: 28px;
  height: 38px;
  border-radius: 10px 10px 0 0;
  background: #e5f2f4;
}

.care-tree {
  position: absolute;
  right: 62px;
  bottom: 35px;
  width: 24px;
  height: 70px;
  border-radius: 999px;
  background: #7d9b69;
}

.care-tree::before {
  content: '';
  position: absolute;
  left: -32px;
  top: -36px;
  width: 88px;
  height: 70px;
  border-radius: 50%;
  background: #82b58b;
}

.care-path {
  position: absolute;
  left: 112px;
  right: 28px;
  bottom: -18px;
  height: 58px;
  border-radius: 50% 50% 0 0;
  background: rgba(93, 176, 191, 0.34);
  transform: skewX(-16deg);
}

.gentle-wizard {
  display: grid;
  grid-template-columns: 220px minmax(0, 1fr);
  gap: 1rem;
  align-items: start;
  margin-top: 1rem;
}

.gentle-progress {
  position: sticky;
  top: 7rem;
  display: grid;
  gap: 0.85rem;
  padding: 1rem;
  border: 1px solid rgba(205, 226, 226, 0.9);
  border-radius: 24px;
  background: rgba(255, 255, 255, 0.95);
  box-shadow: 0 18px 48px rgba(18, 49, 75, 0.08);
}

.gentle-progress .wizard-secondary-btn {
  width: 100%;
  min-height: 3.4rem;
  border-radius: 18px;
  background: #e9f6f5;
  color: #0b625e;
  font-size: 1.05rem;
  border: none;
  cursor: pointer;
  font-weight: 600;
}

.gentle-progress .wizard-secondary-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.gentle-progress-copy {
  display: flex;
  justify-content: space-between;
  gap: 0.75rem;
  color: #40556b;
  font-weight: 700;
}

.gentle-progress-copy strong {
  color: #0b7f79;
}

.gentle-step-rail {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 0.38rem;
}

.gentle-step-rail .step-dot {
  display: block;
  min-height: 0;
  padding: 0;
  border: 0;
  border-radius: 999px;
  background: transparent;
}

.gentle-step-rail .step-dot span {
  width: 100%;
  height: 0.72rem;
  border-radius: 999px;
  background: #dfecee;
  color: transparent;
  font-size: 0;
  display: block;
}

.gentle-step-rail .step-dot.done span,
.gentle-step-rail .step-dot.active span {
  background: #0b7f79;
}

.gentle-question-card {
  min-height: 620px;
  padding: 1.35rem;
  border: 1px solid rgba(205, 226, 226, 0.92);
  border-radius: 30px;
  background: rgba(255, 255, 255, 0.96);
  box-shadow: 0 24px 70px rgba(18, 49, 75, 0.1);
}

.gentle-question-card .question-kicker {
  margin-top: 1rem;
}

.gentle-question-card .question-kicker span {
  width: 3.2rem;
  height: 3.2rem;
  background: #0b7f79;
  font-size: 1rem;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  font-weight: 900;
}

.gentle-question-card .question-kicker strong {
  color: #0b625e;
  font-size: 0.95rem;
  margin-left: 0.5rem;
}

.gentle-question-card h2 {
  max-width: 760px;
  margin: 0.85rem 0 0.35rem;
  font-size: 2.45rem;
  line-height: 1.08;
  font-weight: 700;
  color: #12314b;
}

.gentle-question-card p {
  max-width: 680px;
  color: #40556b;
  font-size: 1.12rem;
}

.tap-note {
  display: inline-flex;
  align-items: center;
  min-height: 2.6rem;
  margin-top: 0.85rem;
  padding: 0.45rem 0.8rem;
  border-radius: 999px;
  background: #eff8f3;
  color: #2e6b37;
  font-size: 1rem;
  font-weight: 700;
}

.question-visual {
  display: grid;
  place-items: center;
  min-height: 150px;
  border-radius: 26px;
  background: linear-gradient(135deg, #e7f5ff, #ecf8f0);
  color: #0b7f79;
  overflow: hidden;
  margin-top: 1rem;
}

.question-visual :deep(svg) {
  width: min(260px, 70%);
  height: 136px;
  animation: float 4.8s ease-in-out infinite;
}

.picture-answer-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 1rem;
  margin-top: 1rem;
}

.picture-answer-grid .answer-option {
  display: grid;
  grid-template-columns: 96px 1fr;
  gap: 0.85rem;
  align-items: center;
  min-height: 126px;
  padding: 0.85rem;
  border: 2px solid #dce9ec;
  border-radius: 24px;
  background: #fff;
  box-shadow: 0 14px 28px rgba(18, 49, 75, 0.07);
  cursor: pointer;
  text-align: left;
  font-family: inherit;
}

.picture-answer-grid .answer-option:hover {
  border-color: #0b7f79;
  transform: translateY(-3px);
}
.picture-answer-grid .answer-option.selected {
  border-color: #3d7b44;
  background: #f3fbf4;
}

.option-picture {
  width: 96px;
  height: 88px;
  /* border-radius: 22px; */
  background: linear-gradient(135deg, #eff9fb, #ecf8f0);
  display: grid;
  place-items: center;
  color: #0b7f79;
}

.option-picture :deep(svg) {
  width: 74px;
  height: 58px;
}

.answer-copy strong {
  margin: 0;
  color: #12314b;
  font-size: 1.24rem;
  line-height: 1.15;
  display: block;
}

.answer-copy small {
  margin-top: 0.3rem;
  color: #51677a;
  font-size: 1rem;
  font-weight: 700;
  line-height: 1.3;
  display: block;
}

.hidden-next-btn {
  display: none;
}

.care-plan {
  margin-top: 1.1rem;
  animation: slideIn 520ms cubic-bezier(0.2, 0.8, 0.2, 1) both;
}

.care-plan-hero {
  display: grid;
  grid-template-columns: 210px minmax(0, 1fr) auto;
  gap: 1.2rem;
  align-items: center;
  padding: 1.25rem;
  border: 2px solid #cfe5d2;
  border-radius: 30px;
  background: linear-gradient(135deg, #f3fbf4, #fff);
  box-shadow: 0 24px 70px rgba(18, 49, 75, 0.1);
}

.care-plan-hero.moderate {
  border-color: #f0dec0;
  background: linear-gradient(135deg, #fff8ed, #fff);
}
.care-plan-hero.high,
.care-plan-hero.urgent {
  border-color: #f0c9be;
  background: linear-gradient(135deg, #fff2ed, #fff);
}

.care-plan-picture {
  position: relative;
    /* min-height: 170px; */
    border-radius: 26px;
    /* background: linear-gradient(180deg, #dff3ff, #f6fbf3); */
    /* overflow: hidden; */
    display: flex;
    justify-content: center;
}

.plan-sun {
  position: absolute;
  width: 68px;
  height: 68px;
  border-radius: 50%;
  background: #f3b45d;
  box-shadow: 0 0 0 12px rgba(243, 180, 93, 0.18);
  top: 22px;
  right: 26px;
}

.plan-person {
  position: absolute;
  left: 70px;
  bottom: 34px;
  width: 38px;
  height: 70px;
  border-radius: 20px 20px 10px 10px;
  background: #0b7f79;
}

.plan-person::before {
  content: '';
  position: absolute;
  left: 6px;
  top: -28px;
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: #12314b;
}

.plan-shade {
  position: absolute;
  left: 32px;
  top: 44px;
  width: 116px;
  height: 52px;
  border-radius: 60px 60px 10px 10px;
  background: rgba(125, 155, 105, 0.88);
}

.plan-small-label {
  margin: 0;
  color: #0b625e;
  font-size: 1rem;
  font-weight: 700;
}

.care-plan-hero h2 {
  margin: 0.2rem 0;
  color: #12314b;
  font-family: Fraunces, Georgia, serif;
  font-size: 3.7rem;
  line-height: 0.96;
}

.care-plan-hero p {
  max-width: 620px;
  margin: 0;
  color: #40556b;
  font-size: 1.15rem;
}

.primary-plan-line {
  display: block;
  margin-top: 0.7rem;
  color: #12314b;
  font-size: 1.3rem;
  line-height: 1.3;
}

.dashboard-score {
  min-width: 112px;
  padding: 0.75rem;
  border-radius: 24px;
  background: #fff;
  text-align: center;
  box-shadow: inset 0 0 0 1px #dce9ec;
}

.dashboard-score strong {
  color: #0b7f79;
  font-size: 3rem;
}

.care-plan-hero.high .dashboard-score strong,
.care-plan-hero.urgent .dashboard-score strong {
  color: #c24735;
}

.do-first-panel,
.simple-factor-panel,
.simple-source-panel,
.simple-emergency-strip,
.visual-plan-card,
.route-card {
  border: 1px solid rgba(205, 226, 226, 0.92);
  border-radius: 24px;
  background: rgba(255, 255, 255, 0.96);
  box-shadow: 0 16px 40px rgba(18, 49, 75, 0.08);
}

.do-first-panel {
  margin-top: 1rem;
  padding: 1rem;
}

.do-first-panel .dashboard-action-grid {
  display: grid;
  grid-template-columns: repeat(5, minmax(0, 1fr));
  gap: 0.85rem;
}

.dashboard-action {
  min-height: 176px;
  padding: 0.75rem;
  border: 2px solid #dce9ec;
  border-radius: 22px;
  background: #fff;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.action-picture {
   width: 100%;
  height: 86px;
  margin-bottom: 0.55rem;
  border-radius: 22px;
  background: linear-gradient(135deg, #eff9fb, #ecf8f0);
  display: grid;
  place-items: center;
  color: #0b7f79;
}

.action-picture img {
  width: 76px;
  height: 58px;
}

.action-picture :deep(svg) {
  width: 76px;
  height: 58px;
}

.dashboard-action.danger .action-picture {
  background: #fff0ee;
  color: #c24735;
}
.dashboard-action.warm .action-picture {
  background: #fff7e7;
  color: #b56916;
}

.dashboard-action strong {
  margin: 0;
  color: #12314b;
  font-size: 1.1rem;
}
.dashboard-action small {
  color: #51677a;
  font-size: 0.95rem;
}

.visual-plan-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  
  gap: 1rem;
  margin-top: 1rem;
}

.visual-plan-card {
  padding: 1rem;
}

.plan-card-picture {
  width: 100%;
  height: 118px;
  margin-bottom: 0.8rem;
  border-radius: 22px;
  background: linear-gradient(135deg, #eff9fb, #ecf8f0);
  display: grid;
  place-items: center;
  color: #0b7f79;
}

.plan-card-picture img {
   width: 100%;
   height: 108px;
   object-fit: cover;
}

.plan-card-picture :deep(svg) {
  width: 108px;
  height: 78px;
}

.visual-plan-card span {
  color: #0b625e;
  font-size: 0.92rem;
  font-weight: 700;
}
.visual-plan-card strong {
  display: block;
  margin-top: 0.22rem;
  color: #12314b;
  font-size: 1.35rem;
  line-height: 1.2;
}
.visual-plan-card p {
  margin: 0.4rem 0 0;
  color: #51677a;
  font-size: 1rem;
  line-height: 1.35;
}

.time-card .dashboard-heat-bars {
  min-height: 94px;
  margin-top: 0.8rem;
  display: grid;
  grid-template-columns: repeat(15, minmax(0, 1fr));
  gap: 0.3rem;
  align-items: flex-end;
}

.dashboard-heat-bar {
  border-radius: 999px 999px 5px 5px;
  background: #83b58a;
  min-height: 18%;
  transition: height 0.3s;
}

.dashboard-heat-bar.warm {
  background: #d69a44;
}
.dashboard-heat-bar.hot {
  background: #c24735;
}
.dashboard-heat-bar.peak {
  box-shadow: 0 0 0 3px rgba(194, 71, 53, 0.13);
}
.dashboard-heat-bar.coolest {
  box-shadow: 0 0 0 3px rgba(61, 123, 68, 0.13);
}

.time-card small {
  display: block;
  margin-top: 0.55rem;
  color: #40556b;
  font-size: 0.95rem;
  font-weight: 700;
}

.picture-route {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 1rem;
  margin-top: 1rem;
}

.route-card {
  padding: 1rem;
}

.route-picture {
  width: 100%;
  height: 96px;
  margin-bottom: 0.75rem;
  border-radius: 22px;
  background: linear-gradient(135deg, #eff9fb, #ecf8f0);
  display: grid;
  place-items: center;
  color: #0b7f79;
  background: transparent !important;
}

.route-picture img {
    width: 100%;
    height: 108px;
    object-fit: cover;
}

.route-picture :deep(svg) {
  width: 86px;
  height: 64px;
}

.route-card span {
  color: #0b625e;
  font-size: 0.92rem;
  font-weight: 700;
}
.route-card strong {
  display: block;
  margin-top: 0.25rem;
  color: #12314b;
  font-size: 1.08rem;
}
.route-card div {
  display: inline-flex;
  margin-top: 0.55rem;
  padding: 0.35rem 0.7rem;
  border-radius: 999px;
  background: #eef7ff;
  color: #174f8a;
  font-weight: 700;
}
.route-card p {
  margin: 0.55rem 0 0;
  color: #51677a;
  font-size: 1rem;
  font-weight: 700;
  line-height: 1.35;
}

.simple-factor-panel,
.simple-source-panel {
  margin-top: 1rem;
  padding: 1rem;
}

.dashboard-card-heading {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.75rem;
}
.dashboard-card-heading span {
  color: #0b625e;
  font-size: 0.92rem;
  font-weight: 700;
}
.dashboard-card-heading strong {
  color: #12314b;
}

.factor-chip-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 0.55rem;
}
.factor-chip-grid span {
  min-height: 3.2rem;
  padding: 0.5rem;
  border-radius: 18px;
  background: #eff8f3;
  font-size: 1rem;
  display: grid;
  place-items: center;
}

.source-chip-grid {
  display: grid;
  grid-template-columns: repeat(5, minmax(0, 1fr));
  gap: 0.55rem;
}
.source-chip-grid a {
  min-height: 3.4rem;
  padding: 0.7rem;
  border-radius: 18px;
  background: #eef7ff;
  color: #174f8a;
  text-decoration: none;
  display: grid;
  place-items: center;
  transition:
    transform 180ms ease,
    background 180ms ease;
}
.source-chip-grid a:hover {
  transform: translateY(-1px);
  background: #dff0ff;
}

.emergency-strip {
  margin-top: 1rem;
  border-width: 2px;
  border-color: #e9bbb6;
  background: #fff2ed;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem;
  flex-wrap: wrap;
  gap: 1rem;
}
.emergency-strip strong {
  font-size: 1.2rem;
  color: #c24735;
}
.emergency-strip p {
  margin: 0.25rem 0 0;
  font-size: 1.02rem;
  color: #40556b;
}
.emergency-buttons {
  display: flex;
  gap: 0.75rem;
}
.emergency-buttons a {
  min-height: 3rem;
  padding: 0.75rem 1.5rem;
  border-radius: 14px;
  font-weight: 900;
  text-decoration: none;
  transition: transform 180ms ease;
  display: grid;
  place-items: center;
}
.emergency-buttons a:first-child {
  background: #c24735;
  color: #fff;
}
.emergency-buttons a:last-child {
  background: #0d3a8f;
  color: #fff;
}
.emergency-buttons a:hover {
  transform: translateY(-1px);
}

.dashboard-actions {
  display: flex;
  justify-content: center;
  gap: 0.75rem;
  max-width: 560px;
  margin: 1rem auto 0;
  flex-wrap: wrap;
}
.dashboard-actions button {
  min-height: 3rem;
  padding: 0.75rem 1.5rem;
  border-radius: 14px;
  background: linear-gradient(135deg, #0d3a8f, #0b7f79);
  color: #fff;
  font-weight: 900;
  border: none;
  cursor: pointer;
  transition:
    transform 180ms ease,
    filter 180ms ease;
}
.dashboard-actions button:hover {
  transform: translateY(-1px);
  filter: brightness(0.95);
}

.plan-status {
  text-align: center;
  color: #40556b;
  font-size: 0.95rem;
  margin-top: 0.5rem;
}

@keyframes float {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-8px);
  }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(24px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 980px) {
  .care-hero,
  .gentle-wizard,
  .care-plan-hero,
  .visual-plan-grid,
  .picture-route {
    grid-template-columns: 1fr;
  }
  .gentle-progress {
    position: static;
  }
  .do-first-panel .dashboard-action-grid,
  .factor-chip-grid,
  .source-chip-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .care-plan-hero {
    display: flex;
    flex-direction: column;
  }
  .care-plan-picture {
    min-height: 120px;
  }
}

@media (max-width: 640px) {
  .senior-plan-app {
    width: 100%;
    margin: 0;
    padding: 0.85rem;
  }
  .care-hero h2 {
    font-size: 2.15rem;
  }
  .care-hero p {
    font-size: 1.08rem;
  }
  .gentle-question-card h2 {
    font-size: 1.85rem;
  }
  .gentle-question-card {
    min-height: auto;
  }
  .picture-answer-grid,
  .dashboard-action-grid,
  .factor-chip-grid,
  .source-chip-grid,
  .dashboard-actions {
    grid-template-columns: 1fr;
  }
  .emergency-strip {
    flex-direction: column;
    align-items: flex-start;
  }
  .care-plan-hero h2 {
    font-size: 2.8rem;
  }
}
</style>

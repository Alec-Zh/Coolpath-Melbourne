<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'
import { useRoute } from 'vue-router'
import NavBar from '../components/NavBar.vue'
import Footer from '../components/Footer.vue'
import OutfitWeatherCard from '../components/OutfitWeatherCard.vue'
import OutfitItemCard from '../components/OutfitItemCard.vue'

const API_BASE = 'https://qcbqul6ys2.execute-api.ap-southeast-2.amazonaws.com'

const route = useRoute()

// ── Suburb data ──────────────────────────────────────────────────────────────
const allSuburbs = ref([])
const loading = ref(true)
const error = ref(null)
const selectedSuburbId = ref('')

const selectedSuburb = computed(
  () => allSuburbs.value.find((s) => s.suburb_id === Number(selectedSuburbId.value)) ?? null,
)

onMounted(async () => {
  try {
    const res = await fetch(`${API_BASE}/suburbs`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    allSuburbs.value = (await res.json()).sort((a, b) => a.suburb_name.localeCompare(b.suburb_name))
    // Prefer suburbId from route query (passed from SuburbDetail),
    // fall back to first suburb in list
    const queryId = route.query.suburbId
    if (queryId && allSuburbs.value.find((s) => s.suburb_id === Number(queryId))) {
      selectedSuburbId.value = String(queryId)
    } else if (allSuburbs.value.length) {
      selectedSuburbId.value = String(allSuburbs.value[0].suburb_id)
    }
    await nextTick()
    autoSelectBestOutfit()
  } catch (e) {
    error.value = 'Could not load suburb data. Please try again.'
    console.error(e)
  } finally {
    loading.value = false
  }
})

// Auto-select the best outfit: per slot pick lowest heatAdj good item, slot-less good items all selected
function autoSelectBestOutfit() {
  const mode = climateMode.value
  const uv = uvHigh.value
  const goodItems = ALL_ITEMS.filter((item) => {
    if (item.modes[mode] !== 'good') return false
    if (item.uvOnly && !uv) return false
    return true
  })
  // For each slot, find the item with the lowest heatAdj (most cooling)
  const slotWinner = {}
  goodItems.forEach((item) => {
    if (item.slot) {
      if (!slotWinner[item.slot] || item.heatAdj < slotWinner[item.slot].heatAdj) {
        slotWinner[item.slot] = item
      }
    }
  })
  const next = new Set()
  goodItems.forEach((item) => {
    if (item.slot) {
      if (slotWinner[item.slot]?.id === item.id) next.add(item.id)
    } else {
      next.add(item.id)
    }
  })
  selectedItems.value = next
}

// AC 4.1.3 — reset expansion and auto-select best outfit when suburb changes
watch(selectedSuburbId, async () => {
  expandedGroups.value = new Set()
  await nextTick()
  autoSelectBestOutfit()
})

// ── Climate mode (temperature-based) ─────────────────────────────────────────
const climateMode = computed(() => {
  const t = selectedSuburb.value?.apparent_temperature ?? 25
  if (t < 18) return 'cool'
  if (t < 28) return 'mild'
  return 'hot'
})

const uvHigh = computed(() => (selectedSuburb.value?.uv_index ?? 0) >= 3)

// ── Master item list ──────────────────────────────────────────────────────────
// modes: which climateMode values this item appears in, and as what category
// uvOnly: if true, only shown when uv_index >= 3 (independent of temperature)
// effect/explanation: keyed by climateMode for context-aware text
const ALL_ITEMS = [
  {
    id: 'hat',
    name: 'Wide-brim hat',
    icon: '🎩',
    modes: { hot: 'good', mild: 'good', cool: 'good' },
    effect: {
      hot: 'Blocks UV and reduces heat stress on your head and neck',
      mild: 'Good sun protection even on mild days',
      cool: 'Keeps head warm and still provides some UV cover',
    },
    explanation: {},
    heatAdj: -1.5,
    layer: 'lh',
    uvOnly: false,
    slot: 'head',
  },
  {
    id: 'shirt',
    name: 'Light cotton shirt',
    icon: '👕',
    modes: { hot: 'good', mild: 'good', cool: 'bad' },
    effect: {
      hot: 'Breathable and reflects heat — ideal for hot weather',
      mild: 'Comfortable and breathable on a mild day',
      cool: 'Too light for today — likely too cold outside',
    },
    explanation: {
      cool: 'Light fabrics provide little insulation in cool conditions.',
    },
    heatAdj: -1.5,
    layer: 'ls',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'pants',
    name: 'Loose light pants',
    icon: '👖',
    modes: { hot: 'good', mild: 'good', cool: 'bad' },
    effect: {
      hot: 'Good airflow and UV coverage for hot conditions',
      mild: 'Comfortable with enough air circulation',
      cool: "Not warm enough for today's temperature",
    },
    explanation: {
      cool: "Light pants won't keep you warm in cool or cold weather.",
    },
    heatAdj: -1.0,
    layer: 'lp',
    uvOnly: false,
    slot: 'bottom',
  },
  {
    id: 'sg',
    name: 'Sunglasses',
    icon: '🕶️',
    modes: { hot: 'good', mild: 'good', cool: 'good' },
    effect: {
      hot: 'UV is high — eye protection is essential today',
      mild: 'UV is moderate or above — sunglasses are recommended',
      cool: 'UV can still be significant in cool weather — protect your eyes',
    },
    explanation: {},
    heatAdj: -0.3,
    layer: 'lsg',
    uvOnly: true,
    slot: null,
  },
  {
    id: 'water',
    name: 'Water bottle',
    icon: '🍶',
    modes: { hot: 'good', mild: 'good', cool: 'good' },
    effect: {
      hot: 'Drink every 15–20 min outdoors — dehydration risk is high',
      mild: "Stay hydrated, especially if you're active outside",
      cool: 'Good habit even in cool weather — easy to forget when not sweating',
    },
    explanation: {},
    heatAdj: -0.8,
    layer: 'lw',
    uvOnly: false,
    slot: null,
  },
  {
    id: 'sc',
    name: 'Sunscreen SPF 50+',
    icon: '🧴',
    modes: { hot: 'good', mild: 'good', cool: 'good' },
    effect: {
      hot: 'Apply before going out and reapply every 2 hours',
      mild: 'UV is moderate or above — sunscreen is still important',
      cool: "UV doesn't disappear in winter — apply before heading out",
    },
    explanation: {},
    heatAdj: -0.3,
    layer: 'lsc',
    uvOnly: true,
    slot: null,
  },
  {
    id: 'warmjacket',
    name: 'Warm jacket',
    icon: '🧥',
    modes: { cool: 'good' },
    effect: {
      cool: 'Good insulation for cool conditions — keeps core temperature stable',
    },
    explanation: {},
    heatAdj: 1.5,
    layer: 'lj',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'jacket',
    name: 'Dark heavy jacket',
    icon: '🧥',
    modes: { hot: 'bad', mild: 'bad' },
    effect: {
      hot: 'Traps heat — significantly raises your body temperature',
      mild: 'Too warm for today — raises heat exposure unnecessarily',
    },
    explanation: {
      hot: 'Heavy dark jackets trap heat and block airflow in warm conditions.',
      mild: 'Even on mild days, heavy jackets can cause overheating during activity.',
    },
    heatAdj: 3.5,
    layer: 'lj',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'dshirt',
    name: 'Tight dark shirt',
    icon: '👔',
    modes: { hot: 'bad', mild: 'bad' },
    effect: {
      hot: 'Absorbs heat and restricts airflow — uncomfortable and risky',
      mild: 'Dark tight fabric absorbs more heat than needed today',
    },
    explanation: {
      hot: 'Dark fabric absorbs sunlight and tight fit reduces air circulation.',
      mild: 'Even on mild days, dark tight clothing can make you warmer than expected.',
    },
    heatAdj: 2.5,
    layer: 'ld',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'longsleeve',
    name: 'Light long-sleeve shirt',
    icon: '🩱',
    modes: { cool: 'good', mild: 'good' },
    effect: {
      cool: 'Adds warmth while keeping a breathable layer',
      mild: 'Good coverage without overheating on a mild day',
    },
    explanation: {},
    heatAdj: -0.5,
    layer: 'llongsleeve',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'scarf',
    name: 'Light scarf',
    icon: '🧣',
    modes: { cool: 'good' },
    effect: {
      cool: 'Protects the neck and face from cold wind',
    },
    explanation: {},
    heatAdj: -0.8,
    layer: 'lscarf',
    uvOnly: false,
    slot: null,
  },
  {
    id: 'cap',
    name: 'Baseball cap',
    icon: '🧢',
    modes: { hot: 'good', mild: 'good' },
    effect: {
      hot: 'Shields face from direct sun — lighter option than a wide-brim hat',
      mild: 'Good sun cover for longer outdoor time on a mild day',
    },
    explanation: {},
    heatAdj: -0.8,
    layer: 'lcap',
    uvOnly: true,
    slot: 'head',
  },
  {
    id: 'umbrella',
    name: 'Sun umbrella',
    icon: '☂️',
    modes: { hot: 'good', mild: 'good' },
    effect: {
      hot: 'Creates portable shade — one of the most effective ways to reduce heat exposure',
      mild: 'Useful if you\'ll be standing or walking in sun for a while',
    },
    explanation: {},
    heatAdj: -1.2,
    layer: 'lumbrella',
    uvOnly: false,
    slot: null,
  },
  {
    id: 'flipflops',
    name: 'Flip flops',
    icon: '🩴',
    modes: { hot: 'bad', mild: 'bad' },
    effect: {
      hot: 'No foot support and pavement can reach 60°C+ in direct sun',
      mild: 'Minimal protection and unstable for longer walks',
    },
    explanation: {
      hot: 'Flip flops offer no arch support and expose feet to extreme pavement heat.',
      mild: 'Better to choose a shoe with grip and support for walking comfort.',
    },
    heatAdj: 0.5,
    layer: 'lflipflops',
    uvOnly: false,
    slot: 'footwear',
  },
  {
    id: 'hoodie',
    name: 'Synthetic hoodie',
    icon: '🧥',
    modes: { cool: 'good', mild: 'bad', hot: 'bad' },
    effect: {
      cool: 'Warm and comfortable for cool conditions',
      mild: 'Synthetic fabric traps warmth unnecessarily on a mild day',
      hot: 'Synthetic layers trap heat and reduce sweat evaporation — dangerous today',
    },
    explanation: {
      mild: 'Synthetic fabric blocks airflow and holds body heat longer than needed.',
      hot: 'Hoodies in heat significantly raise body temperature and risk of heat stress.',
    },
    heatAdj: 2.0,
    layer: 'lhoodie',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'jeans',
    name: 'Tight dark jeans',
    icon: '👖',
    modes: { hot: 'bad', mild: 'bad' },
    effect: {
      hot: 'Dark tight fabric traps heat around the legs — very uncomfortable in the sun',
      mild: 'Tight dark jeans absorb more heat than needed for today\'s temperature',
    },
    explanation: {
      hot: 'Denim is heavy and non-breathable; dark colour absorbs sunlight significantly.',
      mild: 'Looser or lighter fabric will be more comfortable and cooler.',
    },
    heatAdj: 1.2,
    layer: 'ljeans',
    uvOnly: false,
    slot: 'bottom',
  },
  {
    id: 'shorts',
    name: 'Loose light shorts',
    icon: '🩳',
    modes: { hot: 'good', mild: 'good' },
    effect: {
      hot: 'Good airflow around the legs — pair with sunscreen for exposed skin',
      mild: 'Comfortable in mild weather if you prefer them',
    },
    explanation: {},
    heatAdj: -0.5,
    layer: 'lshorts',
    uvOnly: false,
    slot: 'bottom',
  },
  {
    id: 'runners',
    name: 'Breathable runners',
    icon: '👟',
    modes: { cool: 'good', mild: 'good', hot: 'good' },
    effect: {
      cool: 'Supportive and comfortable for walking in cool conditions',
      mild: 'Good support and breathability for a mild day walk',
      hot: 'Breathable mesh keeps feet cooler than heavy footwear',
    },
    explanation: {},
    heatAdj: -0.3,
    layer: 'lrunners',
    uvOnly: false,
    slot: 'footwear',
  },
  {
    id: 'sandals',
    name: 'Supportive sandals',
    icon: '🥿',
    modes: { hot: 'good', mild: 'good' },
    effect: {
      hot: 'Open footwear keeps feet cooler — choose ones with arch support',
      mild: 'Comfortable and breathable for shorter trips in mild weather',
    },
    explanation: {},
    heatAdj: -0.4,
    layer: 'lsandals',
    uvOnly: false,
    slot: 'footwear',
  },
  {
    id: 'vest',
    name: 'Light puffer vest',
    icon: '🦺',
    modes: { cool: 'good', mild: 'good' },
    effect: {
      cool: 'Adds core warmth without restricting arm movement',
      mild: 'Lightweight layer for changeable weather — easy to carry',
    },
    explanation: {},
    heatAdj: -0.6,
    layer: 'lvest',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'linen',
    name: 'Linen trousers',
    icon: '👒',
    modes: { hot: 'good', mild: 'good' },
    effect: {
      hot: 'Loose linen breathes well and protects legs from direct sun',
      mild: 'Comfortable coverage without heat penalty in mild conditions',
    },
    explanation: {},
    heatAdj: -0.7,
    layer: 'llinen',
    uvOnly: false,
    slot: 'bottom',
  },
  {
    id: 'coolingpatch',
    name: 'Cooling neck patch',
    icon: '🧊',
    modes: { hot: 'good' },
    effect: {
      hot: 'Cooling patch on the neck reduces perceived heat significantly during outdoor activity',
    },
    explanation: {},
    heatAdj: -1.0,
    layer: 'lcoolingpatch',
    uvOnly: false,
    slot: null,
  },
  {
    id: 'gloves',
    name: 'Light gloves',
    icon: '🧤',
    modes: { cool: 'good' },
    effect: {
      cool: 'Protects hands from cold — important for older adults who feel the cold more',
    },
    explanation: {},
    heatAdj: -0.5,
    layer: 'lgloves',
    uvOnly: false,
    slot: null,
  },
  {
    id: 'thermals',
    name: 'Thermal base layer',
    icon: '🥼',
    modes: { cool: 'good' },
    effect: {
      cool: 'Traps body heat close to the skin — effective base layer in cold conditions',
    },
    explanation: {},
    heatAdj: -1.0,
    layer: 'lthermals',
    uvOnly: false,
    slot: 'top',
  },
  {
    id: 'raincoat',
    name: 'Light raincoat',
    icon: '🌂',
    modes: { cool: 'good', mild: 'good' },
    effect: {
      cool: 'Wind and rain protection without too much bulk in cool weather',
      mild: 'Useful for unexpected showers — choose breathable fabric',
    },
    explanation: {},
    heatAdj: 0.3,
    layer: 'lraincoat',
    uvOnly: false,
    slot: 'top',
  },
]

// ── Dynamic item lists based on climate mode + UV ─────────────────────────────
const activeItems = computed(() => {
  const mode = climateMode.value
  return ALL_ITEMS.filter((item) => {
    if (!item.modes[mode]) return false
    if (item.uvOnly && !uvHigh.value) return false
    return true
  }).map((item) => ({
    ...item,
    category: item.modes[climateMode.value],
    effect: item.effect[climateMode.value] ?? '',
    explanation: item.explanation[climateMode.value] ?? null,
  }))
})

// ── Grouped items for all-items view ─────────────────────────────────────────
const GROUPS = [
  { key: 'head',      label: 'Head & Sun Protection', ids: ['hat', 'cap', 'sg', 'sc'] },
  { key: 'top',       label: 'Top',                   ids: ['shirt', 'longsleeve', 'vest', 'warmjacket', 'jacket', 'hoodie', 'dshirt'] },
  { key: 'bottom',    label: 'Bottom',                ids: ['pants', 'linen', 'shorts', 'jeans'] },
  { key: 'footwear',  label: 'Footwear',              ids: ['runners', 'sandals', 'flipflops'] },
  { key: 'accessory', label: 'Accessories',           ids: ['water', 'umbrella', 'coolingpatch', 'scarf', 'gloves', 'thermals', 'raincoat'] },
]

const groupedItems = computed(() => {
  const mode = climateMode.value
  return GROUPS.map((group) => ({
    ...group,
    items: group.ids
      .map((id) => {
        const raw = ALL_ITEMS.find((i) => i.id === id)
        if (!raw) return null
        if (raw.uvOnly && !uvHigh.value) return null
        const cat = raw.modes[mode]
        const fallbackMode = Object.keys(raw.modes)[0]
        const displayMode = cat ? mode : fallbackMode
        return {
          ...raw,
          category: cat ?? raw.modes[fallbackMode],
          effect: raw.effect[displayMode] ?? Object.values(raw.effect)[0],
          explanation: raw.explanation[displayMode] ?? null,
          notForToday: !cat,
        }
      })
      .filter(Boolean),
  })).filter((g) => g.items.length > 0)
})

// ── Group expand state (show more options per group) ─────────────────────────
const expandedGroups = ref(new Set())

function toggleGroupExpand(groupKey) {
  const next = new Set(expandedGroups.value)
  if (next.has(groupKey)) {
    next.delete(groupKey)
  } else {
    next.add(groupKey)
  }
  expandedGroups.value = next
}

function isGroupExpanded(groupKey) {
  return expandedGroups.value.has(groupKey)
}

// ── Selection state (AC 4.2.3) ───────────────────────────────────────────────
const selectedItems = ref(new Set())

function toggleItem(id) {
  const next = new Set(selectedItems.value)
  if (next.has(id)) {
    next.delete(id)
  } else {
    const item = ALL_ITEMS.find((i) => i.id === id)
    if (item?.slot) {
      ALL_ITEMS.forEach((other) => {
        if (other.slot === item.slot && other.id !== id) next.delete(other.id)
      })
    }
    next.add(id)
  }
  selectedItems.value = next
}

function isSelected(id) {
  return selectedItems.value.has(id)
}

// ── Heat exposure score (AC 4.2.2) ──────────────────────────────────────────
const exposureTemp = computed(() => {
  const base = selectedSuburb.value?.apparent_temperature ?? null
  if (base === null) return null
  let adj = 0
  selectedItems.value.forEach((id) => {
    const item = ALL_ITEMS.find((i) => i.id === id)
    if (item) adj += item.heatAdj
  })
  return Math.round((base + adj) * 10) / 10
})

const exposureBarPct = computed(() => {
  if (exposureTemp.value === null) return 0
  return Math.max(3, Math.min(97, ((exposureTemp.value - 10) / (45 - 10)) * 100))
})

const exposureConfig = computed(() => {
  const t = exposureTemp.value
  const mode = climateMode.value
  if (t === null) return { color: '#9e9890', label: 'Select a suburb to begin', pulse: false }
  if (mode === 'cool') {
    if (t < 14)
      return { color: '#185FA5', label: 'Cold — dress warmly before heading out', pulse: false }
    if (t < 18)
      return { color: '#4d9e5a', label: 'Cool — comfortable with the right layers', pulse: false }
  }
  if (t < 31) return { color: '#4d9e5a', label: 'Low — good outfit for today', pulse: false }
  if (t < 34) return { color: '#e8903a', label: 'Moderate — take care outside', pulse: false }
  if (t < 37) return { color: '#c0392b', label: 'High — limit time outdoors', pulse: false }
  return { color: '#8b1a12', label: 'Danger — seek cool shelter now', pulse: true }
})

// ── Personalised advice (climate + UV aware) ──────────────────────────────────
const adviceItems = computed(() => {
  const s = selectedItems.value
  const mode = climateMode.value
  const uv = uvHigh.value

  if (!s.size) {
    return [{ color: '#9e9890', text: 'Select clothing items above to see personalised advice.' }]
  }

  const msgs = []

  if (mode === 'cool') {
    if (s.has('warmjacket'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Good choice — a warm jacket helps maintain body temperature in cool conditions.',
      })
    if (s.has('shirt') && !s.has('warmjacket'))
      msgs.push({
        color: '#c0392b',
        text: "A light cotton shirt alone won't keep you warm today — consider adding a warm layer on top.",
      })
    if (s.has('pants'))
      msgs.push({
        color: '#c0392b',
        text: 'Light pants may not be warm enough today. Thicker trousers will be more comfortable.',
      })
    if (s.has('hat'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Good idea — a hat helps retain heat in cool conditions and still provides some UV cover.',
      })
    if (s.has('water'))
      msgs.push({
        color: '#185FA5',
        text: "Good habit — it's easy to forget hydration in cool weather, but it's still important.",
      })
  } else if (mode === 'mild') {
    if (s.has('jacket'))
      msgs.push({
        color: '#e8903a',
        text: "A heavy jacket is too warm for today — you may overheat, especially if you're active.",
      })
    if (s.has('dshirt'))
      msgs.push({
        color: '#e8903a',
        text: 'Tight dark shirts absorb heat and reduce airflow — a lighter top would be more comfortable today.',
      })
    if (s.has('hat'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Wide-brim hat is a good choice — even on mild days, sun protection matters.',
      })
    if (s.has('shirt'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Light cotton is comfortable and breathable on a mild day.',
      })
    if (s.has('water'))
      msgs.push({
        color: '#185FA5',
        text: "Good habit — staying hydrated is important even when it doesn't feel hot.",
      })
  } else {
    if (s.has('jacket'))
      msgs.push({
        color: '#c0392b',
        text: 'Dark heavy jacket is dangerous today — it traps heat and significantly raises your body temperature. Remove it.',
      })
    if (s.has('dshirt'))
      msgs.push({
        color: '#c0392b',
        text: "Tight dark shirt absorbs heat and blocks airflow. This is unsafe in today's heat.",
      })
    if (s.has('hat'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Wide-brim hat is essential in this heat — it blocks UV and reduces heat stress on your head and neck.',
      })
    if (s.has('shirt'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Light cotton shirt is a great choice — it breathes well and reflects sunlight.',
      })
    if (s.has('pants'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Loose light pants keep your legs cool and protected from UV at the same time.',
      })
    if (s.has('water'))
      msgs.push({
        color: '#185FA5',
        text: 'Carry water and drink every 15–20 minutes when outside. Dehydration is a serious risk today.',
      })
  }

  if (uv) {
    if (s.has('sg'))
      msgs.push({
        color: '#2d7a3a',
        text: 'Sunglasses are important today — UV is high enough to cause eye strain and long-term damage.',
      })
    if (s.has('sc'))
      msgs.push({
        color: '#2d7a3a',
        text: 'SPF 50+ sunscreen is the right call. Apply 20 minutes before going out and reapply every 2 hours.',
      })
    if (!s.has('sg') && !s.has('sc') && mode !== 'cool')
      msgs.push({
        color: '#e8903a',
        text: 'UV is high today — add sunglasses and sunscreen before heading out.',
      })
  }

  if (!s.has('water') && mode !== 'cool')
    msgs.push({
      color: '#e8903a',
      text: "Don't forget water — dehydration is easy to miss until it's too late, especially for older adults.",
    })

  return msgs
})
</script>

<template>
  <div class="page">
    <NavBar />

    <div class="content">
      <!-- Page header -->
      <div class="page-header card">
        <h1 class="page-title">Outfit <span class="page-title-accent">Advisor</span></h1>
        <p class="page-desc">
          See what to wear based on today's heat and UV conditions.
          <strong>Select clothing items</strong> to check your personal heat exposure.
        </p>
      </div>

      <div v-if="loading" class="status-msg">Loading suburb data...</div>
      <div v-else-if="error" class="status-msg status-msg--error">{{ error }}</div>

      <template v-else>
        <!-- Suburb selector (AC 4.1.3) -->
        <div class="selector-row card">
          <label for="suburb-select" class="selector-label">
            <svg
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2.5"
              stroke-linecap="round"
              stroke-linejoin="round"
              aria-hidden="true"
            >
              <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z" />
              <circle cx="12" cy="10" r="3" />
            </svg>
            Suburb
          </label>
          <select id="suburb-select" v-model="selectedSuburbId" class="suburb-select">
            <option v-for="s in allSuburbs" :key="s.suburb_id" :value="String(s.suburb_id)">
              {{ s.suburb_name }}
            </option>
          </select>
        </div>

        <!-- AC 4.1.2 — Weather summary card -->
        <OutfitWeatherCard :suburb="selectedSuburb" />

        <!-- Climate mode indicator -->
        <div class="mode-banner" :class="`mode-banner--${climateMode}`">
          <span v-if="climateMode === 'cool'"
            >🧣 Cool day — recommendations are adjusted for lower temperatures</span
          >
          <span v-else-if="climateMode === 'mild'"
            >🌤 Mild day — balanced recommendations for comfortable conditions</span
          >
          <span v-else>☀️ Hot day — heat safety recommendations are active</span>
          <span v-if="!uvHigh" class="uv-note">
            | UV is low — sunglasses and sunscreen not shown</span
          >
        </div>

        <!-- Main grid: mannequin + item list -->
        <!-- How to use — full width above main grid -->
        <div class="guide-card card">
          <p class="guide-title">How to use</p>
          <ul class="guide-list guide-list--row">
            <li>Tap any item to add it to your outfit</li>
            <li>The mannequin updates as you select</li>
            <li>Green items lower your heat exposure</li>
            <li>Red items raise your heat risk</li>
            <li>Your score updates in real time</li>
          </ul>
        </div>

        <div class="main-grid">
          <!-- Left column: mannequin only -->
          <div class="left-col">
            <!-- Mannequin + score (AC 4.2.1 + 4.2.2) -->
            <div class="mannequin-col card">
              <p class="col-label">Outfit preview</p>

              <svg
                viewBox="0 0 160 282"
                xmlns="http://www.w3.org/2000/svg"
                class="mannequin-svg"
                aria-label="Outfit mannequin preview"
                role="img"
              >
                <!-- Base body -->
                <rect x="54" y="154" width="24" height="84" rx="9" fill="#F5C8A8" />
                <rect x="82" y="154" width="24" height="84" rx="9" fill="#F5C8A8" />
                <ellipse cx="66" cy="242" rx="16" ry="7" fill="#E0A882" />
                <ellipse cx="94" cy="242" rx="16" ry="7" fill="#E0A882" />
                <rect x="27" y="90" width="22" height="63" rx="9" fill="#F5C8A8" />
                <rect x="111" y="90" width="22" height="63" rx="9" fill="#F5C8A8" />
                <rect x="50" y="87" width="60" height="72" rx="9" fill="#F5C8A8" />
                <rect x="73" y="74" width="14" height="16" fill="#F5C8A8" />
                <circle cx="80" cy="54" r="27" fill="#F5C8A8" />
                <circle cx="72" cy="50" r="3.5" fill="#2d7a3a" />
                <circle cx="88" cy="50" r="3.5" fill="#2d7a3a" />
                <circle cx="73.5" cy="48.5" r="1.3" fill="white" />
                <circle cx="89.5" cy="48.5" r="1.3" fill="white" />
                <ellipse cx="80" cy="57" rx="2" ry="2.5" fill="#E0A882" />
                <path
                  d="M73 64 Q80 70 87 64"
                  stroke="#C48060"
                  stroke-width="2"
                  fill="none"
                  stroke-linecap="round"
                />
                <!-- Pants -->
                <g :style="{ display: isSelected('pants') ? '' : 'none' }">
                  <rect x="48" y="152" width="64" height="9" rx="3" fill="#7FA4BE" />
                  <rect x="48" y="158" width="30" height="80" rx="9" fill="#A8C4D8" />
                  <rect x="82" y="158" width="30" height="80" rx="9" fill="#A8C4D8" />
                </g>
                <!-- Light shirt -->
                <g
                  :style="{
                    display:
                      isSelected('shirt') && !isSelected('jacket') && !isSelected('dshirt')
                        ? ''
                        : 'none',
                  }"
                >
                  <rect x="25" y="88" width="25" height="65" rx="9" fill="#9FC4E8" />
                  <rect x="110" y="88" width="25" height="65" rx="9" fill="#9FC4E8" />
                  <rect x="48" y="85" width="64" height="74" rx="9" fill="#9FC4E8" />
                  <polygon points="80,85 68,106 92,106" fill="#7DAED8" />
                  <line x1="80" y1="106" x2="80" y2="158" stroke="#7DAED8" stroke-width="2" />
                </g>
                <!-- Jacket (shared layer: dark heavy jacket in hot/mild, warm jacket in cool) -->
                <g
                  :style="{
                    display: isSelected('jacket') || isSelected('warmjacket') ? '' : 'none',
                  }"
                >
                  <rect x="21" y="85" width="30" height="70" rx="10" fill="#2C2C2A" />
                  <rect x="109" y="85" width="30" height="70" rx="10" fill="#2C2C2A" />
                  <rect x="44" y="82" width="72" height="80" rx="10" fill="#2C2C2A" />
                  <polygon points="80,82 62,110 80,118" fill="#3A3A38" />
                  <polygon points="80,82 98,110 80,118" fill="#3A3A38" />
                  <line x1="80" y1="118" x2="80" y2="162" stroke="#3A3A38" stroke-width="2.5" />
                  <rect x="68" y="78" width="24" height="10" rx="5" fill="#3A3A38" />
                </g>
                <!-- Dark shirt (bad) -->
                <g
                  :style="{ display: isSelected('dshirt') && !isSelected('jacket') ? '' : 'none' }"
                >
                  <rect x="25" y="88" width="25" height="65" rx="9" fill="#2A2A2A" />
                  <rect x="110" y="88" width="25" height="65" rx="9" fill="#2A2A2A" />
                  <rect x="48" y="85" width="64" height="74" rx="9" fill="#2A2A2A" />
                  <polygon points="80,85 68,106 92,106" fill="#222" />
                </g>
                <!-- Hat -->
                <g :style="{ display: isSelected('hat') ? '' : 'none' }">
                  <ellipse cx="80" cy="31" rx="44" ry="9" fill="#C4A866" />
                  <rect x="59" y="9" width="42" height="28" rx="6" fill="#D4B886" />
                  <rect x="59" y="32" width="42" height="6" fill="#8B6914" />
                  <ellipse cx="80" cy="31" rx="44" ry="5" fill="#A08030" opacity=".3" />
                </g>
                <!-- Sunglasses -->
                <g :style="{ display: isSelected('sg') ? '' : 'none' }">
                  <rect x="64" y="45" width="15" height="11" rx="5" fill="#1a1a2e" opacity=".9" />
                  <rect x="81" y="45" width="15" height="11" rx="5" fill="#1a1a2e" opacity=".9" />
                  <line x1="79" y1="50" x2="81" y2="50" stroke="#555" stroke-width="1.5" />
                  <line x1="64" y1="50" x2="54" y2="53" stroke="#555" stroke-width="1.5" />
                  <line x1="96" y1="50" x2="106" y2="53" stroke="#555" stroke-width="1.5" />
                  <rect x="67" y="47" width="5" height="3" rx="2" fill="white" opacity=".22" />
                  <rect x="84" y="47" width="5" height="3" rx="2" fill="white" opacity=".22" />
                </g>
                <!-- Water bottle -->
                <g :style="{ display: isSelected('water') ? '' : 'none' }">
                  <rect x="116" y="120" width="13" height="33" rx="5" fill="#4d9e5a" />
                  <rect x="117" y="121" width="5" height="27" rx="3" fill="#7dc486" opacity=".4" />
                  <rect x="117" y="113" width="11" height="9" rx="3" fill="#2d7a3a" />
                  <line
                    x1="121"
                    y1="113"
                    x2="121"
                    y2="102"
                    stroke="#2d7a3a"
                    stroke-width="2.5"
                    stroke-linecap="round"
                  />
                  <rect x="118" y="132" width="9" height="2" rx="1" fill="white" opacity=".35" />
                </g>
                <!-- Sunscreen -->
                <g :style="{ display: isSelected('sc') ? '' : 'none' }">
                  <rect x="22" y="122" width="13" height="28" rx="4" fill="#FAC775" />
                  <rect x="22" y="113" width="13" height="11" rx="3" fill="#EF9F27" />
                  <rect x="27" y="106" width="4" height="9" rx="2" fill="#BA7517" />
                  <text
                    x="28.5"
                    y="137"
                    text-anchor="middle"
                    font-size="5.5"
                    fill="#6B4800"
                    font-weight="bold"
                    font-family="sans-serif"
                  >
                    SPF
                  </text>
                  <rect x="24" y="128" width="9" height="2" rx="1" fill="white" opacity=".35" />
                </g>
                <!-- Cap -->
                <g :style="{ display: isSelected('cap') ? '' : 'none' }">
                  <ellipse cx="80" cy="32" rx="30" ry="7" fill="#e05a2b" />
                  <rect x="59" y="20" width="42" height="18" rx="4" fill="#f07040" />
                  <rect x="82" y="25" width="22" height="5" rx="2" fill="#c04020" />
                  <rect x="59" y="32" width="42" height="4" fill="#c04020" />
                </g>
                <!-- Scarf -->
                <g :style="{ display: isSelected('scarf') ? '' : 'none' }">
                  <rect x="62" y="72" width="36" height="14" rx="6" fill="#c0392b" />
                  <rect x="72" y="82" width="10" height="22" rx="4" fill="#c0392b" />
                  <line x1="65" y1="76" x2="95" y2="76" stroke="#a93226" stroke-width="1.5" />
                  <line x1="65" y1="80" x2="95" y2="80" stroke="#a93226" stroke-width="1.5" />
                </g>
                <!-- Long sleeve shirt -->
                <g :style="{ display: isSelected('longsleeve') && !isSelected('jacket') && !isSelected('warmjacket') && !isSelected('hoodie') && !isSelected('raincoat') ? '' : 'none' }">
                  <rect x="25" y="88" width="25" height="65" rx="9" fill="#7cb9a8" />
                  <rect x="110" y="88" width="25" height="65" rx="9" fill="#7cb9a8" />
                  <rect x="48" y="85" width="64" height="74" rx="9" fill="#7cb9a8" />
                  <polygon points="80,85 68,106 92,106" fill="#5a9a8a" />
                  <line x1="80" y1="106" x2="80" y2="158" stroke="#5a9a8a" stroke-width="2" />
                </g>
                <!-- Vest -->
                <g :style="{ display: isSelected('vest') && !isSelected('jacket') && !isSelected('warmjacket') && !isSelected('hoodie') && !isSelected('raincoat') ? '' : 'none' }">
                  <rect x="50" y="85" width="60" height="74" rx="9" fill="#8B6914" />
                  <polygon points="80,85 68,106 92,106" fill="#6B4F10" />
                  <line x1="80" y1="106" x2="80" y2="158" stroke="#6B4F10" stroke-width="2" />
                  <rect x="51" y="86" width="6" height="72" rx="3" fill="#A07820" opacity=".4" />
                </g>
                <!-- Thermals -->
                <g :style="{ display: isSelected('thermals') && !isSelected('jacket') && !isSelected('warmjacket') && !isSelected('hoodie') && !isSelected('raincoat') && !isSelected('shirt') && !isSelected('longsleeve') && !isSelected('vest') && !isSelected('dshirt') ? '' : 'none' }">
                  <rect x="30" y="90" width="20" height="60" rx="8" fill="#dce8f5" />
                  <rect x="110" y="90" width="20" height="60" rx="8" fill="#dce8f5" />
                  <rect x="50" y="87" width="60" height="72" rx="8" fill="#dce8f5" />
                  <line x1="80" y1="90" x2="80" y2="158" stroke="#b0c8e0" stroke-width="1.5" />
                </g>
                <!-- Hoodie -->
                <g :style="{ display: isSelected('hoodie') && !isSelected('jacket') && !isSelected('warmjacket') ? '' : 'none' }">
                  <rect x="21" y="85" width="30" height="70" rx="10" fill="#555" />
                  <rect x="109" y="85" width="30" height="70" rx="10" fill="#555" />
                  <rect x="44" y="82" width="72" height="80" rx="10" fill="#555" />
                  <ellipse cx="80" cy="84" rx="16" ry="10" fill="#444" />
                  <rect x="72" y="78" width="16" height="12" rx="5" fill="#444" />
                  <rect x="68" y="118" width="24" height="14" rx="7" fill="#4a4a4a" />
                  <line x1="80" y1="100" x2="80" y2="160" stroke="#444" stroke-width="2" />
                </g>
                <!-- Raincoat -->
                <g :style="{ display: isSelected('raincoat') && !isSelected('jacket') && !isSelected('warmjacket') ? '' : 'none' }">
                  <rect x="21" y="85" width="30" height="70" rx="10" fill="#e8c022" />
                  <rect x="109" y="85" width="30" height="70" rx="10" fill="#e8c022" />
                  <rect x="44" y="82" width="72" height="80" rx="10" fill="#e8c022" />
                  <polygon points="80,82 62,110 80,118" fill="#c8a018" />
                  <polygon points="80,82 98,110 80,118" fill="#c8a018" />
                  <line x1="80" y1="118" x2="80" y2="162" stroke="#c8a018" stroke-width="2.5" />
                </g>
                <!-- Shorts -->
                <g :style="{ display: isSelected('shorts') ? '' : 'none' }">
                  <rect x="48" y="152" width="64" height="9" rx="3" fill="#4a7fc1" />
                  <rect x="48" y="158" width="30" height="44" rx="9" fill="#5a8fd1" />
                  <rect x="82" y="158" width="30" height="44" rx="9" fill="#5a8fd1" />
                  <line x1="80" y1="158" x2="80" y2="200" stroke="#4a7fc1" stroke-width="1.5" />
                </g>
                <!-- Jeans -->
                <g :style="{ display: isSelected('jeans') ? '' : 'none' }">
                  <rect x="48" y="152" width="64" height="9" rx="3" fill="#2c3e6b" />
                  <rect x="48" y="158" width="30" height="80" rx="9" fill="#2c4a8b" />
                  <rect x="82" y="158" width="30" height="80" rx="9" fill="#2c4a8b" />
                  <line x1="61" y1="165" x2="61" y2="235" stroke="#1e3060" stroke-width="1.5" stroke-dasharray="3,3" />
                  <line x1="99" y1="165" x2="99" y2="235" stroke="#1e3060" stroke-width="1.5" stroke-dasharray="3,3" />
                </g>
                <!-- Linen trousers -->
                <g :style="{ display: isSelected('linen') ? '' : 'none' }">
                  <rect x="48" y="152" width="64" height="9" rx="3" fill="#c8b89a" />
                  <rect x="48" y="158" width="30" height="80" rx="9" fill="#d8c8aa" />
                  <rect x="82" y="158" width="30" height="80" rx="9" fill="#d8c8aa" />
                  <line x1="80" y1="160" x2="80" y2="235" stroke="#b8a88a" stroke-width="1.5" />
                </g>
                <!-- Runners -->
                <g :style="{ display: isSelected('runners') ? '' : 'none' }">
                  <rect x="46" y="232" width="32" height="12" rx="5" fill="#e8e8e8" />
                  <rect x="46" y="235" width="32" height="6" rx="3" fill="#4d9e5a" />
                  <rect x="80" y="232" width="32" height="12" rx="5" fill="#e8e8e8" />
                  <rect x="80" y="235" width="32" height="6" rx="3" fill="#4d9e5a" />
                  <line x1="52" y1="234" x2="72" y2="234" stroke="#aaa" stroke-width="1" />
                  <line x1="86" y1="234" x2="106" y2="234" stroke="#aaa" stroke-width="1" />
                </g>
                <!-- Sandals -->
                <g :style="{ display: isSelected('sandals') ? '' : 'none' }">
                  <rect x="46" y="238" width="32" height="6" rx="3" fill="#c8a060" />
                  <rect x="52" y="232" width="20" height="4" rx="2" fill="#a07840" />
                  <rect x="80" y="238" width="32" height="6" rx="3" fill="#c8a060" />
                  <rect x="86" y="232" width="20" height="4" rx="2" fill="#a07840" />
                </g>
                <!-- Flip flops -->
                <g :style="{ display: isSelected('flipflops') ? '' : 'none' }">
                  <rect x="44" y="238" width="34" height="6" rx="3" fill="#e05a2b" />
                  <path d="M58 238 Q62 230 66 238" stroke="#c04020" stroke-width="2" fill="none" />
                  <rect x="78" y="238" width="34" height="6" rx="3" fill="#e05a2b" />
                  <path d="M92 238 Q96 230 100 238" stroke="#c04020" stroke-width="2" fill="none" />
                </g>
                <!-- Gloves -->
                <g :style="{ display: isSelected('gloves') ? '' : 'none' }">
                  <ellipse cx="36" cy="148" rx="12" ry="10" fill="#8B4513" />
                  <ellipse cx="124" cy="148" rx="12" ry="10" fill="#8B4513" />
                  <rect x="26" y="140" width="20" height="12" rx="5" fill="#9B5523" />
                  <rect x="114" y="140" width="20" height="12" rx="5" fill="#9B5523" />
                </g>
                <!-- Umbrella -->
                <g :style="{ display: isSelected('umbrella') ? '' : 'none' }">
                  <line x1="130" y1="70" x2="130" y2="170" stroke="#555" stroke-width="2.5" stroke-linecap="round" />
                  <path d="M105 70 Q130 40 155 70" fill="#e05a2b" opacity=".9" />
                  <path d="M105 70 Q117 60 130 70" fill="#c04020" opacity=".5" />
                  <path d="M130 70 Q143 60 155 70" fill="#c04020" opacity=".5" />
                  <path d="M128 168 Q128 175 122 175" stroke="#555" stroke-width="2" fill="none" stroke-linecap="round" />
                </g>
                <!-- Cooling patch -->
                <g :style="{ display: isSelected('coolingpatch') ? '' : 'none' }">
                  <rect x="68" y="76" width="24" height="10" rx="4" fill="#a0d8ef" />
                  <rect x="70" y="78" width="20" height="6" rx="3" fill="#70b8df" opacity=".7" />
                  <text x="80" y="84" text-anchor="middle" font-size="4" fill="#2060a0" font-family="sans-serif" font-weight="bold">COOL</text>
                </g>
              </svg>

              <!-- Heat exposure score display -->
              <div class="score-area">
                <div class="score-row">
                  <span class="score-lbl">Heat exposure</span>
                  <span class="score-val" :style="{ color: exposureConfig.color }">
                    {{ exposureTemp !== null ? exposureTemp.toFixed(1) : '—' }}°C
                  </span>
                </div>
                <div class="score-bar-bg">
                  <div
                    class="score-bar-fill"
                    :class="{ 'score-bar-fill--pulse': exposureConfig.pulse }"
                    :style="{
                      width: exposureBarPct + '%',
                      backgroundColor: exposureConfig.color,
                    }"
                  ></div>
                </div>
                <p class="score-status" :style="{ color: exposureConfig.color }">
                  {{ exposureConfig.label }}
                </p>
              </div>
            </div>

            <!-- Personalised heat advice — sticky with mannequin -->
            <div class="advice-card card">
              <p class="advice-title">Personalised advice</p>
              <ul class="advice-list">
                <li v-for="(msg, i) in adviceItems" :key="i" class="advice-item">
                  <span class="advice-dot" :style="{ backgroundColor: msg.color }"></span>
                  <span class="advice-text">{{ msg.text }}</span>
                </li>
              </ul>
            </div>
          </div>
          <!-- end left-col -->

          <!-- Item list — grouped by category -->
          <div class="items-col">
            <!-- Based on banner -->
            <div class="basis-row">
              <p class="recommendation-basis">
                Based on
                <span class="basis-tag">{{ climateMode }} weather</span>
                <span v-if="uvHigh"> · <span class="basis-tag">UV {{ Math.round(selectedSuburb?.uv_index ?? 0) }}</span></span>
                <span v-else> · <span class="basis-tag basis-tag--muted">UV not a factor</span></span>
              </p>
            </div>

            <div v-for="group in groupedItems" :key="group.key" class="item-group">
              <p class="section-label section-label--group">{{ group.label }}</p>
              <!-- Recommended items for today (good + not notForToday only) -->
              <div class="items-list">
                <OutfitItemCard
                  v-for="item in group.items.filter(i => !i.notForToday && i.category === 'good')"
                  :key="item.id"
                  :item="item"
                  :selected="isSelected(item.id)"
                  @toggle="toggleItem"
                />
              </div>
              <!-- Show more options: bad (avoid) + notForToday items, collapsed by default -->
              <template v-if="group.items.some(i => i.notForToday || i.category === 'bad')">
                <button
                  class="show-more-btn"
                  @click="toggleGroupExpand(group.key)"
                  :aria-expanded="isGroupExpanded(group.key)"
                >
                  <span v-if="!isGroupExpanded(group.key)">
                    Show more options ({{ group.items.filter(i => i.notForToday || i.category === 'bad').length }})
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><polyline points="6 9 12 15 18 9"/></svg>
                  </span>
                  <span v-else>
                    Hide other options
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><polyline points="18 15 12 9 6 15"/></svg>
                  </span>
                </button>
                <div v-if="isGroupExpanded(group.key)" class="items-list items-list--more">
                  <OutfitItemCard
                    v-for="item in group.items.filter(i => i.notForToday || i.category === 'bad')"
                    :key="item.id"
                    :item="item"
                    :selected="isSelected(item.id)"
                    @toggle="toggleItem"
                  />
                </div>
              </template>
            </div>
          </div>
        </div>

        <!-- Full-width navigation row -->
        <div class="nav-links">
          <RouterLink :to="{ path: '/heatmap', query: { suburbId: selectedSuburbId } }" class="nav-link-btn">
            <svg
              width="14"
              height="14"
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
            Check heat map
          </RouterLink>
          <RouterLink :to="{ path: '/trip-coach', query: { suburbId: selectedSuburbId } }" class="nav-link-btn">
            <svg
              width="14"
              height="14"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2.5"
              stroke-linecap="round"
              stroke-linejoin="round"
              aria-hidden="true"
            >
              <circle cx="12" cy="12" r="10" />
              <polyline points="12 6 12 12 16 14" />
            </svg>
            Plan a trip
          </RouterLink>
        </div>
      </template>
    </div>

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
  display: block;
}

.content > * + * {
  margin-top: 1rem;
}

/* main-grid needs block context for sticky to work */
.main-grid-wrap {
  display: block;
}

.card {
  background: #ffffff;
  border-radius: 14px;
  box-shadow:
    0 2px 8px rgba(0, 0, 0, 0.06),
    0 8px 24px rgba(0, 0, 0, 0.06);
  border: 1px solid rgba(0, 0, 0, 0.05);
}

/* Page header */
.page-header {
  padding: 1.75rem 1.6rem;
  background: linear-gradient(135deg, rgba(13, 58, 143, 0.95), rgba(11, 127, 121, 0.88));
  border: none;
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

/* Suburb selector */
.selector-row {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.65rem 1rem;
}

.selector-label {
  display: flex;
  align-items: center;
  gap: 5px;
  font-size: 0.95rem;
  font-weight: 600;
  color: #6b6560;
  white-space: nowrap;
  flex-shrink: 0;
}

.suburb-select {
  flex: 1;
  border: 1px solid #d8eae6;
  border-radius: 8px;
  padding: 6px 10px;
  font-size: 15px;
  background: #f4faf8;
  color: #1a1714;
  cursor: pointer;
  font-family: inherit;
}

.suburb-select:focus {
  outline: 2px solid #4d9e5a;
  outline-offset: 1px;
}

/* Climate mode banner */
.mode-banner {
  padding: 0.65rem 1rem;
  border-radius: 10px;
  font-size: 14px;
  font-weight: 600;
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 4px;
}

.mode-banner--hot {
  background: #fce8e6;
  color: #8b1a12;
}

.mode-banner--mild {
  background: #fef1e0;
  color: #7a4510;
}

.mode-banner--cool {
  background: #e6f0fb;
  color: #0c447c;
}

.uv-note {
  font-weight: 600;
}

/* Main grid */
.main-grid {
  display: grid;
  grid-template-columns: 240px minmax(0, 1fr);
  gap: 1rem;
  align-items: start;
  align-self: stretch;
}

/* Left column — sticky so mannequin stays visible while scrolling items */
.left-col {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  position: sticky;
  top: 80px;
  align-self: start;
  max-height: calc(100vh - 100px);
  overflow-y: auto;
  scrollbar-width: none;
}

.left-col::-webkit-scrollbar {
  display: none;
}

/* Mannequin column */
.mannequin-col {
  padding: 0.875rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
  flex-grow: 1;
}

/* Guide card — full width horizontal strip */
.guide-card {
  padding: 0.875rem 1.25rem;
}

.guide-title {
  font-size: 14px;
  font-weight: 700;
  color: #2d7a3a;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  margin: 0 0 8px;
}

.guide-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.guide-list--row {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 6px 16px;
}

.guide-list li {
  font-size: 15px;
  color: #6b6560;
  line-height: 1.5;
  padding-left: 13px;
  position: relative;
}

.guide-list li::before {
  content: '';
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: #4d9e5a;
  position: absolute;
  left: 0;
  top: 6px;
}

.col-label {
  font-size: 14px;
  font-weight: 700;
  color: #9e9890;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin: 0;
}

.mannequin-svg {
  width: 100%;
  max-width: 190px;
}

/* Score display */
.score-area {
  width: 100%;
  margin-top: 8px;
  padding: 14px 16px 12px;
  border-radius: 12px;
  background: #f4faf8;
  border: 1px solid #d8eae6;
}

.score-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 8px;
}

.score-lbl {
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: #9e9890;
}

.score-val {
  font-size: 2rem;
  font-weight: 800;
  letter-spacing: -0.03em;
  line-height: 1;
  transition: color 0.4s;
}

.score-bar-bg {
  height: 10px;
  background: #dceee9;
  border-radius: 6px;
  overflow: hidden;
  margin: 6px 0 10px;
}

.score-bar-fill {
  height: 100%;
  border-radius: 6px;
  transition:
    width 0.5s ease,
    background-color 0.4s;
}

@keyframes bar-pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

.score-bar-fill--pulse {
  animation: bar-pulse 1.1s infinite;
}

.score-status {
  font-size: 15px;
  font-weight: 700;
  line-height: 1.4;
  margin: 0;
  transition: color 0.4s;
}

/* Items column */
.items-col {
  display: flex;
  flex-direction: column;
  gap: 6px;
}


/* Based on row */
.basis-row {
  margin-bottom: 4px;
}

.recommendation-basis {
  font-size: 13px;
  color: #9e9890;
  margin: 0;
  display: flex;
  align-items: center;
  gap: 5px;
  flex-wrap: wrap;
}

.basis-tag {
  background: #f4faf8;
  border: 1px solid #d8eae6;
  color: #2d7a3a;
  font-weight: 700;
  font-size: 12px;
  padding: 2px 9px;
  border-radius: 20px;
}

.basis-tag--muted {
  color: #9e9890;
  background: #f5f5f5;
  border-color: #e5e5e5;
}

/* Navigation links at bottom of items-col */
.nav-links {
  display: flex;
  gap: 1rem;
}

.nav-link-btn {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 0.9rem 1rem;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 700;
  text-decoration: none;
  transition:
    filter 0.15s,
    transform 0.1s;
  letter-spacing: 0.02em;
}

.nav-link-btn:active {
  transform: scale(0.98);
}

.nav-link-btn:nth-child(1) {
  background: #2d7a3a;
  color: #ffffff;
  border: none;
}

.nav-link-btn:nth-child(1):hover {
  filter: brightness(1.12);
}

.nav-link-btn:nth-child(2) {
  background: #185fa5;
  color: #ffffff;
  border: none;
}

.nav-link-btn:nth-child(2):hover {
  filter: brightness(1.15);
}

.section-label {
  font-size: 14px;
  font-weight: 700;
  letter-spacing: 0.07em;
  text-transform: uppercase;
  margin: 0;
  padding-top: 4px;
}

.section-label--group {
  color: #185FA5;
  margin-top: 8px;
}

.item-group {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.items-list--more {
  margin-top: 3px;
}

.show-more-btn {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  margin-top: 6px;
  padding: 5px 12px 5px 10px;
  background: #f4faf8;
  border: 1px solid #d8eae6;
  border-radius: 20px;
  font-size: 13px;
  font-weight: 600;
  color: #2d7a3a;
  cursor: pointer;
  transition: background-color 0.15s, border-color 0.15s;
  font-family: inherit;
}

.show-more-btn:hover {
  background: #e6f4e8;
  border-color: #4d9e5a;
}

.show-more-btn span {
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.items-list {
  display: flex;
  flex-direction: column;
  gap: 5px;
}



/* Advice card */
.advice-card {
  padding: 1rem 1.125rem;
}

.advice-title {
  font-size: 15px;
  font-weight: 700;
  color: #2d7a3a;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  margin: 0 0 8px;
}

.advice-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.advice-item {
  display: flex;
  gap: 8px;
  align-items: flex-start;
}

.advice-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  flex-shrink: 0;
  margin-top: 5px;
}

.advice-text {
  font-size: 14px;
  line-height: 1.55;
  color: #1a1714;
}

/* Status messages */
.status-msg {
  text-align: center;
  padding: 3rem 0;
  color: #6b6560;
  font-size: 1rem;
}

.status-msg--error {
  color: #c0392b;
}

/* Responsive */
@media (max-width: 600px) {
  .main-grid {
    grid-template-columns: 1fr;
  }

  .left-col {
    flex-direction: row;
    flex-wrap: wrap;
  }

  .mannequin-col {
    flex: 1;
    min-width: 140px;
    flex-direction: row;
    align-items: flex-start;
    flex-wrap: wrap;
    gap: 1rem;
  }

  .guide-card {
    flex: 1;
    min-width: 140px;
  }

  .mannequin-svg {
    max-width: 100px;
  }

  .score-area {
    flex: 1;
    border-top: none;
    border-left: 1px solid #d8eae6;
    padding-top: 0;
    padding-left: 1rem;
    margin-top: 0;
  }
}
</style>
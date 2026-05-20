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

// ── Selection state — declared early so clearSearch can reference it ──────────
const selectedItems = ref(new Set())

// ── Search bar state ──────────────────────────────────────────────────────────
const searchQuery = ref('')
const searchFocused = ref(false)
const searchInputRef = ref(null)

const filteredSuburbs = computed(() => {
  const q = searchQuery.value.trim().toLowerCase()
  if (!q) return allSuburbs.value
  return allSuburbs.value.filter((s) => s.suburb_name.toLowerCase().includes(q))
})

function selectSuburb(suburb) {
  selectedSuburbId.value = String(suburb.suburb_id)
  searchQuery.value = suburb.suburb_name
  searchFocused.value = false
}

function clearSearch() {
  searchQuery.value = ''
  selectedSuburbId.value = ''
  selectedItems.value = new Set()
  searchInputRef.value?.focus()
}

function onSearchBlur() {
  setTimeout(() => { searchFocused.value = false }, 150)
}

// ── Guided tour ───────────────────────────────────────────────────────────────
// Steps: 1 = pick suburb, 2 = read weather, 3 = pick items, 4 = check score, 0 = done
const TOUR_KEY = 'coolpath_outfit_tour_done'
const tourStep = ref(0)

const tourActive = computed(() => tourStep.value >= 1)

// Refs for scrollIntoView on each highlighted section
const refWeatherCard = ref(null)
const refItemsCol = ref(null)
const refLeftCol = ref(null)

function scrollToRef(el) {
  if (!el) return
  el.scrollIntoView({ behavior: 'smooth', block: 'start' })
}

function tourNext() {
  if (tourStep.value >= 4) {
    endTour()
    return
  }
  tourStep.value++
  nextTick(() => {
    if (tourStep.value === 2) scrollToRef(refWeatherCard.value)
    if (tourStep.value === 3) scrollToRef(refItemsCol.value)
    if (tourStep.value === 4) scrollToRef(refLeftCol.value)
  })
}

function endTour() {
  tourStep.value = 0
  try { sessionStorage.setItem(TOUR_KEY, '1') } catch {}
}

function restartTour() {
  tourStep.value = selectedSuburbId.value ? 2 : 1
  try { sessionStorage.removeItem(TOUR_KEY) } catch {}
  nextTick(() => {
    if (tourStep.value === 2) scrollToRef(refWeatherCard.value)
    else scrollToRef(searchInputRef.value)
  })
}

// Advance tour step 1 → 2 automatically when suburb is selected
watch(selectedSuburbId, (val) => {
  if (val && tourStep.value === 1) {
    tourStep.value = 2
    nextTick(() => scrollToRef(refWeatherCard.value))
  }
})

onMounted(async () => {
  // Start tour unless already completed this session
  let tourDone = false
  try { tourDone = !!sessionStorage.getItem(TOUR_KEY) } catch {}
  if (!tourDone) tourStep.value = 1

  try {
    const res = await fetch(`${API_BASE}/suburbs`)
    if (!res.ok) throw new Error(`HTTP ${res.status}`)
    allSuburbs.value = (await res.json()).sort((a, b) => a.suburb_name.localeCompare(b.suburb_name))
    // If suburbId passed from another page, auto-select it
    const queryId = route.query.suburbId
    if (queryId && allSuburbs.value.find((s) => s.suburb_id === Number(queryId))) {
      selectedSuburbId.value = String(queryId)
      const match = allSuburbs.value.find((s) => s.suburb_id === Number(queryId))
      if (match) searchQuery.value = match.suburb_name
      // Skip step 1 if suburb pre-selected via query param
      if (tourStep.value === 1) tourStep.value = 2
      await nextTick()
      autoSelectBestOutfit()
    }
    // No fallback — user must select a suburb themselves
  } catch (e) {
    error.value = 'Could not load suburb data. Please try again.'
    console.error(e)
  } finally {
    loading.value = false
  }
})

// Mandatory layers: always select one item, even if only bad options exist
const MANDATORY_LAYERS = ['base-top', 'bottom', 'footwear']

// Auto-select best outfit: good items preferred, mandatory layers always filled
function autoSelectBestOutfit() {
  const mode = climateMode.value
  const uv = uvHigh.value
  const next = new Set()

  // Helper: get heatAdj for current mode
  const getAdj = (item) => (typeof item.heatAdj === 'object' ? item.heatAdj[mode] ?? 0 : item.heatAdj)

  // For each exclusive layer, pick the best item
  // Preference: good items first, then fall back to any item for mandatory layers
  const layersToFill = [...EXCLUSIVE_LAYERS, 'base-top']
  layersToFill.forEach((layer) => {
    const candidates = ALL_ITEMS.filter((item) => {
      if (item.layer !== layer) return false
      if (item.uvOnly && !uv) return false
      if (!item.modes[mode]) return false
      return true
    })
    const goodCandidates = candidates.filter((i) => i.modes[mode] === 'good')
    const pool = goodCandidates.length ? goodCandidates : candidates

    if (!pool.length && MANDATORY_LAYERS.includes(layer)) {
      // Fallback: include items not in modes[mode] for mandatory layers
      const fallback = ALL_ITEMS.filter((i) => i.layer === layer && !(i.uvOnly && !uv))
      if (fallback.length) {
        // Pick the one whose heatAdj best moves toward comfort zone (18–22°C)
        const base = selectedSuburb.value?.apparent_temperature ?? 22
        fallback.sort((a, b) => {
          const targetA = Math.abs((base + getAdj(a)) - 20)
          const targetB = Math.abs((base + getAdj(b)) - 20)
          return targetA - targetB
        })
        next.add(fallback[0].id)
      }
      return
    }

    if (!pool.length) return

    // Sort: for hot/mild, minimise heatAdj; for cool, maximise toward comfort
    const base = selectedSuburb.value?.apparent_temperature ?? 22
    pool.sort((a, b) => {
      const distA = Math.abs((base + getAdj(a)) - 20)
      const distB = Math.abs((base + getAdj(b)) - 20)
      return distA - distB
    })
    next.add(pool[0].id)
  })

  // Add all good null-layer accessories (water, sunscreen etc.)
  ALL_ITEMS.forEach((item) => {
    if (item.layer !== null) return
    if (!item.modes[mode] || item.modes[mode] !== 'good') return
    if (item.uvOnly && !uv) return
    next.add(item.id)
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
// layer: controls mutual exclusion within a group
//   'base-top'  — inner upper layer (shirt, thermals, longsleeve) — coexists with outer-top
//   'outer-top' — outer upper layer (jacket, vest, hoodie, raincoat) — mutually exclusive
//   'bottom'    — lower body, mutually exclusive
//   'head'      — head slot, mutually exclusive
//   'footwear'  — foot slot, mutually exclusive
//   null        — accessories, no exclusion
// heatAdj: positive = raises body temp, negative = lowers it (both directions move toward comfort)
// modes: category per climateMode ('good' | 'bad', omit if not relevant)
const ALL_ITEMS = [
  // ── HEAD ──────────────────────────────────────────────────────────────────
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
    heatAdj: { hot: -1.5, mild: -1.0, cool: 1.5 },
    uvOnly: false,
    layer: 'head',
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
    heatAdj: { hot: -0.8, mild: -0.5, cool: 0 },
    uvOnly: true,
    layer: 'head',
  },
  {
    id: 'beanie',
    name: 'Beanie',
    icon: '🧶',
    modes: { cool: 'good' },
    effect: {
      cool: 'Retains significant body heat — essential for keeping warm on cold days',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: 0, cool: 2.0 },
    uvOnly: false,
    layer: 'head',
  },
  // ── BASE TOP (inner layer — coexists with outer-top) ──────────────────────
  {
    id: 'shirt',
    name: 'Light cotton shirt',
    icon: '👕',
    modes: { hot: 'good', mild: 'good', cool: 'bad' },
    effect: {
      hot: 'Breathable and reflects heat — ideal for hot weather',
      mild: 'Comfortable and breathable on a mild day',
      cool: 'Too light alone for cool conditions — layer it under a warm jacket',
    },
    explanation: {
      cool: 'Light fabrics provide little insulation in cool conditions on their own.',
    },
    heatAdj: { hot: -1.5, mild: -1.0, cool: 0.5 },
    uvOnly: false,
    layer: 'base-top',
  },
  {
    id: 'longsleeve',
    name: 'Light long-sleeve shirt',
    icon: '🩱',
    modes: { cool: 'good', mild: 'good' },
    effect: {
      cool: 'Adds warmth while keeping a breathable base layer',
      mild: 'Good coverage without overheating on a mild day',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: -0.5, cool: 1.5 },
    uvOnly: false,
    layer: 'base-top',
  },
  {
    id: 'thermals',
    name: 'Thermal base layer',
    icon: '🥼',
    modes: { cool: 'good' },
    effect: {
      cool: 'Traps body heat close to the skin — most effective base layer in cold conditions',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: 0, cool: 2.5 },
    uvOnly: false,
    layer: 'base-top',
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
    heatAdj: { hot: 2.5, mild: 1.5, cool: 0 },
    uvOnly: false,
    layer: 'base-top',
  },
  // ── OUTER TOP (outer layer — mutually exclusive with each other) ──────────
  {
    id: 'warmjacket',
    name: 'Warm jacket',
    icon: '🧥',
    modes: { cool: 'good', mild: 'good' },
    effect: {
      cool: 'Good insulation for cool conditions — keeps core temperature stable',
      mild: 'Useful layer for changeable or breezy mild days',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: 1.0, cool: 3.5 },
    uvOnly: false,
    layer: 'outer-top',
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
    heatAdj: { hot: 0, mild: 0.5, cool: 2.0 },
    uvOnly: false,
    layer: 'outer-top',
  },
  {
    id: 'hoodie',
    name: 'Fleece hoodie',
    icon: '🧥',
    modes: { cool: 'good', mild: 'bad', hot: 'bad' },
    effect: {
      cool: 'Warm and comfortable for cool conditions',
      mild: 'Too warm for today — may cause overheating during activity',
      hot: 'Traps heat and reduces sweat evaporation — avoid in hot weather',
    },
    explanation: {
      mild: 'Synthetic fabric blocks airflow and holds body heat longer than needed.',
      hot: 'Hoodies in heat significantly raise body temperature and risk of heat stress.',
    },
    heatAdj: { hot: 3.0, mild: 2.0, cool: 2.5 },
    uvOnly: false,
    layer: 'outer-top',
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
    heatAdj: { hot: 0, mild: 0.3, cool: 1.0 },
    uvOnly: false,
    layer: 'outer-top',
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
    heatAdj: { hot: 4.0, mild: 2.5, cool: 0 },
    uvOnly: false,
    layer: 'outer-top',
  },
  // ── BOTTOM ────────────────────────────────────────────────────────────────
  {
    id: 'lighttrousers',
    name: 'Light trousers',
    icon: '👖',
    modes: { hot: 'good', mild: 'good' },
    effect: {
      hot: 'Good airflow and UV coverage — better than shorts for sun protection',
      mild: 'Comfortable with enough air circulation for a mild day',
    },
    explanation: {},
    heatAdj: { hot: -1.0, mild: -0.5, cool: 0 },
    uvOnly: false,
    layer: 'bottom',
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
    heatAdj: { hot: -0.5, mild: -0.3, cool: 0 },
    uvOnly: false,
    layer: 'bottom',
  },
  {
    id: 'warmtrousers',
    name: 'Warm trousers',
    icon: '👖',
    modes: { cool: 'good', mild: 'good' },
    effect: {
      cool: 'Heavier fabric keeps legs warm — much better than light trousers in cool conditions',
      mild: 'Good option for cooler mild days, especially in the evening',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: 0.5, cool: 2.5 },
    uvOnly: false,
    layer: 'bottom',
  },
  {
    id: 'jeans',
    name: 'Tight dark jeans',
    icon: '👖',
    modes: { hot: 'bad', mild: 'bad' },
    effect: {
      hot: 'Dark tight fabric traps heat around the legs — very uncomfortable in the sun',
      mild: "Tight dark jeans absorb more heat than needed for today's temperature",
    },
    explanation: {
      hot: 'Denim is heavy and non-breathable; dark colour absorbs sunlight significantly.',
      mild: 'Looser or lighter fabric will be more comfortable and cooler.',
    },
    heatAdj: { hot: 2.0, mild: 1.0, cool: 0 },
    uvOnly: false,
    layer: 'bottom',
  },
  {
    id: 'leggings',
    name: 'Thermal leggings',
    icon: '🩲',
    modes: { cool: 'good' },
    effect: {
      cool: 'Worn under trousers, they add significant warmth without bulk — great for very cold days',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: 0, cool: 1.5 },
    uvOnly: false,
    layer: 'base-bottom',
  },
  // ── FOOTWEAR ──────────────────────────────────────────────────────────────
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
    heatAdj: { hot: -0.3, mild: -0.2, cool: 0.3 },
    uvOnly: false,
    layer: 'footwear',
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
    heatAdj: { hot: -0.5, mild: -0.3, cool: 0 },
    uvOnly: false,
    layer: 'footwear',
  },
  {
    id: 'warmboots',
    name: 'Warm boots',
    icon: '🥾',
    modes: { cool: 'good' },
    effect: {
      cool: 'Insulated boots keep feet warm and dry — important for comfort on cold days',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: 0, cool: 1.5 },
    uvOnly: false,
    layer: 'footwear',
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
    heatAdj: { hot: 0.5, mild: 0.3, cool: 0 },
    uvOnly: false,
    layer: 'footwear',
  },
  // ── ACCESSORIES (no layer exclusion) ──────────────────────────────────────
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
    heatAdj: { hot: 0, mild: 0, cool: 0 },
    uvOnly: true,
    layer: null,
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
    heatAdj: { hot: 0, mild: 0, cool: 0 },
    uvOnly: true,
    layer: null,
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
    heatAdj: { hot: 0, mild: 0, cool: 0 },
    uvOnly: false,
    layer: null,
  },
  {
    id: 'umbrella',
    name: 'Sun umbrella',
    icon: '☂️',
    modes: { hot: 'good', mild: 'good' },
    effect: {
      hot: 'Creates portable shade — one of the most effective ways to reduce heat exposure',
      mild: "Useful if you'll be standing or walking in sun for a while",
    },
    explanation: {},
    heatAdj: { hot: -1.5, mild: -0.8, cool: 0 },
    uvOnly: false,
    layer: null,
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
    heatAdj: { hot: -1.2, mild: 0, cool: 0 },
    uvOnly: false,
    layer: null,
  },
  {
    id: 'scarf',
    name: 'Warm scarf',
    icon: '🧣',
    modes: { cool: 'good' },
    effect: {
      cool: 'Protects the neck and face from cold wind — retains significant body heat',
    },
    explanation: {},
    heatAdj: { hot: 0, mild: 0, cool: 1.5 },
    uvOnly: false,
    layer: null,
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
    heatAdj: { hot: 0, mild: 0, cool: 1.0 },
    uvOnly: false,
    layer: null,
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
  { key: 'head',       label: 'Head & Sun Protection', ids: ['hat', 'cap', 'beanie', 'sg', 'sc'] },
  { key: 'base-top',   label: 'Base Layer (Top)',       ids: ['thermals', 'longsleeve', 'shirt', 'dshirt'] },
  { key: 'outer-top',  label: 'Outer Layer (Top)',      ids: ['warmjacket', 'vest', 'hoodie', 'raincoat', 'jacket'] },
  { key: 'bottom',     label: 'Bottom',                 ids: ['warmtrousers', 'lighttrousers', 'shorts', 'jeans'] },
  { key: 'base-bottom',label: 'Base Layer (Bottom)',    ids: ['leggings'] },
  { key: 'footwear',   label: 'Footwear',               ids: ['warmboots', 'runners', 'sandals', 'flipflops'] },
  { key: 'accessory',  label: 'Accessories',            ids: ['water', 'umbrella', 'coolingpatch', 'scarf', 'gloves'] },
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

// Layers that are mutually exclusive (only one item per layer allowed)
const EXCLUSIVE_LAYERS = new Set(['head', 'outer-top', 'bottom', 'footwear'])

function toggleItem(id) {
  const next = new Set(selectedItems.value)
  if (next.has(id)) {
    next.delete(id)
  } else {
    const item = ALL_ITEMS.find((i) => i.id === id)
    // Deselect other items in the same exclusive layer
    if (item?.layer && EXCLUSIVE_LAYERS.has(item.layer)) {
      ALL_ITEMS.forEach((other) => {
        if (other.layer === item.layer && other.id !== id) next.delete(other.id)
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
  const mode = climateMode.value
  let adj = 0
  selectedItems.value.forEach((id) => {
    const item = ALL_ITEMS.find((i) => i.id === id)
    if (item) {
      adj += typeof item.heatAdj === 'object' ? (item.heatAdj[mode] ?? 0) : item.heatAdj
    }
  })
  return Math.round((base + adj) * 10) / 10
})

const exposureBarPct = computed(() => {
  if (exposureTemp.value === null) return 0
  // Bar maps 5°C–45°C range; centred on comfort zone 18–22°C
  return Math.max(3, Math.min(97, ((exposureTemp.value - 5) / (45 - 5)) * 100))
})

const exposureConfig = computed(() => {
  const t = exposureTemp.value
  if (t === null) return { color: '#9e9890', label: 'Select a suburb to begin', pulse: false }
  // Unified comfort-zone logic: target 18–22°C regardless of season
  if (t < 10)  return { color: '#0c447c', label: 'Very cold — add more layers', pulse: false }
  if (t < 14)  return { color: '#185FA5', label: 'Cold — dress more warmly', pulse: false }
  if (t < 18)  return { color: '#5b9bd5', label: 'A little cool — consider an extra layer', pulse: false }
  if (t <= 26) return { color: '#4d9e5a', label: 'Comfortable — good outfit for today', pulse: false }
  if (t <= 30) return { color: '#c8a020', label: 'Slightly warm — consider lighter clothing', pulse: false }
  if (t <= 34) return { color: '#e8903a', label: 'Warm — take care outside', pulse: false }
  if (t <= 37) return { color: '#c0392b', label: 'Hot — limit time outdoors', pulse: false }
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
  const G = '#2d7a3a'
  const B = '#185FA5'
  const O = '#e8903a'
  const R = '#c0392b'

  // ── Cool mode ──────────────────────────────────────────────────────────────
  if (mode === 'cool') {
    // Outer layer
    if (s.has('warmjacket'))
      msgs.push({ color: G, text: 'Good choice — a warm jacket keeps your core temperature stable in cool conditions.' })
    else if (s.has('vest'))
      msgs.push({ color: G, text: 'A puffer vest adds core warmth. Consider pairing it with a long-sleeve base layer.' })
    else if (s.has('hoodie'))
      msgs.push({ color: G, text: 'A fleece hoodie works well on cool days — good warmth without being too heavy.' })
    else if (s.has('raincoat'))
      msgs.push({ color: O, text: 'A light raincoat helps with wind and rain, but may not be warm enough alone — layer underneath.' })
    else
      msgs.push({ color: R, text: "No outer layer selected — today's conditions call for a warm jacket or similar." })

    // Base top
    if (s.has('thermals'))
      msgs.push({ color: G, text: 'Thermal base layer is excellent — traps body heat right against the skin.' })
    else if (s.has('longsleeve'))
      msgs.push({ color: G, text: 'Long-sleeve base layer is a good foundation — pair with a warm outer layer.' })
    else if (s.has('shirt'))
      msgs.push({ color: O, text: 'Light cotton shirt alone may not be enough — make sure you have a warm layer on top.' })
    else if (s.has('dshirt'))
      msgs.push({ color: R, text: 'Tight dark shirt is a poor base layer for cold weather — it provides little insulation.' })

    // Bottom
    if (s.has('warmtrousers'))
      msgs.push({ color: G, text: 'Warm trousers are the right call — light fabric would leave your legs too cold.' })
    else if (s.has('leggings'))
      msgs.push({ color: G, text: 'Thermal leggings add meaningful warmth. Pair them with warm trousers on very cold days.' })
    else if (s.has('lighttrousers') || s.has('shorts'))
      msgs.push({ color: R, text: 'Light bottoms are not warm enough for today — warm trousers will be much more comfortable.' })

    // Head
    if (s.has('beanie'))
      msgs.push({ color: G, text: 'Beanie is a great choice — significant heat is lost through the head in cold weather.' })
    else if (s.has('hat'))
      msgs.push({ color: G, text: 'Wide-brim hat helps retain heat and still provides UV cover.' })

    // Footwear
    if (s.has('warmboots'))
      msgs.push({ color: G, text: 'Warm boots keep feet insulated — cold feet are one of the first signs of heat loss.' })
    else if (s.has('runners'))
      msgs.push({ color: O, text: 'Runners are manageable on cool days but warm boots would keep your feet more comfortable.' })

    // Accessories
    if (s.has('scarf'))
      msgs.push({ color: G, text: 'Scarf protects your neck and face from wind — makes a noticeable difference in the cold.' })
    if (s.has('gloves'))
      msgs.push({ color: G, text: 'Gloves are important — hands and extremities lose heat quickly in cool conditions.' })
    if (s.has('water'))
      msgs.push({ color: B, text: "Good habit — it's easy to forget hydration in cool weather, but it's still important." })
  }

  // ── Mild mode ──────────────────────────────────────────────────────────────
  else if (mode === 'mild') {
    if (s.has('jacket'))
      msgs.push({ color: O, text: "Dark heavy jacket is too warm for today — you may overheat, especially if you're active." })
    if (s.has('hoodie'))
      msgs.push({ color: O, text: 'Fleece hoodie may be a bit warm for a mild day — a light vest or shirt would be more comfortable.' })
    if (s.has('dshirt'))
      msgs.push({ color: O, text: 'Tight dark shirts absorb heat and reduce airflow — a lighter top would be more comfortable today.' })
    if (s.has('warmjacket') || s.has('vest') || s.has('raincoat'))
      msgs.push({ color: G, text: 'Good layering choice — easy to take off if you warm up during the day.' })
    if (s.has('shirt') || s.has('longsleeve'))
      msgs.push({ color: G, text: 'Comfortable and breathable choice for a mild day.' })
    if (s.has('hat'))
      msgs.push({ color: G, text: 'Wide-brim hat is a good call — sun protection matters even on mild days.' })
    if (s.has('lighttrousers'))
      msgs.push({ color: G, text: 'Light trousers are a great choice — comfortable coverage without heat penalty.' })
    if (s.has('warmtrousers'))
      msgs.push({ color: O, text: 'Warm trousers may be a bit heavy for today — light trousers would be more comfortable.' })
    if (s.has('jeans'))
      msgs.push({ color: O, text: 'Tight dark jeans absorb more heat than needed today — lighter fabric would be more comfortable.' })
    if (s.has('water'))
      msgs.push({ color: B, text: "Staying hydrated is important even when it doesn't feel hot." })
  }

  // ── Hot mode ───────────────────────────────────────────────────────────────
  else {
    if (s.has('jacket'))
      msgs.push({ color: R, text: 'Dark heavy jacket is dangerous today — it traps heat and significantly raises your body temperature.' })
    if (s.has('hoodie'))
      msgs.push({ color: R, text: 'Hoodie traps heat and reduces sweat evaporation — remove it immediately in this heat.' })
    if (s.has('dshirt'))
      msgs.push({ color: R, text: "Tight dark shirt absorbs heat and blocks airflow — unsafe in today's conditions." })
    if (s.has('jeans'))
      msgs.push({ color: R, text: 'Tight dark jeans absorb sunlight and trap heat — switch to light trousers or shorts.' })
    if (s.has('flipflops'))
      msgs.push({ color: O, text: 'Flip flops offer no support — pavement can reach 60°C+ in direct sun.' })
    if (s.has('shirt'))
      msgs.push({ color: G, text: 'Light cotton shirt is a great choice — breathes well and reflects sunlight.' })
    if (s.has('lighttrousers'))
      msgs.push({ color: G, text: 'Light trousers protect your legs from UV while keeping airflow good.' })
    if (s.has('shorts'))
      msgs.push({ color: G, text: 'Shorts keep legs cool — remember to apply sunscreen to any exposed skin.' })
    if (s.has('hat'))
      msgs.push({ color: G, text: 'Wide-brim hat is essential today — blocks UV and reduces heat stress on your head and neck.' })
    if (s.has('umbrella'))
      msgs.push({ color: G, text: 'Sun umbrella creates portable shade — one of the most effective ways to reduce heat exposure.' })
    if (s.has('coolingpatch'))
      msgs.push({ color: G, text: 'Cooling neck patch can reduce perceived heat noticeably during outdoor activity.' })
    if (s.has('water'))
      msgs.push({ color: B, text: 'Carry water and drink every 15–20 minutes when outside. Dehydration is a serious risk today.' })
    else
      msgs.push({ color: R, text: "Don't go out without water — dehydration is easy to miss until it's too late, especially for older adults." })
  }

  // ── UV advice (all modes) ──────────────────────────────────────────────────
  if (uv) {
    if (s.has('sg'))
      msgs.push({ color: G, text: 'Sunglasses are the right call — UV is high enough to cause eye strain and long-term damage.' })
    if (s.has('sc'))
      msgs.push({ color: G, text: 'SPF 50+ sunscreen is important. Apply 20 minutes before going out and reapply every 2 hours.' })
    if (!s.has('sg') && !s.has('sc'))
      msgs.push({ color: O, text: 'UV is high today — add sunglasses and sunscreen before heading out.' })
  }

  return msgs.length ? msgs : [{ color: G, text: 'Looking good — your outfit is well suited to today\'s conditions.' }]
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
        <!-- Suburb search bar -->
        <div class="selector-row card" :class="{ 'tour-highlight': tourActive && tourStep === 1 }">
          <div class="search-wrap">
            <svg class="search-icon" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <circle cx="11" cy="11" r="8" /><line x1="21" y1="21" x2="16.65" y2="16.65" />
            </svg>
            <input
              ref="searchInputRef"
              v-model="searchQuery"
              type="text"
              class="search-input"
              :placeholder="tourActive && tourStep === 1 ? 'Type your suburb to begin...' : 'Search suburb...'"
              autocomplete="off"
              @focus="searchFocused = true"
              @blur="onSearchBlur"
            />
            <button v-if="searchQuery" class="clear-btn" @click="clearSearch" aria-label="Clear suburb">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><line x1="18" y1="6" x2="6" y2="18"/><line x1="6" y1="6" x2="18" y2="18"/></svg>
            </button>
            <ul v-if="searchFocused && filteredSuburbs.length" class="search-dropdown">
              <li
                v-for="s in filteredSuburbs"
                :key="s.suburb_id"
                class="search-option"
                :class="{ 'search-option--active': s.suburb_id === Number(selectedSuburbId) }"
                @mousedown.prevent="selectSuburb(s)"
              >{{ s.suburb_name }}</li>
            </ul>
            <p v-if="searchFocused && searchQuery && !filteredSuburbs.length" class="search-empty">No suburbs found</p>
          </div>
        </div>

        <!-- Tour step 1: pick a suburb (shown before any suburb is selected) -->
        <div v-if="tourActive && tourStep === 1" class="tour-card tour-card--step1">
          <div class="tour-step-pip">Step 1 of 4</div>
          <p class="tour-heading">Start by choosing your suburb</p>
          <p class="tour-body">
            Type the name of your suburb in the highlighted search box above.
            We'll look up today's temperature, UV level, and tree shade for that area —
            then build outfit recommendations just for those conditions.
          </p>
          <button class="tour-skip-btn" @click="endTour">Skip guide</button>
        </div>

        <!-- Content shown after suburb selected -->
        <template v-if="selectedSuburbId">

        <!-- AC 4.1.2 — Weather summary card -->
        <div ref="refWeatherCard" :class="{ 'tour-highlight-wrap': tourActive && tourStep === 2 }">
          <OutfitWeatherCard :suburb="selectedSuburb" />
        </div>

        <!-- Tour step 2: understand the weather card -->
        <div v-if="tourActive && tourStep === 2" class="tour-card">
          <div class="tour-step-pip">Step 2 of 4</div>
          <p class="tour-heading">Understanding today's conditions</p>
          <p class="tour-body">
            The card above shows four things about your suburb right now:
          </p>
          <ul class="tour-list">
            <li><strong>Feels like</strong> — the temperature your body actually experiences, accounting for humidity and wind. This drives the outfit recommendations.</li>
            <li><strong>UV Index</strong> — how strong the sun is. When UV is 3 or above, sun protection items like a hat and sunscreen appear.</li>
            <li><strong>Tree coverage</strong> — how much shade your suburb has. More shade means lower heat risk when outdoors.</li>
            <li><strong>Risk level</strong> — an overall heat safety rating combining temperature and shade.</li>
          </ul>
          <div class="tour-actions">
            <button class="tour-next-btn" @click="tourNext">Got it, show me the clothes →</button>
            <button class="tour-skip-btn" @click="endTour">Skip guide</button>
          </div>
        </div>

        <!-- Climate mode indicator -->
        <div class="mode-banner" :class="`mode-banner--${climateMode}`">
          <span v-if="climateMode === 'cool'">🧣 Cool day — recommendations are adjusted for lower temperatures</span>
          <span v-else-if="climateMode === 'mild'">🌤 Mild day — balanced recommendations for comfortable conditions</span>
          <span v-else>☀️ Hot day — heat safety recommendations are active</span>
          <span v-if="!uvHigh" class="uv-note"> | UV is low — sunglasses and sunscreen not shown</span>
        </div>

        <!-- Tour step 3: picking clothing items — guide moved inside items-col below -->

        <div class="main-grid">
          <!-- Left column: mannequin + score + advice -->
          <div ref="refLeftCol" class="left-col" :class="{ 'tour-highlight-wrap': tourActive && tourStep === 4 }">
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
                <g :style="{ display: isSelected('lighttrousers') || isSelected('warmtrousers') ? '' : 'none' }">
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
                <g :style="{ display: isSelected('thermals') && !isSelected('shirt') && !isSelected('longsleeve') && !isSelected('dshirt') ? '' : 'none' }">
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
                <g :style="{ display: false ? '' : 'none' }">
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
                <!-- Beanie -->
                <g :style="{ display: isSelected('beanie') ? '' : 'none' }">
                  <!-- Brim band -->
                  <rect x="58" y="30" width="44" height="9" rx="4" fill="#8B2020" />
                  <!-- Main dome -->
                  <ellipse cx="80" cy="26" rx="23" ry="22" fill="#A83030" />
                  <!-- Ribbing lines -->
                  <line x1="68" y1="8" x2="65" y2="32" stroke="#8B2020" stroke-width="1.5" opacity=".6" />
                  <line x1="74" y1="5" x2="72" y2="32" stroke="#8B2020" stroke-width="1.5" opacity=".6" />
                  <line x1="80" y1="4" x2="80" y2="32" stroke="#8B2020" stroke-width="1.5" opacity=".6" />
                  <line x1="86" y1="5" x2="88" y2="32" stroke="#8B2020" stroke-width="1.5" opacity=".6" />
                  <line x1="92" y1="8" x2="95" y2="32" stroke="#8B2020" stroke-width="1.5" opacity=".6" />
                  <!-- Pom-pom -->
                  <circle cx="80" cy="5" r="5" fill="#C84040" />
                </g>
                <!-- Leggings (base-bottom layer — shown under warm trousers) -->
                <g :style="{ display: isSelected('leggings') ? '' : 'none' }">
                  <!-- Left leg -->
                  <rect x="52" y="154" width="22" height="84" rx="7" fill="#2e3a50" />
                  <!-- Right leg -->
                  <rect x="86" y="154" width="22" height="84" rx="7" fill="#2e3a50" />
                  <!-- Waistband -->
                  <rect x="50" y="152" width="60" height="7" rx="3" fill="#1e2a3a" />
                  <!-- Subtle seam lines -->
                  <line x1="63" y1="160" x2="63" y2="236" stroke="#1e2a3a" stroke-width="1" opacity=".5" />
                  <line x1="97" y1="160" x2="97" y2="236" stroke="#1e2a3a" stroke-width="1" opacity=".5" />
                </g>
                <!-- Warm boots -->
                <g :style="{ display: isSelected('warmboots') ? '' : 'none' }">
                  <!-- Left boot shaft -->
                  <rect x="47" y="210" width="30" height="32" rx="5" fill="#5c3a1e" />
                  <!-- Left boot sole -->
                  <rect x="44" y="238" width="34" height="8" rx="4" fill="#3a2010" />
                  <!-- Left boot toe cap -->
                  <ellipse cx="61" cy="242" rx="17" ry="5" fill="#3a2010" />
                  <!-- Right boot shaft -->
                  <rect x="83" y="210" width="30" height="32" rx="5" fill="#5c3a1e" />
                  <!-- Right boot sole -->
                  <rect x="82" y="238" width="34" height="8" rx="4" fill="#3a2010" />
                  <!-- Right boot toe cap -->
                  <ellipse cx="99" cy="242" rx="17" ry="5" fill="#3a2010" />
                  <!-- Lace lines left -->
                  <line x1="54" y1="218" x2="70" y2="218" stroke="#c8a870" stroke-width="1.2" />
                  <line x1="54" y1="224" x2="70" y2="224" stroke="#c8a870" stroke-width="1.2" />
                  <line x1="54" y1="230" x2="70" y2="230" stroke="#c8a870" stroke-width="1.2" />
                  <!-- Lace lines right -->
                  <line x1="90" y1="218" x2="106" y2="218" stroke="#c8a870" stroke-width="1.2" />
                  <line x1="90" y1="224" x2="106" y2="224" stroke="#c8a870" stroke-width="1.2" />
                  <line x1="90" y1="230" x2="106" y2="230" stroke="#c8a870" stroke-width="1.2" />
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

            <!-- Tour step 4: score + mannequin + advice -->
            <div v-if="tourActive && tourStep === 4" class="tour-card">
              <div class="tour-step-pip">Step 4 of 4</div>
              <p class="tour-heading">Your outfit preview and score</p>
              <p class="tour-body">The highlighted panel on the left shows three things:</p>
              <ul class="tour-list">
                <li><strong>Mannequin</strong> — updates in real time as you tap items. You can see exactly what you've selected at a glance.</li>
                <li><strong>Heat exposure score</strong> — an estimated temperature your body will feel in this outfit. Lower is safer. The bar colour changes from green to red as risk rises.</li>
                <li><strong>Personalised advice</strong> — plain-language tips based on the specific items you've chosen and today's conditions.</li>
              </ul>
              <p class="tour-body">Try tapping different items in the list to see how your score changes.</p>
              <div class="tour-actions">
                <button class="tour-next-btn" @click="endTour">Done — let me try it ✓</button>
              </div>
            </div>
          </div>
          <!-- end left-col -->

          <!-- Item list — grouped by category -->
          <div ref="refItemsCol" class="items-col" :class="{ 'tour-highlight-wrap': tourActive && tourStep === 3 }">
            <!-- Tour step 3: picking items — guide at top of items col -->
            <div v-if="tourActive && tourStep === 3" class="tour-card">
              <div class="tour-step-pip">Step 3 of 4</div>
              <p class="tour-heading">Choose your clothing items</p>
              <p class="tour-body">The list to the right is your wardrobe for today. Here's how to read it:</p>
              <ul class="tour-list">
                <li><span class="tour-tag tour-tag--good">Green items</span> are recommended for today's heat and UV — tap to add them to your outfit.</li>
                <li><span class="tour-tag tour-tag--bad">Red items</span> are best avoided — they raise your heat exposure. They're shown so you know what to leave at home.</li>
                <li>Each group (Head, Top, Bottom, Footwear) only lets you pick <strong>one item per slot</strong> — selecting a new one swaps the old one out.</li>
                <li>Tap <strong>Show more options</strong> in any group to see all available items, including less common ones.</li>
              </ul>
              <div class="tour-actions">
                <button class="tour-next-btn" @click="tourNext">Got it, show me my score →</button>
                <button class="tour-skip-btn" @click="endTour">Skip guide</button>
              </div>
            </div>

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
        <!-- end v-if="selectedSuburbId" -->

        <!-- Restart tour button — always visible after tour is done -->
        <div v-if="!tourActive" class="tour-restart-row">
          <button class="tour-restart-btn" @click="restartTour" aria-label="Restart guide">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true"><circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/></svg>
            How to use this page
          </button>
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

/* Suburb search bar */
.selector-row {
  display: flex;
  align-items: center;
  padding: 0.5rem 0.75rem;
}

.search-wrap {
  position: relative;
  display: flex;
  align-items: center;
  width: 100%;
}

.search-icon {
  position: absolute;
  left: 12px;
  color: #9e9890;
  pointer-events: none;
  flex-shrink: 0;
}

.search-input {
  flex: 1;
  width: 100%;
  border: 1px solid #d8eae6;
  border-radius: 10px;
  padding: 9px 40px 9px 38px;
  font-size: 15px;
  background: #f4faf8;
  color: #1a1714;
  font-family: inherit;
  transition: border-color 0.15s, box-shadow 0.15s;
}

.search-input::placeholder {
  color: #b0aaa4;
}

.search-input:focus {
  outline: none;
  border-color: #4d9e5a;
  box-shadow: 0 0 0 3px rgba(77, 158, 90, 0.12);
}

.clear-btn {
  position: absolute;
  right: 8px;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: none;
  background: #e8e4e0;
  color: #6b6560;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: background-color 0.15s;
}

.clear-btn:hover {
  background: #d8d4d0;
}

.search-dropdown {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: #ffffff;
  border: 1px solid #d8eae6;
  border-radius: 10px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.1);
  list-style: none;
  margin: 0;
  padding: 4px 0;
  z-index: 100;
  max-height: 240px;
  overflow-y: auto;
}

.search-option {
  padding: 9px 14px;
  font-size: 14px;
  color: #1a1714;
  cursor: pointer;
  transition: background-color 0.1s;
}

.search-option:hover {
  background: #f4faf8;
}

.search-option--active {
  background: #e6f4e8;
  color: #2d7a3a;
  font-weight: 600;
}

.search-empty {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: #ffffff;
  border: 1px solid #d8eae6;
  border-radius: 10px;
  padding: 10px 14px;
  font-size: 14px;
  color: #9e9890;
  z-index: 100;
  margin: 0;
}

/* Tour highlight */
@keyframes tour-glow-pulse {
  0%, 100% { box-shadow: 0 0 0 3px #f0c040, 0 0 14px 5px rgba(240, 180, 0, 0.30); }
  50%       { box-shadow: 0 0 0 3px #f0c040, 0 0 24px 10px rgba(240, 180, 0, 0.50); }
}

.tour-highlight {
  outline: none;
  border-color: #f0c040 !important;
  animation: tour-glow-pulse 2s ease-in-out infinite;
  border-radius: 14px;
}

.tour-highlight-wrap {
  border-radius: 14px;
  animation: tour-glow-pulse 2s ease-in-out infinite;
}

/* Tour list */
.tour-list {
  list-style: none;
  padding: 0;
  margin: 4px 0 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.tour-list li {
  font-size: 14px;
  line-height: 1.6;
  color: #3a3530;
  padding-left: 14px;
  position: relative;
}

.tour-list li::before {
  content: '';
  position: absolute;
  left: 0;
  top: 8px;
  width: 5px;
  height: 5px;
  border-radius: 50%;
  background: #4d9e5a;
}

/* Guided tour cards */
.tour-card {
  background: #ffffff;
  border: 2px solid #4d9e5a;
  border-radius: 14px;
  padding: 1.25rem 1.4rem 1.1rem;
  box-shadow: 0 4px 20px rgba(45, 122, 58, 0.12);
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.tour-card--step1 {
  border-left: 4px solid #2d7a3a;
  border-top-color: #d8eae6;
  border-right-color: #d8eae6;
  border-bottom-color: #d8eae6;
  border-width: 1px;
  border-left-width: 4px;
  box-shadow: none;
  background: #f7fbf8;
}

.tour-step-pip {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #4d9e5a;
}

.tour-heading {
  font-size: 1.1rem;
  font-weight: 700;
  color: #1a1714;
  margin: 0;
  line-height: 1.3;
}

.tour-body {
  font-size: 15px;
  line-height: 1.65;
  color: #3a3530;
  margin: 0;
}

.tour-tag {
  display: inline-block;
  font-weight: 700;
  font-size: 13px;
  padding: 2px 8px;
  border-radius: 20px;
  margin: 0 1px;
}

.tour-tag--good {
  background: #e6f4e8;
  color: #1e5c28;
}

.tour-tag--bad {
  background: #fce8e6;
  color: #8b1a12;
}

.tour-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 0.25rem;
  flex-wrap: wrap;
}

.tour-next-btn {
  padding: 0.7rem 1.4rem;
  background: #2d7a3a;
  color: #ffffff;
  border: none;
  border-radius: 10px;
  font-size: 15px;
  font-weight: 700;
  font-family: inherit;
  cursor: pointer;
  transition: filter 0.15s, transform 0.1s;
}

.tour-next-btn:hover {
  filter: brightness(1.1);
}

.tour-next-btn:active {
  transform: scale(0.97);
}

.tour-skip-btn {
  background: none;
  border: none;
  font-size: 13px;
  color: #9e9890;
  cursor: pointer;
  font-family: inherit;
  padding: 0;
  text-decoration: underline;
  text-underline-offset: 2px;
}

.tour-skip-btn:hover {
  color: #6b6560;
}

/* Restart tour row */
.tour-restart-row {
  display: flex;
  justify-content: flex-end;
}

.tour-restart-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: none;
  border: 1px solid #d8eae6;
  border-radius: 20px;
  padding: 6px 14px;
  font-size: 13px;
  font-weight: 600;
  color: #6b6560;
  cursor: pointer;
  font-family: inherit;
  transition: background-color 0.15s, border-color 0.15s, color 0.15s;
}

.tour-restart-btn:hover {
  background: #f4faf8;
  border-color: #4d9e5a;
  color: #2d7a3a;
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
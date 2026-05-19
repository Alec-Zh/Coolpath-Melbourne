<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  suburbs: { type: Array, default: () => [] },
  selectedSuburbId: { type: [Number, String], default: '' },
  transportMode: { type: String, default: 'walk' },
  durationMins: { type: Number, default: 30 },
  departureMinutes: { type: Number, default: 540 },
  loading: { type: Boolean, default: false },
})

const emit = defineEmits([
  'update:selectedSuburbId',
  'update:transportMode',
  'update:durationMins',
  'update:departureMinutes',
  'submit',
])

// ── Searchable suburb dropdown ────────────────────────────────────────────────
const searchQuery = ref('')
const dropdownOpen = ref(false)

const selectedSuburb = computed(
  () => props.suburbs.find((s) => String(s.suburb_id) === String(props.selectedSuburbId)) ?? null,
)

const filteredSuburbs = computed(() => {
  const q = searchQuery.value.trim().toLowerCase()
  if (!q) return props.suburbs
  return props.suburbs.filter((s) => s.suburb_name.toLowerCase().includes(q)).slice(0, 8)
})

function selectSuburb(suburb) {
  emit('update:selectedSuburbId', String(suburb.suburb_id))
  searchQuery.value = ''
  dropdownOpen.value = false
}

function onBlur() {
  setTimeout(() => {
    dropdownOpen.value = false
  }, 150)
}

function clearSuburb() {
  emit('update:selectedSuburbId', '')
  searchQuery.value = ''
  dropdownOpen.value = false
}

// ── Transport ─────────────────────────────────────────────────────────────────
const transportOptions = [
  { value: 'walk', label: 'Walk', icon: '🚶' },
  { value: 'tram', label: 'Tram', icon: '🚃' },
  { value: 'bus', label: 'Bus', icon: '🚌' },
  { value: 'drive', label: 'Drive', icon: '🚗' },
]

// ── Duration stepper ──────────────────────────────────────────────────────────
function stepDuration(delta) {
  const next = Math.min(120, Math.max(5, props.durationMins + delta))
  emit('update:durationMins', next)
}

// ── Departure time ────────────────────────────────────────────────────────────
const sliderMin = 540
const sliderMax = 1200
const sliderStep = 10

function minutesToLabel(mins) {
  const h = Math.floor(mins / 60)
  const m = mins % 60
  const period = h >= 12 ? 'pm' : 'am'
  const displayH = h > 12 ? h - 12 : h === 0 ? 12 : h
  return m === 0
    ? `${displayH}:00 ${period}`
    : `${displayH}:${String(m).padStart(2, '0')} ${period}`
}

const departureLabel = computed(() => minutesToLabel(props.departureMinutes))
const sliderFill = computed(
  () => ((props.departureMinutes - sliderMin) / (sliderMax - sliderMin)) * 100,
)

const timePresets = [
  { label: 'Morning', sublabel: '9:00 am', mins: 540, icon: '🌅' },
  { label: 'Midday', sublabel: '12:00 pm', mins: 720, icon: '☀️' },
  { label: 'Afternoon', sublabel: '3:00 pm', mins: 900, icon: '🌤' },
  { label: 'Evening', sublabel: '5:00 pm', mins: 1020, icon: '🌇' },
]

const sliderTicks = [
  { mins: 540, label: '9am' },
  { mins: 720, label: '12pm' },
  { mins: 900, label: '3pm' },
  { mins: 1020, label: '5pm' },
  { mins: 1200, label: '8pm' },
]

const isFormValid = computed(() => !!props.selectedSuburbId && !!props.transportMode)
</script>

<template>
  <div class="card">
    <h2 class="section-title">Plan your trip</h2>

    <!-- 1. Destination -->
    <div class="field">
      <label class="field-label">
        <span class="step-num">1</span>
        Where are you going?
      </label>
      <div class="search-wrap" @blur.capture="onBlur">
        <div
          v-if="selectedSuburb && !dropdownOpen"
          class="selected-display"
          @click="dropdownOpen = true"
        >
          <span class="selected-name">{{ selectedSuburb.suburb_name }}</span>
          <button class="clear-btn" @click.stop="clearSuburb" aria-label="Clear selection">
            <svg
              width="15"
              height="15"
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
        </div>
        <div v-else class="input-wrap">
          <svg
            class="search-icon"
            width="18"
            height="18"
            viewBox="0 0 24 24"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
          >
            <circle cx="11" cy="11" r="8" />
            <line x1="21" y1="21" x2="16.65" y2="16.65" />
          </svg>
          <input
            class="search-input"
            type="text"
            :placeholder="loading ? 'Loading…' : 'Type a suburb name…'"
            v-model="searchQuery"
            @focus="dropdownOpen = true"
            autocomplete="off"
          />
        </div>
        <ul v-if="dropdownOpen" class="dropdown" role="listbox">
          <li v-if="filteredSuburbs.length === 0" class="dropdown-empty">No suburbs found</li>
          <li
            v-for="s in filteredSuburbs"
            :key="s.suburb_id"
            class="dropdown-item"
            role="option"
            @mousedown.prevent="selectSuburb(s)"
          >
            {{ s.suburb_name }}
          </li>
        </ul>
      </div>
    </div>

    <!-- 2. Transport -->
    <div class="field">
      <span class="field-label">
        <span class="step-num">2</span>
        How are you getting there?
      </span>
      <div class="toggle-group">
        <button
          v-for="opt in transportOptions"
          :key="opt.value"
          type="button"
          class="toggle-btn"
          :class="{ active: transportMode === opt.value }"
          @click="emit('update:transportMode', opt.value)"
        >
          <span class="toggle-icon">{{ opt.icon }}</span>
          <span class="toggle-label">{{ opt.label }}</span>
        </button>
      </div>
    </div>

    <!-- 3. Duration — stepper -->
    <div class="field">
      <span class="field-label">
        <span class="step-num">3</span>
        How long will the trip take?
      </span>
      <div class="stepper">
        <button
          class="stepper-btn"
          :disabled="durationMins <= 5"
          @click="stepDuration(-5)"
          aria-label="Decrease by 5 minutes"
        >
          −
        </button>
        <div class="stepper-display">
          <span class="stepper-value">{{ durationMins }}</span>
          <span class="stepper-unit">min</span>
        </div>
        <button
          class="stepper-btn"
          :disabled="durationMins >= 120"
          @click="stepDuration(5)"
          aria-label="Increase by 5 minutes"
        >
          +
        </button>
      </div>
      <div class="stepper-hint">Tap − or + to change in 5-minute steps</div>
    </div>

    <!-- 4. Departure time — presets + fine slider -->
    <div class="field">
      <span class="field-label">
        <span class="step-num">4</span>
        What time are you leaving?
      </span>

      <div class="time-presets">
        <button
          v-for="p in timePresets"
          :key="p.mins"
          class="preset-btn"
          :class="{ active: departureMinutes === p.mins }"
          @click="emit('update:departureMinutes', p.mins)"
        >
          <span class="preset-icon">{{ p.icon }}</span>
          <span class="preset-label">{{ p.label }}</span>
          <span class="preset-sub">{{ p.sublabel }}</span>
        </button>
      </div>

      <div class="time-fine">
        <div class="time-fine-label">
          Fine-tune: <strong>{{ departureLabel }}</strong>
        </div>
        <div class="slider-wrap">
          <input
            type="range"
            class="slider"
            :min="sliderMin"
            :max="sliderMax"
            :step="sliderStep"
            :value="departureMinutes"
            :style="{ '--fill': sliderFill + '%' }"
            @input="emit('update:departureMinutes', Number($event.target.value))"
          />
          <div class="slider-ticks">
            <span
              v-for="tick in sliderTicks"
              :key="tick.mins"
              :style="{ left: ((tick.mins - sliderMin) / (sliderMax - sliderMin)) * 100 + '%' }"
              >{{ tick.label }}</span
            >
          </div>
        </div>
      </div>
    </div>

    <!-- Submit -->
    <button class="submit-btn" :disabled="!isFormValid" @click="emit('submit')">
      Check my trip →
    </button>
    <p v-if="!isFormValid" class="submit-hint">Please choose a suburb to continue</p>
  </div>
</template>

<style scoped>
.card {
  background: #fff;
  border-radius: 14px;
  padding: 1.75rem 1.5rem;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.07);
}
.section-title {
  font-size: 1.2rem;
  font-weight: 700;
  color: #1c2e2a;
  margin: 0 0 1.75rem;
}

/* Step label */
.field {
  margin-bottom: 1.75rem;
}
.field:last-of-type {
  margin-bottom: 0;
}
.field-label {
  display: flex;
  align-items: center;
  gap: 0.55rem;
  font-size: 1.05rem;
  font-weight: 700;
  color: #1c2e2a;
  margin-bottom: 0.85rem;
  flex-wrap: wrap;
}
.step-num {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: #2d7a3a;
  color: #fff;
  font-size: 0.82rem;
  font-weight: 800;
  flex-shrink: 0;
}

/* Suburb search */
.search-wrap {
  position: relative;
}
.selected-display {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.9rem 1rem;
  border: 2.5px solid #2d7a3a;
  border-radius: 10px;
  cursor: pointer;
  background: #f5fdf6;
}
.selected-name {
  font-weight: 700;
  font-size: 1.05rem;
  color: #1c2e2a;
}
.clear-btn {
  background: #e8f0ee;
  border: none;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  color: #5a6e6a;
  flex-shrink: 0;
}
.clear-btn:hover {
  background: #d0e8e4;
}
.input-wrap {
  position: relative;
}
.search-icon {
  position: absolute;
  left: 0.85rem;
  top: 50%;
  transform: translateY(-50%);
  color: #888;
  pointer-events: none;
}
.search-input {
  width: 100%;
  padding: 0.9rem 1rem 0.9rem 2.6rem;
  border: 2px solid #ddd;
  border-radius: 10px;
  font-size: 1.05rem;
  color: #222;
  outline: none;
  box-sizing: border-box;
  transition: border-color 0.2s;
}
.search-input:focus {
  border-color: #2d7a3a;
}
.dropdown {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background: #fff;
  border: 1.5px solid #ddd;
  border-radius: 10px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.12);
  list-style: none;
  margin: 0;
  padding: 0.3rem 0;
  z-index: 100;
  max-height: 260px;
  overflow-y: auto;
}
.dropdown-empty {
  padding: 0.85rem 1rem;
  font-size: 1rem;
  color: #888;
}
.dropdown-item {
  padding: 0.85rem 1rem;
  font-size: 1rem;
  font-weight: 500;
  color: #1a1a1a;
  cursor: pointer;
  transition: background 0.15s;
}
.dropdown-item:hover {
  background: #f0faf2;
}

/* Transport toggles */
.toggle-group {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 0.5rem;
}
.toggle-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.3rem;
  padding: 0.85rem 0.5rem;
  border: 2.5px solid #ddd;
  border-radius: 12px;
  background: #fff;
  cursor: pointer;
  color: #555;
  transition:
    border-color 0.15s,
    background 0.15s,
    color 0.15s;
}
.toggle-btn:hover {
  border-color: #2d7a3a;
  color: #2d7a3a;
}
.toggle-btn.active {
  border-color: #2d7a3a;
  background: #2d7a3a;
  color: #fff;
}
.toggle-icon {
  font-size: 1.6rem;
}
.toggle-label {
  font-weight: 700;
  font-size: 0.9rem;
}

/* Duration stepper */
.stepper {
  display: flex;
  align-items: center;
  gap: 0;
  max-width: 260px;
  border: 2px solid #ddd;
  border-radius: 12px;
  overflow: hidden;
}
.stepper-btn {
  background: #f5f5f5;
  border: none;
  font-size: 1.6rem;
  font-weight: 400;
  line-height: 1;
  width: 64px;
  height: 64px;
  cursor: pointer;
  color: #2d7a3a;
  flex-shrink: 0;
  transition: background 0.15s;
}
.stepper-btn:hover:not(:disabled) {
  background: #e8f5ea;
}
.stepper-btn:disabled {
  color: #ccc;
  cursor: not-allowed;
}
.stepper-display {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0;
  padding: 0.5rem;
  border-left: 2px solid #eee;
  border-right: 2px solid #eee;
}
.stepper-value {
  font-size: 2rem;
  font-weight: 800;
  color: #1c2e2a;
  line-height: 1;
}
.stepper-unit {
  font-size: 0.85rem;
  color: #7a9490;
  font-weight: 600;
}
.stepper-hint {
  margin-top: 0.5rem;
  font-size: 0.82rem;
  color: #9ab5b2;
}

/* Time presets */
.time-presets {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 0.5rem;
  margin-bottom: 1rem;
}
.preset-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.2rem;
  padding: 0.75rem 0.25rem;
  border: 2.5px solid #ddd;
  border-radius: 12px;
  background: #fff;
  cursor: pointer;
  transition:
    border-color 0.15s,
    background 0.15s;
}
.preset-btn:hover {
  border-color: #2d7a3a;
}
.preset-btn.active {
  border-color: #2d7a3a;
  background: #f0faf2;
}
.preset-icon {
  font-size: 1.4rem;
}
.preset-label {
  font-size: 0.82rem;
  font-weight: 700;
  color: #1c2e2a;
}
.preset-sub {
  font-size: 0.72rem;
  color: #7a9490;
}

/* Fine-tune slider */
.time-fine {
  border-top: 1px solid #e8f0ee;
  padding-top: 0.85rem;
}
.time-fine-label {
  font-size: 0.9rem;
  color: #5a6e6a;
  margin-bottom: 0.6rem;
}
.time-fine-label strong {
  color: #1c2e2a;
  font-size: 1rem;
}
.slider-wrap {
  position: relative;
  padding-bottom: 1.4rem;
}
.slider {
  width: 100%;
  height: 8px;
  appearance: none;
  border-radius: 4px;
  background: linear-gradient(to right, #2d7a3a var(--fill, 0%), #ddd var(--fill, 0%));
  outline: none;
  cursor: pointer;
}
.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #2d7a3a;
  border: 3px solid #fff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  cursor: pointer;
}
.slider-ticks {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 1.2rem;
}
.slider-ticks span {
  position: absolute;
  transform: translateX(-50%);
  font-size: 0.78rem;
  color: #9ab5b2;
  white-space: nowrap;
}

/* Submit */
.submit-btn {
  width: 100%;
  margin-top: 1.25rem;
  padding: 1.1rem;
  background: #2d7a3a;
  color: #fff;
  border: none;
  border-radius: 12px;
  font-size: 1.1rem;
  font-weight: 700;
  cursor: pointer;
  transition:
    filter 0.2s,
    opacity 0.2s;
  letter-spacing: 0.01em;
}
.submit-btn:hover:not(:disabled) {
  filter: brightness(1.1);
}
.submit-btn:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}
.submit-hint {
  text-align: center;
  font-size: 0.85rem;
  color: #9ab5b2;
  margin: 0.5rem 0 0;
}
</style>
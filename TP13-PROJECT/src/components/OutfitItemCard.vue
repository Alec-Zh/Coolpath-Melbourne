<script setup>
import { ref } from 'vue'

defineProps({
  item: { type: Object, required: true },
  selected: { type: Boolean, default: false },
})

defineEmits(['toggle'])

const imageMap = {
  hat: 'hat', shirt: 'shirt', sg: 'sunglasses',
  water: 'water', sc: 'sunscreen', warmjacket: 'warmjacket',
  jacket: 'jacket', dshirt: 'dshirt', longsleeve: 'longsleeve',
  scarf: 'scarf', cap: 'cap', umbrella: 'umbrella', flipflops: 'flipflops',
  hoodie: 'hoodie', jeans: 'jeans', shorts: 'shorts', runners: 'runners',
  sandals: 'sandals', vest: 'vest', coolingpatch: 'coolingpatch',
  gloves: 'gloves', thermals: 'thermals', raincoat: 'raincoat',
  lighttrousers: 'lighttrousers',
  warmtrousers: 'warmtrousers',
  leggings: 'leggings',
  beanie: 'beanie',
  warmboots: 'warmboots',
}

function imgSrc(id) {
  const name = imageMap[id]
  return name ? `/outfit/${name}.webp` : null
}

const zoomVisible = ref(false)
const zoomStyle = ref({})

function showZoom(e) {
  const rect = e.currentTarget.getBoundingClientRect()
  const ZOOM_W = 156
  const ZOOM_H = 156
  let left = rect.right + 10
  let top = rect.top + rect.height / 2 - ZOOM_H / 2
  // flip left if not enough room on right
  if (left + ZOOM_W > window.innerWidth - 8) {
    left = rect.left - ZOOM_W - 10
  }
  // clamp vertically
  top = Math.max(8, Math.min(top, window.innerHeight - ZOOM_H - 8))
  zoomStyle.value = { top: top + 'px', left: left + 'px' }
  zoomVisible.value = true
}

function hideZoom() {
  zoomVisible.value = false
}
</script>

<template>
  <button
    class="citem"
    :class="{
      'citem--good': selected && item.category === 'good',
      'citem--bad':  selected && item.category === 'bad',
      'citem--warn': !selected && item.category === 'bad',
      'citem--dim':  item.notForToday && !selected,
    }"
    @click="$emit('toggle', item.id)"
    @mouseenter="imgSrc(item.id) && showZoom($event)"
    @mouseleave="hideZoom"
    :aria-pressed="selected"
  >
    <!-- Icon -->
    <span class="ci-ico-wrap">
      <img v-if="imgSrc(item.id)" :src="imgSrc(item.id)" :alt="item.name" class="ci-img" />
      <span v-else class="ci-ico" aria-hidden="true">{{ item.icon }}</span>
    </span>

    <!-- Info -->
    <div class="ci-inf">
      <div class="ci-name">
        {{ item.name }}
        <span v-if="item.category === 'bad'" class="warn-icon" aria-label="Unsafe item">⚠</span>
        <span v-if="item.notForToday" class="not-today-tag">Not for today</span>
        <span v-else-if="item.category === 'good' && !item.notForToday" class="today-tag">✓ Today</span>
      </div>
      <div class="ci-eff">{{ item.effect }}</div>
      <div v-if="item.category === 'bad' && item.explanation" class="ci-explain">
        {{ item.explanation }}
      </div>
    </div>

    <!-- Selection indicator -->
    <span class="ci-check" aria-hidden="true">
      <svg v-if="selected && item.category === 'good'" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round">
        <polyline points="20 6 9 17 4 12" />
      </svg>
      <svg v-else-if="selected && item.category === 'bad'" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3" stroke-linecap="round" stroke-linejoin="round">
        <line x1="18" y1="6" x2="6" y2="18" />
        <line x1="6" y1="6" x2="18" y2="18" />
      </svg>
    </span>
  </button>

  <!-- Zoom overlay — fixed position, outside button -->
  <Teleport to="body">
    <div v-if="zoomVisible && imgSrc(item.id)" class="ci-zoom-portal" :style="zoomStyle" aria-hidden="true">
      <img :src="imgSrc(item.id)" :alt="item.name" class="ci-zoom-img" />
    </div>
  </Teleport>
</template>

<style scoped>
.citem {
  width: 100%;
  background: #ffffff;
  border: 1px solid #d8eae6;
  border-radius: 10px;
  padding: 11px 14px;
  cursor: pointer;
  display: flex;
  align-items: flex-start;
  gap: 10px;
  transition: border-color 0.15s, background-color 0.15s;
  text-align: left;
  font-family: inherit;
}

.citem:hover { border-color: #4d9e5a; background: #f4faf8; }
.citem--good { border-color: #4d9e5a; background: #f4faf8; }
.citem--bad  { border-color: #c0392b; background: #fce8e6; }
.citem--warn { border-color: #f5a78a; }
.citem--warn:hover { border-color: #c0392b; background: #fce8e6; }
.citem--dim  { opacity: 0.55; }
.citem--dim:hover { opacity: 1; }

.ci-ico-wrap {
  flex-shrink: 0;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.ci-img {
  width: 36px;
  height: 36px;
  object-fit: contain;
  border-radius: 6px;
  display: block;
}

.ci-ico {
  font-size: 22px;
  text-align: center;
}

.ci-inf { flex: 1; min-width: 0; }

.ci-name {
  font-size: 15px;
  font-weight: 600;
  color: #1a1714;
  line-height: 1.2;
  display: flex;
  align-items: center;
  gap: 5px;
  flex-wrap: wrap;
}

.warn-icon { font-size: 13px; color: #c0392b; flex-shrink: 0; }

.ci-eff {
  font-size: 13px;
  color: #9e9890;
  margin-top: 2px;
  line-height: 1.4;
}

.citem--good .ci-eff { color: #2d7a3a; }
.citem--bad  .ci-eff { color: #8b1a12; }

.ci-explain {
  font-size: 13px;
  color: #c0392b;
  margin-top: 3px;
  font-style: italic;
  line-height: 1.4;
}

.ci-check {
  width: 20px;
  height: 20px;
  border: 1.5px solid #d8eae6;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  margin-top: 2px;
  transition: all 0.15s;
}

.citem--good .ci-check { background: #4d9e5a; border-color: #4d9e5a; color: #ffffff; }
.citem--bad  .ci-check { background: #c0392b; border-color: #c0392b; color: #ffffff; }

.today-tag {
  font-size: 10px;
  font-weight: 700;
  background: #e6f4e8;
  color: #2d7a3a;
  border-radius: 20px;
  padding: 1px 7px;
  white-space: nowrap;
  flex-shrink: 0;
}

.not-today-tag {
  font-size: 10px;
  font-weight: 600;
  background: #f0f0f0;
  color: #9e9890;
  border-radius: 20px;
  padding: 1px 7px;
  white-space: nowrap;
  flex-shrink: 0;
}
</style>

<style>
/* Global — zoom portal outside scoped component */
.ci-zoom-portal {
  position: fixed;
  z-index: 9999;
  background: #ffffff;
  border: 1.5px solid #d8eae6;
  border-radius: 12px;
  padding: 8px;
  box-shadow: 0 8px 24px rgba(0,0,0,0.14);
  pointer-events: none;
}

.ci-zoom-img {
  width: 140px;
  height: 140px;
  object-fit: contain;
  display: block;
}
</style>
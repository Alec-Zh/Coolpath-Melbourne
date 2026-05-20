<script setup>
import { ref, computed } from 'vue'

const props = defineProps({
  modelValue: {
    type: String,
    default: '',
  },
  options: {
    type: Array,
    required: true,
  },
  placeholder: {
    type: String,
    default: 'Search...',
  },
  loading: {
    type: Boolean,
    default: false,
  },
})

const emit = defineEmits(['update:modelValue', 'change'])

const searchQuery = ref('')
const dropdownOpen = ref(false)

const filteredOptions = computed(() => {
  const q = searchQuery.value.trim().toLowerCase()
  if (!q) return props.options
  return props.options.filter((o) => o.toLowerCase().includes(q)).slice(0, 8)
})

function selectOption(option) {
  emit('update:modelValue', option)
  searchQuery.value = ''
  dropdownOpen.value = false
  emit('change', option)
}

function onBlur() {
  setTimeout(() => {
    dropdownOpen.value = false
  }, 150)
}

function clearSelection() {
  emit('update:modelValue', '')
  searchQuery.value = ''
  dropdownOpen.value = false
  emit('change', '')
}

function openDropdown() {
  dropdownOpen.value = true
}
</script>

<template>
  <div class="search-wrap" @blur.capture="onBlur">
    <div
      v-if="modelValue && !dropdownOpen"
      class="selected-display"
      @click="openDropdown"
    >
      <span class="selected-name">{{ modelValue }}</span>
      <button class="clear-btn" @click.stop="clearSelection" aria-label="Clear selection">
        <svg width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round">
          <line x1="18" y1="6" x2="6" y2="18" />
          <line x1="6" y1="6" x2="18" y2="18" />
        </svg>
      </button>
    </div>
    <div v-else class="input-wrap">
      <svg class="search-icon" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round">
        <circle cx="11" cy="11" r="8" />
        <line x1="21" y1="21" x2="16.65" y2="16.65" />
      </svg>
      <input
        class="search-input"
        type="text"
        :placeholder="loading ? 'Loading…' : placeholder"
        v-model="searchQuery"
        @focus="openDropdown"
        autocomplete="off"
      />
    </div>
    <ul v-if="dropdownOpen" class="dropdown" role="listbox">
      <li v-if="filteredOptions.length === 0" class="dropdown-empty">No options found</li>
      <li
        v-for="option in filteredOptions"
        :key="option"
        class="dropdown-item"
        role="option"
        @mousedown.prevent="selectOption(option)"
      >
        {{ option }}
      </li>
    </ul>
  </div>
</template>

<style scoped>
.search-wrap {
  position: relative;
  flex: 1;
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
  z-index: 1;
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
</style>

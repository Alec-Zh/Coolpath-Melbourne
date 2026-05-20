<script setup>
import { ref, onMounted, computed, watch, nextTick } from 'vue'
import NavBar from '@/components/NavBar.vue'
import Footer from '@/components/Footer.vue'
import WelcomeModal from '@/components/WelcomeModal.vue'
import TemperatureAlert from '@/components/TemperatureAlert.vue'
import Directions from '@/components/DIRECTIONS.vue'
import SearchableSelect from '@/components/SearchableSelect.vue'
import mapboxgl from 'mapbox-gl'
import 'mapbox-gl/dist/mapbox-gl.css'
const photoObj = {
  library: '/libraryCover.jpg',
  museum: '/museumCover.jpg',
  park: '/parkCover.jpg',
  community: '/communityCover.jpg',
  shopping: '/shoppingCover.jpg',
}
const mapPhotoObj = {
  library: '/library.png',
  museum: '/museum.png',
  park: '/park.png',
  community: '/community.jpg',
  shopping: '/shopping.png',
}

const facilitiesObj = {
  library: '❄️  🛜  ☕️',
  museum: '🚻  🛜  ☕️',
  park: '🚻  🅿️ 💧',
  community: '🛜  ☕️  🅿️',
  shopping: '❄️  🛜  🚻',
}
const openingHoursObj = {
  library: '9:00-17:00',
  museum: '11:00-17:00',
  park: 'Open 24 hours',
  community: 'Open 24 hours',
  shopping: '10:00-20:00',
}

// API 基础地址
// const API_BASE_URL = 'http://114.132.125.151:3000/api'
const API_BASE_URL = 'https://qcbqul6ys2.execute-api.ap-southeast-2.amazonaws.com/api'

// 响应式数据
const refuges = ref([])
const loading = ref(false)
const error = ref(null)
const selectedCity = ref('')
const cityOptions = ref([])
const selectedType = ref('all')
const openStatusFilter = ref('all')
const userLocation = ref(null)
const locationPermissionDenied = ref(false)
const currentView = ref('list') // 'list' 或 'map'
const mapContainer = ref(null)
let map = null
let markers = []

// 选中的 refuge（用于地图标记点击显示详情）
const selectedRefuge = ref(null)

// 温度弹窗相关数据
const showTempAlert = ref(false)
const currentTemp = ref(0)

// 路线弹窗相关数据
const showDirections = ref(false)
const directionsRefuge = ref(null)
const showTempAlertInDirections = ref(false)
const showLocationModal = ref(false)

// Mapbox access token（从 Lambda 获取）
const initMapboxToken = async () => {
  try {
    const response = await fetch(
      'https://qcbqul6ys2.execute-api.ap-southeast-2.amazonaws.com/mapbox-token',
    )
    const data = await response.json()
    mapboxgl.accessToken = data.token
  } catch (err) {
    console.error('Failed to fetch Mapbox token:', err)
  }
}

// 获取指定位置的温度
const fetchTemperature = async (latitude, longitude) => {
  try {
    const response = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&current=temperature_2m`,
    )
    const data = await response.json()
    if (data.current && data.current.temperature_2m !== undefined) {
      return data.current.temperature_2m
    }
    return null
  } catch (err) {
    console.error('Error fetching temperature:', err)
    return null
  }
}

// 显示温度弹窗
const showTemperatureAlert = async (refuge) => {
  const temp = await fetchTemperature(refuge.latitude, refuge.longitude)
  if (temp !== null) {
    currentTemp.value = temp
    showTempAlert.value = true
  }
}

// 类型配置
const typeConfig = {
  all: { label: 'All', icon: '' },
  library: { label: 'Library', icon: '📚' },
  museum: { label: 'Museum', icon: '🏛️' },
  park: { label: 'Park', icon: '🌳' },
  community: { label: 'Community', icon: '🏘️' },
  shopping: { label: 'Shopping', icon: '🛍️' },
}

// 打开路线弹窗（从列表点击，显示温度预警）
const openDirections = (refuge) => {
  if (!userLocation.value) {
    showLocationModal.value = true
    return
  }
  directionsRefuge.value = refuge
  showTempAlertInDirections.value = true
  showDirections.value = true
}

// 打开路线弹窗（从地图点击，不显示温度预警）
const openDirectionsFromMap = (refuge) => {
  if (!userLocation.value) {
    showLocationModal.value = true
    return
  }
  directionsRefuge.value = refuge
  showTempAlertInDirections.value = false
  showDirections.value = true
}

const getDirections = (refuge) => {
  // 如果有用户位置，使用实际经纬度作为起始点
  // 否则使用默认位置（墨尔本市中心）
  let origin
  if (userLocation.value) {
    origin = `${userLocation.value.latitude},${userLocation.value.longitude}`
  } else {
    // origin = '-37.8136,144.9631' // 墨尔本市中心默认位置
  }

  const url = `https://www.google.com/maps/dir/?api=1&origin=${origin}&destination=${refuge.latitude},${refuge.longitude}`
  window.open(url, '_blank')
}

// 获取设施列表
const getFacilities = (type) => {
  const facilitiesMap = {
    '❄️': { icon: '❄️', name: 'COOL AIR' },
    '🛜': { icon: '📶', name: 'WIFI' },
    '☕️': { icon: '☕', name: 'CAFE' },
    '🚻': { icon: '🚻', name: 'TOILET' },
    '🅿️': { icon: '🅿️', name: 'PARKING' },
    '💧': { icon: '💧', name: 'WATER' },
  }

  const facilityStr = facilitiesObj[type] || ''
  const facilityEmojis = facilityStr.split(/\s+/).filter((e) => e)

  return facilityEmojis.map((emoji) => facilitiesMap[emoji])
}

// 判断是否营业中
const isOpenNow = (type) => {
  const hours = openingHoursObj[type]

  // 24小时开放
  if (hours === 'Open 24 hours') {
    return true
  }

  // 解析营业时间，如 "9:00-17:00"
  const match = hours.match(/(\d+):(\d+)-(\d+):(\d+)/)
  if (!match) return false

  const [, openHour, openMin, closeHour, closeMin] = match.map(Number)

  const now = new Date()
  const currentHour = now.getHours()
  const currentMin = now.getMinutes()

  // 转换为分钟数便于比较
  const currentTime = currentHour * 60 + currentMin
  const openTime = openHour * 60 + openMin
  const closeTime = closeHour * 60 + closeMin

  return currentTime >= openTime && currentTime < closeTime
}

// 获取类型标签样式
const getTypeClass = (type) => {
  const typeClasses = {
    library: 'library',
    museum: 'museum',
    park: 'park',
    community: 'community',
    shopping: 'shopping',
  }
  return typeClasses[type] || ''
}

// 获取类型对应颜色
const getTypeColor = (type) => {
  const colors = {
    all: '#0d3a8f',
    library: '#3b82f6',
    museum: '#8b5cf6',
    park: '#22c55e',
    community: '#f59e0b',
    shopping: '#ec4899',
  }
  return colors[type] || '#0d3a8f'
}

// 计算两点之间的距离（使用 Haversine 公式）
const calculateDistance = (lat1, lon1, lat2, lon2) => {
  const R = 6371 // 地球半径（公里）
  const dLat = ((lat2 - lat1) * Math.PI) / 180
  const dLon = ((lon2 - lon1) * Math.PI) / 180
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos((lat1 * Math.PI) / 180) *
      Math.cos((lat2 * Math.PI) / 180) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2)
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  return R * c
}

// 获取用户位置 - 只调用一次浏览器弹框
const getUserLocation = () => {
  if (!navigator.geolocation) {
    console.log('Geolocation is not supported by your browser')
    return
  }

  navigator.geolocation.getCurrentPosition(
    (position) => {
      userLocation.value = {
        latitude: position.coords.latitude,
        longitude: position.coords.longitude,
      }
      // userLocation.value = {
      //   latitude: -37.8136,
      //   longitude: 144.9631
      // }
      locationPermissionDenied.value = false
      console.log('Location obtained:', userLocation.value)
    },
    (error) => {
      // 用户拒绝权限
      if (error.code === error.PERMISSION_DENIED) {
        locationPermissionDenied.value = true
        console.log('Location permission denied by user')
      } else {
        console.log('Location access unavailable')
      }
    },
    {
      enableHighAccuracy: true,
      timeout: 10000,
      maximumAge: 60000,
    },
  )
}

// 计算距离显示文本
const getDistanceText = (refuge) => {
  if (!userLocation.value || !refuge.latitude || !refuge.longitude) {
    return ''
  }

  const distance = calculateDistance(
    userLocation.value.latitude,
    userLocation.value.longitude,
    refuge.latitude,
    refuge.longitude,
  )

  if (distance < 1) {
    return `${Math.round(distance * 1000)} m`
  } else {
    return `${distance.toFixed(1)} km`
  }
}

// 计算距离数值（用于排序）
const getDistanceValue = (refuge) => {
  if (!userLocation.value || !refuge.latitude || !refuge.longitude) {
    return Infinity
  }

  return calculateDistance(
    userLocation.value.latitude,
    userLocation.value.longitude,
    refuge.latitude,
    refuge.longitude,
  )
}

// 获取所有城市列表
const fetchCities = async () => {
  try {
    const response = await fetch(`${API_BASE_URL}/cool-refuges`)
    const result = await response.json()
    if (result.success && result.data) {
      const cities = [...new Set(result.data.map((r) => r.city).filter(Boolean))]
      cityOptions.value = cities.sort()
    }
  } catch (err) {
    console.error('Error fetching cities:', err)
  }
}

// 获取数据
const fetchRefuges = async () => {
  loading.value = true
  error.value = null

  // 数据更新时关闭弹框
  selectedRefuge.value = null
  showTempAlert.value = false

  try {
    let url = `${API_BASE_URL}/cool-refuges`
    const params = new URLSearchParams()

    // 添加搜索参数
    if (selectedCity.value) {
      params.append('city', selectedCity.value)
    }

    // 添加类型参数
    if (selectedType.value !== 'all') {
      params.append('type', selectedType.value)
    }

    if (params.toString()) {
      url += `?${params.toString()}`
    }

    const response = await fetch(url)
    const result = await response.json()

    if (result.success) {
      refuges.value = result.data
    } else {
      error.value = 'Failed to load refuges'
    }
  } catch (err) {
    console.error('Error fetching refuges:', err)
    error.value = 'Failed to connect to server'
  } finally {
    loading.value = false
  }
}

// 筛选城市
const handleSearch = async () => {
  currentPage.value = 1
  await fetchRefuges()

  // 如果在地图视图，刷新标记
  if (currentView.value === 'map' && map) {
    nextTick(() => {
      addMarkers()
    })
  }
}

// 选择类型
const selectType = async (type) => {
  console.log('Selecting type:', type)
  selectedType.value = type
  currentPage.value = 1
  await fetchRefuges()
  console.log('Fetched refuges:', refuges.value.length)

  // 如果在地图视图，刷新标记
  if (currentView.value === 'map') {
    console.log('In map view, refreshing markers...')
    nextTick(() => {
      if (map) {
        console.log('Map exists, calling addMarkers')
        addMarkers()
      } else {
        console.log('Map does not exist')
      }
    })
  }
}

// 分页相关
const currentPage = ref(1)
const pageSize = 6

// 按营业状态筛选后的数据
const filteredRefuges = computed(() => {
  if (openStatusFilter.value === 'all') {
    return refuges.value
  }

  return refuges.value.filter((refuge) => {
    const open = isOpenNow(refuge.type)
    return openStatusFilter.value === 'open' ? open : !open
  })
})

// 计算属性：统计数量
const refugeCount = computed(() => filteredRefuges.value.length)

// 分页后的数据（按距离从小到大排序）
const paginatedRefuges = computed(() => {
  // 先排序
  const sorted = [...filteredRefuges.value].sort((a, b) => {
    const distA = getDistanceValue(a)
    const distB = getDistanceValue(b)
    if (distA === Infinity && distB === Infinity) return 0
    if (distA === Infinity) return 1
    if (distB === Infinity) return -1
    return distA - distB
  })

  const start = (currentPage.value - 1) * pageSize
  return sorted.slice(start, start + pageSize)
})

// 总页数
const totalPages = computed(() => Math.ceil(filteredRefuges.value.length / pageSize))

// 显示的页码（最多显示5个）
const visiblePages = computed(() => {
  const total = totalPages.value
  const current = currentPage.value
  if (total <= 5) return Array.from({ length: total }, (_, i) => i + 1)
  let start = Math.max(1, current - 2)
  let end = Math.min(total, start + 4)
  if (end - start < 4) start = Math.max(1, end - 4)
  return Array.from({ length: end - start + 1 }, (_, i) => start + i)
})

const goToPage = (page) => {
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page
    window.scrollTo({ top: 0, behavior: 'smooth' })
  }
}

// 初始化地图
const initMap = () => {
  if (!mapContainer.value) return

  // 默认中心点（墨尔本）
  const center = [144.9631, -37.8136]

  map = new mapboxgl.Map({
    container: mapContainer.value,
    style: 'mapbox://styles/mapbox/streets-v12',
    center: center,
    zoom: 12,
  })

  // 添加导航控件
  map.addControl(new mapboxgl.NavigationControl())

  // 地图加载完成后添加标记
  map.on('load', () => {
    console.log('Map loaded event fired')
    addMarkers()
  })

  // 添加错误处理
  map.on('error', (e) => {
    console.error('Map error:', e)
  })
}

// 添加标记到地图
const addMarkers = () => {
  console.log('Adding markers, refuges count:', filteredRefuges.value.length)
  console.log('Map exists:', !!map)

  // 清除现有标记
  markers.forEach((marker) => marker.remove())
  markers = []

  if (!map) {
    console.log('No map to display')
    return
  }

  if (!filteredRefuges.value.length) {
    console.log('No refuges to display')
    return
  }

  console.log('Adding markers to map...')

  // 10km 距离限制
  const MAX_DISTANCE_KM = 5

  // 过滤出 10km 以内的避暑场所
  const nearbyRefuges = filteredRefuges.value.filter((refuge) => {
    // if (!refuge.longitude || !refuge.latitude) {
    //   return false
    // }

    // 如果没有用户位置，显示所有标记
    // if (!userLocation.value) {
    //   return true
    // }

    // 计算距离
    const distance = calculateDistance(
      userLocation.value.latitude,
      userLocation.value.longitude,
      refuge.latitude,
      refuge.longitude,
    )

    return distance <= MAX_DISTANCE_KM
  })

  console.log(`Showing ${nearbyRefuges.length} refuges within ${MAX_DISTANCE_KM}km`)

  nearbyRefuges.forEach((refuge) => {
    if (!refuge.longitude || !refuge.latitude) {
      return
    }

    // 创建标记元素
    const el = document.createElement('div')
    el.className = 'map-marker'
    el.style.width = '36px'
    el.style.height = '36px'
    el.style.borderRadius = '50%'
    el.style.border = '2px solid #0056b3'
    el.style.boxShadow = '0 2px 4px rgba(0,0,0,0.3)'
    el.style.cursor = 'pointer'
    el.style.backgroundImage = `url(${mapPhotoObj[refuge.type]})`
    el.style.backgroundSize = 'cover'
    el.style.backgroundPosition = 'center'

    // 点击标记显示详情卡片和温度弹窗
    el.addEventListener('click', (e) => {
      e.stopPropagation()
      selectedRefuge.value = refuge
      // 显示温度弹窗
      showTemperatureAlert(refuge)
    })

    // 添加标记
    const marker = new mapboxgl.Marker(el).setLngLat([refuge.longitude, refuge.latitude]).addTo(map)

    markers.push(marker)
  })

  console.log('Added', markers.length, 'markers')

  // 如果有标记，调整地图视野以包含所有标记
  if (markers.length > 0) {
    const bounds = new mapboxgl.LngLatBounds()
    nearbyRefuges.forEach((refuge) => {
      if (refuge.longitude && refuge.latitude) {
        bounds.extend([refuge.longitude, refuge.latitude])
      }
    })
    map.fitBounds(bounds, {
      padding: 50,
      maxZoom: 14,
      duration: 0,
    })
  }
}

// 获取标记颜色
const getMarkerColor = (type) => {
  const colors = {
    library: '#3b82f6',
    museum: '#8b5cf6',
    park: '#22c55e',
    community: '#f59e0b',
    shopping: '#ec4899',
  }
  return colors[type] || '#6b7280'
}

// 切换视图
const switchView = (view) => {
  console.log('Switching to view:', view)
  currentView.value = view
  if (view === 'map') {
    nextTick(() => {
      console.log('In nextTick, mapContainer:', mapContainer.value)
      if (!map) {
        console.log('Initializing map...')
        initMap()
      } else {
        console.log('Resizing map and adding markers...')
        map.resize()
        addMarkers()
      }
    })
  }
}

// 监听数据变化，更新地图标记
watch(refuges, () => {
  if (currentView.value === 'map' && map) {
    addMarkers()
  }
})

// 监听营业状态筛选变化，重置分页并刷新地图
watch(openStatusFilter, () => {
  currentPage.value = 1
  if (currentView.value === 'map' && map) {
    addMarkers()
  }
})

// 页面加载时获取用户位置和数据
onMounted(async () => {
  // 先获取 Mapbox token
  await initMapboxToken()

  // 自动请求位置权限（会触发浏览器弹框）
  getUserLocation()

  // 获取城市列表
  fetchCities()

  // 获取数据
  fetchRefuges()
})
</script>

<template>
  <div class="page">
    <WelcomeModal />
    <NavBar :show-alert-button="false" />

    <div class="refuges-header">
      <div class="refuges-header-content">
        <div class="header-top">
          <div class="header-text">
            <h1 class="refuges-title">Stay Safe and Cool Today</h1>
            <p class="refuges-description">
              Find public spaces with air conditioning nearby to escape the summer heat safely.
            </p>
            <!-- <router-link to="/safety-plan" class="safety-plan-btn">Safety Plan</router-link> -->
          </div>
          <div class="current-location" v-if="userLocation">
            <span class="location-icon">📍</span>
            <span class="location-text">CURRENT LOCATION</span>
            <span class="location-result">{{ refugeCount }} Refuges found</span>
          </div>
        </div>

        <div class="search-container">
          <SearchableSelect
            v-model="selectedCity"
            :options="cityOptions"
            placeholder="Search by city…"
            :loading="loading"
            @change="handleSearch"
          />
          <button class="search-button" @click="handleSearch" :disabled="loading">
            {{ loading ? 'Loading...' : 'FIND' }}
          </button>
        </div>

        <div class="controls-container">
          <div class="view-toggle">
            <button
              class="view-button"
              :class="{ active: currentView === 'list' }"
              @click="switchView('list')"
            >
              📋 List
            </button>
            <button
              class="view-button"
              :class="{ active: currentView === 'map' }"
              @click="switchView('map')"
            >
              🗺️ Map
            </button>
          </div>
          <div class="category-filters">
            <select class="status-filter-select" v-model="openStatusFilter">
              <option value="all">All Status</option>
              <option value="open">🟢 Open Now</option>
              <option value="closed">🔴 Closed</option>
            </select>
            <button
              v-for="(config, type) in typeConfig"
              :key="type"
              class="filter-button"
              :class="{ active: selectedType === type }"
              :style="
                selectedType === type
                  ? {
                      backgroundColor: getTypeColor(type),
                      borderColor: getTypeColor(type),
                      color: '#ffffff',
                    }
                  : {}
              "
              @click="selectType(type)"
            >
              <span v-if="config.icon">{{ config.icon }}</span>
              {{ config.label }}
            </button>
          </div>
        </div>

        <!-- 位置权限被拒绝提示 -->
        <div v-if="locationPermissionDenied" class="location-denied-prompt">
          <div class="location-denied-content">
            <span class="location-icon-large">📍</span>
            <h3>Location Access Required</h3>
            <p>
              You have denied location access. To see distances to cool refuges, please enable
              location permission in your browser settings:
            </p>
            <div class="browser-instructions">
              <div class="instruction-item">
                <strong>Chrome / Edge:</strong> Click the 🔒 lock icon in the address bar → Site
                settings → Location → Allow
              </div>
              <div class="instruction-item">
                <strong>Firefox:</strong> Click the 🔒 lock icon in the address bar → Permissions →
                Location → Allow
              </div>
              <div class="instruction-item">
                <strong>Safari:</strong> Safari menu → Settings → Websites → Location → Allow for
                this website
              </div>
            </div>
            <button @click="locationPermissionDenied = false" class="btn-dismiss">Got it</button>
          </div>
        </div>

        <!-- 无位置权限弹窗 -->
        <div
          v-if="showLocationModal"
          class="location-modal-overlay"
          @click.self="showLocationModal = false"
        >
          <div class="location-modal">
            <span class="location-icon-large">📍</span>
            <h3>Location Required</h3>
            <p>Please enable location permission to get directions.</p>
            <div class="browser-instructions">
              <div class="instruction-item">
                <strong>Chrome / Edge:</strong> Click the 🔒 lock icon in the address bar → Site
                settings → Location → Allow
              </div>
              <div class="instruction-item">
                <strong>Firefox:</strong> Click the 🔒 lock icon in the address bar → Permissions →
                Location → Allow
              </div>
              <div class="instruction-item">
                <strong>Safari:</strong> Safari menu → Settings → Websites → Location → Allow for
                this website
              </div>
            </div>
            <button @click="showLocationModal = false" class="btn-dismiss">Got it</button>
          </div>
        </div>

        <!-- 错误提示 -->
        <div v-if="error" class="error-message">
          {{ error }}
          <button @click="fetchRefuges" class="retry-button">Retry</button>
        </div>

        <!-- 加载状态 -->
        <div v-if="loading && currentView === 'list'" class="loading-message">
          Loading refuges...
        </div>

        <div class="content-container" v-if="!error">
          <!-- 列表视图 -->
          <div v-if="currentView === 'list'" class="refuges-list">
            <!-- 加载状态 -->
            <div v-if="loading" class="loading-message">Loading refuges...</div>
            <!-- 没有数据时显示 -->
            <div v-if="!loading && filteredRefuges.length === 0" class="no-data">
              No refuges found. Try adjusting your search or filters.
            </div>

            <!-- 数据列表 -->
            <div
              v-for="refuge in paginatedRefuges"
              :key="refuge.id"
              v-show="!loading"
              class="refuge-card"
              :class="getTypeClass(refuge.type)"
            >
              <div class="refuge-image-wrapper">
                <span class="image-type-label" :class="getTypeClass(refuge.type)">
                  {{ typeConfig[refuge.type]?.label }}
                </span>
                <div class="refuge-image">
                  <img :src="photoObj[refuge.type]" :alt="refuge.name" />
                </div>
                <span class="refuge-distance-badge" v-if="getDistanceText(refuge)">
                  {{ getDistanceText(refuge) }}
                </span>
              </div>
              <div class="refuge-info">
                <div class="refuge-header">
                  <h3 class="refuge-name">{{ refuge.name }}</h3>
                </div>
                <div class="refuge-address">
                  <span>📍 {{ refuge.street }}, {{ refuge.city }}</span>
                </div>
                <div class="refuge-tags">
                  <!-- <span class="tag">{{ getTagText(refuge.type) }}</span> -->
                  <span class="tag" v-if="refuge.openingHours">{{
                    openingHoursObj[refuge.type]
                  }}</span>
                </div>
                <div class="refuge-facilities">
                  <span class="facility-label">AVAILABLE FACILITIES</span>
                  <div class="facility-text">{{ facilitiesObj[refuge.type] }}</div>
                </div>
                <div class="refuge-buttons">
                  <button class="btn-direction" @click="openDirections(refuge)">
                    GET DIRECTIONS
                  </button>
                  <!-- <button class="btn-details">DETAILS</button> -->
                </div>
              </div>
            </div>

            <!-- 分页组件 -->
            <div v-if="!loading && filteredRefuges.length > 0" class="pagination">
              <button
                class="page-btn prev-next"
                :disabled="currentPage === 1"
                @click="goToPage(currentPage - 1)"
              >
                ‹ Previous
              </button>

              <button
                v-for="page in visiblePages"
                :key="page"
                class="page-btn"
                :class="{ active: page === currentPage }"
                @click="goToPage(page)"
              >
                {{ page }}
              </button>

              <button
                class="page-btn prev-next"
                :disabled="currentPage === totalPages"
                @click="goToPage(currentPage + 1)"
              >
                Next ›
              </button>
            </div>
            <p class="pagination-info" v-if="!loading && filteredRefuges.length > 0">
              Showing
              <strong
                >{{ (currentPage - 1) * pageSize + 1 }}-{{
                  Math.min(currentPage * pageSize, filteredRefuges.length)
                }}</strong
              >
              of <strong>{{ filteredRefuges.length }}</strong> entries
            </p>
          </div>

          <!-- 地图视图 -->
          <div v-show="currentView === 'map'" class="map-view">
            <div ref="mapContainer" class="map-container">
              <!-- 温度预警弹窗 - 在地图内部左下角 -->
              <TemperatureAlert
                v-if="showTempAlert"
                :temperature="currentTemp"
                :show="showTempAlert"
                @close="showTempAlert = false"
                class="map-temp-alert"
              />
            </div>

            <!-- 地图标记详情卡片 -->
            <div v-if="selectedRefuge" class="map-detail-card">
              <button class="close-btn" @click="selectedRefuge = null">×</button>
              <div class="detail-image">
                <img :src="photoObj[selectedRefuge.type]" :alt="selectedRefuge.name" />
                <span class="open-badge" :class="{ closed: !isOpenNow(selectedRefuge.type) }">
                  {{ isOpenNow(selectedRefuge.type) ? 'OPEN NOW' : 'CLOSED' }}
                </span>
              </div>
              <div class="detail-content">
                <h3 class="detail-name">{{ selectedRefuge.name }}</h3>
                <p class="detail-address">
                  <span v-if="getDistanceText(selectedRefuge)"
                    >{{ getDistanceText(selectedRefuge) }} •
                  </span>
                  {{ selectedRefuge.street }}, {{ selectedRefuge.city }}
                </p>

                <div class="detail-hours">
                  <div class="hours-icon">🕐</div>
                  <div class="hours-text">
                    <span class="hours-label">FULL HOURS</span>
                    <span class="hours-value"
                      >Open today: {{ openingHoursObj[selectedRefuge.type] }}</span
                    >
                  </div>
                </div>

                <div class="detail-facilities">
                  <div
                    v-for="(facility, index) in getFacilities(selectedRefuge.type)"
                    :key="index"
                    class="facility-item"
                  >
                    <span class="facility-icon">{{ facility.icon }}</span>
                    <span class="facility-name">{{ facility.name }}</span>
                  </div>
                </div>

                <button class="btn-directions" @click="openDirectionsFromMap(selectedRefuge)">
                  <span class="direction-icon">🚶</span>
                  GET DIRECTIONS
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <Footer />

    <!-- 路线弹窗 -->
    <Directions
      v-if="showDirections && directionsRefuge"
      :refuge="directionsRefuge"
      :user-location="userLocation"
      :show-temp-alert="showTempAlertInDirections"
      @close="showDirections = false"
    />
  </div>
</template>

<style scoped>
.page {
  min-height: 100vh;
  background-color: #f5f5f5;
}

.refuges-header {
  background-color: #ffffff;
  padding: 2rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  width: 100%;
}

.refuges-header-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0;
}

.header-top {
  display: flex;
  border-radius: 14px;
  background:
    linear-gradient(135deg, rgba(13, 58, 143, 0.95), rgba(11, 127, 121, 0.88)),
    linear-gradient(
      45deg,
      rgba(255, 255, 255, 0.12) 25%,
      transparent 25% 50%,
      rgba(255, 255, 255, 0.12) 50% 75%,
      transparent 75%
    );
  color: #fff;
  justify-content: space-between;
  padding: 1.25rem;
  align-items: flex-start;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.header-text {
  flex: 1;
  min-width: 300px;
}

.refuges-title {
  font-size: 2rem;
  font-weight: 700;
  /* color: #1a3a8f; */
  margin-bottom: 0.5rem;
}

.refuges-description {
  font-size: 1rem;
  /* color: #666; */
  margin-bottom: 1.5rem;
}

.safety-plan-btn {
  display: inline-flex;
  align-items: center;
  background-color: #0d3a8f;
  color: #ffffff;
  border: none;
  padding: 1rem 2rem;
  border-radius: 50px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  text-decoration: none;
}

.safety-plan-btn:hover {
  background-color: #1a4bb8;
  transform: translateY(-2px);
  color: #ffffff;
}

.current-location {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background-color: #e6f0ff;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  width: fit-content;
}

.location-icon {
  font-size: 1rem;
}

.location-text {
  font-size: 0.75rem;
  font-weight: 600;
  color: #1a3a8f;
  letter-spacing: 0.5px;
}

.location-result {
  font-size: 0.875rem;
  color: #333;
  font-weight: 500;
}

.search-container {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
  width: 100%;
}

.search-button {
  background-color: #0d3a8f;
  color: #ffffff;
  border: none;
  padding: 0.75rem 2rem;
  border-radius: 4px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.3s ease;
  white-space: nowrap;
}

.search-button:hover:not(:disabled) {
  background-color: #1a4bb8;
}

.search-button:disabled {
  background-color: #94a3b8;
  cursor: not-allowed;
}

.controls-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 1rem;
}

.view-button {
  padding: 0.5rem 1.5rem;
  border: 1px solid #e2e8f0;
  background-color: #ffffff;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.view-button.active {
  background-color: #0056b3;
  color: #ffffff;
  border-color: #0056b3;
}

.view-button:hover:not(.active) {
  background-color: #f8f9fa;
  border-color: #dee2e6;
}

.view-toggle {
  display: flex;
  gap: 0;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.view-toggle .view-button {
  border-radius: 0;
  border: none;
  border-right: 1px solid #e2e8f0;
}

.view-toggle .view-button:last-child {
  border-right: none;
}

.category-filters {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  align-items: center;
}

.status-filter-select {
  padding: 0.5rem 1rem;
  border: 1px solid #e2e8f0;
  background-color: #f8fafc;
  border-radius: 10px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  outline: none;
  color: #1a1a1a;
  appearance: auto;
}

.status-filter-select:focus {
  border-color: #0d3a8f;
}

.filter-button {
  padding: 0.5rem 1rem;
  border: 1px solid #e2e8f0;
  background-color: #f8fafc;
  border-radius: 10px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.filter-button.active {
  background-color: #0d3a8f;
  color: #ffffff;
  border-color: #0d3a8f;
}

.filter-button:hover:not(.active) {
  background-color: #e2e8f0;
}

.content-container {
  display: flex;
  gap: 2rem;
  margin-top: 2rem;
  flex-wrap: wrap;
}

.refuges-list {
  flex: 1;
  min-width: 0;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem;
  align-content: start;
}

.map-view {
  flex: 1;
  min-width: 300px;
  height: 600px;
  position: relative;
}

.map-container {
  width: 100%;
  height: 100%;
  border-radius: 8px;
  overflow: hidden;
  max-height: 100vh;
  overflow-y: auto;
  position: relative;
}

/* 地图内的温度弹窗 */
.map-temp-alert {
  position: absolute !important;
  bottom: 20px;
  left: 20px;
}

/* 地图详情卡片 */
.map-detail-card {
  position: absolute;
  bottom: 20px;
  right: 20px;
  width: 320px;
  background: white;
  border-radius: 16px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.15);
  overflow: hidden;
  z-index: 1000;
  animation: slideUp 0.3s ease;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.map-detail-card .close-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: none;
  background: rgba(255, 255, 255, 0.9);
  font-size: 20px;
  cursor: pointer;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #333;
}

.map-detail-card .detail-image {
  position: relative;
  width: 100%;
  height: 140px;
  overflow: hidden;
}

.map-detail-card .detail-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.map-detail-card .open-badge {
  position: absolute;
  top: 10px;
  left: 10px;
  background: #22c55e;
  color: white;
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 11px;
  font-weight: 600;
}

.map-detail-card .open-badge.closed {
  background: #ef4444;
}

.map-detail-card .favorite-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: none;
  background: white;
  font-size: 18px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #1a3a8f;
}

.map-detail-card .detail-content {
  padding: 16px;
}

.map-detail-card .detail-name {
  font-size: 18px;
  font-weight: 700;
  color: #1a1a1a;
  margin: 0 0 6px 0;
}

.map-detail-card .detail-address {
  font-size: 13px;
  color: #666;
  margin: 0 0 16px 0;
}

.map-detail-card .detail-hours {
  display: flex;
  align-items: center;
  gap: 12px;
  background: #f5f5f5;
  padding: 12px;
  border-radius: 12px;
  margin-bottom: 16px;
}

.map-detail-card .hours-icon {
  width: 36px;
  height: 36px;
  background: #1a3a8f;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
}

.map-detail-card .hours-text {
  display: flex;
  flex-direction: column;
}

.map-detail-card .hours-label {
  font-size: 10px;
  color: #999;
  letter-spacing: 0.5px;
}

.map-detail-card .hours-value {
  font-size: 14px;
  font-weight: 600;
  color: #1a1a1a;
}

.map-detail-card .detail-facilities {
  display: flex;
  justify-content: space-between;
  margin-bottom: 16px;
}

.map-detail-card .facility-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.map-detail-card .facility-icon {
  width: 48px;
  height: 48px;
  background: #f0f4ff;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
}

.map-detail-card .facility-name {
  font-size: 10px;
  color: #666;
  font-weight: 500;
}

.map-detail-card .btn-directions {
  width: 100%;
  padding: 14px;
  background: #1a3a8f;
  color: white;
  border: none;
  border-radius: 12px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  transition: background 0.2s;
}

.map-detail-card .btn-directions:hover {
  background: #142d73;
}

.map-detail-card .direction-icon {
  font-size: 16px;
}

.sidebar {
  width: 300px;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.refuge-card {
  background-color: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1rem;
  display: flex;
  gap: 1rem;
  transition: all 0.3s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.refuge-card:hover {
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

.refuge-card.library {
  border-top: 4px solid #3b82f6;
}

.refuge-card.museum {
  border-top: 4px solid #8b5cf6;
}

.refuge-card.park {
  border-top: 4px solid #22c55e;
}

.refuge-card.community {
  border-top: 4px solid #f59e0b;
}

.refuge-card.shopping {
  border-top: 4px solid #ec4899;
}

.refuge-image-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

.image-type-label {
  padding: 2px 12px;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 700;
  color: #ffffff;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.image-type-label.library {
  background-color: #3b82f6;
}

.image-type-label.museum {
  background-color: #8b5cf6;
}

.image-type-label.park {
  background-color: #22c55e;
}

.image-type-label.community {
  background-color: #f59e0b;
}

.image-type-label.shopping {
  background-color: #ec4899;
}

.refuge-image {
  width: 90px;
  height: 90px;
  border-radius: 8px;
  overflow: hidden;
  flex-shrink: 0;
}

.refuge-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.refuge-info {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.refuge-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 0.5rem;
}

.refuge-name {
  font-size: 1.25rem;
  font-weight: 600;
  color: #1a3a8f;
  margin: 0;
}

.refuge-distance-badge {
  font-size: 25px;
  font-weight: 600;
  color: #3b82f6;
  background-color: #e0f2fe;
  padding: 10px;
  border-radius: 12px;
  white-space: nowrap;
}

.refuge-address {
  font-size: 0.875rem;
  color: #666;
  margin-bottom: 0.75rem;
}

.refuge-features {
  display: flex;
  gap: 0.5rem;
  flex-wrap: wrap;
  margin-bottom: 0.75rem;
}

.feature {
  font-size: 0.75rem;
  color: #666;
  display: flex;
  align-items: center;
  gap: 0.25rem;
}

.refuge-hours {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
  font-size: 0.875rem;
}

.hours-label {
  font-weight: 500;
  color: #333;
}

.hours-value {
  color: #666;
}

.refuge-buttons {
  display: flex;
  gap: 0.75rem;
  margin-top: auto;
}

.btn-direction {
  background-color: #0d3a8f;
  color: #ffffff;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.btn-direction:hover {
  background-color: #1a4bb8;
}

.btn-details {
  background-color: #f8fafc;
  color: #333;
  border: 1px solid #e2e8f0;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-details:hover {
  background-color: #e2e8f0;
}

.refuge-tags {
  display: flex;
  gap: 0.75rem;
  margin-bottom: 0.75rem;
  flex-wrap: wrap;
}

.tag {
  background-color: #e0f2fe;
  color: #0284c7;
  padding: 0.25rem 0.5rem;
  border-radius: 12px;
  font-size: 0.75rem;
  font-weight: 500;
}

.refuge-facilities {
  margin-bottom: 1rem;
}

.facility-label {
  display: block;
  font-size: 0.75rem;
  font-weight: 600;
  color: #666;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.facility-text {
  font-size: 20px;
  color: #333;
  line-height: 1.4;
  letter-spacing: 5px;
}

.sidebar-card {
  border-radius: 8px;
  padding: 1.5rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.health-card {
  background-color: #fecaca;
  border-top: 4px solid #f87171;
}

.map-card {
  background-color: #ffffff;
  border-top: 4px solid #3b82f6;
}

.card-title {
  font-size: 1rem;
  font-weight: 600;
  color: #333;
  margin-bottom: 1rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.health-card .card-title {
  color: #dc2626;
  font-size: 1.125rem;
  font-weight: 700;
}

.contact-item {
  margin-bottom: 1rem;
  padding: 1rem;
  background-color: #ffffff;
  border-radius: 6px;
  border: 1px solid #fecaca;
}

.contact-item:last-child {
  margin-bottom: 0;
}

.contact-item.emergency {
  background-color: #ffffff;
  padding: 1rem;
  border-radius: 6px;
  margin-top: 0.5rem;
  border: 1px solid #fecaca;
}

.contact-label {
  display: block;
  font-size: 0.75rem;
  font-weight: 600;
  color: #dc2626;
  margin-bottom: 0.25rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.contact-number {
  display: block;
  font-size: 1.25rem;
  font-weight: 700;
  color: #dc2626;
  margin-bottom: 0.25rem;
}

.contact-number.emergency {
  color: #dc2626;
  font-size: 2rem;
  font-weight: 800;
}

.contact-note {
  display: block;
  font-size: 0.75rem;
  color: #dc2626;
}

.map-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.view-full {
  font-size: 0.75rem;
  font-weight: 500;
  color: #3b82f6;
  text-decoration: none;
  transition: color 0.3s ease;
}

.view-full:hover {
  color: #1d4ed8;
}

.map-image {
  position: relative;
  border-radius: 6px;
  overflow: hidden;
}

.map-image img {
  width: 100%;
  height: auto;
  display: block;
}

.map-info {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background-color: #3b82f6;
  color: #ffffff;
  padding: 0.5rem;
  font-size: 0.75rem;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.map-info::before {
  content: '📍';
  font-size: 0.875rem;
}

.error-message {
  background-color: #fee2e2;
  color: #dc2626;
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
  display: flex;
  align-items: center;
  gap: 1rem;
}

.retry-button {
  background-color: #dc2626;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.875rem;
}

.retry-button:hover {
  background-color: #b91c1c;
}

.location-denied-prompt {
  background-color: #fef3c7;
  border: 2px solid #f59e0b;
  border-radius: 12px;
  padding: 1.5rem;
  margin: 1.5rem 0;
}

.location-denied-content {
  text-align: center;
}

.location-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 3000;
  padding: 20px;
}

.location-modal {
  background: white;
  border-radius: 16px;
  padding: 2rem;
  max-width: 480px;
  width: 100%;
  text-align: center;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
}

.location-icon-large {
  font-size: 2.5rem;
  display: block;
  margin-bottom: 0.75rem;
}

.location-modal h3 {
  font-size: 1.125rem;
  font-weight: 600;
  color: #92400e;
  margin-bottom: 0.5rem;
}

.location-modal p {
  font-size: 0.875rem;
  color: #a16207;
  margin-bottom: 1rem;
  line-height: 1.5;
}

.browser-instructions {
  background-color: #ffffff;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1rem;
  text-align: left;
}

.instruction-item {
  font-size: 0.8rem;
  color: #713f12;
  margin-bottom: 0.75rem;
  padding: 0.5rem;
  background-color: #fef9c3;
  border-radius: 6px;
  line-height: 1.4;
}

.instruction-item:last-child {
  margin-bottom: 0;
}

.instruction-item strong {
  color: #92400e;
  display: block;
  margin-bottom: 0.25rem;
}

.btn-dismiss {
  background-color: #f59e0b;
  color: white;
  border: none;
  padding: 0.5rem 1.5rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.btn-dismiss:hover {
  background-color: #d97706;
}

.pagination {
  grid-column: 1 / -1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.4rem;
  padding: 1.5rem 0 0.5rem;
  width: 100%;
}

.page-btn {
  min-width: 40px;
  height: 40px;
  padding: 0 0.75rem;
  border: 1px solid #d1d5db;
  background-color: #ffffff;
  color: #374151;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.page-btn:hover:not(:disabled) {
  border-color: #1d3f99;
  color: #1d3f99;
  background-color: #eff6ff;
}

.page-btn.active {
  background-color: #1d3f99;
  border-color: #1d3f99;
  color: #ffffff;
  font-weight: 700;
}

.page-btn.prev-next {
  padding: 0 1rem;
  color: #374151;
}

.page-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.pagination-info {
  grid-column: 1 / -1;
  text-align: center;
  font-size: 0.875rem;
  color: #6b7280;
}

.pagination-info strong {
  color: #111827;
}

.loading-message {
  text-align: center;
  padding: 2rem;
  color: #666;
  font-size: 1rem;
}

.no-data {
  text-align: center;
  padding: 3rem;
  color: #666;
  background-color: #f8fafc;
  border-radius: 8px;
  font-size: 1rem;
}

@media (max-width: 768px) {
  .content-container {
    flex-direction: column;
  }

  .refuges-list {
    grid-template-columns: 1fr;
  }

  .refuge-card {
    flex-direction: column;
  }

  .refuge-image-wrapper {
    width: 100%;
  }

  .refuge-image {
    width: 100%;
    height: 200px;
  }

  .tips-container {
    flex-direction: column;
  }

  .tip-item {
    width: 100%;
  }
}

.using-refuges-section {
  background-color: #ebeff7;
  border-radius: 12px;
  padding: 2rem;
  margin-top: 3rem;
  margin-bottom: 3rem;
}

.section-title {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0619ab;
  text-align: center;
  margin-bottom: 2rem;
}

.tips-container {
  display: flex;
  justify-content: space-between;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.tip-item {
  flex: 1;
  min-width: 200px;
  text-align: center;
  padding: 1.5rem;
}

.tip-icon {
  width: 60px;
  height: 60px;
  background-color: #ffffff;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 1rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  border: 1px solid #d1e7ff;
}

.tip-icon .icon {
  font-size: 1.5rem;
  color: #0056b3;
}

.tip-title {
  font-size: 1rem;
  font-weight: 600;
  color: #000;
  margin-bottom: 0.5rem;
}

.tip-description {
  font-size: 0.875rem;
  color: #333333;
  line-height: 1.4;
}
</style>

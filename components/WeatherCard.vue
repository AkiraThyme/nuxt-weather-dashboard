<template>
  <div :class="['weather-container', theme]">
    <div class="weather-grid">
      <div class="weather-header">
        <h1>Weather</h1>
        <p class="subtitle">Fast city forecasts for any screen size.</p>

        <form class="search-bar" @submit.prevent="fetchWeather()">
          <label for="city" class="sr-only">City</label>
          <div class="search-input-wrap">
            <input
              id="city"
              v-model.trim="city"
              type="text"
              inputmode="search"
              autocomplete="off"
              placeholder="Search places worldwide..."
              :disabled="isLoading"
              @input="onCityInput"
              @focus="showSuggestions = suggestions.length > 0"
              @blur="onInputBlur"
              @keydown.down.prevent="moveActiveSuggestion(1)"
              @keydown.up.prevent="moveActiveSuggestion(-1)"
              @keydown.enter.prevent="submitActiveOrCurrent"
            />

            <ul
              v-if="showSuggestions && suggestions.length"
              class="suggestions"
              role="listbox"
              aria-label="Place suggestions"
            >
              <li
                v-for="(item, index) in suggestions"
                :key="`${item.name}-${item.lat}-${item.lon}-${index}`"
                :class="{ active: index === activeSuggestion }"
                @mousedown.prevent="selectSuggestion(item)"
                @mouseenter="activeSuggestion = index"
              >
                <span>{{ formatSuggestion(item) }}</span>
              </li>
            </ul>
          </div>

          <button type="submit" :disabled="isLoading || !city">
            <Icon name="mdi-magnify" />
            <span class="sr-only">Search weather</span>
          </button>
        </form>

        <button class="theme-toggle" type="button" @click="toggleTheme">
          <Icon :name="theme === 'dark' ? 'mdi-weather-sunny' : 'mdi-weather-night'" />
          <span>{{ theme === 'dark' ? 'Light mode' : 'Dark mode' }}</span>
        </button>
      </div>

      <div class="separator" aria-hidden="true"></div>

      <div class="weather-content" aria-live="polite">
        <Transition name="fade-slide" mode="out-in">
          <div v-if="error" key="error" class="status error">
            <p>{{ error }}</p>
          </div>

          <div v-else-if="isLoading" key="loading" class="status loading">
            <p>Loading weather...</p>
          </div>

          <div v-else-if="weather" key="weather" class="weather-result">
            <p class="city-name">{{ weather.name }}, {{ weather.sys.country }}</p>
            <div class="weather-main">
              <img :src="iconUrl" :alt="weather.weather[0].description" />
              <div>
                <h2>{{ celsius(weather.main.temp) }}°C</h2>
                <p>{{ weather.weather[0].main }}</p>
              </div>
            </div>
            <div class="weather-details-card">
              <div class="detail-card">
                <Icon name="mdi-water-percent" />
                <div class="separator"></div>
                <label>Humidity</label>
                <strong>{{ weather.main.humidity }}%</strong>
              </div>
              <div class="detail-card">
                <Icon name="mdi-weather-windy" />
                <div class="separator"></div>
                <label>Wind Speed</label>
                <strong>{{ weather.wind.speed }} m/s</strong>
              </div>
              <div class="detail-card">
                <Icon name="mdi-cloud-outline" />
                <div class="separator"></div>
                <label>Cloudiness</label>
                <strong>{{ weather.clouds.all }}%</strong>
              </div>
              <div class="detail-card">
                <Icon name="mdi-gauge" />
                <div class="separator"></div>
                <label>Pressure</label>
                <strong>{{ weather.main.pressure }} hPa</strong>
              </div>
            </div>
          </div>

          <div v-else key="empty" class="status empty">
            <p>Search for a city to see the latest weather.</p>
          </div>
        </Transition>
      </div>
    </div>

    <footer class="credits">
      <p>
        © 2025 <a href="https://github.com/AkiraThyme" target="_blank" rel="noopener noreferrer">Jerold</a>
      </p>
    </footer>
  </div>
</template>

<script setup>
const config = useRuntimeConfig()
const WEATHER_API = config.public.weatherApi

const weather = ref(null)
const city = ref('')
const theme = ref('light')
const isLoading = ref(false)
const error = ref('')
const suggestions = ref([])
const showSuggestions = ref(false)
const activeSuggestion = ref(-1)
let suggestionsRequestId = 0

const getApiKey = () => {
  try {
    const parsed = new URL(WEATHER_API)
    return parsed.searchParams.get('appid') || ''
  } catch {
    return ''
  }
}

const weatherApiKey = getApiKey()

const fetchSuggestions = async () => {
  const query = city.value.trim()
  const requestId = ++suggestionsRequestId

  if (!query || query.length < 2 || !weatherApiKey) {
    suggestions.value = []
    showSuggestions.value = false
    return
  }

  try {
    const response = await fetch(
      `https://api.openweathermap.org/geo/1.0/direct?q=${encodeURIComponent(query)}&limit=6&appid=${weatherApiKey}`,
    )

    if (!response.ok) {
      suggestions.value = []
      showSuggestions.value = false
      return
    }

    const data = await response.json()
    if (requestId !== suggestionsRequestId) return
    suggestions.value = Array.isArray(data) ? data : []
    activeSuggestion.value = suggestions.value.length ? 0 : -1
    showSuggestions.value = suggestions.value.length > 0
  } catch {
    if (requestId !== suggestionsRequestId) return
    suggestions.value = []
    showSuggestions.value = false
  }
}

const onCityInput = () => {
  fetchSuggestions()
}

const formatSuggestion = (item) => {
  const state = item.state ? `, ${item.state}` : ''
  return `${item.name}${state}, ${item.country}`
}

const selectSuggestion = (item) => {
  city.value = formatSuggestion(item)
  showSuggestions.value = false
  suggestions.value = []
  fetchWeather(item)
}

const moveActiveSuggestion = (direction) => {
  if (!suggestions.value.length) return
  const next = activeSuggestion.value + direction

  if (next < 0) {
    activeSuggestion.value = suggestions.value.length - 1
    return
  }

  if (next >= suggestions.value.length) {
    activeSuggestion.value = 0
    return
  }

  activeSuggestion.value = next
}

const submitActiveOrCurrent = () => {
  if (showSuggestions.value && activeSuggestion.value >= 0 && suggestions.value[activeSuggestion.value]) {
    selectSuggestion(suggestions.value[activeSuggestion.value])
    return
  }

  fetchWeather()
}

const onInputBlur = () => {
  setTimeout(() => {
    showSuggestions.value = false
  }, 120)
}

const fetchWeather = async (selectedPlace) => {
  if ((!city.value && !selectedPlace) || isLoading.value) return

  isLoading.value = true
  error.value = ''
  showSuggestions.value = false

  try {
    const query = selectedPlace
      ? `lat=${selectedPlace.lat}&lon=${selectedPlace.lon}`
      : `q=${encodeURIComponent(city.value)}`

    const separator = WEATHER_API.includes('?') ? '&' : '?'
    const res = await fetch(`${WEATHER_API}${separator}${query}`)
    const data = await res.json()

    if (!res.ok || Number(data.cod) >= 400) {
      weather.value = null
      error.value = data.message ? `Could not find weather: ${data.message}` : 'Unable to fetch weather right now.'
      return
    }

    weather.value = data
  } catch {
    weather.value = null
    error.value = 'Network error. Please try again.'
  } finally {
    isLoading.value = false
  }
}

const celsius = (k) => (k - 273.15).toFixed(1)
const iconUrl = computed(() =>
  weather.value
    ? `https://openweathermap.org/img/wn/${weather.value.weather[0].icon}@2x.png`
    : '',
)

const toggleTheme = () => {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
}
</script>

<template>
  <div :class="['weather-container', theme]">
    <div class="weather-grid">
      <div class="weather-header">
        <h1>Weather</h1>
        <p class="subtitle">Fast city forecasts for any screen size.</p>

        <form class="search-bar" @submit.prevent="fetchWeather">
          <label for="city" class="sr-only">City</label>
          <input
            id="city"
            v-model.trim="city"
            type="text"
            inputmode="search"
            autocomplete="off"
            placeholder="Enter city..."
            :disabled="isLoading"
          />
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
        <div v-if="error" class="status error">
          <p>{{ error }}</p>
        </div>

        <div v-else-if="isLoading" class="status loading">
          <p>Loading weather...</p>
        </div>

        <div v-else-if="weather" class="weather-result">
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

        <div v-else class="status empty">
          <p>Search for a city to see the latest weather.</p>
        </div>
      </div>
    </div>
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

const fetchWeather = async () => {
  if (!city.value || isLoading.value) return

  isLoading.value = true
  error.value = ''

  try {
    const res = await fetch(`${WEATHER_API}&q=${encodeURIComponent(city.value)}`)
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

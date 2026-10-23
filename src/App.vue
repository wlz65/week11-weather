<script setup>
import { computed, onMounted, ref } from 'vue'

const city = ref('Melbourne')
const location = ref(null)
const weather = ref(null)
const loading = ref(false)
const errorMessage = ref('')

const weatherDescriptions = {
  0: ['Clear sky', '☀️'],
  1: ['Mainly clear', '🌤️'],
  2: ['Partly cloudy', '⛅'],
  3: ['Overcast', '☁️'],
  45: ['Fog', '🌫️'],
  48: ['Rime fog', '🌫️'],
  51: ['Light drizzle', '🌦️'],
  53: ['Drizzle', '🌦️'],
  55: ['Heavy drizzle', '🌧️'],
  61: ['Light rain', '🌦️'],
  63: ['Rain', '🌧️'],
  65: ['Heavy rain', '🌧️'],
  71: ['Light snow', '🌨️'],
  73: ['Snow', '❄️'],
  75: ['Heavy snow', '❄️'],
  80: ['Rain showers', '🌦️'],
  81: ['Rain showers', '🌧️'],
  82: ['Heavy showers', '⛈️'],
  95: ['Thunderstorm', '⛈️'],
  96: ['Thunderstorm with hail', '⛈️'],
  99: ['Thunderstorm with hail', '⛈️'],
}

const condition = computed(() => {
  if (!weather.value) return { label: '', icon: '' }
  const [label, icon] = weatherDescriptions[weather.value.weather_code] ?? [
    'Mixed conditions',
    '🌡️',
  ]
  return { label, icon }
})

const localTime = computed(() => {
  if (!weather.value?.time) return ''
  return new Intl.DateTimeFormat('en-AU', {
    dateStyle: 'full',
    timeStyle: 'short',
  }).format(new Date(weather.value.time))
})

async function getWeather() {
  const query = city.value.trim()
  if (!query) {
    errorMessage.value = 'Please enter a city name.'
    return
  }

  loading.value = true
  errorMessage.value = ''

  try {
    const geocodingUrl = new URL('https://geocoding-api.open-meteo.com/v1/search')
    geocodingUrl.searchParams.set('name', query)
    geocodingUrl.searchParams.set('count', '1')
    geocodingUrl.searchParams.set('language', 'en')
    geocodingUrl.searchParams.set('format', 'json')

    const geocodingResponse = await fetch(geocodingUrl)
    if (!geocodingResponse.ok) throw new Error('Unable to search for that city.')
    const geocodingData = await geocodingResponse.json()
    const result = geocodingData.results?.[0]
    if (!result) throw new Error('No matching city was found. Try another name.')
    location.value = result

    const forecastUrl = new URL('https://api.open-meteo.com/v1/forecast')
    forecastUrl.searchParams.set('latitude', result.latitude)
    forecastUrl.searchParams.set('longitude', result.longitude)
    forecastUrl.searchParams.set(
      'current',
      'temperature_2m,relative_humidity_2m,apparent_temperature,weather_code,wind_speed_10m',
    )
    forecastUrl.searchParams.set('timezone', 'auto')

    const forecastResponse = await fetch(forecastUrl)
    if (!forecastResponse.ok) throw new Error('The weather service is temporarily unavailable.')
    weather.value = (await forecastResponse.json()).current
  } catch (error) {
    weather.value = null
    location.value = null
    errorMessage.value = error instanceof Error ? error.message : 'Something went wrong.'
  } finally {
    loading.value = false
  }
}

onMounted(getWeather)
</script>

<template>
  <main class="weather-page">
    <section class="weather-shell" aria-labelledby="page-title">
      <header class="hero">
        <p class="eyebrow">FIT5032 Week 11</p>
        <h1 id="page-title">Get Weather</h1>
        <p>Search for a city to view its current weather conditions.</p>
      </header>

      <form class="search-form" @submit.prevent="getWeather">
        <label for="city">City</label>
        <div class="search-row">
          <input
            id="city"
            v-model="city"
            name="city"
            type="search"
            autocomplete="address-level2"
            placeholder="Enter a city, for example Melbourne"
          />
          <button type="submit" :disabled="loading">
            {{ loading ? 'Loading…' : 'Get Weather' }}
          </button>
        </div>
      </form>

      <p v-if="errorMessage" class="message error" role="alert">{{ errorMessage }}</p>

      <article v-if="weather && location" class="weather-card" aria-live="polite">
        <div class="location-heading">
          <div>
            <p class="status">Current conditions</p>
            <h2>{{ location.name }}, {{ location.country }}</h2>
            <p>{{ localTime }}</p>
          </div>
          <span class="weather-icon" aria-hidden="true">{{ condition.icon }}</span>
        </div>

        <div class="temperature-row">
          <p class="temperature">{{ Math.round(weather.temperature_2m) }}°</p>
          <div>
            <p class="condition">{{ condition.label }}</p>
            <p>Feels like {{ Math.round(weather.apparent_temperature) }}°C</p>
          </div>
        </div>

        <dl class="weather-details">
          <div>
            <dt>Humidity</dt>
            <dd>{{ weather.relative_humidity_2m }}%</dd>
          </div>
          <div>
            <dt>Wind speed</dt>
            <dd>{{ weather.wind_speed_10m }} km/h</dd>
          </div>
          <div>
            <dt>Coordinates</dt>
            <dd>{{ location.latitude.toFixed(2) }}, {{ location.longitude.toFixed(2) }}</dd>
          </div>
        </dl>
      </article>

      <footer>Weather data provided by Open-Meteo · Cloudflare Pages deployment exercise</footer>
    </section>
  </main>
</template>

<style scoped>
:global(*) {
  box-sizing: border-box;
}
:global(body) {
  margin: 0;
  min-width: 320px;
  min-height: 100vh;
  color: #10233f;
  background:
    radial-gradient(circle at 12% 10%, rgba(255, 255, 255, 0.9), transparent 25rem),
    linear-gradient(145deg, #b8ddff 0%, #e8f4ff 48%, #fff7e8 100%);
  font-family:
    Inter,
    ui-sans-serif,
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    'Segoe UI',
    sans-serif;
}
button,
input {
  font: inherit;
}
.weather-page {
  display: grid;
  min-height: 100vh;
  place-items: center;
  padding: 40px 20px;
}
.weather-shell {
  width: min(760px, 100%);
}
.hero {
  margin-bottom: 28px;
}
.eyebrow,
.status {
  margin: 0 0 8px;
  color: #1764a5;
  font-size: 0.78rem;
  font-weight: 800;
  letter-spacing: 0.14em;
  text-transform: uppercase;
}
h1,
h2,
p {
  margin-top: 0;
}
h1 {
  margin-bottom: 10px;
  font-size: clamp(2.6rem, 8vw, 5rem);
  line-height: 0.98;
  letter-spacing: -0.055em;
}
.hero > p:last-child {
  max-width: 42rem;
  color: #4c6079;
  font-size: 1.08rem;
}
.search-form,
.weather-card {
  border: 1px solid rgba(255, 255, 255, 0.8);
  border-radius: 24px;
  background: rgba(255, 255, 255, 0.74);
  box-shadow: 0 20px 60px rgba(28, 75, 116, 0.14);
  backdrop-filter: blur(18px);
}
.search-form {
  margin-bottom: 18px;
  padding: 22px;
}
label {
  display: block;
  margin-bottom: 8px;
  font-size: 0.9rem;
  font-weight: 700;
}
.search-row {
  display: flex;
  gap: 10px;
}
input {
  width: 100%;
  border: 1px solid #b9c9da;
  border-radius: 14px;
  outline: none;
  padding: 13px 15px;
  background: #fff;
  color: #10233f;
}
input:focus {
  border-color: #1764a5;
  box-shadow: 0 0 0 4px rgba(23, 100, 165, 0.12);
}
button {
  flex: 0 0 auto;
  border: 0;
  border-radius: 14px;
  padding: 13px 20px;
  color: #fff;
  background: #1764a5;
  cursor: pointer;
  font-weight: 800;
}
button:hover {
  background: #0e4f86;
}
button:disabled {
  cursor: wait;
  opacity: 0.65;
}
.message {
  border-radius: 14px;
  padding: 13px 16px;
}
.error {
  color: #8f241f;
  background: #fff0ef;
}
.weather-card {
  padding: clamp(24px, 6vw, 42px);
}
.location-heading,
.temperature-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20px;
}
.location-heading h2 {
  margin-bottom: 5px;
  font-size: clamp(1.5rem, 4vw, 2.2rem);
}
.location-heading p:last-child,
.temperature-row p:last-child {
  margin-bottom: 0;
  color: #5e7188;
}
.weather-icon {
  font-size: clamp(3.5rem, 10vw, 6rem);
  filter: drop-shadow(0 10px 12px rgba(55, 98, 136, 0.18));
}
.temperature-row {
  justify-content: flex-start;
  margin: 30px 0;
}
.temperature {
  margin-bottom: 0;
  font-size: clamp(4.6rem, 15vw, 8rem);
  font-weight: 800;
  line-height: 0.8;
  letter-spacing: -0.08em;
}
.condition {
  margin-bottom: 4px;
  font-size: 1.3rem;
  font-weight: 800;
}
.weather-details {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px;
  margin: 0;
}
.weather-details div {
  border-radius: 15px;
  padding: 16px;
  background: rgba(220, 238, 253, 0.68);
}
dt {
  margin-bottom: 5px;
  color: #5e7188;
  font-size: 0.78rem;
  font-weight: 700;
  text-transform: uppercase;
}
dd {
  margin: 0;
  font-weight: 800;
}
footer {
  padding-top: 18px;
  color: #66798f;
  font-size: 0.78rem;
  text-align: center;
}
@media (max-width: 620px) {
  .search-row,
  .temperature-row {
    align-items: stretch;
    flex-direction: column;
  }
  button {
    width: 100%;
  }
  .weather-details {
    grid-template-columns: 1fr;
  }
}
</style>

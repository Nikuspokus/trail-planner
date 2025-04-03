<template>
  <div class="h-screen w-full">
    <div id="map" class="h-full w-full"></div>
    <div class="absolute top-4 left-4 z-50 bg-white p-4 shadow-xl rounded max-w-sm">
      <label for="speed">Vitesse moyenne (km/h):</label>
      <input
        v-model.number="speed"
        type="number"
        id="speed"
        min="1"
        class="border ml-2 px-2 py-1 rounded w-full"
      />

      <p v-if="distanceKm !== null && arrivalTime !== null" class="mt-4 text-sm">
        🚶 Distance : {{ distanceKm.toFixed(2) }} km — 🕒 Heure d'arrivée estimée :
        {{ arrivalTime }}
      </p>

      <p v-if="currentWeather" class="mt-2 text-sm text-gray-700">
        🌡️ Météo actuelle à votre position : {{ currentWeather }}
      </p>

      <ul class="mt-2 text-xs text-gray-600 max-h-40 overflow-y-auto">
        <li v-for="(entry, index) in weatherHistory" :key="index">
          🕒 {{ entry.time }} — 🌡️ {{ entry.temperature }}
        </li>
      </ul>

      <button
        @click="resetMap"
        class="mt-4 px-3 py-1 rounded bg-red-500 text-white hover:bg-red-600"
      >
        Réinitialiser
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import axios from 'axios'

const speed = ref(6)
const startCoords: [number, number] = [48.858093, 2.294694] // Paris
const distanceKm = ref<number | null>(null)
const arrivalTime = ref<string | null>(null)
const userCoords = ref<[number, number] | null>(null)
const userMarker = ref<L.Marker | null>(null)
let map: L.Map
let routeLayer: L.Polyline | null = null
let destinationMarker: L.Marker | null = null

const currentWeather = ref<string | null>(null)

type WeatherEntry = {
  time: string
  temperature: string
}
const weatherHistory = ref<WeatherEntry[]>([])

async function calculateRoute(destCoords: [number, number]) {
  try {
    const coords = `${startCoords[1]},${startCoords[0]};${destCoords[1]},${destCoords[0]}`
    const url = `https://router.project-osrm.org/route/v1/foot/${coords}?overview=full&geometries=geojson`

    const response = await axios.get(url)
    if (!response.data.routes || response.data.routes.length === 0)
      throw new Error('Aucune route trouvée par OSRM.')

    const geometry = response.data.routes[0].geometry.coordinates
    const distance = response.data.routes[0].distance / 1000
    const durationHours = distance / speed.value
    const now = new Date()
    const arrival = new Date(now.getTime() + durationHours * 3600000)

    const latlngs = geometry.map(([lng, lat]: [number, number]) => [lat, lng])

    if (routeLayer) map.removeLayer(routeLayer)
    routeLayer = L.polyline(latlngs, { color: 'blue' }).addTo(map)
    map.fitBounds(routeLayer.getBounds())

    const intervals = Math.floor(durationHours * 2)
    const weatherPromises = []
    for (let i = 0; i <= intervals; i++) {
      const point = latlngs[Math.floor((i / intervals) * (latlngs.length - 1))]
      const time = new Date(now.getTime() + i * 1800000).toISOString()
      weatherPromises.push(fetchWeather(point[0], point[1], time))
    }

    const results = await Promise.allSettled(weatherPromises)
    const weatherData = results.map((res, i) => {
      if (res.status === 'fulfilled') return res.value
      console.warn(`Erreur météo à l'étape ${i} :`, res.reason)
      return null
    })

    weatherData.forEach((weather, i) => {
      if (!weather) return
      const point = latlngs[Math.floor((i / weatherData.length) * (latlngs.length - 1))]
      L.marker(point)
        .addTo(map)
        .bindPopup(`${weather.temperature}°C à ${new Date(weather.time).toLocaleTimeString()}`)
    })

    distanceKm.value = distance
    arrivalTime.value = arrival.toLocaleTimeString()
  } catch (error: any) {
    console.error('Erreur OSRM :', error.message || error)
  }
}

async function fetchWeather(lat: number, lon: number, time: string) {
  const response = await axios.get('https://api.open-meteo.com/v1/forecast', {
    params: {
      latitude: lat,
      longitude: lon,
      hourly: 'temperature_2m',
      start: time,
      end: time,
      timezone: 'auto',
    },
  })
  return {
    temperature: response.data.hourly.temperature_2m[0],
    time,
  }
}

function resetMap() {
  if (routeLayer) {
    map.removeLayer(routeLayer)
    routeLayer = null
  }
  if (destinationMarker) {
    map.removeLayer(destinationMarker)
    destinationMarker = null
  }
  distanceKm.value = null
  arrivalTime.value = null
}

async function updateCurrentWeather(lat: number, lon: number) {
  try {
    const now = new Date().toISOString()
    const response = await axios.get('https://api.open-meteo.com/v1/forecast', {
      params: {
        latitude: lat,
        longitude: lon,
        hourly: 'temperature_2m',
        start: now,
        end: now,
        timezone: 'auto',
      },
    })
    const temp = `${response.data.hourly.temperature_2m[0]}°C`
    currentWeather.value = temp
    weatherHistory.value.unshift({
      time: new Date().toLocaleTimeString(),
      temperature: temp,
    })
    if (weatherHistory.value.length > 20) weatherHistory.value.pop()
  } catch (error) {
    console.error('Erreur météo en direct :', error)
  }
}

onMounted(() => {
  map = L.map('map').setView(startCoords, 13)

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors',
  }).addTo(map)

  L.marker(startCoords).addTo(map).bindPopup('Départ : Paris').openPopup()

  map.on('click', (e: L.LeafletMouseEvent) => {
    const destCoords: [number, number] = [e.latlng.lat, e.latlng.lng]
    if (destinationMarker) {
      destinationMarker.setLatLng(e.latlng)
    } else {
      destinationMarker = L.marker(e.latlng).addTo(map).bindPopup('Arrivée')
    }
    destinationMarker.openPopup()
    calculateRoute(destCoords)
  })

  trackUserPosition()

  setInterval(
    () => {
      if (userCoords.value) {
        updateCurrentWeather(userCoords.value[0], userCoords.value[1])
      }
    },
    30 * 60 * 1000,
  )
})

function trackUserPosition() {
  if (!navigator.geolocation) {
    alert('La géolocalisation n’est pas disponible.')
    return
  }

  navigator.geolocation.watchPosition(
    (position) => {
      const lat = position.coords.latitude
      const lng = position.coords.longitude
      userCoords.value = [lat, lng]

      if (userMarker.value) {
        userMarker.value.setLatLng([lat, lng])
      } else {
        userMarker.value = L.marker([lat, lng], {
          icon: L.icon({ iconUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png' }),
        })
          .addTo(map)
          .bindPopup('📍 Vous êtes ici')
      }

      updateCurrentWeather(lat, lng)
    },
    (error) => {
      console.error('Erreur de géolocalisation :', error)
    },
    { enableHighAccuracy: true, maximumAge: 0, timeout: 5000 },
  )
}
</script>

<style>
body {
  background-color: #fff;
  color: black;
}

#map {
  height: 40em;
  max-width: 80em;
  min-width: 20em;
  margin: 1em;
  border: 2px solid #ccc;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease-in-out;
}

/* Tablettes et petits laptops */
@media (max-width: 768px) {
  #map {
    height: 60vh;
    max-width: 47em;
    min-width: 29em;
    border-radius: 10px;
  }
}

/* Smartphones */
@media (max-width: 480px) {
  #map {
    height: 50vh;
    min-width: 5em;
    max-width: 28em;
    border-radius: 8px;
  }
}

ul {
  margin-top: 1rem;
  padding-left: 1rem;
  padding-right: 1rem;
  border-top: 1px solid #eee;
  max-height: 150px;
  overflow-y: auto;
}

li {
  padding: 0.25rem 0;
  border-bottom: 1px dashed #ddd;
}
</style>

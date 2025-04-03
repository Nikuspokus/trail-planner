<template>
  <div class="h-screen w-full">
    <div id="map" class="h-full w-full"></div>
    <div class="absolute top-4 left-4 z-50 bg-white p-4 shadow-xl rounded">
      <label for="speed">Vitesse moyenne (km/h):</label>
      <input
        v-model.number="speed"
        type="number"
        id="speed"
        min="1"
        class="border ml-2 px-2 py-1 rounded"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, ref } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import axios from 'axios'

const speed = ref(5) // km/h
const startCoords: [number, number] = [48.858093, 2.294694] // Paris
let map: L.Map
let routeLayer: L.Polyline | null = null

async function calculateRoute(destCoords: [number, number]) {
  try {
    const coords = `${startCoords[1]},${startCoords[0]};${destCoords[1]},${destCoords[0]}`
    const url = `https://router.project-osrm.org/route/v1/foot/${coords}?overview=full&geometries=geojson`

    const response = await axios.get(url)
    console.log('Réponse OSRM brute :', response.data)

    if (!response.data.routes || response.data.routes.length === 0) {
      throw new Error('Aucune route trouvée par OSRM.')
    }

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

    alert(
      `Distance : ${distance.toFixed(2)} km\\nHeure d'arrivée estimée : ${arrival.toLocaleTimeString()}`,
    )
  } catch (error: any) {
    console.error('Erreur OSRM :', error.message || error)
    alert('Échec du calcul de l’itinéraire avec OSRM.')
  }
}

async function fetchWeather(lat: number, lon: number, time: string) {
  const response = await axios.get(`https://api.open-meteo.com/v1/forecast`, {
    params: {
      latitude: lat,
      longitude: lon,
      hourly: 'temperature_2m',
      start: time,
      end: time,
      timezone: 'auto',
    },
  })
  const hourIndex = 0
  return {
    temperature: response.data.hourly.temperature_2m[hourIndex],
    time,
  }
}

let destinationMarker: L.Marker | null = null

onMounted(() => {
  map = L.map('map').setView(startCoords, 13)

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors',
  }).addTo(map)

  L.marker(startCoords).addTo(map).bindPopup('Départ : Paris').openPopup()

  map.on('click', (e: L.LeafletMouseEvent) => {
    const destCoords: [number, number] = [e.latlng.lat, e.latlng.lng]

    // Affiche ou met à jour le marqueur d’arrivée
    if (destinationMarker) {
      destinationMarker.setLatLng(e.latlng)
    } else {
      destinationMarker = L.marker(e.latlng).addTo(map).bindPopup('Arrivée')
    }
    destinationMarker.openPopup()

    calculateRoute(destCoords)
  })
})
</script>

<style>
#map {
  height: 60em;
  width: 90em;
}
</style>

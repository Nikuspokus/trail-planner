🗺️ Vue 3 - Application de planification de trajet pédestre

Cette application interactive utilise **Vue 3**, **Leaflet.js**, **OSRM** et **Open-Meteo** pour :

- Afficher une carte interactive
- Définir un point de départ fixe (Paris)
- Sélectionner un point d’arrivée par clic sur la carte
- Calculer l’itinéraire pédestre via OSRM
- Estimer la distance et l’heure d’arrivée en fonction d’une vitesse moyenne saisie
- Afficher la météo à intervalles réguliers sur l’itinéraire avec Open-Meteo
- Afficher la météo actuelle à la position GPS de l’utilisateur

---

## 🧱 Technologies utilisées

- **Vue 3 + `<script setup>`**
- **TypeScript**
- **Leaflet.js** – carte interactive
- **OSRM API** – calcul d'itinéraire
- **Open-Meteo API** – météo actuelle et future
- **Axios** – appels HTTP

---

## 🚀 Lancer le projet en local

### Prérequis

- [Node.js](https://nodejs.org/) >= 18.x
- [npm](https://www.npmjs.com/) ou [pnpm](https://pnpm.io)

### Installation

```bash
# Clone du projet
git clone https://github.com/ton-user/ton-projet.git
cd ton-projet

# Installation des dépendances
npm install


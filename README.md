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
git clone https://github.com/Nikuspokus/trail-planner-2.git
cd trail-planner-2

# Installation des dépendances
npm install

Démarrage du serveur
bash
Copier
Modifier
npm run dev
📸 Fonctionnalités
🌍 Carte interactive centrée sur Paris

📍 Sélection du point d’arrivée par clic

🧮 Calcul de distance / temps estimé

🏃‍♂️ Entrée de la vitesse de marche personnalisée

🌡️ Météo en temps réel + météo toutes les 30 minutes sur le parcours

📍 Suivi de position utilisateur via l'API navigateur

💡 Améliorations possibles
🗺️ Affichage du profil d’altitude (via API Elevation)

🗓️ Ajout d’un sélecteur de date pour prévoir un itinéraire futur

📦 Export PDF ou impression de l’itinéraire

🧭 Mode mobile responsive amélioré

🔐 APIs utilisées
OSRM (Open Source Routing Machine)
URL utilisée : https://router.project-osrm.org/route/v1/foot/...

Gratuit et sans clé

Open-Meteo
URL utilisée : https://api.open-meteo.com/v1/forecast

Gratuit et sans clé API

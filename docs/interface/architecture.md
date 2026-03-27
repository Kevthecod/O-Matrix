# Architecture de l'Interface React

## Stack technologique

| Composant | Technologie | Rôle |
|-----------|-------------|------|
| Framework UI | React 18 + Vite 5 | Interface principale |
| Vue 3D | Three.js | Rendu WebGL du cube |
| WebSocket | API native navigateur | Communication temps réel |
| Serveur | Node.js + `ws` 8 | Relay WebSocket |
| Proxy Vite | `vite.config.js` | `/ws` → `ws://localhost:8080` |

## Structure du projet

```
led-cube-controller/
├── package.json        ← dépendances + scripts
├── vite.config.js      ← proxy WebSocket
├── server.js           ← serveur relay Node.js
├── index.html
└── src/
    ├── main.jsx        ← point d'entrée React
    └── App.jsx         ← toute l'application
```

## Démarrage

```bash
npm install
npm start
# Lance concurrently : node server.js + vite
```

```bash
# Si port 8080 occupé
npx kill-port 8080 && npm start
```

## Onglets de l'interface

=== "Panneau gauche"

    | Onglet | Contenu |
    |--------|---------|
    | **Formes** | Formes 3D prédéfinies mono/multi-couleurs |
    | **Dessin** | Éditeur plan par plan avec frames d'animation |
    | **Texte** | Affichage de texte défilant avec choix de couleur |
    | **Anim.** | Animations dynamiques prédéfinies |
    | **Script** | Éditeur de code JavaScript pour formes/animations custom |
    | **OBJ** | Import et voxélisation de fichiers .obj |
    | **Couleur** | Color picker 12 bits avec sliders R/G/B |
    | **Détect.** | Détection des couleurs utilisées dans la scène |

=== "Centre"

    Vue 3D WebGL du cube (Three.js) :
    
    - Rotation par drag souris
    - Zoom par molette
    - 1000 sphères — allumées = couleur active, éteintes = gris transparent
    - Compteur LEDs actives en superposition

=== "Panneau droit"

    | Onglet | Contenu |
    |--------|---------|
    | **Tranches** | Vue 2D plan par plan avec toggle LED |
    | **WebSocket ▶** | Connexion, envoi, statistiques temps réel |

## Format des payloads envoyés

```typescript
interface Payload {
  size: 10,
  count: number,      // Nombre de LEDs dans ce payload
  leds: Array<{
    pos: number,      // x + y*10 (0-99)
    z:   number,      // plan (0-9)
    r:   number,      // 0-4095 (12 bits)
    g:   number,
    b:   number,
  }>
}
```

### Modes d'envoi

| Mode | Description |
|------|-------------|
| **Full** | Toutes les LEDs allumées envoyées (count peut être < 1000) |
| **Delta** | Uniquement les LEDs modifiées depuis le dernier envoi |
| **Auto-send** | Envoi automatique à chaque changement de la scène |
| **Manuel** | Bouton "Envoyer" explicite |

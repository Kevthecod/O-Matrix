# WebSocket Client (React)

## Hook useWebSocket

Le hook `useWebSocket(fps)` gère la connexion, reconnexion automatique et l'envoi throttlé.

```typescript
const { status, stats, connect, disconnect, sendGrid } = useWebSocket(fps);

// Connexion : passe l'URL au moment du clic
connect(`ws://${ip}:${port}`);

// Envoi de la scène complète
sendGrid(grid, true);

// Envoi delta (seules les LEDs modifiées)
sendGrid(grid, false);
```

## États de connexion

| État | Couleur | Description |
|------|---------|-------------|
| `idle` | Gris-bleu | Non connecté |
| `connecting` | Orange | Tentative en cours |
| `online` | Vert | Connecté et opérationnel |
| `error` | Rouge | Erreur — reconnexion automatique |
| `sending` | Cyan | Envoi en cours |

## Reconnexion automatique

En cas d'erreur ou de déconnexion, le hook tente une reconnexion avec **backoff exponentiel** :

```
1ère tentative : 1 s
2ème           : 2 s
3ème           : 4 s
...
Maximum        : 30 s
```

## Message d'identification

Dès l'ouverture de la connexion, le navigateur envoie :

```json
{"type": "register", "role": "browser"}
```

Le serveur répond avec le statut de l'ESP32 :

```json
{"type": "esp32_status", "connected": true}
```

## Throttle d'envoi

Le hook limite la fréquence d'envoi au FPS configuré :

```javascript
const minInterval = 1000 / fps;
// Si un payload est prêt avant minInterval, il est mis en attente
// Le dernier payload reçu pendant l'attente est envoyé à l'expiration
```

!!! warning "Connexion indépendante de l'auto-send"
    La connexion WebSocket (`connect()`) est **indépendante** de l'option "Envoi automatique". On peut se connecter sans activer l'envoi auto, et envoyer manuellement.

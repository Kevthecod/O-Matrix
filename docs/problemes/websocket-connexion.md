# Problèmes de Connexion WebSocket

## Problème 1 : ESP32 — Erreur 404 au handshake

### Symptôme

```
E (156627) transport_ws: Sec-WebSocket-Accept not found
E (156627) websocket_client: esp_transport_connect() failed with -1,
           esp_ws_handshake_status_code=404
E (156637) WS_CLIENT: Erreur WebSocket
I (161667) transport_ws: Reconnect after 5000 ms
```

### Cause

Le serveur Node.js utilisait `new WebSocketServer({ port: 8080 })` qui crée son propre serveur TCP interne. Ce mode ne gère pas toujours correctement le handshake HTTP→WebSocket attendu par `esp_websocket_client`.

De plus, la config ESP-IDF manquait du champ `.path = "/"` obligatoire.

### Solution

**Côté serveur** — utiliser `http.createServer` + WebSocketServer attaché :

```javascript
// AVANT (problématique avec esp_websocket_client)
const wss = new WebSocketServer({ port: 8080 });

// APRÈS (compatible)
const httpServer = http.createServer(...);
const wss = new WebSocketServer({ server: httpServer });
httpServer.listen(8080, '0.0.0.0');
```

**Côté ESP32** — ajouter `.path` et `.transport` explicites :

```c
esp_websocket_client_config_t cfg = {
    .uri       = "ws://192.168.x.x:8080",
    .path      = "/",                              // ← Obligatoire
    .transport = WEBSOCKET_TRANSPORT_OVER_TCP,     // ← Pas de TLS
    // ...
};
```

**Vérification rapide** :
```bash
curl http://192.168.x.x:8080
# Doit répondre : LED Cube WebSocket Server OK
```

---

## Problème 2 : Interface — Bouton "Connecter" sans effet

### Symptôme

Cliquer sur le bouton **CONNECTER** dans l'interface ne déclenche aucune connexion. Le statut reste "NON CONNECTÉ".

### Cause

Le hook `useWebSocket` recevait l'URL comme paramètre statique, et celle-ci était conditionnée à l'état de la case "Envoi automatique" :

```jsx
// AVANT — bug : url=null tant que autoSend est false
const { connect } = useWebSocket(autoSend ? wsUrl : null, fps);
// connect() faisait if (!url) return; → rien ne se passait
```

### Solution

Séparer complètement la **connexion** de l'**envoi automatique** :

```jsx
// APRÈS — url passée au moment du clic
const { connect } = useWebSocket(fps);

// Bouton connecter
<button onClick={() => connect(wsUrl)}>CONNECTER</button>

// Envoi automatique indépendant
useEffect(() => {
    if (!autoSend || status !== 'online') return;
    sendGrid(grid, !deltaMode);
}, [grid, autoSend, status]);
```

---

## Problème 3 : Serveur — Port 8080 déjà utilisé

### Symptôme

```
[0] Erreur serveur : listen EADDRINUSE: address already in use 0.0.0.0:8080
[0] npm run server exited with code 1
```

### Cause

Une ancienne instance du serveur Node.js tourne encore en arrière-plan (mauvais arrêt précédent).

### Solution

**Windows — trouver et tuer le process :**

```powershell
# Trouver le PID
netstat -ano | findstr :8080
# → TCP  0.0.0.0:8080  LISTENING  12345

# Tuer le process
taskkill /PID 12345 /F
```

**Solution automatique dans package.json :**

```json
"start": "npx kill-port 8080 && concurrently \"npm run server\" \"npm run dev\""
```

`kill-port` libère le port avant chaque démarrage, évitant l'erreur de façon permanente.

---

## Problème 4 : Terminal Node.js — Serveur ne démarre pas

### Diagnostic par les messages d'erreur

Le serveur affiche maintenant des messages explicites :

```
❌ Module "ws" introuvable. Lance : npm install
❌ Port 8080 déjà utilisé. Tue le process : npx kill-port 8080
❌ Erreur : [message détaillé]
```

| Message | Solution |
|---------|----------|
| `Module "ws" introuvable` | `npm install` dans le dossier projet |
| `EADDRINUSE` | `npx kill-port 8080` puis relancer |
| Rien dans le terminal | Vérifier que `node server.js` est bien exécuté |

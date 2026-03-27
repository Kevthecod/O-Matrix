# Client WebSocket ESP-IDF

## Configuration

```c
esp_websocket_client_config_t cfg = {
    .uri                  = SERVER_URI,  // "ws://192.168.x.x:8080"
    .path                 = "/",         // Obligatoire en ESP-IDF v5.x
    .reconnect_timeout_ms = 5000,        // Reconnexion auto toutes les 5s
    .network_timeout_ms   = 10000,
    .buffer_size          = 4096,        // Taille d'un fragment TCP
    .task_stack           = 8192,
    .transport            = WEBSOCKET_TRANSPORT_OVER_TCP,
};
```

!!! warning "`.path = \"/\"` obligatoire"
    Sans ce champ, certaines versions d'ESP-IDF envoient un header `Host` malformé → le serveur Node.js répond **404** et le handshake WebSocket échoue.

---

## Accumulation des fragments TCP

Un payload de 1000 LEDs en JSON fait **~41 KB**. Le buffer WebSocket (`buffer_size = 4096`) ne peut recevoir qu'un fragment à la fois. Il faut **accumuler** tous les fragments avant de parser.

```c
#define ACCUM_SIZE  65536   // 64 KB

static char  *s_accum     = NULL;   // Alloué une fois au démarrage
static int    s_accum_len = 0;

// Dans ws_event_handler, case WEBSOCKET_EVENT_DATA :
if (data->payload_offset == 0) accum_reset();  // Nouveau message
accum_append(data->data_ptr, data->data_len);

// Dernier fragment détecté par :
if ((data->payload_offset + data->data_len) >= data->payload_len) {
    parse_payload(s_accum, s_accum_len);
    accum_reset();
}
```

### Champs utilisés (ESP-IDF v5.x)

| Champ | Rôle |
|-------|------|
| `payload_len` | Taille totale du message WS annoncée dans le header |
| `payload_offset` | Position du début de ce fragment dans le message |
| `data_len` | Taille de ce fragment |
| `op_code` | `0x01` = TEXT, `0x00` = fragment de continuation |

!!! danger "fin_flag inexistant en v5.x"
    Le champ `fin_flag` n'existe **pas** dans `esp_websocket_event_data_t` de ESP-IDF v5.x. Utiliser `payload_offset + data_len >= payload_len` à la place.

---

## Gestion des événements

```c
switch (event_id) {
case WEBSOCKET_EVENT_CONNECTED:
    // Réinitialise le buffer + s'identifie comme ESP32
    accum_reset();
    esp_websocket_client_send_text(s_ws,
        "{\"type\":\"register\",\"role\":\"esp32\"}", ...);
    break;

case WEBSOCKET_EVENT_DATA:
    // Accumulation + parsing au dernier fragment
    break;

case WEBSOCKET_EVENT_DISCONNECTED:
    accum_reset();
    // Reconnexion automatique gérée par le client
    break;
}
```

---

## Identification auprès du serveur relay

Immédiatement après connexion, l'ESP32 envoie :

```json
{"type": "register", "role": "esp32"}
```

Le serveur répond :

```json
{"type": "ack", "role": "esp32"}
```

Et notifie les navigateurs connectés :

```json
{"type": "esp32_status", "connected": true}
```

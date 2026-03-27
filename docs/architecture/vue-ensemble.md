# Vue d'ensemble du système

## Schéma synoptique

```mermaid
graph TD
    subgraph SW["Logiciel (PC)"]
        UI[Interface React<br/>:5173]
        SRV[Serveur relay Node.js<br/>:8080]
        UI -->|WebSocket| SRV
    end

    subgraph FW["Firmware ESP32"]
        WS_CLI[Client WebSocket]
        PARSER[Parser JSON<br/>zéro-alloc]
        BUF[Frame Buffer<br/>matrix 10×304]
        DISP[display_task<br/>Core 1 - Prio 5]
        WS_CLI --> PARSER --> BUF --> DISP
    end

    subgraph HW["Matériel"]
        TLC[19× TLC5940<br/>SPI2 - 5MHz]
        HC595[74HC595<br/>SPI3 - 5MHz]
        MOSFET[10× MOSFET P-canal<br/>SQ3493EV]
        LEDS[1000 LEDs RGB<br/>Anode commune]
        TLC -->|300 cathodes| LEDS
        HC595 --> MOSFET -->|10 plans anodes| LEDS
    end

    SRV -->|WebSocket Wi-Fi| WS_CLI
    DISP -->|update_tlc| TLC
    DISP -->|select_plane| HC595
```

---

## Blocs fonctionnels

### Interface Web (PC)

L'interface, développée en **React.js**, tourne dans le navigateur et communique avec le serveur relay via WebSocket. Elle intègre :

- Une **vue 3D WebGL** (Three.js) du cube en temps réel
- Des onglets : Formes, Dessin, Texte, Animations, Script, Import OBJ
- Un **éditeur plan par plan** pour dessiner manuellement des formes
- Un panneau WebSocket pour la connexion et l'envoi des données

### Serveur relay Node.js

Le serveur `server.js` fait le pont entre le navigateur et l'ESP32. Il identifie chaque client par un message `register` et relaie les payloads LED du navigateur vers l'ESP32.

```
Navigateur  →  register {"role":"browser"}  →  Serveur
ESP32       →  register {"role":"esp32"}    →  Serveur
Navigateur  →  payload JSON LEDs            →  Serveur → ESP32
```

### Firmware ESP32

Le firmware ESP-IDF gère deux tâches FreeRTOS :

| Tâche | Core | Priorité | Rôle |
|-------|------|----------|------|
| `app_main` / WebSocket | Core 0 | Normale | Réception et parsing des payloads |
| `display_task` | Core 1 | 5 (haute) | Scan des 10 plans en boucle continue |

Un **mutex** protège le buffer `matrix[][]` contre les accès simultanés des deux tâches.

### Hardware

| Composant | Rôle | Interface |
|-----------|------|-----------|
| 19× TLC5940 | PWM 12 bits sur 300 canaux cathodes | SPI2, 5 MHz |
| 74HC595 | Sélection du plan actif (0-9) | SPI3, 5 MHz |
| 10× MOSFET SQ3493EV | Commutation courant anode commune | GPIO via 74HC595 |
| LEDC ESP32 | Génération GSCLK 1 MHz | GPIO21 |

---

## Flux de données complet

```mermaid
sequenceDiagram
    participant UI as Interface React
    participant SRV as Serveur Node.js
    participant ESP as ESP32
    participant HW as TLC5940 + 74HC595

    UI->>SRV: JSON {size:10, count:N, leds:[...]}
    SRV->>ESP: Relay WebSocket (fragmenté TCP)
    Note over ESP: Accumulation fragments<br/>(buffer 64KB)
    ESP->>ESP: parse_payload() — zéro alloc
    ESP->>ESP: matrix_clear() + matrix_set_led()
    loop Toutes les ~120µs
        ESP->>HW: update_tlc(matrix[p]) via SPI2
        ESP->>HW: select_plane(p) via SPI3
        Note over HW: Plan p actif 1.2ms
        ESP->>HW: update_tlc(black) — anti-ghosting
    end
```

# Références

## Documentation technique

| Document | Description |
|----------|-------------|
| [TLC5940 Datasheet](https://www.ti.com/lit/ds/symlink/tlc5940.pdf) | Texas Instruments — Spécifications complètes |
| [74HC595 Datasheet](https://www.nxp.com/docs/en/data-sheet/74HC_HCT595.pdf) | NXP — Registre à décalage 8 bits |
| [ESP-IDF v5.4 Docs](https://docs.espressif.com/projects/esp-idf/en/v5.4/) | Espressif — Documentation officielle ESP-IDF |
| [esp_websocket_client API](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/protocols/esp_websocket_client.html) | API WebSocket client ESP-IDF |
| [SQ3493EV Datasheet](https://www.vishay.com/docs/77089/sq3493ev.pdf) | Vishay — MOSFET P-canal |

## Ressources logicielles

| Ressource | URL |
|-----------|-----|
| ESP-IDF GitHub | https://github.com/espressif/esp-idf |
| ws (Node.js) | https://github.com/websockets/ws |
| React | https://react.dev |
| Three.js | https://threejs.org |
| MkDocs Material | https://squidfunk.github.io/mkdocs-material |
| KiCad | https://www.kicad.org |

## Projet

| Élément | Information |
|---------|-------------|
| Institution | EPAC — École Polytechnique d'Abomey-Calavi, UAC |
| Département | Génie Électrique — 5e Année |
| UE | Projet en Tutorat |
| Superviseur | Dr. KIKI Probus |
| Auteurs | HOUNSA Kévin & YEHOUENOU Chrysostome |
| Année académique | 2024-2025 |
| Nom du projet | O'Matrix — LED Matrix |

## Calculs de référence

### Résistance Rref (TLC5940)

```
Is = (39.06 × Vrref) / Rrref   (selon datasheet TLC5940)
Pour Is = 20 mA, Vrref = 1.024 V :
Rref = (39.06 × 1.024) / 0.020 ≈ 2 000 Ω → valeur normalisée 2 kΩ (E24)
```

### Résistances de protection LEDs

```
Alimentation : 5 V
Courant cible : 18 mA

Rouge  : R = (5 - 2.1) / 0.018 ≈ 161 Ω → 150 Ω ou 180 Ω (E24)
Vert   : R = (5 - 3.1) / 0.018 ≈ 106 Ω → 100 Ω ou 120 Ω (E24)
Bleu   : R = (5 - 3.1) / 0.018 ≈ 106 Ω → 100 Ω ou 120 Ω (E24)
```

### Fréquence de refresh

```
GSCLK = 1 MHz
Période PWM = 4096 cycles GSCLK = 4.096 ms
Fréquence refresh = 1 / 4.096ms ≈ 244 Hz

Avec 10 plans en multiplexage :
Fréquence apparente par plan = 244 Hz (le PWM TLC gère tout)
Durée par plan ≈ 1.2 ms + temps SPI
```

### Consommation maximale

```
1 plan entier allumé (luminosité max) :
100 LEDs × 3 canaux × 20 mA = 6 000 mA = 6 A @ 5V

En multiplexage (1 plan à la fois) :
Consommation moyenne = 6 A / 10 plans × duty ≈ 600 mA à 1.2A selon animations
```

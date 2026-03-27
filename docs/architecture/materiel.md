# Spécifications Matérielles

## ESP32-WROOM-32

Le microcontrôleur central du système.

| Caractéristique | Valeur |
|----------------|--------|
| CPU | Xtensa LX6 dual-core 240 MHz |
| RAM | 520 KB SRAM |
| Wi-Fi | 802.11 b/g/n |
| BLE | Bluetooth 4.2 |
| Interfaces | SPI×3, I2C×2, UART×3, LEDC, RMT, MCPWM |

**Ressources ESP32 utilisées dans ce projet :**

| Signal | GPIO | Périphérique ESP-IDF |
|--------|------|---------------------|
| TLC MOSI | 23 | SPI2_HOST (MOSI) |
| TLC SCLK | 18 | SPI2_HOST (CLK) |
| TLC XLAT | 22 | GPIO output |
| TLC BLANK | 4 | GPIO output |
| TLC GSCLK | 21 | LEDC canal 0 |
| TLC VPRG | 15 | GPIO output |
| TLC DCPRG | 12 | GPIO output |
| 74HC595 MOSI | 13 | SPI3_HOST (MOSI) |
| 74HC595 SCLK | 14 | SPI3_HOST (CLK) |
| 74HC595 STCP | 27 | GPIO output |

---

## TLC5940 — Driver PWM LED

Le TLC5940 est un driver LED 16 canaux avec résolution **12 bits** (4 096 niveaux) par canal.

### Caractéristiques utilisées

| Paramètre | Valeur |
|-----------|--------|
| Nombre de TLC | 19 |
| Canaux totaux | 19 × 16 = **304 canaux** |
| Canaux utilisés | 300 (10 couches × 10 LEDs × 3 couleurs) |
| Résolution PWM | 12 bits (0–4095) |
| Courant max par canal | 20 mA |
| Résistance Rref | 2 kΩ |
| Vitesse SPI max | 30 MHz |
| Vitesse SPI utilisée | 5 MHz |

### Signaux de contrôle

```
GSCLK : Horloge PWM — 1 MHz via LEDC ESP32
        4096 cycles = 1 période PWM → refresh LEDs à 244 Hz

BLANK : Passe HAUT tous les 4096 cycles GSCLK
        Réinitialise le compteur PWM interne

XLAT  : Impulsion après envoi SPI
        Transfère shift register → output register

VPRG  : LOW  = écriture Grayscale (mode normal)
        HIGH = écriture Dot Correction (calibration)

DCPRG : HIGH permanent → utilise DC stocké en RAM
```

### Trame SPI Grayscale

Pour **19 TLC5940** en cascade :

| Paramètre | Valeur |
|-----------|--------|
| Bits par TLC | 16 canaux × 12 bits = 192 bits |
| Octets par TLC | 24 octets |
| Taille trame totale | 19 × 24 = **456 octets** |
| Durée @ 5 MHz | ~730 µs |

### Initialisation Dot Correction

Au démarrage, les données DC (6 bits par canal) sont envoyées avant d'activer les LEDs :

```c
// Séquence DC (luminosité maximale sur tous les canaux)
gpio_set_level(TLC_VPRG, 1);          // Mode DC
// Envoi de 19 × 12 octets = 228 octets (tous à 0xFF)
spi_device_transmit(spi_tlc, &t);
gpio_set_level(TLC_XLAT, 1/0);        // Verrouillage
gpio_set_level(TLC_VPRG, 0);          // Retour mode GS
```

!!! warning "Important"
    Ne jamais laisser `VPRG` flottant. Un bruit parasite peut faire basculer les TLC en mode programmation DC de façon aléatoire, provoquant des clignotements erratiques.

---

## 74HC595 + MOSFET P-canal — Driver Plans

### Principe

Le **74HC595** est un registre à décalage 8 bits. Deux registres sont utilisés en cascade pour couvrir 10 plans.

La sortie de chaque bit commande la **grille d'un MOSFET SQ3493EV** (P-canal). Quand la sortie du 74HC595 est **LOW**, le MOSFET conduit et alimente l'anode commune du plan.

```
74HC595 sortie Qn = 0  →  MOSFET passant  →  Plan n ACTIF
74HC595 sortie Qn = 1  →  MOSFET bloqué   →  Plan n ÉTEINT
```

### Sélection d'un plan

```c
void select_plane(int p) {
    uint16_t mask = ~(1 << p);  // Bit p à 0, tous les autres à 1
    uint8_t data[2] = { mask >> 8, mask & 0xFF };
    spi_device_transmit(spi_595, &t);
    // Impulsion STCP → verrouillage
}
// Plan 11 (0x07FF) → tous les plans éteints (anti-ghosting)
```

### Courant maximal par plan

Un plan entier allumé (100 LEDs × 3 canaux × 20 mA) = **6 A**. Le SQ3493EV supporte jusqu'à -8A en continu — largement dimensionné.

| Composant | Valeur |
|-----------|--------|
| MOSFET | SQ3493EV |
| Résistance de grille | 100 Ω |
| Résstance de pull-down de grlle | 10 kΩ |
| Courant max plan | 6 A |
| Tension VGS seuil | –4 V max |

---

## Matrice LED RGB

| Paramètre | Valeur |
|-----------|--------|
| Type | Anode commune, 5 mm, diffuse |
| Quantité | 1 000 |
| VF rouge | ≈ 2,1 V |
| VF vert/bleu | ≈ 3,1 V |
| Courant nominal | 18–20 mA par canal |
| Résistance protection rouge | Calculée selon VF = 2,1 V |
| Résistance protection vert/bleu | Calculée selon VF = 3,1 V |

!!! note "Anode commune"
    Les **anodes** sont connectées par plan (10 par couche). Les **cathodes** R, G, B de chaque LED sont connectées aux sorties des TLC5940.

---

## Alimentation

| Paramètre | Valeur |
|-----------|--------|
| Tension | 5 V continu |
| Courant maximum | 6 A (plan entier allumé) |
| Découplage | 100 nF + 10 µF par carte |

!!! warning "Dimensionnement"
    En pratique avec le multiplexage (1 plan actif à la fois), la consommation instantanée est divisée par 10. Cependant, l'alimentation doit être dimensionnée pour le pic instantané d'un plan entier à luminosité maximale.

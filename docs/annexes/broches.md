# Broches ESP32 — Tableau de Câblage

## Attribution des GPIO

| GPIO | Signal | Direction | Périphérique | Destination |
|------|--------|-----------|-------------|-------------|
| 4 | TLC_BLANK | OUT | GPIO | TLC5940 broche BLANK |
| 12 | TLC_DCPRG | OUT | GPIO | TLC5940 broche DCPRG |
| 13 | REG_MOSI | OUT | SPI3 MOSI | 74HC595 broche SER |
| 14 | REG_SCLK | OUT | SPI3 CLK | 74HC595 broche SRCLK |
| 15 | TLC_VPRG | OUT | GPIO | TLC5940 broche VPRG |
| 18 | TLC_SCLK | OUT | SPI2 CLK | TLC5940 broche SCLK (chaîne) |
| 21 | TLC_GSCLK | OUT | LEDC ch.0 | TLC5940 broche GSCLK (chaîne) |
| 22 | TLC_XLAT | OUT | GPIO | TLC5940 broche XLAT (chaîne) |
| 23 | TLC_MOSI | OUT | SPI2 MOSI | TLC5940 broche SIN (premier TLC) |
| 27 | REG_STCP | OUT | GPIO | 74HC595 broche RCLK |

## Niveaux logiques

| Signal | Niveau actif | État par défaut |
|--------|-------------|----------------|
| BLANK | HIGH | HIGH (LEDs éteintes au démarrage) |
| XLAT | Impulsion HIGH | LOW |
| VPRG | LOW = GS, HIGH = DC | LOW (mode GS) |
| DCPRG | HIGH | HIGH permanent |
| STCP | Impulsion HIGH | LOW |

## Schéma de connexion simplifié

```
ESP32                    TLC5940 (premier de la chaîne)
GPIO23 (MOSI SPI2) ─────► SIN
GPIO18 (CLK  SPI2) ─────► SCLK
GPIO22             ─────► XLAT
GPIO4              ─────► BLANK
GPIO21 (LEDC)      ─────► GSCLK ──► TLC2 ──► ... ──► TLC19
GPIO15             ─────► VPRG
GPIO12             ─────► DCPRG

                   TLC1 SOUT ──► TLC2 SIN ──► ... ──► TLC19 SOUT (NC)

ESP32                    74HC595
GPIO13 (MOSI SPI3) ─────► SER
GPIO14 (CLK  SPI3) ─────► SRCLK
GPIO27             ─────► RCLK

                   74HC595_1 Q7S ──► 74HC595_2 SER
                   Q0..Q7 ──► MOSFET grilles (plans 0-7)
                   74HC595_2 Q0..Q1 ──► MOSFET grilles (plans 8-9)
```

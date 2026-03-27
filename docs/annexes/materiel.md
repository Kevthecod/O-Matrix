# Liste du Matériel

## Composants électroniques

| Composant | Quantité | Modèle / Valeur | Remarque |
|-----------|----------|----------------|----------|
| LEDs RGB anode commune 5mm diffuse | 1 000 | — | VF rouge ≈ 2,1 V ; VF vert/bleu ≈ 3,1 V |
| TLC5940 driver PWM 16 ch | 19–20 | TLC5940NT | Is = 18–20 mA, Rref = 2 kΩ |
| Registre à décalage 74HC595 | 4 | SOP-16 | Sélection des 10 plans |
| MOSFET P-canal | 10 | IRF9540 ou SQ3465EV | Commutation courant de plan |
| Microcontrôleur | 1 | ESP32-WROOM-32 | Wi-Fi + BLE intégrés |
| Régulateur 3,3 V | 1 | AMS1117-3.3 | Alimentation logique |
| Résistance Rref TLC5940 | 19–20 | 2 kΩ (E24) | Fixe Is max = 20 mA |
| Résistances protection LEDs rouge | 1 000 | Calculée | Selon VF = 2,1 V, Is = 18 mA |
| Résistances protection LEDs vert/bleu | 2 000 | Calculée | Selon VF = 3,1 V, Is = 18 mA |
| Résistances grille MOSFET | 10 | 10 kΩ | Contrôle commutation |
| Résistances pull-up drain MOSFET | 10 | 100 Ω | — |
| Résistances tirage signaux | ~5 | 220 Ω | XLAT, BLANK, signaux critiques |
| Condensateurs découplage | ~30 | 100 nF CMS 0402 | Près de chaque CI |
| Condensateurs électrolytiques | ~10 | 10 µF CMS | Découplage alimentation |
| Alimentation | 1 | 5 V – 6 A | Courant max plan = 6 A |
| Buzzer | 1 | — | Signalisation sonore |

## PCBs (JLCPCB)

| Carte | Référence | Quantité | Matériau |
|-------|-----------|----------|---------|
| Carte principale | `Gerber_Main_Board_Y2` | 5 | FR4 |
| Driver TLC5940 | `Gerber_TLC5940_Y3` | 20 | FR4 |
| Driver plans 74HC595 | `Gerber_74HC595_Y4` | 5 | FR4 |

## Outillage logiciel

| Outil | Usage |
|-------|-------|
| VS Code + ESP-IDF Extension | Développement firmware C |
| ESP-IDF v5.4.3 | Framework embarqué |
| KiCad | Conception PCBs |
| Node.js 18+ | Serveur relay WebSocket |
| React 18 + Vite 5 | Interface web |
| Autodesk Fusion 360 | Modélisation 3D structure |
| JLCPCB | Fabrication PCBs |

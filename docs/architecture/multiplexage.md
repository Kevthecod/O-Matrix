# Principe de Multiplexage

## Concept

Le multiplexage temporel permet de piloter **1 000 LEDs** avec un nombre limité de lignes d'E/S. À tout instant, **un seul plan** est alimenté par les MOSFET. Les TLC5940 envoient simultanément les données de couleur des 100 LEDs de ce plan.

```
t=0 : Plan 0 actif  → TLC envoient data couche 0..9, colonne 0
t=1 : Plan 1 actif  → TLC envoient data couche 0..9, colonne 1
...
t=9 : Plan 9 actif  → TLC envoient data couche 0..9, colonne 9
t=10: Plan 0 actif  → cycle recommence
```

À **244 Hz de refresh PWM** et **10 plans**, la fréquence apparente est **> 24 Hz** — bien au-dessus du seuil de persistance rétinienne (~50 Hz recommandé). En pratique la durée par plan de 1,2 ms donne ~833 Hz de scan.

---

## Définitions des axes

```
       Plan (Z)   0→9  sélectionné par 74HC595
       │
       │   Couche (Y)  0→9  axe vertical
       │   │
       │   │   Colonne (X)  0→9  = position dans la couche = plan Z
       │   │   │
       ▼   ▼   ▼
    matrix[plan][couche * 30 + colonne * 3 + {B,G,R}]
```

!!! info "Plan = Colonne physique"
    Dans ce cube, le **plan Z** sélectionné par le 74HC595 correspond à la **colonne physique X**. La LED à la position `(x=3, y=2, z=5)` n'est allumée physiquement **que lorsque le plan Z=5 est actif**.

---

## Indexation LED → Canal TLC

### Format du payload WebSocket

```json
{
  "pos": 23,   // x + y×10  →  x=3, y=2
  "z": 3,      // plan actif (colonne physique)
  "r": 4095,
  "g": 0,
  "b": 0
}
```

### Calcul du canal TLC

```
pos = x + y × 10
x   = pos % 10      (colonne = plan Z attendu)
y   = pos / 10      (couche verticale 0-9)

canal_TLC = y × 30 + x × 3

matrix[z][canal + 0] = B   (bleu en premier)
matrix[z][canal + 1] = G
matrix[z][canal + 2] = R
```

### Exemple complet

| pos | z | x | y | canal_TLC | B | G | R |
|-----|---|---|---|-----------|---|---|---|
| 0 | 0 | 0 | 0 | 0 | [0] | [1] | [2] |
| 1 | 1 | 1 | 0 | 3 | [3] | [4] | [5] |
| 10 | 0 | 0 | 1 | 30 | [30] | [31] | [32] |
| 23 | 3 | 3 | 2 | 69 | [69] | [70] | [71] |
| 99 | 9 | 9 | 9 | 297 | [297] | [298] | [299] |

### Couverture TLC

| Couche | LEDs | Canaux TLC |
|--------|------|------------|
| 0 | 0–9 | 0–29 |
| 1 | 10–19 | 30–59 |
| 2 | 20–29 | 60–89 |
| ... | ... | ... |
| 9 | 90–99 | 270–299 |
| — | inutilisés | 300–303 |

19 TLC × 16 canaux = 304 canaux → 4 canaux inutilisés (toujours à 0).

---

## Timing du scan

```
Pour chaque plan p (0→9) :
  ┌─────────────────────────────────────────────────┐
  │ 1. update_tlc(matrix[p])  ← SPI2, ~730µs        │
  │ 2. select_plane(p)        ← SPI3, impulsion STCP │
  │ 3. esp_rom_delay_us(1200) ← plan actif 1.2ms     │
  │ 4. update_tlc(black)      ← éteindre les TLC     │
  │ 5. select_plane(11)       ← tous plans éteints   │
  └─────────────────────────────────────────────────┘
  Durée totale par plan : ~1.93ms
  Fréquence de scan : ~52 Hz par plan  (10 plans → ~5.2 Hz apparent)
```

!!! tip "Anti-ghosting"
    L'étape 4 (`update_tlc(black)`) et l'étape 5 (`select_plane(11)`) sont essentielles pour éviter le **ghosting** (persistance lumineuse d'un plan sur le suivant due au temps de commutation des MOSFET).

# Erreur d'Indexation des LEDs

## Symptôme

Les LEDs s'allument aux mauvaises positions. Toutes les LEDs d'une forme semblent mappées sur les mêmes canaux TLC, ou les couleurs apparaissent décalées verticalement.

## Cause — Oubli de la couche Y dans le calcul du canal

### Code incorrect

```c
// AVANT — bug
static void matrix_set_led(int pos, int z, uint16_t r, uint16_t g, uint16_t b) {
    int base = (pos % CUBE_SIZE) * 3;   // ← ignorait la couche y !
    matrix[z][base + 0] = b;
    matrix[z][base + 1] = g;
    matrix[z][base + 2] = r;
}
```

**Problème** : pour `pos=0` (couche 0) et `pos=10` (couche 1), le résultat était identique :
```
pos=0  → base = (0%10)*3 = 0
pos=10 → base = (10%10)*3 = 0  ← MÊME canal !
```

Toutes les couches partageaient les mêmes canaux TLC 0–29, écrasant les données des couches précédentes.

### Architecture physique rappel

```
Chaîne TLC continue :
  Couche 0 → canaux   0–29  (10 LEDs × 3 canaux RGB)
  Couche 1 → canaux  30–59
  Couche 2 → canaux  60–89
  ...
  Couche 9 → canaux 270–299
```

### Code correct

```c
// APRÈS — corrigé
static void matrix_set_led(int pos, int z, uint16_t r, uint16_t g, uint16_t b) {
    if (pos < 0 || pos >= CUBE_SIZE * CUBE_SIZE) return;
    if (z   < 0 || z   >= NUM_PLANES)            return;

    int y    = pos / CUBE_SIZE;   // Couche  (0-9)
    int x    = pos % CUBE_SIZE;   // Colonne (0-9)
    int base = y * 30 + x * 3;   // Canal TLC correct

    matrix[z][base + 0] = b;     // Ordre câblage : B, G, R
    matrix[z][base + 1] = g;
    matrix[z][base + 2] = r;
}
```

## Tableau de vérification

| pos | z | y (couche) | x (col) | base | Canaux TLC |
|-----|---|-----------|---------|------|-----------|
| 0 | 0 | 0 | 0 | 0 | [0,1,2] |
| 9 | 9 | 0 | 9 | 27 | [27,28,29] |
| 10 | 0 | 1 | 0 | 30 | [30,31,32] |
| 50 | 5 | 5 | 0 | 150 | [150,151,152] |
| 99 | 9 | 9 | 9 | 297 | [297,298,299] |

## Correspondance interface ↔ hardware

```
Interface envoie :
  pos = x + y×10   (colonne + couche×10)
  z   = plan actif (= colonne physique sélectionnée par 74HC595)

Hardware reçoit :
  Plan z actif → 74HC595 active MOSFET du plan z
  TLC envoient canaux y×30+x×3 allumés → LED (x,y) de ce plan s'allume

Cohérence : x dans le payload doit = z (le plan sélectionne la colonne)
```

!!! info "Pourquoi x = z ?"
    Dans ce cube, les **plans** correspondent aux **colonnes physiques**. La LED à la position `(x=3, y=2)` n'est allumée que lorsque le plan `z=3` est actif. L'interface respecte cette convention : `pos = x + y×10` et `z = x`.

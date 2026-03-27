# Buffer Hardware & Indexation

## Structure du buffer

```c
static uint16_t matrix[NUM_PLANES][NUM_OUTPUTS];
//                      ^           ^
//                      10 plans    304 canaux TLC (19×16)
```

Le buffer `matrix[p][c]` contient la valeur PWM 12 bits (0–4095) du canal TLC `c` lorsque le plan `p` est actif.

---

## Fonction matrix_set_led

```c
static void matrix_set_led(int pos, int z, uint16_t r, uint16_t g, uint16_t b)
{
    if (pos < 0 || pos >= 100) return;
    if (z   < 0 || z   >= 10) return;

    int y    = pos / 10;      // Couche verticale (0-9)
    int x    = pos % 10;      // Colonne (0-9) = plan Z attendu
    int base = y * 30 + x * 3;

    matrix[z][base + 0] = b;  // Bleu en premier (câblage BGR)
    matrix[z][base + 1] = g;
    matrix[z][base + 2] = r;
}
```

### Justification du mapping

La chaîne TLC est **continue** sur les 10 couches :

```
Couche 0  →  canaux   0–29   (10 LEDs × 3 canaux)
Couche 1  →  canaux  30–59
Couche 2  →  canaux  60–89
...
Couche 9  →  canaux 270–299
Inutilisés→  canaux 300–303
```

Pour chaque couche `y`, les 10 positions (colonnes) occupent 3 canaux chacune :

```
Colonne x dans couche y  →  canal y×30 + x×3
```

### Exemple de vérification

LED à `pos=23, z=3` (rouge pur, 4095) :

```
y    = 23 / 10 = 2      (couche 2)
x    = 23 % 10 = 3      (colonne 3)
base = 2×30 + 3×3 = 69

matrix[3][69] = 0       (B=0)
matrix[3][70] = 0       (G=0)
matrix[3][71] = 4095    (R=4095)

→ Quand plan z=3 est actif, les TLC allument
  le canal 71 à 4095, canal 69 et 70 à 0
  → LED rouge à la couche 2, colonne 3 ✓
```

---

## Ordre B-G-R

Le câblage physique des cathodes sur les cartes TLC5940 suit l'ordre **B, G, R** (et non R, G, B). Cet ordre est documenté dans `push_frame()` de l'exemple hardware original :

```c
// push_frame() original — référence de câblage
matrix[y][base + 0] = frame[y][x].b;   // B en premier
matrix[y][base + 1] = frame[y][x].g;
matrix[y][base + 2] = frame[y][x].r;   // R en dernier
```

!!! danger "Ne pas inverser B et R"
    Inverser cet ordre produira des couleurs incorrectes : le rouge apparaîtra bleu et vice-versa. Le mapping BGR est imposé par le câblage physique des PCBs.

---

## Protection par mutex

Le buffer `matrix[][]` est accédé par deux tâches :

- **`display_task`** (Core 1) : lecture lors du scan, ~toutes les 120 µs
- **`parse_payload`** (Core 0) : écriture lors de la réception d'un payload

Un mutex FreeRTOS protège chaque accès :

```c
// Dans display_task — lecture rapide
xSemaphoreTake(s_matrix_mutex, pdMS_TO_TICKS(1));
memcpy(plane_buf, matrix[p], sizeof(plane_buf));  // <1µs
xSemaphoreGive(s_matrix_mutex);

// Dans parse_payload — écriture longue mais atomique
xSemaphoreTake(s_matrix_mutex, pdMS_TO_TICKS(20));
matrix_clear();
// ... matrix_set_led() pour chaque LED ...
xSemaphoreGive(s_matrix_mutex);
```

Le timeout de `1 ms` dans `display_task` est un filet de sécurité : si le mutex est tenu par le parser, `display_task` affiche du noir pour ce plan plutôt que de bloquer.

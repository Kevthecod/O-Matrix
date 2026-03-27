# Parser JSON — Zéro Allocation Heap

## Pourquoi ne pas utiliser cJSON ?

Un payload de 1 000 LEDs en JSON fait **~41 KB**. La bibliothèque cJSON crée un nœud dynamique pour chaque valeur JSON. Pour 1 000 objets LED × 5 champs, cela représente **~5 000 allocations** pour une consommation heap de **~200 KB** — supérieure à la RAM disponible (~190 KB libres).

**Symptôme observé :**
```
E (666072) LED_CUBE: JSON parse error (len=40933)
```

**Solution :** Parser directement le JSON octet par octet dans le buffer d'accumulation déjà alloué. **Zéro allocation heap supplémentaire.**

---

## Format du payload attendu

```json
{
  "size": 10,
  "count": 1000,
  "leds": [
    {"pos": 0,  "z": 0, "r": 0,    "g": 4095, "b": 4095},
    {"pos": 1,  "z": 1, "r": 4095, "g": 0,    "b": 0   },
    ...
  ]
}
```

| Champ | Description |
|-------|-------------|
| `size` | Taille du cube (10) |
| `count` | Nombre de LEDs dans ce payload |
| `pos` | `x + y×10` (0–99) |
| `z` | Plan actif (0–9) |
| `r`, `g`, `b` | Valeurs 12 bits (0–4095) |

---

## Algorithme du parser

```c
static void parse_payload(const char *buf, int len)
{
    // 1. Détection rapide des messages de contrôle
    if (memmem(buf, min(len,60), "\"type\"", 6)) return;

    // 2. Localisation du tableau "leds"
    const char *leds_ptr = memmem(buf, len, "\"leds\"", 6);
    // Avance jusqu'au '['

    // 3. Prise du mutex — une seule fois pour tout le payload
    xSemaphoreTake(s_matrix_mutex, 20ms);
    matrix_clear();   // Scène propre

    // 4. Parcours objet par objet
    while (p < len) {
        // Cherche '{'
        // Lit les clés : pos, z, r, g, b
        // → matrix_set_led(pos, z, r, g, b)
    }

    xSemaphoreGive(s_matrix_mutex);
    led_buffer_apply();
}
```

### Fonctions auxiliaires

**`next_key(buf, len, &p, key, key_max)`**
: Avance jusqu'au prochain `"`, lit la clé dans `key`.

**`read_int_after_colon(buf, len, &p, &val)`**
: Avance jusqu'au `:`, lit l'entier qui suit. Gère les négatifs.

### Discrimination des clés par premier caractère

```c
if      (key[0]=='p') pos = val;   // "pos"
else if (key[0]=='z') z   = val;   // "z"
else if (key[0]=='r') r   = val;   // "r"
else if (key[0]=='g') g   = val;   // "g"
else if (key[0]=='b') b   = val;   // "b"
```

!!! tip "Performance"
    Le parser traite ~41 KB en quelques millisecondes sur l'ESP32 @ 240 MHz. Le mutex est tenu pendant toute la durée du parsing pour garantir l'atomicité de la mise à jour du buffer.

---

## Gestion de la mémoire

| Élément | Allocation | Taille |
|---------|-----------|--------|
| Buffer accumulation `s_accum` | `malloc` au démarrage | 64 KB |
| Buffer TLC `gs` dans `update_tlc` | Stack (local) | 456 octets |
| `matrix[][]` | BSS (statique) | 6 080 octets |
| Mutex | FreeRTOS heap | ~88 octets |

**Total consommation RAM supplémentaire pour le WebSocket/JSON : ~65 KB**

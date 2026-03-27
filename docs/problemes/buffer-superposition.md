# Superposition de Formes dans le Buffer

## Symptôme

Après l'envoi de plusieurs formes successives, des **LEDs fantômes** restent allumées — résidu des formes précédentes superposé à la forme courante.

## Cause 1 — Buffer non effacé avant frame complète

### Explication

En mode **delta**, l'interface n'envoie que les LEDs qui **changent**. Si une LED était allumée et ne change pas, elle n'est pas dans le payload delta.

L'ancien code distinguait frame complète (count ≥ 1000) et delta (count < 1000) :

```c
// AVANT — comportement partiel
bool is_full = (count >= total);
if (is_full) {
    led_buffer_clear();   // Effacement seulement si full !
}
// Les payloads delta ne nettoyaient pas les anciennes LEDs
```

**Résultat** : les LEDs allumées par la forme précédente restaient dans le buffer même si la nouvelle forme ne les incluait pas.

## Cause 2 — Buffer non effacé à la reconnexion

Si l'ESP32 se reconnecte (reset, perte WiFi), le buffer `matrix[][]` conservait les données de la session précédente. Le premier payload reçu après reconnexion se superposait aux résidus.

## Solution appliquée

**Effacer systématiquement le buffer avant chaque payload**, qu'il soit full ou delta :

```c
// APRÈS — effacement systématique
static void parse_payload(const char *buf, int len) {
    // ...
    xSemaphoreTake(s_matrix_mutex, pdMS_TO_TICKS(20));

    matrix_clear();   // ← TOUJOURS, sans condition

    // Applique le payload par-dessus le buffer vide
    cJSON_ArrayForEach(led, j_leds) {
        matrix_set_led(...);
    }

    xSemaphoreGive(s_matrix_mutex);
}
```

**Et à la reconnexion WebSocket :**

```c
case WEBSOCKET_EVENT_CONNECTED:
    matrix_clear();   // ← Reset à chaque reconnexion
    accum_reset();
    // ...
    break;
```

## Logique finale

```
Paquet reçu complet
    → matrix_clear()          ← Scène vierge garantie
    → matrix_set_led(...)     ← Applique les LEDs du paquet
    → led_buffer_apply()      ← Envoie au driver physique
```

Chaque paquet décrit **intégralement** la scène à afficher. Le buffer est toujours propre avant écriture.

!!! note "Impact sur le mode delta"
    Cette approche signifie que le mode delta côté interface n'est utile que pour **réduire la bande passante réseau** (envoyer moins de données). Côté ESP32, le buffer est toujours entièrement recalculé. Pour de vraies animations à haute fréquence, envisager d'envoyer des frames complètes plutôt que des deltas.

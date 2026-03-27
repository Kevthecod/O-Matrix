# Signaux Dégradés — GSCLK & SCLK

## Problème 1 : Signal GSCLK bruité au 6ème TLC

### Symptôme observé

À l'oscilloscope (UNI-T UTD2102CEX+, 200 ns/div) :

- **Fréquence mesurée** : 1.00001 MHz ✓ (correcte)
- **Amplitude** : ~1.12 V ✗ (devrait être 3.3 V)
- **Forme** : très perturbée — pics parasites, oscillations, pas un carré propre

<figure markdown>
  ![gsclk_sig](../assets/courbe (6).jpg){ width=640 height=480 }
  <figcaption>Signal GSCLK aux bornes du 6ème TLC — cliquer pour agrandir</figcaption>
</figure>

### Causes identifiées

**Cause principale — Charge capacitive excessive**

19 TLC5940 en parallèle sur la même ligne GSCLK représentent une capacité parasite cumulée importante. Un GPIO ESP32 ne peut sourcer que **~12 mA** — insuffisant pour driver 19 entrées CMOS à 1 MHz.

**Cause secondaire — Longueur des pistes**

Des pistes longues sans terminaison créent des **réflexions** à 1 MHz, visibles comme des oscillations sur les fronts.

### Solutions

=== "Solution matérielle — Buffer 74HC244"

    Insérer un **buffer de ligne 74HC244** entre l'ESP32 et la chaîne TLC :
    
    ```
    ESP32 GPIO21 ──► [74HC244] ──► GSCLK de tous les TLC
    ESP32 GPIO18 ──► [74HC244] ──► SCLK de tous les TLC
    ESP32 GPIO23 ──► [74HC244] ──► MOSI de tous les TLC
    ```
    
    Le 74HC244 peut sourcer **±35 mA** par sortie, largement suffisant pour 19 TLC.

=== "Solution électronique — Résistance + condensateur"

    Pour amortir les réflexions sans buffer :
    
    ```
    ESP32 GPIO21 ──[33Ω]──┬── TLC1 GSCLK
                           ├── TLC2 GSCLK
                           │   ...
                           └── TLC19 GSCLK
    
    + 100 nF entre GSCLK et GND au plus près de chaque TLC
    ```
<figure markdown>
  ![gsclk_sig](../assets/sig (1).jpg){ width=640 height=480 }
  <figcaption>Signal GSCLK aux bornes de l'esp32 — cliquer pour agrandir</figcaption>
</figure>
<figure markdown>
  ![gsclk_sig](../assets/sig (4).jpg){ width=640 height=480 }
  <figcaption>Signal GSCLK aux bornes du 8ème TLC — cliquer pour agrandir</figcaption>
</figure>
On observe toujours une dégradation à partir du 8ème TLC5940.
### Checklist de diagnostic

- [ ] Mesurer VCC des TLC (broche 21) → doit être 3.3 V ± 0.1 V
- [ ] Mesurer GND des TLC (broche 22) → doit être 0 V
- [ ] Tester le signal GSCLK **sans** les TLC connectés → doit être un beau carré 3.3 V
- [ ] Ajouter 100 nF de découplage sur chaque carte TLC si pas encore fait

---

## Problème 2 : Signal SCLK sinusoïdal à 2.4 kHz

### Symptôme observé

À l'oscilloscope (50 ns/div) :

- **Fréquence mesurée** : 2.40003 MHz ✗ (devrait être 5 MHz)
- **Amplitude** : ~1.12 V 
- **Forme** : **sinusoïdale / triangulaire** — pas du tout un signal numérique

<figure markdown>
  ![gsclk_sig](../assets/courbe (3).jpg){ width=640 height=480 }
  <figcaption>Signal CLK aux bornes du 8ème TLC — cliquer pour agrandir</figcaption>
</figure>

<figure markdown>
  ![gsclk_sig](../assets/sig (5).jpg){ width=640 height=480 }
  <figcaption>Signal CLK aux bornes du 8ème TLC (après ajout de la résistance série) — cliquer pour agrandir</figcaption>
</figure>

!!! danger "Critique"
    Ce n'est plus du tout un signal SPI CLK utilisable. À 2.4 MHz, les TLC ne peuvent pas recevoir de données correctement.

### Causes identifiées

**Cause 1 — Court-circuit ou surcharge sur SCLK**

La capacité cumulée de 19 TLC sur le SCLK (5 MHz) est encore plus critique que pour le GSCLK. Le GPIO ESP32 ne peut pas charger/décharger cette capacité assez vite.

**Cause 2 — GND flottant**

Un signal oscillant autour de 0 V de façon asymétrique indique un **GND mal connecté** entre l'ESP32 et les TLC (connexion de masse manquante ou résistance élevée).

**Cause 3 — Court-circuit accidentel**

Un court-circuit entre SCLK et une autre ligne peut provoquer une chute de tension et une fréquence parasite.

### Solutions

=== "Diagnostic — Test sans charge"

    1. Déconnecter **toutes** les cartes TLC de la ligne SCLK
    2. Mesurer le SCLK sur GPIO18 directement
    3. Si le signal est propre → problème de charge/court-circuit dans le câblage
    4. Si le signal est toujours mauvais → problème côté ESP32

=== "Solution — Buffer 74HC244 (recommandé)"

    Même solution que pour le GSCLK. Un seul 74HC244 peut buffer **4 signaux** en parallèle → un chip pour SCLK, MOSI, GSCLK, et un signal libre.

=== "Solution — Réduire la vitesse SPI"

    ```c
    spi_device_interface_config_t dev_tlc = {
        .clock_speed_hz = 1000000,   // 1 MHz pour le debug
        // Remonter progressivement : 2M → 5M → 10M
    };
    ```

=== "Solution — Vérifier les masses"

    - Mesurer la résistance entre GND ESP32 et GND cartes TLC (doit être < 1 Ω)
    - Vérifier toutes les connexions de masse entre les cartes
    - S'assurer que les condensateurs de découplage sont bien reliés au GND local de chaque carte

### Condensateurs de découplage recommandés

| Position | Valeur | Type |
|----------|--------|------|
| Près de chaque TLC (VCC-GND) | 100 nF | CMS 0402 |
| Sur rail d'alimentation | 10 µF | Électrolytique CMS |
| En entrée du buffer 74HC244 | 100 nF | CMS 0402 |

!!! tip "Règle d'or"
    Placer les condensateurs de découplage **le plus près possible** des broches VCC des circuits intégrés, avec les pistes les plus courtes possible vers GND.

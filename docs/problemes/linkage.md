# Erreur de Linkage ESP-IDF

## Symptôme

```
ld.exe: esp-idf/main/libmain.a(main.c.obj): in function `app_main':
main.c:104: undefined reference to `ws_client_start'
collect2.exe: error: ld returned 1 exit status
ninja: build stopped: subcommand failed.
```

## Contexte

Le projet était initialement structuré en plusieurs fichiers :

```
main/
├── main.c          ← appelle ws_client_start()
├── ws_client.c     ← définit ws_client_start()
├── ws_client.h
├── led_buffer.c
├── led_buffer.h
└── CMakeLists.txt
```

Malgré la présence physique de `ws_client.c` dans le dossier, le linker ne trouvait pas `ws_client_start`.

## Causes identifiées

### Cause 1 — Fichiers absents du projet réel

Les fichiers `ws_client.c` et `led_buffer.c` étaient présents sur la machine de développement mais **non copiés** dans le dossier du projet ESP-IDF réel (`Animation_3D/main/`).

**Vérification** :
```powershell
Get-ChildItem "C:\...\Animation_3D\main\" | Select-Object Name, Length
# ws_client.c absente ou taille = 0
```

### Cause 2 — Cache CMake obsolète

Même avec les fichiers présents, le dossier `build/` contenait un `libmain.a` ancien compilé sans `ws_client.c`.

**Solution** :
```powershell
Remove-Item -Recurse -Force build
idf.py build
```

### Cause 3 — idf_component.yml

La présence d'un fichier `idf_component.yml` dans `main/` peut interférer avec la résolution des dépendances et provoquer des comportements inattendus lors du linkage.

## Solution définitive appliquée

**Fusion de tout le code dans un seul `main.c`** — élimine toute possibilité de problème de linkage :

```cmake
# main/CMakeLists.txt — version finale
idf_component_register(
    SRCS "main.c"        # ← un seul fichier source
    INCLUDE_DIRS "."
    REQUIRES
        esp_wifi esp_event esp_netif nvs_flash
        esp_websocket_client freertos log driver esp_rom
)
```

Tout le code (WiFi, WebSocket, TLC5940, 74HC595, parser JSON, buffer) est dans `main.c`.

## Procédure de résolution complète

```powershell
# 1. Vérifier les fichiers présents
Get-ChildItem "C:\...\Animation_3D\main\"

# 2. Supprimer le cache
cd C:\...\Animation_3D
Remove-Item -Recurse -Force build

# 3. Recompiler
idf.py build

# 4. Flasher
idf.py -p COM3 flash monitor
```

!!! tip "Bonne pratique ESP-IDF"
    Pour les projets simples (< 2000 lignes), un seul `main.c` est plus simple à maintenir et évite les problèmes de linkage. Pour les projets plus complexes, utiliser des **composants ESP-IDF** séparés avec leur propre `CMakeLists.txt` dans des dossiers dédiés (pas dans `main/`).

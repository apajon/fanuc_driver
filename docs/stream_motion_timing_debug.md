# Instrumentation de timing Stream Motion — `FANUC_SM_TIMING_DEBUG`

## Objectif

Prouver expérimentalement le comportement du pipeline UDP Stream Motion en production, en
particulier :

- identifier la **branche réellement prise** à chaque appel de `getStatusPacket()` :
  `normal_recv`, `catch_up`, `exceeded_error`, `timeout_error` ;
- mesurer le **temps d'attente socket** (`receive_duration_us`) qui cadence le thread interne ;
- corréler chaque cycle avec le **`sendCommand()` correspondant** et son délai ;
- détecter les **cycles de rattrapage** (`catch_up`) et mesurer leur durée réelle.

Ce flag est **désactivé par défaut** (`OFF`). Il ne modifie aucun comportement fonctionnel.

---

## Contexte architectural

```
controller_manager (125 Hz)
  ├─ read()  → readJointAngles() → try_dequeue(robot_state_queue_)  [non bloquant]
  └─ write() → writeJointTarget() → enqueue(command_queue_)         [non bloquant]

rt_thread_  (FanucClient::streamMotionThread)
  while (is_streaming_):
    (1) getStatusPacket()  ← RECV UDP   ← pace la boucle (~8 ms par feedback robot)
    (2) interpolation depuis command_queue_
    (3) sendCommand()      ← SEND UDP
    (4) enqueue(robot_state_queue_)
```

La **cadence UDP est pilotée par le robot** (retour status ~8 ms), pas par le
`controller_manager`. Quand un paquet status est manqué ou retardé, la branche
`catch_up` de `getStatusPacket()` envoie une commande supplémentaire **sans attendre
de nouveau paquet**, produisant 1-2 cycles très courts.

---

## Fichiers modifiés

| Fichier | Nature de la modification |
|---------|--------------------------|
| `fanuc_libs/stream_motion/include/stream_motion/stream.hpp` | `enum SmDebugBranch`, `struct SmDebugRow`, prototype `dumpTimingCsv()`, membres gardés `sm_debug_rows_` / `sm_debug_cycle_` |
| `fanuc_libs/stream_motion/src/stream.cpp` | Helper `SmNowNs()`, instrumentation de `getStatusPacket()` et `sendCommand()`, flush CSV dans le destructeur, `reserve(200000)` dans le constructeur |
| `fanuc_libs/stream_motion/CMakeLists.txt` | `option(FANUC_SM_TIMING_DEBUG ... OFF)` + `target_compile_definitions(... PUBLIC ...)` |

> **Note CMake — `PUBLIC` obligatoire :** `fanuc_client` construit `StreamMotionConnection`
> et inclut le header. Le flag doit être identique dans les deux unités de compilation :
> avec `PRIVATE`, le buffer membre gardé serait présent dans `stream.cpp` mais absent pour
> `fanuc_client.cpp`, ce qui change le `sizeof` de la classe et provoque une **violation
> d'ODR / corruption mémoire** silencieuse.

---

## Colonnes CSV produites

Le fichier est écrit à la **destruction de `StreamMotionConnection`** (donc lors du
`on_cleanup` / reset du hardware interface, ou à la fin du process). Chemin configurable
via la variable d'environnement `FANUC_SM_TIMING_CSV` (défaut : `/tmp/fanuc_sm_timing.csv`).

| Colonne | Type | Description |
|---------|------|-------------|
| `cycle` | entier | Index monotone par appel à `getStatusPacket()` |
| `t_enter_ns` | ns | `steady_clock` à l'entrée de `getStatusPacket()` |
| `t_recv_start_ns` | ns | Juste avant `receive()` (branche `normal_recv` uniquement) |
| `t_recv_end_ns` | ns | Juste après `receive()` (branche `normal_recv` uniquement) |
| `t_exit_ns` | ns | À la sortie de `getStatusPacket()` |
| `branch` | string | `normal_recv` / `catch_up` / `exceeded_error` / `timeout_error` |
| `cmd_seq_before` | uint32 | `command_sequence_no_` avant l'appel |
| `status_seq_before` | uint32 | `status_sequence_no_` avant l'appel |
| `recv_status_seq` | uint32 | `status.sequence_no` reçu du robot (0 si pas de recv) |
| `cmd_seq_after` | uint32 | `command_sequence_no_` après l'appel |
| `status_seq_after` | uint32 | `status_sequence_no_` après l'appel |
| `recv_executed` | 0/1 | `1` si un paquet UDP a été reçu ce cycle, `0` en `catch_up` |
| `receive_duration_us` | µs | Durée totale du recv (≈ délai d'attente robot) ; 0 si `recv_executed=0` |
| `t_send_start_ns` | ns | Juste avant `socket_impl_->send()` dans `sendCommand()` |
| `t_send_end_ns` | ns | Juste après `socket_impl_->send()` dans `sendCommand()` |
| `send_cmd_seq` | uint32 | `command_sequence_no_` correspondant au paquet envoyé (valeur host-order) |
| `send_recorded` | 0/1 | `1` si `sendCommand()` a bien complété ce cycle |
| `send_duration_us` | µs | Durée du `send()` UDP |

> **`cmd_seq_before` vs `send_cmd_seq` :** `cmd_seq_before` est la valeur avant
> `getStatusPacket()`. `send_cmd_seq` est `command_sequence_no_` au moment de
> `sendCommand()`, soit après l'incrément (`cmd_seq_before + 1` en cas normal).

---

## Signature d'un cycle de rattrapage dans le CSV

Le pattern à chercher :

```
...,branch,          ...,recv_executed,receive_duration_us,...
...,normal_recv,     ...,1,            ~8000,             ...  ← cycle normal
...,catch_up,        ...,0,            0,                 ...  ← rattrapage
...,normal_recv,     ...,1,            ~8000,             ...  ← retour normal
```

Calcul du delta inter-send (non directement dans le CSV) :

```python
import pandas as pd
df = pd.read_csv("/tmp/fanuc_sm_timing.csv")
df["send_to_send_us"] = df["t_send_end_ns"].diff() / 1000
print(df[["cycle", "branch", "recv_executed", "receive_duration_us", "send_to_send_us"]].to_string())
```

Résultat attendu sur un catch-up :

| cycle | branch | recv_executed | receive_duration_us | send_to_send_us |
|-------|--------|---------------|---------------------|-----------------|
| N-1   | normal_recv | 1 | ~8000 | ~8000 |
| N     | normal_recv | 1 | ~16000 | ~16000 |  ← lag robot |
| N+1   | catch_up    | 0 | 0     | ~0–500 |  ← envoi immédiat |
| N+2   | normal_recv | 1 | ~8000 | ~8000 |  ← retour normal |

---

## Capacité du buffer

```
sm_debug_rows_.reserve(200000);  // ~26 min @ 125 Hz (200000 / 125 = 1600 s)
```

Au-delà de 200 000 cycles, les nouvelles lignes sont **silencieusement ignorées** (garde
`size() < capacity()` dans `sm_dbg_finalize`). Aucune réallocation, aucun crash, aucune
perturbation du timing. Pour un diagnostic de 30–60 s, la capacité est largement
suffisante.

---

## Activation et build

```bash
# Reconstruire avec le flag (--packages-up-to pour reconstruire aussi les dépendants
# qui partagent le layout de classe StreamMotionConnection)
colcon build --packages-up-to fanuc_hardware_interface \
  --cmake-clean-cache \
  --cmake-args -DFANUC_SM_TIMING_DEBUG=ON

source install/setup.bash
```

---

## Collecte des données

```bash
# Optionnel : chemin CSV personnalisé (défaut : /tmp/fanuc_sm_timing.csv)
export FANUC_SM_TIMING_CSV=/tmp/fanuc_sm_timing.csv

# Lancer le driver normalement (launch, ros2 control, etc.)
# ...

# Arrêt propre pour déclencher le flush CSV dans le destructeur
ros2 control set_hardware_component_state <hardware_name> inactive
ros2 control set_hardware_component_state <hardware_name> unconfigured

# Vérification immédiate
ls -lh /tmp/fanuc_sm_timing.csv
head -3 /tmp/fanuc_sm_timing.csv
grep -c catch_up /tmp/fanuc_sm_timing.csv
grep catch_up   /tmp/fanuc_sm_timing.csv | head -5
```

---

## Désactivation (build de production)

Sans `-DFANUC_SM_TIMING_DEBUG=ON`, le préprocesseur supprime **tous** les blocs gardés :
aucune donnée membre ajoutée, aucun appel `SmNowNs()`, aucune allocation, aucun I/O
fichier. Le binaire et le comportement sont strictement identiques à la version sans patch.

---

## Limites connues

| Limitation | Impact |
|------------|--------|
| Flush uniquement à la destruction | Pas de données si le process est tué (`SIGKILL`). Utiliser `SIGTERM` ou un arrêt propre ros2_control. |
| `send_cmd_seq` = host-order | Valeur correcte pour la corrélation, mais différente de la valeur byte-swappée présente dans le paquet UDP sur le fil. |
| `recv_status_seq = 0` en `catch_up` | Non ambigu : `status_seq_before` et `status_seq_after` suffisent à reconstituer l'état des deux compteurs. |
| Buffer non thread-safe | `sm_debug_rows_` est écrit par `rt_thread_` uniquement et lu uniquement dans le destructeur après join. Pas de data race. |

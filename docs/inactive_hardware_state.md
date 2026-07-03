# Rapport de bug & correctif — `hardware_components_initial_state: inactive` non fonctionnel

## Problème

Lorsque `FanucHardwareInterface` est configuré en état initial `inactive` via
`hardware_components_initial_state` dans `ros2_controllers.yaml`, le composant retombe
immédiatement en `unconfigured` dès le premier cycle de contrôle, rendant l'objectif
inatteignable.

**Symptômes observés dans les logs :**

```
[resource_manager]: 'configure' hardware 'FanucJoints'
[FR_HW_Interface]: Successfully connected to the robot.
[FR_HW_Interface]: FANUC ROS2 HW no longer streaming (this is normal during shutdown).
[controller_manager]: Deactivating following hardware components as their read cycle
                      resulted in an error: [ FanucJoints ]
[planning_scene_monitor]: The complete state of the robot is not yet known.
                          Missing crx_joint_1, crx_joint_2, ...
```

Overrun constaté : `Read time: 1000258 µs` (attendu < 8000 µs) — causé par le
`stopRealtimeStream()` bloquant appelé depuis `read()` pendant l'état `INACTIVE`.

## Contexte : pourquoi vouloir `inactive` ?

La motivation est de **démarrer le `controller_manager` sans robot disponible** tout en
gardant les state interfaces joints visibles pour `joint_state_broadcaster`. Cela permet :

- `joint_state_broadcaster` de spawner et publier `/joint_states`
- MoveIt / `planning_scene_monitor` de démarrer sans timeout
- La connexion réelle au robot d'être différée à une activation manuelle explicite
  (`ros2 control set_hardware_component_state FanucJoints active`) une fois le robot présent

L'état `unconfigured` atteint en cas d'échec de `on_configure` (robot absent) ne publie
aucune interface, ce qui bloque toute la stack MoveIt au démarrage.

## Cause racine

### Comportement ros2_control (Jazzy)

Contrairement à l'intuition, ros2_control appelle `read()` **pour les composants `INACTIVE`
ET `ACTIVE`** (source : [`hardware_component.cpp`][hw_comp] lignes 352–374) :

```cpp
// hardware_component.cpp — ros2_control Jazzy
return_type HardwareComponent::read(...)
{
  if (lifecycleStateThatRequiresNoAction(...))  // UNCONFIGURED / FINALIZED → OK sans appel
    return return_type::OK;
  if (state == PRIMARY_STATE_INACTIVE || state == PRIMARY_STATE_ACTIVE)
  {
    const auto trigger_result = impl_->trigger_read(...);  // ← appelé en INACTIVE aussi !
    if (trigger_result.result == return_type::ERROR)
      error();  // → transition INACTIVE → UNCONFIGURED
  }
}
```

`UNCONFIGURED` et `FINALIZED` sont court-circuités avec OK. `INACTIVE` ne l'est pas.

### Comportement du driver

`FanucHardwareInterface::read()` (avant correctif) effectuait le test suivant dès son entrée :

```cpp
robot_status_.is_connected = fanuc_client_ != nullptr && fanuc_client_->isStreaming();
if (!robot_status_.is_connected)
{
    fanuc_client_->stopRealtimeStream();  // ← BLOQUANT ~1 s (attente UDP qui n'arrive pas)
    return hardware_interface::return_type::ERROR;  // ← déclenche error() du composant
}
```

`isStreaming()` retourne `false` en `INACTIVE` (le stream UDP n'a pas encore démarré, c'est
normal). Le driver interprète cela comme une perte de connexion en cours d'opération, alors
qu'il s'agit simplement d'un composant intentionnellement non activé.

### Bug secondaire — déréférencement null avec `use_rmi=0`

Avec `use_rmi=0`, `rmi_connection_` est `nullptr`. L'appel à `stopRealtimeStream()`
depuis `read()` en état `INACTIVE` appelle `rmi_connection_->abort()` inconditionnellement
([`fanuc_client.cpp:710`][fanuc_client]), provoquant un déréférencement null à chaque cycle.

## Correctif appliqué

### `fanuc_hardware_interface/src/hardware_interface.cpp`

Ajout d'un garde lifecycle-state en tête de `read()` **et** de `write()`, avant tout accès
au stream :

```cpp
// read() :
if (get_lifecycle_state().id() ==
    lifecycle_msgs::msg::State::PRIMARY_STATE_INACTIVE)
{
    return hardware_interface::return_type::OK;
}

// write() : garde identique
```

**Comportement après correctif :**

| État lifecycle | Robot présent | Comportement |
|----------------|--------------|--------------|
| `INACTIVE` | Oui ou Non | `read()`/`write()` retournent `OK` silencieusement. Aucun appel réseau. Le composant reste `INACTIVE`, ses state interfaces restent visibles. |
| `ACTIVE` | Oui | Comportement inchangé — données cycliques Stream Motion. |
| `ACTIVE` | Non (perte de stream) | Comportement inchangé — `ERROR` → désactivation des controllers. |

### `fanuc_hardware_interface/CMakeLists.txt` et `package.xml`

Ajout de la dépendance `lifecycle_msgs` (nécessaire pour `lifecycle_msgs::msg::State::PRIMARY_STATE_INACTIVE`) :

```cmake
find_package(lifecycle_msgs REQUIRED)
# …
target_link_libraries(fanuc_robot_driver PUBLIC
  …
  lifecycle_msgs::lifecycle_msgs__rosidl_generator_cpp  # INTERFACE, headers-only
)
```

```xml
<depend>lifecycle_msgs</depend>
```

### `fanuc_hardware_interface/src/hardware_interface.cpp` — include

```cpp
#include "lifecycle_msgs/msg/state.hpp"
```

## Neutralité RMI

Le correctif est **indépendant du flag `use_rmi`** :

- Le chemin cyclique `read()`/`write()` est 100% Stream Motion UDP dans les deux modes.
  `use_rmi` ne concerne que le bootstrap `on_configure`/`on_activate` (TCP RMI, handshake,
  `FRC_Initialize`, `STREAM_MOTN.TP`).
- Le garde s'applique sur l'**état lifecycle**, pas sur `use_rmi` ni sur `isStreaming()`.
- En `use_rmi=1`, le comportement en `ACTIVE` (là où RMI a un effet) est strictement inchangé.

## Configuration YAML requise

Pour que le composant Stream Motion démarre en `INACTIVE` :

```yaml
# ros2_controllers.yaml
controller_manager:
  ros__parameters:
    hardware_components_initial_state:
      unconfigured:
        - "FanucEtherCAT"
        - "FestoEtherCAT"
      inactive:
        - "FanucJoints"   # ← plugin fanuc_robot_driver/FanucHardwareInterface
```

> ⚠️ Vérifier que le nom correspond exactement à l'attribut `name=""` de la balise
> `<ros2_control>` dans le fichier XACRO assembleur.

## Activation manuelle du robot

Une fois le robot présent et prêt :

```bash
ros2 control set_hardware_component_state FanucJoints active
```

Cette commande déclenche `on_activate()` → `startRealtimeStream()` → attente du status
packet UDP (2 s max). Si le robot ne répond pas, le composant reste `INACTIVE` (pas de crash
du `controller_manager`).

## Validation

```bash
colcon build --packages-select fanuc_hardware_interface
# → Finished <<< fanuc_hardware_interface [2.4s] — EXIT:0
```

## Lien avec l'issue upstream

Ce comportement a été signalé au dépôt officiel `FANUC-CORPORATION/fanuc_driver` :

> **Issue : `FanucHardwareInterface: read()/write() return ERROR when INACTIVE, preventing
> hardware_components_initial_state: inactive from working`**
>
> Branche concernée : `feat/use-rmi-flag`
> ROS 2 Jazzy — `use_rmi=0`

[hw_comp]: https://github.com/ros-controls/ros2_control/blob/jazzy/hardware_interface/src/hardware_component.cpp#L352
[fanuc_client]: ../fanuc_libs/fanuc_client/src/fanuc_client.cpp

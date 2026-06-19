# Rapport d'implémentation — flag `use_rmi`

## Objectif

Permettre au driver ros2_control FANUC de fonctionner en **Stream Motion uniquement** (UDP),
sans aucune connexion RMI (TCP). Le bootstrap côté contrôleur normalement réalisé par RMI
(`FRC_Initialize`, lancement de `STREAM_MOTN.TP`, activation du remote-motion) est alors
fourni séparément via **EtherCAT** sur un autre canal.

Le flag `use_rmi` vaut **`true` par défaut** : le comportement existant est inchangé.
À `false` : **zéro RMI / zéro TCP**, Stream Motion seul.

## Contexte architectural

Le driver utilise deux canaux réseau distincts :

| Canal | Transport | Rôle |
|-------|-----------|------|
| Stream Motion | UDP port 60015 (`sockpp::udp_socket`) | Toutes les données cycliques temps réel (cibles articulaires, feedback position/vitesse, IO, force, limites, capability) |
| RMI | TCP port 16001 (`sockpp::tcp_connector`) | Bootstrap uniquement : `FRC_Initialize` + `programCallNonBlocking("STREAM_MOTN")`. Aucune donnée cyclique |

Point clé : `RMISingleton::creatNewRMIInstance(...)` construit un `rmi::RMIConnection` qui
**ouvre la socket TCP dès sa construction**. Pour garantir zéro TCP, `rmi_connection_` doit
donc rester `nullptr` quand `use_rmi=false` (et non simplement court-circuiter `connect()`).

## Modifications

### Cœur C++ — `fanuc_client`

- **`fanuc_libs/fanuc_client/include/fanuc_client/fanuc_client.hpp`**
  - 6ᵉ paramètre constructeur : `bool use_rmi = true`.
  - Membre `const bool use_rmi_;` déclaré **avant** `rmi_connection_` (ordre d'initialisation).

- **`fanuc_libs/fanuc_client/src/fanuc_client.cpp`**
  - Constructeur : `rmi_connection_` reste `nullptr` quand `use_rmi=false`
    (aucun appel à `creatNewRMIInstance` → aucune ouverture TCP) ; `connect(5)` gardé par `if (use_rmi_)`.
  - Destructeur : `abort()` et `disconnect()` gardés par `if (rmi_connection_)`.
  - `startRMI()` : early-return si `!use_rmi_` (filet de sécurité contre le déréférencement null).
  - `startRealtimeStream()` : bootstrap RMI (`startRMI()` + `programCallNonBlocking("STREAM_MOTN")`)
    conditionné à `do_motn_ctrl_ && use_rmi_`. La boucle d'attente `status & 0x1` est **inchangée**
    (le TP démarré via EtherCAT positionne ce bit).

### HW interface

- **`fanuc_hardware_interface/include/fanuc_robot_driver/hardware_interface.hpp`**
  - Membre `bool use_rmi_{ true };`.

- **`fanuc_hardware_interface/src/hardware_interface.cpp`**
  - Parsing du paramètre `use_rmi` via `.find()` (défaut `true` si absent → configs existantes intactes).
  - Valeur transmise au constructeur `FanucClient`.

### Xacro / launch

- Paramètre `use_rmi` ajouté aux deux macros :
  - `fanuc_hardware_interface/config/crx_physical_ros2_control_macro.xacro`
  - `fanuc_hardware_interface/config/6dof_robot_physical_ros2_control_macro.xacro`
- `<xacro:arg name="use_rmi" default="1"/>` ajouté aux 7 robots `fanuc_hardware_interface/robot/*.urdf.xacro`
  (`crx3ia`, `crx5ia`, `crx10ia`, `crx10ia_l`, `crx20ia_l`, `crx30ia`, `6dof_robot`).
- `fanuc_hardware_interface/launch/fanuc_physical_control.launch.py` :
  `LaunchConfiguration("use_rmi")` + mapping xacro + `DeclareLaunchArgument("use_rmi", default_value="1")`.

## Utilisation

```bash
# Comportement standard (RMI activé) — défaut
ros2 launch fanuc_hardware_interface fanuc_physical_control.launch.py

# Stream Motion uniquement, sans RMI/TCP (bootstrap fourni via EtherCAT)
ros2 launch fanuc_hardware_interface fanuc_physical_control.launch.py use_rmi:=0
```

## Validation

- `colcon build --packages-select fanuc_libs fanuc_hardware_interface --cmake-args -DCMAKE_BUILD_TYPE=Release` : **PASS** (exit 0).
- Seul message stderr : un avertissement de dépréciation **préexistant** sur `on_init` (sans rapport avec ces changements).

## Points d'attention

- Les APIs RMI explicites **hors chemin HW** (`startMotionControl`, `writeJointTargetRMI`,
  `readJointAnglesRMI`, `stopMotionControl`) sont **inchangées** : les appeler avec `use_rmi=false`
  déréférencerait un `nullptr`. Hors périmètre de l'interface HW, mais à garder en tête.
- **Risque ouvert à valider sur contrôleur (réel ou virtuel)** : que `STREAM_MOTN.TP` lancé via
  EtherCAT **sans** `FRC_Initialize` autorise réellement le mouvement (`motion_possible=true`).
  Sinon le flux est accepté mais aucun mouvement n'est produit. Le bootstrap EtherCAT doit donc
  aussi fournir l'équivalent du remote-motion (`FRC_Initialize`), pas uniquement lancer le `.TP`.

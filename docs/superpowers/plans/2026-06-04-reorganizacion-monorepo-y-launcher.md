# Reorganización monorepo + launcher maestro — Plan de implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidar el fork `ETM2097/Proyecto` en un monorepo de una sola rama `main` organizado por subsistemas, con un launch file maestro que arranca sim + bridge + SCADA, y limpiar las ramas obsoletas.

**Architecture:** Se parte de `autonomous_movement` (paquete ROS2 + scada). Se reubica el paquete a `navegacion/`, se sincroniza con el código actualizado de `~/test_ros2_pro/src/mir2`, se copian planos los subsistemas `comunicaciones` y `database`, se añade `bringup.launch.py`, README global y `.gitignore`. Todo queda en el working tree para un único commit del usuario; las ramas remotas obsoletas se borran del fork.

**Tech Stack:** Git, ROS2 Jazzy, colcon/ament_cmake, Python launch, bash.

> **NOTA DE COMMITS (instrucción del usuario, prioritaria):** Claude **NO** hace `git commit` en ningún paso. Los pasos usan `git mv` / `git add` para *stagear*, pero el commit final (uno solo) lo hace el usuario. Donde un plan TDD normal pondría "commit", aquí ponemos "stage + verify".

> **Working dir:** todos los comandos se ejecutan desde `/home/enric_talens/Documents/pol/Proyecto` salvo que se indique otra cosa.

---

## File Structure

| Path | Responsabilidad | Acción |
|---|---|---|
| `navegacion/` | Paquete ROS2 `mir_nav2_robodk` (todo el contenido actual del repo raíz) | Crear (mover contenido) |
| `navegacion/launch/bringup.launch.py` | Launcher maestro: sim + bridge + scada | Crear |
| `navegacion/launch/simulation.launch.py` | Sim Gazebo+Nav2+RViz | Mover (intacto) |
| `navegacion/scripts/robodk_bridge.py` | Bridge RoboDK-MQTT-ROS2 | Mover + sobrescribir con versión actualizada + default host=localhost |
| `navegacion/config/nav2_params.yaml` | Params Nav2 | Mover + sobrescribir con versión actualizada |
| `navegacion/scada/` | App Qt SCADA | Mover (desde `scada/` raíz) |
| `navegacion/CMakeLists.txt` | Build/install del paquete | Mover + añadir install de `scada/` |
| `comunicaciones/` | Subsistema comunicaciones (c/ + d/) | Crear (copia plana de `origin/comunicaciones`) |
| `database/` | Subsistema DB (bridge.py + .sql) | Crear (copia plana de `origin/database`) |
| `README.md` (raíz) | Visión global + cómo lanzar | Crear nuevo (el viejo va a `navegacion/README.md`) |
| `.gitignore` (raíz) | Ignorar build/install/log/__pycache__/.vscode | Crear |
| `docs/superpowers/...` | Spec + este plan | Ya existen, se quedan en raíz |

---

## Task 1: Crear la rama consolidada `main`

**Files:** ninguno (solo refs git)

- [ ] **Step 1: Verificar estado y rama actual**

Run:
```bash
cd /home/enric_talens/Documents/pol/Proyecto
git status -sb
```
Expected: `## autonomous_movement...origin/autonomous_movement` y, como mucho, untracked `.vscode/`, `scada/__pycache__/`, `docs/`.

- [ ] **Step 2: Asegurar que todas las refs remotas están fetchadas**

Run:
```bash
git fetch --all --prune
```
Expected: lista de ramas remotas incluyendo `origin/comunicaciones`, `origin/database`, `origin/objetos`, `origin/robots`, `origin/Plantilla`, `origin/main`, `origin/autonomous_movement`.

- [ ] **Step 3: Borrar la rama local `main` vieja (la ref remota `origin/main` se conserva como backup)**

Run:
```bash
git branch -D main
```
Expected: `Deleted branch main (was a4893ad).`

- [ ] **Step 4: Crear y cambiar a la nueva rama `main` desde `autonomous_movement`**

Run:
```bash
git checkout -b main autonomous_movement
git status -sb
```
Expected: `## main` y el working tree con el paquete + `scada/` presentes.

---

## Task 2: Reubicar el paquete ROS2 a `navegacion/`

**Files:** Mover todo el contenido del paquete (raíz) a `navegacion/`.

- [ ] **Step 1: Crear la carpeta destino**

Run:
```bash
mkdir -p navegacion
```

- [ ] **Step 2: Mover (git mv) todos los items del paquete a `navegacion/`**

Run:
```bash
git mv CMakeLists.txt package.xml README.md config launch maps meshes models rviz scripts urdf worlds scada navegacion/
```
Expected: sin salida (éxito). `scada/__pycache__` y `scripts/__pycache__` son untracked; si `git mv scada` se queja por contenido untracked, mover el resto y luego: `mv scada navegacion/` para la carpeta entera.

- [ ] **Step 3: Verificar el layout**

Run:
```bash
ls navegacion/
ls -d */ | grep -vE 'docs|navegacion'
```
Expected: `navegacion/` contiene `CMakeLists.txt package.xml README.md config launch maps meshes models rviz scripts urdf worlds scada`. En la raíz, además de `docs/` y `navegacion/`, no debería quedar ningún directorio del paquete.

---

## Task 3: Sincronizar `navegacion/` con el código actualizado de `~/test_ros2_pro/src/mir2`

**Files:**
- Modify: `navegacion/scripts/robodk_bridge.py` (sobrescribir con versión actualizada)
- Modify: `navegacion/config/nav2_params.yaml` (sobrescribir con versión actualizada)

- [ ] **Step 1: Copiar los dos archivos actualizados (working tree de mir2)**

Run:
```bash
cp ~/test_ros2_pro/src/mir2/scripts/robodk_bridge.py navegacion/scripts/robodk_bridge.py
cp ~/test_ros2_pro/src/mir2/config/nav2_params.yaml navegacion/config/nav2_params.yaml
```

- [ ] **Step 2: Verificar que entraron los cambios actualizados**

Run:
```bash
grep -n "len(coords) == 3" navegacion/scripts/robodk_bridge.py
grep -n "inflation_radius: 0.55" navegacion/config/nav2_params.yaml
```
Expected: ambos `grep` devuelven una línea (confirma yaw-por-destino en el bridge e inflation actualizado en params).

- [ ] **Step 3: Cambiar el default de `robodk_host` a `localhost`**

El archivo actualizado trae IPs de laboratorio como default. Sustituir en los dos sitios.

Run:
```bash
grep -n "robodk_host" navegacion/scripts/robodk_bridge.py
```
Expected: dos líneas — el docstring (`- robodk_host RoboDK API host (default '10.51.67.104')`) y `self.declare_parameter('robodk_host', '192.168.137.8')`.

Editar ambas para que el default sea `'localhost'`:
- Docstring → `  - robodk_host RoboDK API host (default 'localhost')`
- `self.declare_parameter('robodk_host', 'localhost')`

- [ ] **Step 4: Verificar el cambio de host**

Run:
```bash
grep -n "robodk_host" navegacion/scripts/robodk_bridge.py
python3 -m py_compile navegacion/scripts/robodk_bridge.py && echo "SYNTAX OK"
```
Expected: ambas líneas muestran `localhost`; imprime `SYNTAX OK`.

---

## Task 4: Copia plana de `comunicaciones` y `database`

**Files:**
- Create: `comunicaciones/` (contenido de `origin/comunicaciones`)
- Create: `database/` (contenido de `origin/database`)

- [ ] **Step 1: Extraer el árbol de `origin/comunicaciones` a `comunicaciones/`**

Run:
```bash
mkdir -p comunicaciones database
git archive origin/comunicaciones | tar -x -C comunicaciones/
git archive origin/database | tar -x -C database/
```
Expected: sin errores.

- [ ] **Step 2: Verificar el contenido importado**

Run:
```bash
ls comunicaciones/
ls database/
```
Expected: `comunicaciones/` contiene `c d README.md .gitignore`; `database/` contiene `bridge.py db_structure.sql queries.sql requirements.txt .env`.

- [ ] **Step 3: Confirmar que el `.env` no tiene secretos reales (decisión ya tomada, verificación rápida)**

Run:
```bash
cat database/.env
```
Expected: revisar visualmente; ya confirmado por el usuario que no hay secretos. Si apareciera algo sensible, parar y avisar.

---

## Task 5: Crear `bringup.launch.py`

**Files:**
- Create: `navegacion/launch/bringup.launch.py`

- [ ] **Step 1: Crear el launcher maestro**

Crear `navegacion/launch/bringup.launch.py` con este contenido exacto:

```python
"""
Master bringup: full simulation + RoboDK-MQTT bridge + SCADA.

Includes simulation.launch.py (Gazebo + MiR + Nav2 + RViz) and adds the
RoboDK-MQTT-ROS2 bridge node and the SCADA Qt app. Each extra piece can be
toggled with its own argument.

Usage:
  ros2 launch mir_nav2_robodk bringup.launch.py
  ros2 launch mir_nav2_robodk bringup.launch.py bridge:=false scada:=false
  ros2 launch mir_nav2_robodk bringup.launch.py mqtt_host:=localhost robodk_host:=192.168.1.50
"""

import sys

from launch import LaunchDescription
from launch.actions import (
    DeclareLaunchArgument,
    IncludeLaunchDescription,
    ExecuteProcess,
)
from launch.conditions import IfCondition
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration, PathJoinSubstitution
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare


def generate_launch_description():
    pkg_share = FindPackageShare('mir_nav2_robodk')

    # --- arguments ---
    slam_arg = DeclareLaunchArgument(
        'slam', default_value='True',
        description='Run SLAM (True) or localization with a map (False)')
    world_arg = DeclareLaunchArgument(
        'world',
        default_value=PathJoinSubstitution([pkg_share, 'worlds', 'station.sdf']),
        description='Path to Gazebo world SDF file')
    map_arg = DeclareLaunchArgument(
        'map',
        default_value=PathJoinSubstitution([pkg_share, 'maps', 'warehouse.yaml']),
        description='Map yaml for localization (used when slam:=False)')
    rviz_arg = DeclareLaunchArgument(
        'rviz', default_value='true', description='Launch RViz2')
    bridge_arg = DeclareLaunchArgument(
        'bridge', default_value='true', description='Launch the RoboDK-MQTT bridge')
    scada_arg = DeclareLaunchArgument(
        'scada', default_value='true', description='Launch the SCADA Qt app')
    mqtt_host_arg = DeclareLaunchArgument(
        'mqtt_host', default_value='broker.hivemq.com',
        description='MQTT broker host for the bridge')
    robodk_host_arg = DeclareLaunchArgument(
        'robodk_host', default_value='localhost',
        description='RoboDK API host for the bridge')

    # --- simulation (reuse the existing launch) ---
    simulation = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            PathJoinSubstitution([pkg_share, 'launch', 'simulation.launch.py'])
        ),
        launch_arguments={
            'slam': LaunchConfiguration('slam'),
            'world': LaunchConfiguration('world'),
            'map': LaunchConfiguration('map'),
            'rviz': LaunchConfiguration('rviz'),
        }.items(),
    )

    # --- RoboDK-MQTT-ROS2 bridge ---
    bridge_node = Node(
        condition=IfCondition(LaunchConfiguration('bridge')),
        package='mir_nav2_robodk',
        executable='robodk_bridge.py',
        name='robodk_bridge',
        output='screen',
        parameters=[{
            'use_sim_time': True,
            'mqtt_host': LaunchConfiguration('mqtt_host'),
            'robodk_host': LaunchConfiguration('robodk_host'),
        }],
    )

    # --- SCADA Qt app (installed under share/<pkg>/scada) ---
    scada_proc = ExecuteProcess(
        condition=IfCondition(LaunchConfiguration('scada')),
        cmd=[sys.executable, PathJoinSubstitution([pkg_share, 'scada', 'main.py'])],
        output='screen',
    )

    return LaunchDescription([
        slam_arg, world_arg, map_arg, rviz_arg,
        bridge_arg, scada_arg, mqtt_host_arg, robodk_host_arg,
        simulation,
        bridge_node,
        scada_proc,
    ])
```

- [ ] **Step 2: Verificar sintaxis Python**

Run:
```bash
python3 -m py_compile navegacion/launch/bringup.launch.py && echo "SYNTAX OK"
```
Expected: `SYNTAX OK`.

---

## Task 6: Actualizar `CMakeLists.txt` para instalar `scada/`

**Files:**
- Modify: `navegacion/CMakeLists.txt`

- [ ] **Step 1: Añadir `scada` a la lista de `install(DIRECTORY ...)`**

En `navegacion/CMakeLists.txt`, en el bloque `install(DIRECTORY ... DESTINATION share/${PROJECT_NAME})`, añadir `scada` junto a las demás carpetas. El bloque queda:

```cmake
install(DIRECTORY
  config
  launch
  maps
  meshes
  models
  rviz
  scada
  urdf
  worlds
  DESTINATION share/${PROJECT_NAME}
)
```

- [ ] **Step 2: Verificar**

Run:
```bash
grep -n "scada" navegacion/CMakeLists.txt
```
Expected: una línea con `  scada` dentro del bloque install.

---

## Task 7: Crear `.gitignore` y `README.md` global

**Files:**
- Create: `.gitignore` (raíz)
- Create: `README.md` (raíz)

- [ ] **Step 1: Crear `.gitignore`**

Crear `.gitignore` en la raíz con este contenido:

```gitignore
# ROS2 / colcon
build/
install/
log/

# Python
__pycache__/
*.py[cod]

# Editores
.vscode/
*.swp

# SO
.DS_Store
```

- [ ] **Step 2: Crear el README global**

Crear `README.md` en la raíz con este contenido:

```markdown
# Proyecto PR2-A1 — Célula robotizada (monorepo)

Monorepo del proyecto, organizado por subsistemas.

## Estructura

| Carpeta | Subsistema |
|---|---|
| `navegacion/` | Paquete ROS2 `mir_nav2_robodk`: simulación Gazebo del AMR (MiR), Nav2, bridge RoboDK-MQTT-ROS2 y SCADA. |
| `comunicaciones/` | Firmware ESP-IDF/Arduino (`c/`) y documentación (`d/`). |
| `database/` | Puente MQTT↔SQL/noSQL y esquema de base de datos. |

## Lanzar la simulación completa

Requisitos: ROS2 Jazzy, Gazebo Harmonic, Nav2. El paquete `navegacion/` se
compila con colcon (se encuentra automáticamente aunque esté anidado).

```bash
# Desde la raíz del repo (o desde tu workspace colcon que contenga este repo en src/)
colcon build --packages-select mir_nav2_robodk
source install/setup.bash

# Arranca TODO: Gazebo + MiR + Nav2 + RViz + bridge RoboDK-MQTT + SCADA
ros2 launch mir_nav2_robodk bringup.launch.py
```

### Apagar piezas concretas

```bash
ros2 launch mir_nav2_robodk bringup.launch.py bridge:=false scada:=false rviz:=false
```

### Argumentos principales

| Argumento | Default | Descripción |
|---|---|---|
| `slam` | `True` | SLAM (True) o localización con mapa (False) |
| `world` | `station.sdf` | Mundo de Gazebo |
| `map` | `warehouse.yaml` | Mapa para localización (si `slam:=False`) |
| `rviz` | `true` | Lanzar RViz |
| `bridge` | `true` | Lanzar el bridge RoboDK-MQTT |
| `scada` | `true` | Lanzar la app SCADA |
| `mqtt_host` | `broker.hivemq.com` | Broker MQTT |
| `robodk_host` | `localhost` | Host de la API de RoboDK |

## Otros subsistemas

- **comunicaciones**: ver `comunicaciones/README.md`.
- **database**: ver scripts en `database/` (`bridge.py`, `db_structure.sql`, `queries.sql`).
```

- [ ] **Step 3: Verificar**

Run:
```bash
ls -a | grep -E '^\.gitignore$|^README.md$'
head -5 README.md
```
Expected: aparecen `.gitignore` y `README.md`; el README empieza por el título del monorepo.

---

## Task 8: Verificación de build y del launch file

**Files:** ninguno (verificación)

- [ ] **Step 1: Compilar el paquete con colcon**

Run:
```bash
cd /home/enric_talens/Documents/pol/Proyecto
colcon build --packages-select mir_nav2_robodk
```
Expected: `Finished <<< mir_nav2_robodk` y `Summary: 1 package finished`. (Crea `build/ install/ log/`, ya ignorados.)

- [ ] **Step 2: Comprobar que el launch file es válido (carga sin ejecutar)**

Run:
```bash
source /opt/ros/jazzy/setup.bash
source install/setup.bash
ros2 launch mir_nav2_robodk bringup.launch.py --show-args
```
Expected: lista los argumentos (`slam`, `world`, `map`, `rviz`, `bridge`, `scada`, `mqtt_host`, `robodk_host`) sin trazas de error de import. (No arranca Gazebo con `--show-args`.)

- [ ] **Step 3: Comprobar que el SCADA y los scripts se instalaron**

Run:
```bash
ls install/mir_nav2_robodk/share/mir_nav2_robodk/scada/main.py
ls install/mir_nav2_robodk/lib/mir_nav2_robodk/robodk_bridge.py
```
Expected: ambas rutas existen.

---

## Task 9: Stagear todo para el commit único del usuario

**Files:** ninguno (git staging)

- [ ] **Step 1: Stagear todos los cambios (movimientos, nuevos archivos, ediciones)**

Run:
```bash
cd /home/enric_talens/Documents/pol/Proyecto
git add -A
git status -s | head -40
```
Expected: renames de los archivos del paquete a `navegacion/`, nuevos `comunicaciones/`, `database/`, `README.md`, `.gitignore`, `navegacion/launch/bringup.launch.py`, `docs/...`. NO debe aparecer `build/`, `install/`, `log/` ni `__pycache__/` (los ignora `.gitignore`).

- [ ] **Step 2: Revisar el árbol final**

Run:
```bash
git ls-files | sed 's|/.*||' | sort -u
```
Expected: top-level tracked = `.gitignore README.md comunicaciones database docs navegacion`.

- [ ] **Step 3: NO commitear — dejar para el usuario**

El usuario hará un único commit, por ejemplo:
```bash
git commit -m "Reorganización en monorepo + launcher maestro"
```
(Claude no ejecuta este commit.)

---

## Task 10: Limpieza de ramas remotas en el fork

**Files:** ninguno (refs remotas). **Destructivo, autorizado por el usuario.** Las ramas originales siguen existiendo en `PR2-A1/Proyecto`.

> Ejecutar SOLO tras verificar (Task 8) que el contenido necesario está copiado en el working tree.

- [ ] **Step 1: Confirmar el remoto correcto (el fork)**

Run:
```bash
git remote -v
```
Expected: `origin git@github.com:ETM2097/Proyecto.git`. **Si apuntara a `PR2-A1`, PARAR** (sería el repo original, no el fork).

- [ ] **Step 2: Borrar las ramas remotas obsoletas del fork**

Run:
```bash
git push origin --delete Plantilla objetos robots autonomous_movement comunicaciones database
```
Expected: `- [deleted]` por cada rama. (No se borra `main`: se sobrescribirá con la consolidada tras el commit del usuario.)

- [ ] **Step 3 (post-commit del usuario): publicar `main` consolidada**

Tras el commit del usuario:
```bash
git push origin main --force
```
Expected: `main` del fork actualizada con el monorepo. Verificar que `main` sigue siendo la rama por defecto en GitHub (Settings → Branches); como conservamos el nombre `main`, no hace falta cambiar el default.

---

## Self-Review (cobertura del spec)

- Monorepo por subsistemas → Tasks 2, 4 ✅
- Incluir autonomous_movement/comunicaciones/database → Tasks 2, 4 ✅
- Descartar objetos/robots/Plantilla/main vieja → Task 10 (no se importan; se borran) ✅
- Copia plana sin historial → Task 4 ✅
- SCADA dentro del paquete → Tasks 2, 6 ✅
- Fuente de verdad = mir2 working tree + host localhost → Task 3 ✅
- `bringup.launch.py` (sim+bridge+scada, args) → Task 5 ✅
- Broker hivemq por defecto, host configurable → Task 5 ✅
- colcon encuentra el paquete anidado → Task 8 ✅
- README global + .gitignore → Task 7 ✅
- Sin commits de Claude, un único commit del usuario → Task 9 ✅
- Borrado de ramas en el fork, default sigue main → Task 10 ✅
```

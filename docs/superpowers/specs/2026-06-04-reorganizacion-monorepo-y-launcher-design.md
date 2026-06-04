# Reorganización del repositorio en monorepo + launcher maestro

**Fecha:** 2026-06-04
**Autor:** Enric Talens (con Claude Code)
**Repo:** fork `git@github.com:ETM2097/Proyecto.git` (origen: `PR2-A1/Proyecto`)

## Problema

El fork está desorganizado. Las ramas (`main`, `objetos`, `Plantilla`, `robots`,
`comunicaciones`, `database`, `autonomous_movement`) son subsistemas distintos de
un proyecto de equipo, con estructuras de archivos incompatibles e historiales
**sin ancestro común**. Hace falta:

1. Consolidar el contenido útil en un único monorepo organizado por subsistemas.
2. Descartar las ramas que no aportan.
3. Crear un launcher que arranque todo el sistema de simulación de una vez.

Además, el código ROS2/bridge **más actualizado no está en ninguna rama git**: vive
como cambios sin commitear en el working tree de `~/test_ros2_pro/src/mir2`.

## Objetivos

- Una sola rama `main` con una carpeta por subsistema.
- Importar el contenido útil de `comunicaciones` y `database` a sus carpetas.
- Que `colcon build` siga encontrando y compilando el paquete ROS2.
- Un único `bringup.launch.py` que levante Gazebo + Nav2 + RViz + SCADA + bridge RoboDK-MQTT.
- Limpiar el fork dejando solo `main`.

## No-objetivos (YAGNI)

- No se incluye `objetos`, `robots`, `Plantilla` ni la `main` vieja (obsoletos).
- No se preserva el historial de las ramas importadas (copia plana; ver Decisiones).
- No se reescribe la lógica de `simulation.launch.py`, `robodk_bridge.py` ni del SCADA;
  solo se reubican y se orquestan (única excepción: el default de `robodk_host`, ver abajo).
- No se monta broker MQTT local (mosquitto): se mantiene `broker.hivemq.com` público.
- No se toca el repo original `PR2-A1/Proyecto`, solo el fork `ETM2097/Proyecto`.

## Decisiones tomadas

| Decisión | Elección |
|---|---|
| Estructura final | Monorepo por subsistemas |
| Ramas a incluir | `autonomous_movement` (ROS2), `comunicaciones`, `database` |
| Ramas a descartar | `objetos`, `robots`, `Plantilla`, `main` (vieja) |
| Limpieza remota | Borrarlas del fork; dejar solo `main` consolidada |
| Mecánica de import | **Copia plana** (archivos en su carpeta, sin historial) |
| Ubicación SCADA | **Dentro del paquete ROS2** (`navegacion/scada/`), instalado en `share/` |
| Forma del launcher | Launch file ROS2 maestro (`bringup.launch.py`) |
| Componentes del launcher | Sim (Gazebo+Nav2+RViz) + SCADA + bridge RoboDK-MQTT |
| Broker MQTT | `broker.hivemq.com` público (host configurable por argumento) |
| `robodk_host` default | **`localhost`** (cambiar el default actual, que apunta a IPs de lab) |
| Rama consolidada | `main` (recreada limpia, marcada como default) |
| Commits | **Claude NO commitea**; deja todo en el working tree y el usuario hace **un único commit** |

## Fuente de verdad por subsistema

| Subsistema | Origen | Notas |
|---|---|---|
| `navegacion/` (paquete `mir_nav2_robodk`) | `~/test_ros2_pro/src/mir2` **working tree** | Incluye cambios sin commitear: bridge con yaw por destino + coords TOLVA calibradas; `nav2_params.yaml` con `inflation_radius: 0.55`, `cost_scaling_factor: 3.0`. `simulation.launch.py` es idéntico al del fork. Tras copiar, se ajusta el default de `robodk_host` a `localhost`. |
| `navegacion/scada/` | fork `Proyecto` (commit "added scada") | Único sitio donde existe. App Qt independiente (no-ROS), pero vive dentro del paquete para que el launcher la localice vía `share/`. |
| `comunicaciones/` | rama `origin/comunicaciones` (copia plana) | Contiene `c/` (ESP-IDF/Arduino) y `d/` (documentación). |
| `database/` | rama `origin/database` (copia plana) | `bridge.py` MQTT↔SQL/noSQL, `db_structure.sql`, `queries.sql`, `requirements.txt`, `.env`. Confirmado: el `.env` no tiene secretos reales → se publica tal cual. |

## Layout final del repo

```
Proyecto/                       (rama: main)
├── README.md                   ← nuevo: visión global + cómo lanzar todo
├── .gitignore                  ← consolidado (build/, install/, log/, __pycache__/, .vscode/)
├── docs/
│   └── superpowers/specs/      ← este spec y el plan de implementación
├── navegacion/                 ← paquete ROS2 mir_nav2_robodk
│   ├── package.xml             (name sigue siendo mir_nav2_robodk)
│   ├── CMakeLists.txt          (instala launch/, config/, ..., scripts/ y scada/)
│   ├── config/  (gz_bridge.yaml, nav2_params.yaml [actualizado], slam_toolbox_params.yaml)
│   ├── launch/
│   │   ├── simulation.launch.py    (intacto)
│   │   └── bringup.launch.py       ← NUEVO launcher maestro
│   ├── maps/  meshes/  models/  rviz/  urdf/  worlds/
│   ├── scripts/  (robodk_bridge.py [actualizado, host=localhost], cap_color_*, manual_goal_sender.py, colors.yaml)
│   └── scada/   (main.py, mqtt_client.py, config.py, widgets.py, ...)
├── comunicaciones/             ← c/ + d/  (copia plana)
└── database/                   ← bridge.py + .sql  (copia plana)
```

Notas del layout:
- `colcon build` recursa buscando `package.xml`, así que encuentra el paquete aunque
  esté anidado en `navegacion/`. Las carpetas `comunicaciones/` y `database/` no tienen
  `package.xml` → colcon las ignora.
- El nombre interno del paquete NO cambia (`mir_nav2_robodk`); solo cambia dónde vive la
  carpeta. Por tanto `ros2 launch mir_nav2_robodk bringup.launch.py` sigue siendo válido.
- `CMakeLists.txt` se actualiza para instalar `bringup.launch.py` (ya cubierto por
  `install(DIRECTORY launch ...)`) y la carpeta `scada/` (`install(DIRECTORY scada ...)`).

## Launcher maestro: `navegacion/launch/bringup.launch.py`

Orquesta todo el sistema reutilizando lo existente:

1. **Simulación** — `IncludeLaunchDescription(simulation.launch.py)`: Gazebo Harmonic +
   MiR + ros_gz_bridge + Nav2 (+ RViz). Reexpone sus argumentos: `slam`, `world`, `map`,
   `rviz`, `mir_type`, `spawn_*`, `nav2_params_file`.
2. **Bridge RoboDK-MQTT-ROS2** — `Node` que ejecuta `robodk_bridge.py`. Tolera que RoboDK
   o MQTT no estén disponibles (se conecta si puede). Argumentos expuestos: `mqtt_host`
   (default `broker.hivemq.com`), `robodk_host` (default `localhost`).
3. **SCADA** — `ExecuteProcess` que lanza `python3 <share>/scada/main.py`, resolviendo la
   ruta con `FindPackageShare('mir_nav2_robodk')` (por eso SCADA se instala en `share/`).

### Argumentos del launcher

| Argumento | Default | Efecto |
|---|---|---|
| `slam` | `True` | SLAM vs localización (heredado de la sim) |
| `world` | `station.sdf` | Mundo Gazebo (heredado) |
| `map` | `warehouse.yaml` | Mapa para localización (heredado) |
| `rviz` | `true` | Lanzar RViz (heredado) |
| `bridge` | `true` | Lanzar `robodk_bridge.py` |
| `scada` | `true` | Lanzar la app SCADA |
| `mqtt_host` | `broker.hivemq.com` | Broker MQTT del bridge |
| `robodk_host` | `localhost` | Host de la API de RoboDK |

Por defecto `ros2 launch mir_nav2_robodk bringup.launch.py` arranca **todo**; cada pieza
se puede apagar con su flag (`bridge:=false`, `scada:=false`, `rviz:=false`).

## Plan de consolidación (git)

Trabajamos sobre el fork `ETM2097/Proyecto`. **Claude no hace commits**: deja el working
tree preparado para que el usuario haga un único commit. Pasos de alto nivel (el detalle
fino irá en el plan de implementación):

1. **Crear la rama consolidada** `main` (nueva, limpia) a partir de `autonomous_movement`,
   que ya tiene el paquete + scada del fork.
2. **Reubicar** el paquete ROS2 a `navegacion/` (scada se queda dentro, en `navegacion/scada/`).
3. **Sincronizar navegación** con el working tree de `~/test_ros2_pro/src/mir2`: copiar los
   cambios actualizados de `robodk_bridge.py` y `nav2_params.yaml` (y cualquier otra
   diferencia) a `navegacion/`. Luego ajustar el default de `robodk_host` a `localhost`.
4. **Copiar el contenido** de `origin/comunicaciones` → `comunicaciones/` y de
   `origin/database` → `database/` (extracción plana de archivos, sin merge de historia).
5. **Crear** `bringup.launch.py`, actualizar `CMakeLists.txt`, y crear el `README.md` global
   y el `.gitignore` consolidado.
6. **Verificar** que el árbol coincide con el layout y que `bringup.launch.py` no tiene
   errores de sintaxis (y `colcon build` si hay entorno ROS2 disponible).
7. **Limpiar el remoto**: borrar del fork `Plantilla`, `objetos`, `robots`,
   `autonomous_movement` y la `main` vieja, dejando solo la `main` consolidada como default.
   (El borrado se hace una vez el contenido necesario ya está copiado en el working tree.)
8. **Dejar todo sin commitear** y avisar al usuario para que haga su único commit.

## Verificación / criterios de éxito

- El árbol del repo coincide con el layout objetivo.
- `colcon build --packages-select mir_nav2_robodk` termina sin errores (si hay entorno ROS2).
- `bringup.launch.py` carga sin errores de sintaxis Python / launch.
- En el fork remoto solo queda la rama `main` y es la default.
- El `README.md` global explica la estructura y el comando único para lanzar todo.
- El working tree queda con todos los cambios listos para un único commit del usuario.

## Riesgos y mitigaciones

- **Cambios sin commitear en `mir2`**: son la única copia del código actualizado. Mitigación:
  copiarlos explícitamente a `navegacion/` antes de cualquier operación destructiva.
- **Borrado de ramas remotas**: destructivo, pero autorizado explícitamente por el usuario.
  Mitigación: hacerlo solo tras copiar el contenido necesario; las ramas originales siguen
  existiendo en `PR2-A1/Proyecto`.
- **Sin historial de subsistemas**: decisión consciente (copia plana) a cambio de un repo
  limpio en un único commit.

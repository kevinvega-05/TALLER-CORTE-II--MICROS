# Taller — Simulación robótica con PyBullet y consola de mandos ESP32

**Universidad Militar Nueva Granada (UMNG) — Ingeniería Mecatrónica**

Este repositorio reúne los tres puntos del taller:

| Punto | Tema | Carpeta |
|---|---|---|
| **1** | Simulación de drones con `gym-pybullet-drones` (control y aprendizaje por refuerzo) | [`punto1/`](punto1/) |
| **2** | Consola de mandos ESP32 para un brazo de Baxter: movimiento X/Y/Z y agarre de un objeto | [`punto2/`](punto2/) |
| **3** | Consola de mandos ESP32 para los dos brazos de Baxter con visor 3D ligero | [`punto3/`](punto3/) |

---

## Índice

- [Requisitos generales](#requisitos-generales)
- [Estructura del repositorio](#estructura-del-repositorio)
- [Punto 1 — Simulación de drones (gym-pybullet-drones)](#punto-1--simulación-de-drones-gym-pybullet-drones)
- [Punto 2 — Consola ESP32 → Baxter (un brazo + agarre)](#punto-2--consola-esp32--baxter-un-brazo--agarre)
- [Punto 3 — Consola ESP32 → Baxter (dos brazos, visor 3D ligero)](#punto-3--consola-esp32--baxter-dos-brazos-visor-3d-ligero)
- [Referencias](#referencias)

---

## Requisitos generales

- Python 3 con `pip` (o `conda` para el Punto 1)
- `pybullet`, `numpy`, `pyserial`
- Arduino IDE con el paquete de placas **ESP32** (Puntos 2 y 3)
- Placa ESP32 DevKit (30 o 38 pines), potenciómetros, joysticks y pulsadores

## Estructura del repositorio

```
.
├── README.md
├── punto1/
│   ├── arduino/
│   │   └── punto1.ino
│   └── evidencias/          (fotos, GIF, capturas)
├── punto2/
│   ├── baxter_console_control.py
│   ├── esp32_console/
│   │   └── esp32_console.ino
│   └── evidencias/
└── punto3/
    ├── esp32_baxter_console.ino
    ├── generar_urdf_ligero.py
    ├── baxter_console_control_3d_ligero.py
    ├── baxter_console_control_ligero.py
    ├── baxter_console_control.py
    └── evidencias/
```

---

# Punto 1 — Simulación de drones (gym-pybullet-drones)

Se usa [`gym-pybullet-drones`](https://github.com/learnsyslab/gym-pybullet-drones),
una versión minimalista del entorno original, compatible con
[`gymnasium`](https://github.com/Farama-Foundation/Gymnasium),
[`stable-baselines3` 2.0](https://github.com/DLR-RM/stable-baselines3) y
[`betaflight`](https://github.com/betaflight/betaflight) SITL.

> **Alternativas relacionadas**
> - Dinámica simbólica y restricciones: [`safe-control-gym`](https://github.com/learnsyslab/safe-control-gym)
> - Simulación diferenciable en GPU (JAX): [`crazyflow`](https://github.com/learnsyslab/crazyflow)
> - Despliegue real PX4/ArduPilot + ROS2 + JetPack: [`aerial-autonomy-stack`](https://github.com/JacopoPan/aerial-autonomy-stack)

> **Nota:** el código original del artículo IROS 2021 está disponible con `git checkout [paper|master]`.

### 1.1 Instalación

Probado en Intel x64/Ubuntu 24.04 y Apple Silicon/macOS 26.

```sh
git clone https://github.com/learnsyslab/gym-pybullet-drones.git
cd gym-pybullet-drones/

conda create -n drones python=3.12
conda activate drones

# A partir de Python 3.10, pybullet no tiene wheel precompilado
# En Ubuntu, instalar gcc para que pip pueda compilar pybullet:
sudo apt install build-essential
# En macOS, compilar e instalar pybullet con:
CFLAGS="-Dfdopen=fdopen" pip install pybullet --no-cache-dir

pip3 install -e .
```

Comandos útiles del entorno: `conda list` (ver paquetes),
`conda deactivate` (salir), `conda remove -n drones --all` (eliminar).

### 1.2 Ejemplos de control

```sh
cd gym_pybullet_drones/examples/
python3 pid.py
python3 pid_velocity.py
python3 mrac.py
```

### 1.3 Efecto *downwash*

```sh
cd gym_pybullet_drones/examples/
python3 downwash.py
```

### 1.4 Aprendizaje por refuerzo (PPO de Stable-Baselines3)

```sh
cd gym_pybullet_drones/examples/

# Un agente — tarea: un dron en hover a z = 1.0
python learn.py
LATEST_MODEL=$(ls -t results | head -n 1) && python play.py --model_path "results/${LATEST_MODEL}/best_model.zip"

# Multiagente — tarea: 2 drones en hover a z = 1.2 y z = 0.7
python learn.py --multiagent true
LATEST_MODEL=$(ls -t results | head -n 1) && python play.py --multiagent true --model_path "results/${LATEST_MODEL}/best_model.zip"
```

### 1.5 Pruebas

```sh
# desde la carpeta raíz del repo
cd gym-pybullet-drones/
pytest tests/
```

### 1.6 Betaflight SITL (solo Ubuntu)

```sh
# Configuración única: compilar un ejecutable SITL por dron (ej. 2); si hace falta, `apt install curl`
cd gym-pybullet-drones/
./gym_pybullet_drones/assets/clone_bfs.sh 2

# Ejecutar el ejemplo
cd gym_pybullet_drones/examples/
python3 beta.py --num_drones 2
# --num_drones debe ser <= al número usado en clone_bfs.sh
```

### 1.7 Solución de problemas

- En Ubuntu con tarjeta NVIDIA, si aparece *"Failed to create an OpenGL context"*:
  abrir `nvidia-settings`, en **PRIME Profiles** elegir
  **NVIDIA (Performance Mode)**, reiniciar y volver a intentar.

### 1.8 Código Arduino IDE

<!-- Pega tu código entre las líneas ```cpp y ``` y guarda el archivo .ino en punto1/arduino/ -->

📄 Archivo: [`punto1/arduino/punto1.ino`](punto1/arduino/punto1.ino)

```cpp
// ===== Punto 1 — Código Arduino IDE =====
// Pega aquí tu código

void setup() {

}

void loop() {

}
```

**Explicación del código:**

- _Describe aquí qué hace cada parte del código._

### 1.9 Evidencias del funcionamiento

<!-- Guarda tus fotos/GIF en punto1/evidencias/ y reemplaza los nombres de archivo -->

| Montaje | Funcionamiento |
|---|---|
| ![Montaje punto 1](punto1/evidencias/montaje.jpg) | ![Funcionamiento punto 1](punto1/evidencias/funcionamiento.gif) |

🎥 Video: [Ver demostración del Punto 1](PEGA_AQUI_EL_ENLACE_DEL_VIDEO)

### 1.10 Cita

```bibtex
@INPROCEEDINGS{panerati2021learning,
      title={Learning to Fly---a Gym Environment with PyBullet Physics for Reinforcement Learning of Multi-agent Quadcopter Control},
      author={Jacopo Panerati and Hehui Zheng and SiQi Zhou and James Xu and Amanda Prorok and Angela P. Schoellig},
      booktitle={2021 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS)},
      year={2021},
      pages={7512-7519},
      doi={10.1109/IROS51168.2021.9635857}
}
```

---

# Punto 2 — Consola ESP32 → Baxter (un brazo + agarre)

Controla el brazo del robot **Baxter** en simulación (PyBullet) con una
consola física basada en **ESP32**: movimiento fluido del efector final
en X/Y/Z y control de la pinza para coger y mover un objeto.

Se probó de extremo a extremo en modo *headless* (sin ventana gráfica)
contra el URDF real del repositorio
[`erwincoumans/pybullet_robots`](https://github.com/erwincoumans/pybullet_robots):
carga de Baxter, cinemática inversa, apertura/cierre de pinza, agarre y
traslado de un cubo.

### 2.0 ⚠️ Dónde debe vivir el script

El URDF de Baxter se carga con una ruta **relativa** que solo existe
dentro de la carpeta `data/` del repositorio (ahí también está
`plane.urdf`). El script debe copiarse y ejecutarse **dentro** de esa carpeta:

```bash
git clone https://github.com/erwincoumans/pybullet_robots.git
cp punto2/baxter_console_control.py pybullet_robots/data/
cd pybullet_robots/data
python baxter_console_control.py
```

Si se ejecuta desde la raíz del repo, PyBullet no encuentra el URDF.

### 2.1 Esquema de conexión (ESP32)

```
                     ESP32 DevKit
                 ┌───────────────────┐
   Pot X (10kΩ)  │                   │
   extremo A ────┤ 3V3               │
   cursor  ──────┤ GPIO34 (ADC1_CH6) │
   extremo B ────┤ GND               │
                 │                   │
   Pot Y (10kΩ)  │                   │
   extremo A ────┤ 3V3               │
   cursor  ──────┤ GPIO35 (ADC1_CH7) │
   extremo B ────┤ GND               │
                 │                   │
   Pot Z (10kΩ)  │                   │
   extremo A ────┤ 3V3               │
   cursor  ──────┤ GPIO32 (ADC1_CH4) │
   extremo B ────┤ GND               │
                 │                   │
   Pulsador ─────┤ GPIO25            │
   (otro pin) ───┤ GND               │
                 │                   │
                 │ USB ──────────────┼── al PC (alimentación + serial)
                 └───────────────────┘
```

- Los 3 potenciómetros funcionan como divisores de tensión: extremos a
  3V3 y GND, cursor (pin central) a la entrada ADC.
- El pulsador usa `INPUT_PULLUP`: solo GPIO25 y GND, sin resistencia
  externa. Presionado = pinza **cerrada**.
- Solo se usan pines **ADC1** (32 a 39): con Wi-Fi activo, ADC2 no es
  confiable. Este firmware no usa Wi-Fi.
- Sirve igual en ESP32 de 30 o 38 pines (GPIO 25, 32, 34 y 35 existen en ambas).

Con un **joystick analógico XY** de 2 ejes, se puede usar para X/Y y
dejar un potenciómetro solo para Z.

### 2.2 Firmware del ESP32

Archivo: `punto2/esp32_console/esp32_console.ino`

1. Abrirlo en Arduino IDE (con el paquete de placas ESP32) o en PlatformIO.
2. Seleccionar la placa ESP32 y el puerto.
3. Subir el código.
4. Abrir el Monitor Serial a **115200 baudios** y verificar líneas como
   `0.2000,0.0500,-0.1000,1`.
5. **Cerrar el Monitor Serial** antes de correr Python (el puerto solo
   puede estar abierto por un programa a la vez).

### 2.3 Simulación en PyBullet

Archivo: `punto2/baxter_console_control.py`

```bash
pip install pybullet numpy pyserial
```

Ubicar el script como se indica en el paso 2.0 y ajustar al inicio del archivo:

```python
SERIAL_PORT = "COM5"   # Windows: ver Administrador de dispositivos
                        # Linux: normalmente "/dev/ttyUSB0" o "/dev/ttyACM0"
BAUD_RATE = 115200
USE_SERIAL = False      # False para la primera prueba (ver 2.4)
```

```bash
python baxter_console_control.py
```

### 2.4 Primera prueba sin la ESP32 (modo demo con sliders)

Con `USE_SERIAL = False` se abre PyBullet con Baxter, una plataforma de
apoyo y un cubo (`cube_small.urdf`), más 4 sliders (`targetPosX/Y/Z` y
`gripper`). Así se prueban el movimiento y el agarre **antes** de
conectar el hardware, separando errores de código de errores de cableado.

Cuando funcione, cambiar `USE_SERIAL = True` y poner el puerto real en `SERIAL_PORT`.

### 2.5 Optimizado para equipos modestos

- Simula a **60 Hz** en vez de 240 Hz (`SIM_HZ`).
- Menos iteraciones del solver de físicas (`SOLVER_ITERATIONS`).
- Sombras desactivadas.
- IK con el solver **nativo** de PyBullet (`maxNumIterations=5`), no con
  un bucle manual en Python.
- Listas de articulaciones precalculadas una sola vez.
- En modo headless, cada paso tarda ~0.3–0.5 ms: el cuello de botella
  real es el renderizado, no la física ni la IK.

Si aún se siente pesado:

```python
SIM_HZ = 30              # menos pasos de física por segundo
IK_EVERY_N_STEPS = 2      # recalcula la IK cada 2 pasos
```

### 2.6 Cómo se logra el movimiento fluido

- **No** se usa `resetJointState` en bucle (como el demo original),
  porque teletransporta las articulaciones.
- Se calcula la IK y se aplica `setJointMotorControl2` en modo
  `POSITION_CONTROL`: el motor virtual se mueve suavemente al objetivo.
- El objetivo se filtra con un promedio móvil exponencial
  (`SMOOTHING_ALPHA`) tanto en la ESP32 como en Python.

### 2.7 Cómo funciona el agarre

El plano de suelo del demo original está ~90 cm por debajo del punto de
trabajo del brazo, así que el script crea una **plataforma de apoyo**
(caja fija sin masa) cuya cara superior coincide con la base del cubo.

Al cerrar la pinza a menos de `GRASP_DISTANCE` (6 cm) del objeto, se crea
una restricción rígida (`p.createConstraint`) que lo une al efector
final — el método estándar en PyBullet para simular agarre sin depender
de la fricción de los dedos. Al abrir la pinza, la restricción se
elimina y el objeto queda sujeto a la gravedad (si se suelta en el aire, cae).

### 2.8 Archivos del punto 2

- `baxter_console_control.py` — simulación PyBullet + lectura serial + IK + agarre
- `esp32_console/esp32_console.ino` — firmware de la consola física

### 2.9 Código Arduino IDE

<!-- Pega tu código entre las líneas ```cpp y ``` -->

📄 Archivo: [`punto2/esp32_console/esp32_console.ino`](punto2/esp32_console/esp32_console.ino)

```cpp
// ===== Punto 2 — Consola ESP32 (un brazo + agarre) =====
// Pega aquí tu código

void setup() {

}

void loop() {

}
```

**Explicación del código:**

- _Lectura de los potenciómetros (GPIO34, 35, 32):_
- _Lectura del pulsador de la pinza (GPIO25):_
- _Filtrado y envío por serial a 115200 baudios:_

### 2.10 Evidencias del funcionamiento

<!-- Guarda tus fotos/GIF en punto2/evidencias/ y reemplaza los nombres de archivo -->

| Consola física | Baxter en PyBullet |
|---|---|
| ![Consola punto 2](punto2/evidencias/consola.jpg) | ![Simulación punto 2](punto2/evidencias/simulacion.gif) |

| Monitor Serial | Agarre del cubo |
|---|---|
| ![Monitor serial punto 2](punto2/evidencias/monitor_serial.png) | ![Agarre punto 2](punto2/evidencias/agarre.gif) |

🎥 Video: [Ver demostración del Punto 2](PEGA_AQUI_EL_ENLACE_DEL_VIDEO)

---

# Punto 3 — Consola ESP32 → Baxter (dos brazos, visor 3D ligero)

> Actividad c: *"Desarrollar una consola de mandos con la ESP32 para un
> movimiento fluido del robot Baxter que permita una movilidad real."*

Usa el repo de apoyo
[`erwincoumans/pybullet_robots`](https://github.com/erwincoumans/pybullet_robots)
(carpeta `baxter_common/` y el ejemplo `baxter_ik_demo.py`) como base del
modelo y de la cinemática inversa real de Baxter.

### 3.0 El problema y la solución

El URDF original de Baxter trae **28 mallas 3D detalladas** (~60 MB en
archivos .DAE/.STL). Dibujarlas con OpenGL en una GPU integrada satura
el equipo.

Solución: generar una copia del **mismo** robot —mismas articulaciones,
límites y cinemática— reemplazando las mallas por primitivas
(cilindros, cajas, esferas). Se comprobó que:

| Métrica | URDF original | URDF ligero |
|---|---|---|
| Error de posición con IK | 0.00089 m | 0.00089 m (idéntico) |
| Tiempo de carga | 3.25 s | 0.017 s (~190× más rápido) |
| Articulaciones (número, nombres, índices) | — | Exactamente iguales |

El robot sigue siendo 3D, rotable con el mouse y con el mismo
movimiento; solo que la GPU dibuja formas simples.

### 3.1 Archivos

| Archivo | Descripción |
|---|---|
| `esp32_baxter_console.ino` | Firmware ESP32 |
| `generar_urdf_ligero.py` | Se ejecuta **una sola vez**: crea el Baxter con formas simples |
| `baxter_console_control_3d_ligero.py` | **Recomendado.** Visor 3D real, pero ligero |
| `baxter_console_control_ligero.py` | Alternativa sin ventana 3D (panel 2D con matplotlib) |
| `baxter_console_control.py` | Visor 3D con las mallas originales (requiere GPU dedicada) |

### 3.2 Hardware de la consola

| Elemento | Pin ESP32 | Función |
|---|---|---|
| Joystick derecho — eje X | GPIO34 | velocidad X brazo derecho |
| Joystick derecho — eje Y | GPIO35 | velocidad Y brazo derecho |
| Potenciómetro altura derecha | GPIO32 | altura (Z) absoluta brazo derecho |
| Joystick izquierdo — eje X | GPIO33 | velocidad X brazo izquierdo |
| Joystick izquierdo — eje Y | GPIO36 | velocidad Y brazo izquierdo |
| Potenciómetro altura izquierda | GPIO39 | altura (Z) absoluta brazo izquierdo |
| Potenciómetro cabeza | GPIO25 | giro `head_pan` |
| Pulsador home derecho | GPIO18 | regresa el brazo derecho a home |
| Pulsador pinza derecha | GPIO19 | abre/cierra pinza derecha |
| Pulsador home izquierdo | GPIO21 | regresa el brazo izquierdo a home |
| Pulsador pinza izquierda | GPIO22 | abre/cierra pinza izquierda |
| Pulsador parada de emergencia | GPIO23 | congela todo el movimiento mientras se mantiene |

- Joysticks y potenciómetros: alimentar con **3V3** (no 5V) y GND; el cursor al pin analógico.
- Pulsadores: `INPUT_PULLUP`, un terminal al pin y el otro a GND.
- Compatible con la DOIT ESP32 DevKit V1 de 30 pines (CH9102F); se evitan
  los pines de arranque (0, 2, 5, 12, 15).

### 3.3 Paso a paso

#### Paso 1 — Cargar el firmware

1. Abrir `punto3/esp32_baxter_console.ino` en Arduino IDE. Si no está el
   paquete ESP32: `Archivo > Preferencias > URLs adicionales` →
   `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`,
   luego `Herramientas > Placa > Gestor de placas` → instalar "esp32".
2. En `Herramientas > Placa`, elegir **"ESP32 Dev Module"**.
3. Conectar la ESP32 por USB y elegir el puerto en `Herramientas > Puerto`.
4. Subir el sketch (`Ctrl+U`).
5. Abrir el Monitor Serial (`Ctrl+Mayús+M`) a **115200 baudios**. Deben
   verse líneas como `0.000,0.000,0.502,0.000,0.000,0.498,0.501,0,0,0,0,0`.
6. **Cerrar el Monitor Serial** antes del siguiente paso.

#### Paso 2 — Instalar Python y librerías

Instalar Python desde [python.org](https://www.python.org/downloads/)
marcando **"Add Python to PATH"**. Luego:

```bash
pip install pybullet pyserial numpy
```

#### Paso 3 — Descargar el modelo de Baxter

```bash
git clone https://github.com/erwincoumans/pybullet_robots.git
cd pybullet_robots
```

Copiar `baxter_console_control_3d_ligero.py` dentro de `pybullet_robots/`.

#### Paso 4 — Generar el Baxter ligero (una sola vez)

Copiar `generar_urdf_ligero.py` en
`pybullet_robots/data/baxter_common/baxter_description/urdf/` y ejecutarlo ahí:

```bash
cd data/baxter_common/baxter_description/urdf
python generar_urdf_ligero.py
```

Salida esperada:

```
Listo: se generó 'toms_baxter_ligero.urdf'
  Links totales:               57
  Visuales simplificados:      28 (de mallas 3D a figuras simples)
  ...
```

Volver a la raíz del repo:

```bash
cd ../../../..
```

#### Paso 5 — Probar sin la ESP32 (sliders)

```bash
python baxter_console_control_3d_ligero.py --test
```

Se abre la ventana 3D con Baxter ligero y sliders que simulan la consola:

- `R vx`, `R vy`, `R potZ` → brazo derecho
- `L vx`, `L vy`, `L potZ` → brazo izquierdo
- `head` → giro de cabeza
- `Grip R`, `Grip L`, `Home R`, `Home L`, `ESTOP` → subir el slider por
  encima de la mitad equivale a "presionar" el botón

Cámara: **click izquierdo + arrastrar** rota, **rueda** hace zoom,
**click derecho + arrastrar** desplaza.

#### Paso 6 — Ejecutar con la ESP32

1. Ver en el Administrador de dispositivos el puerto COM de la ESP32 (ej. `COM5`).
2. Ejecutar:

```bash
python baxter_console_control_3d_ligero.py --port COM5
```

(en Linux/Mac: `--port /dev/ttyUSB0`). Baxter debe responder igual que
con los sliders, ahora con el hardware real.

### 3.4 Cómo se logra la movilidad fluida

1. **Filtrado en la ESP32** (media móvil exponencial + zona muerta): elimina
   el ruido del ADC y da un centro estable al joystick.
2. **Control de velocidad en joysticks:** la deflexión se integra como
   velocidad del efector (`target += v · dt`); al soltar, se detiene. Los
   potenciómetros fijan una **posición absoluta** de altura.
3. **Cinemática inversa por brazo** (`calculateInverseKinematics`, como
   `baxter_ik_demo.py`) sobre `right_endpoint` / `left_endpoint`.
4. **`POSITION_CONTROL` con `maxVelocity` limitado:** cada articulación va
   suavemente al ángulo objetivo, como un motor con control PD.
5. **Visor 3D aligerado:** primitivas, sin sombras y con resolución de
   ventana reducida.

### 3.5 Uso con GPU dedicada

`baxter_console_control.py` usa el URDF original completo. La trama de
la ESP32 es idéntica, así que se puede cambiar de script sin tocar el
firmware ni el cableado.

### 3.6 Parámetros a ajustar

`HOME_POS_R/L` y `WORKSPACE_*` son un punto de partida. Conviene probarlos
en modo `--test` y ajustarlos hasta cubrir el espacio de trabajo deseado.

### 3.7 Solución de problemas

| Síntoma | Causa probable / solución |
|---|---|
| `No se encontró 'toms_baxter_ligero.urdf'` | No se ejecutó el Paso 4, o se corrió en la carpeta equivocada |
| El puerto serie no abre / "Access is denied" | Cerrar el Monitor Serial de Arduino IDE u otro programa que use el puerto |
| La ventana 3D sigue lenta | Verificar que se ejecuta `_3d_ligero.py` y que la consola indica que cargó `toms_baxter_ligero.urdf` |
| El brazo tiembla o no llega al punto | Normal cerca del límite del espacio de trabajo; ajustar `WORKSPACE_*` o `HOME_POS_*` |

### 3.8 Posibles extensiones

- Incluir en el informe la comparación de tiempos de carga (original vs. ligero).
- Registrar en `.csv` las trayectorias del efector para analizar velocidad y aceleración.
- Hacer la consola inalámbrica con una segunda ESP32 o Bluetooth/ESP-NOW.

### 3.9 Código Arduino IDE

<!-- Pega tu código entre las líneas ```cpp y ``` -->

📄 Archivo: [`punto3/esp32_baxter_console.ino`](punto3/esp32_baxter_console.ino)

```cpp
// ===== Punto 3 — Consola ESP32 (dos brazos, visor 3D ligero) =====
// Pega aquí tu código

void setup() {

}

void loop() {

}
```

**Explicación del código:**

- _Lectura de joysticks y potenciómetros (GPIO34, 35, 32, 33, 36, 39, 25):_
- _Lectura de pulsadores home, pinzas y emergencia (GPIO18, 19, 21, 22, 23):_
- _Filtro EMA + zona muerta:_
- _Trama serial enviada al PC (12 valores separados por comas):_

### 3.10 Evidencias del funcionamiento

<!-- Guarda tus fotos/GIF en punto3/evidencias/ y reemplaza los nombres de archivo -->

| Consola física | Baxter ligero en PyBullet |
|---|---|
| ![Consola punto 3](punto3/evidencias/consola.jpg) | ![Simulación punto 3](punto3/evidencias/simulacion.gif) |

| Monitor Serial | Movimiento de ambos brazos |
|---|---|
| ![Monitor serial punto 3](punto3/evidencias/monitor_serial.png) | ![Dos brazos punto 3](punto3/evidencias/dos_brazos.gif) |

🎥 Video: [Ver demostración del Punto 3](PEGA_AQUI_EL_ENLACE_DEL_VIDEO)

---

## Referencias

- J. Panerati et al. (2021). *Learning to Fly — a Gym Environment with PyBullet Physics for Reinforcement Learning of Multi-agent Quadcopter Control*. IROS 2021. [arXiv](https://arxiv.org/abs/2103.02142)
- E. Coumans y Y. Bai (2023). [*PyBullet Quickstart Guide*](https://docs.google.com/document/d/10sXEhzFRSnvFcl3XxNGhnD4N2SedqwdAvK3dsihxVUA/edit)
- E. Coumans. [`pybullet_robots`](https://github.com/erwincoumans/pybullet_robots) — modelo URDF de Baxter y `baxter_ik_demo.py`
- C. Luis y J. Le Ny (2016). [*Design of a Trajectory Tracking Controller for a Nanoquadcopter*](https://arxiv.org/pdf/1608.05786.pdf)
- N. Michael, D. Mellinger, Q. Lindsey, V. Kumar (2010). [*The GRASP Multiple Micro-UAV Testbed*](https://ieeexplore.ieee.org/document/5569026)
- B. Landry (2014). [*Planning and Control for Quadrotor Flight through Cluttered Environments*](http://groups.csail.mit.edu/robotics-center/public_papers/Landry15)
- J. Forster (2015). [*System Identification of the Crazyflie 2.0 Nano Quadrocopter*](https://www.research-collection.ethz.ch/handle/20.500.11850/214143)
- A. Raffin et al. (2019). [*Stable Baselines3*](https://github.com/DLR-RM/stable-baselines3)
- G. Shi et al. (2019). [*Neural Lander: Stable Drone Landing Control Using Learned Dynamics*](https://arxiv.org/pdf/1811.08027.pdf)
- C. K. Liu y D. Negrut (2020). [*The Role of Physics-Based Simulators in Robotics*](https://www.annualreviews.org/doi/pdf/10.1146/annurev-control-072220-093055)
- Y. Song et al. (2020). [*Flightmare: A Flexible Quadrotor Simulator*](https://arxiv.org/pdf/2009.00563.pdf)

---

> Créditos del entorno de drones: UTIAS / [Learning Systems and Robotics Lab](https://github.com/learnsyslab) / [Vector Institute](https://github.com/VectorInstitute) / [Prorok Lab](https://github.com/proroklab), University of Cambridge.

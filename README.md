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

### 1.8 Codigo de Visual (Python)

 ```cpp
 import serial
import numpy as np
# Importaciones hipotéticas basadas en gym-pybullet-drones
# from envs.HoverAviary import HoverAviary
# from control.DSLPIDControl import DSLPIDControl

def iniciar_simulacion():
    # 1. Conectar con la ESP32 (ajusta el puerto 'COM3' o '/dev/ttyUSB0' según tu OS)
    try:
        esp32 = serial.Serial('COM3', 115200, timeout=0.1)
    except:
        print("Error: No se pudo conectar a la ESP32")
        return

    # 2. Inicializar entorno de PyBullet y controladores
    # env = HoverAviary(...)
    # ctrl = DSLPIDControl(...)
    
    # Coordenada objetivo inicial (Lugar A)
    target_pos = np.array([0.0, 0.0, 1.0]) 

    print("Simulación iniciada. Esperando comandos de la ESP32...")

    # Bucle de simulación
    # for i in range(duracion_simulacion):
        # 3. Leer datos de la ESP32 (si los hay)
        if esp32.in_waiting > 0:
            datos_recibidos = esp32.readline().decode('utf-8').strip()
            if datos_recibidos:
                # Se espera un formato como "x,y,z"
                try:
                    x, y, z = map(float, datos_recibidos.split(','))
                    target_pos = np.array([x, y, z])
                    print(f"Nuevo objetivo (ESP32): {target_pos}")
                except ValueError:
                    pass # Ignorar tramas corruptas

        # 4. Calcular acción de control para ir al target_pos y avanzar simulación
        # action = ctrl.computeControlFromState(..., target_pos=target_pos)
        # obs, reward, done, info = env.step(action)
        
        # env.render()
 
 ```

### 1.9 Código Arduino IDE

<!-- Pega tu código entre las líneas ```cpp y ``` y guarda el archivo .ino en punto1/arduino/ -->

📄 Archivo: [`punto1/arduino/punto1.ino`](punto1/arduino/punto1.ino)

```cpp

// Definición de los estados o lugares
int lugar_actual = 0; // 0 = Lugar A, 1 = Lugar B, 2 = Lugar C

// Tiempos para controlar el cambio de lugar sin usar delay() que bloquee el procesador
unsigned long tiempo_anterior = 0;
const long intervalo = 10000; // Tiempo en milisegundos (10 segundos por lugar)

void setup() {
  // Iniciar la comunicación Serial. 
  // IMPORTANTE: Debe coincidir exactamente con el BAUD_RATE de Python (115200)
  Serial.begin(115200); 
  
  // Pequeña pausa para estabilizar la conexión serial
  delay(1000); 
}

void loop() {
  unsigned long tiempo_actual = millis();

  // Comprobar si han pasado 10 segundos para cambiar de objetivo
  if (tiempo_actual - tiempo_anterior >= intervalo) {
    tiempo_anterior = tiempo_actual;

    // Enviar la coordenada correspondiente al lugar actual
    // El formato debe ser estrictamente "X,Y,Z" para que Python lo lea bien
    switch (lugar_actual) {
      case 0:
        Serial.println("0.0,0.0,1.0");  // Coordenadas Lugar A
        break;
      case 1:
        Serial.println("1.5,1.5,1.5");  // Coordenadas Lugar B
        break;
      case 2:
        Serial.println("-1.5,1.5,1.0"); // Coordenadas Lugar C
        break;
    }

    // Avanzar al siguiente lugar
    lugar_actual++;
    
    // Si ya pasamos por los 3 lugares, reiniciar el ciclo al Lugar A
    if (lugar_actual > 2) {
      lugar_actual = 0; 
    }
  }
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

### 2.9 Código Visual (Python)
```cpp
import time
import numpy as np
import pybullet as p
import pybullet_data

try:
    import serial
except ImportError:
    serial = None
    print("[AVISO] pyserial no está instalado (pip install pyserial). "
          "El script funcionará en modo DEMO con sliders en vez de la ESP32.")

# ---------------------------------------------------------------------------
# CONFIGURACIÓN — AJUSTAR SEGÚN TU EQUIPO
# ---------------------------------------------------------------------------
SERIAL_PORT = "COM5"        # Windows: "COM5", "COM7"...  Linux/Mac: "/dev/ttyUSB0"
BAUD_RATE = 115200          # Debe coincidir con el firmware del ESP32
USE_SERIAL = False          # Empieza en False para probar con sliders. Cambia a True con la ESP32 conectada.

SMOOTHING_ALPHA = 0.08      # 0 < alpha <= 1. Más bajo = movimiento más suave/lento
GRASP_DISTANCE = 0.06       # metros. Distancia máx. para considerar "agarrado"
OBJECT_START_POS = [0.2, 0.0, -0.1]   # mismo punto TARGET del demo original
OBJECT_HALF_EXTENT = 0.025  # mitad del lado de cube_small.urdf (cubo de 5 cm)

# El demo original de Erwin Coumans NO tiene piso a la altura del brazo: el
# plano (suelo) está a z=-1, muy por debajo del punto TARGET (z=-0.1). Si se
# soltara el cubo ahí sin apoyo, caería ~90 cm antes de que el brazo lo
# alcance. Por eso se agrega una plataforma pequeña justo debajo del cubo.
PLATFORM_HALF_EXTENTS = [0.15, 0.15, 0.02]  # 30x30x4 cm

# --- Ajustes de rendimiento (bajar estos valores si el PC se siente lento) ---
SIM_HZ = 60                 # pasos de física por segundo (demo original usa 240)
SOLVER_ITERATIONS = 8       # iteraciones del solver de contacto (default de PyBullet: 50)
IK_EVERY_N_STEPS = 1        # recalcular IK cada N pasos de simulación (sube a 2-3 si hace falta)

# ---------------------------------------------------------------------------


def setUpWorld(initialSimSteps=100):
    """Carga el plano y a Baxter, igual que el demo original, pero con
    ajustes de físicas más livianos para no sobrecargar equipos modestos."""
    p.resetSimulation()
    p.setAdditionalSearchPath(pybullet_data.getDataPath())
    p.setPhysicsEngineParameter(fixedTimeStep=1.0 / SIM_HZ, numSolverIterations=SOLVER_ITERATIONS)

    p.loadURDF("plane.urdf", [0, 0, -1], useFixedBase=True)

    p.configureDebugVisualizer(p.COV_ENABLE_RENDERING, 0)
    p.configureDebugVisualizer(p.COV_ENABLE_SHADOWS, 0)  # las sombras son lo más pesado de renderizar
    baxterId = p.loadURDF(
        "baxter_common/baxter_description/urdf/toms_baxter.urdf",
        useFixedBase=True,
    )
    p.resetBasePositionAndOrientation(
        baxterId, [0.5, -0.8, 0.0], [0., 0., -1., -1.]
    )
    p.configureDebugVisualizer(p.COV_ENABLE_RENDERING, 1)

    endEffectorId = 48  # gripper izquierda (mismo índice que el demo original)
    p.setGravity(0., 0., -10.)

    for _ in range(initialSimSteps):
        p.stepSimulation()

    return baxterId, endEffectorId


def getJointRanges(bodyId, includeFixed=False):
    """
    Igual que en el demo original, pero además devuelve la lista de
    articulaciones móviles y su qIndex, precalculada UNA sola vez para
    no volver a llamar getJointInfo dentro del bucle principal.
    """
    lowerLimits, upperLimits, jointRanges, restPoses = [], [], [], []
    movableJoints = []   # [(jointIndex, qIndex), ...]
    numJoints = p.getNumJoints(bodyId)

    for i in range(numJoints):
        jointInfo = p.getJointInfo(bodyId, i)
        if includeFixed or jointInfo[3] > -1:
            rp = p.getJointState(bodyId, i)[0]
            lowerLimits.append(-2)
            upperLimits.append(2)
            jointRanges.append(2)
            restPoses.append(rp)
            if jointInfo[3] > -1:
                movableJoints.append((i, jointInfo[3]))

    return lowerLimits, upperLimits, jointRanges, restPoses, movableJoints


def findGripperFingerJoints(bodyId, endEffectorId):
    """
    Busca automáticamente las articulaciones de los dedos de la pinza
    que corresponde al mismo brazo que endEffectorId, para poder
    abrir/cerrar la pinza sin depender de índices fijos.

    En el URDF de Baxter el link del efector final se llama
    "left_gripper" / "right_gripper", pero las articulaciones de los
    dedos usan la abreviatura corta: "l_gripper_l_finger_joint",
    "l_gripper_r_finger_joint" (y su equivalente "r_..." del otro
    brazo). Por eso se deriva la abreviatura de lado (l/r) en vez de
    comparar el nombre completo.
    """
    eeInfo = p.getJointInfo(bodyId, endEffectorId)
    eeLinkName = eeInfo[12].decode("utf-8")  # child link name, p.ej "left_gripper"
    side = "l" if eeLinkName.startswith("left") else "r"
    prefix = side + "_gripper"  # p.ej "l_gripper"

    fingerJoints = []
    for i in range(p.getNumJoints(bodyId)):
        info = p.getJointInfo(bodyId, i)
        jointName = info[1].decode("utf-8")
        if jointName.startswith(prefix) and "finger" in jointName and info[3] > -1:
            lower, upper = info[8], info[9]
            fingerJoints.append({"index": i, "lower": lower, "upper": upper})

    return fingerJoints


def setGripper(bodyId, fingerJoints, openFraction):
    """
    openFraction: 0.0 = totalmente cerrada, 1.0 = totalmente abierta.
    Mueve ambos dedos con control de posición (movimiento suave).
    """
    openFraction = max(0.0, min(1.0, openFraction))
    for f in fingerJoints:
        target = f["lower"] + openFraction * (f["upper"] - f["lower"])
        p.setJointMotorControl2(
            bodyIndex=bodyId,
            jointIndex=f["index"],
            controlMode=p.POSITION_CONTROL,
            targetPosition=target,
            force=20,
        )


def calculateAndApplyIK(bodyId, endEffectorId, targetPosition,
                         lowerLimits, upperLimits, jointRanges, restPoses,
                         movableJoints):
    """
    Calcula la IK con el solver NATIVO de PyBullet en una sola llamada
    (maxNumIterations=5 en vez de un bucle manual en Python que reevalúa
    la IK decenas o cientos de veces por frame — eso es lo que suele
    congelar la simulación) y aplica control de POSICIÓN sobre la lista
    de articulaciones ya precalculada.
    """
    jointPoses = p.calculateInverseKinematics(
        bodyId, endEffectorId, targetPosition,
        lowerLimits=lowerLimits, upperLimits=upperLimits,
        jointRanges=jointRanges, restPoses=restPoses,
        maxNumIterations=5, residualThreshold=1e-3,
    )

    for jointIndex, qIndex in movableJoints:
        p.setJointMotorControl2(
            bodyIndex=bodyId,
            jointIndex=jointIndex,
            controlMode=p.POSITION_CONTROL,
            targetPosition=jointPoses[qIndex - 7],
            force=200,
            positionGain=0.3,
            velocityGain=1.0,
        )


def createSupportPlatform(xy, topZ, halfExtents=PLATFORM_HALF_EXTENTS):
    """
    Crea una plataforma estática (caja fija, sin masa) cuya cara
    superior queda exactamente en topZ, para que el objeto a coger
    tenga dónde apoyarse en vez de caer hasta el plano del suelo.
    """
    centerZ = topZ - halfExtents[2]
    colId = p.createCollisionShape(p.GEOM_BOX, halfExtents=halfExtents)
    visId = p.createVisualShape(p.GEOM_BOX, halfExtents=halfExtents, rgbaColor=[0.55, 0.4, 0.25, 1])
    return p.createMultiBody(
        baseMass=0,
        baseCollisionShapeIndex=colId,
        baseVisualShapeIndex=visId,
        basePosition=[xy[0], xy[1], centerZ],
    )


class ConsoleReader:
    """
    Lee líneas de la ESP32 con formato:  X,Y,Z,G\\n
        X, Y, Z -> floats, posición objetivo del efector final (metros)
        G       -> 0 (pinza cerrada) o 1 (pinza abierta)

    Se queda solo con la ÚLTIMA línea disponible en el buffer para
    evitar que se acumule retraso (lag) entre la consola y el robot.
    """

    def __init__(self, port, baud):
        self.ser = None
        self.last_target = [0.2, 0.0, -0.1]
        self.last_gripper_open = True
        if serial is None:
            return
        try:
            self.ser = serial.Serial(port, baud, timeout=0)
            time.sleep(2)  # esperar reset del ESP32 al abrir el puerto
            print(f"[OK] Conectado a la ESP32 en {port} @ {baud} baudios")
        except Exception as e:
            print(f"[ERROR] No se pudo abrir {port}: {e}")
            self.ser = None

    def read_latest(self):
        if self.ser is None:
            return self.last_target, self.last_gripper_open

        newest_line = None
        try:
            while self.ser.in_waiting:
                line = self.ser.readline().decode("utf-8", errors="ignore").strip()
                if line:
                    newest_line = line
        except Exception as e:
            print(f"[ERROR serial] {e}")
            return self.last_target, self.last_gripper_open

        if newest_line:
            try:
                x, y, z, g = newest_line.split(",")
                self.last_target = [float(x), float(y), float(z)]
                self.last_gripper_open = int(g) == 1
            except ValueError:
                pass  # línea corrupta / incompleta -> se ignora

        return self.last_target, self.last_gripper_open


def main():
    guiClient = p.connect(p.GUI)
    p.configureDebugVisualizer(p.COV_ENABLE_GUI, 1)
    p.resetDebugVisualizerCamera(2., 180, 0., [0.52, 0.2, np.pi / 4.])

    baxterId, endEffectorId = setUpWorld()
    lowerLimits, upperLimits, jointRanges, restPoses, movableJoints = getJointRanges(baxterId, includeFixed=False)
    fingerJoints = findGripperFingerJoints(baxterId, endEffectorId)

    # Plataforma de apoyo + objeto que el robot debe poder coger y mover
    platformTopZ = OBJECT_START_POS[2] - OBJECT_HALF_EXTENT - 0.002
    createSupportPlatform(OBJECT_START_POS[:2], platformTopZ)
    objectId = p.loadURDF("cube_small.urdf", OBJECT_START_POS)
    p.addUserDebugText("TARGET", OBJECT_START_POS, textColorRGB=[1, 0, 0], textSize=1.5)

    # Sliders de respaldo (modo DEMO sin ESP32)
    targetPosXId = p.addUserDebugParameter("targetPosX", -1, 1, OBJECT_START_POS[0])
    targetPosYId = p.addUserDebugParameter("targetPosY", -1, 1, OBJECT_START_POS[1])
    targetPosZId = p.addUserDebugParameter("targetPosZ", -1, 1, OBJECT_START_POS[2])
    gripperId = p.addUserDebugParameter("gripper (1=abierta)", 0, 1, 1)

    reader = ConsoleReader(SERIAL_PORT, BAUD_RATE) if USE_SERIAL else None

    currentTarget = np.array(OBJECT_START_POS, dtype=float)
    grasped = False
    constraintId = None
    stepCount = 0
    frameDuration = 1.0 / SIM_HZ

    print("Simulación iniciada. Cierra la ventana o Ctrl+C en la consola para salir.")

    try:
        while p.isConnected():
            loopStart = time.time()

            if reader is not None:
                rawTarget, gripperOpen = reader.read_latest()
            else:
                rawTarget = [
                    p.readUserDebugParameter(targetPosXId),
                    p.readUserDebugParameter(targetPosYId),
                    p.readUserDebugParameter(targetPosZId),
                ]
                gripperOpen = p.readUserDebugParameter(gripperId) > 0.5

            # --- Movimiento fluido: filtro paso-bajo hacia el objetivo real ---
            rawTarget = np.array(rawTarget, dtype=float)
            currentTarget += SMOOTHING_ALPHA * (rawTarget - currentTarget)

            if stepCount % IK_EVERY_N_STEPS == 0:
                calculateAndApplyIK(
                    baxterId, endEffectorId, currentTarget.tolist(),
                    lowerLimits, upperLimits, jointRanges, restPoses,
                    movableJoints,
                )
                openFraction = 1.0 if gripperOpen else 0.0
                setGripper(baxterId, fingerJoints, openFraction)

            # --- Lógica de agarre ---
            eeState = p.getLinkState(baxterId, endEffectorId)
            eePos = np.array(eeState[4])
            objPos, _ = p.getBasePositionAndOrientation(objectId)
            dist = np.linalg.norm(eePos - np.array(objPos))

            if (not gripperOpen) and (not grasped) and dist < GRASP_DISTANCE:
                constraintId = p.createConstraint(
                    parentBodyUniqueId=baxterId,
                    parentLinkIndex=endEffectorId,
                    childBodyUniqueId=objectId,
                    childLinkIndex=-1,
                    jointType=p.JOINT_FIXED,
                    jointAxis=[0, 0, 0],
                    parentFramePosition=[0, 0, 0],
                    childFramePosition=[0, 0, 0],
                )
                grasped = True
                print("[AGARRE] Objeto sujetado por la pinza.")

            if gripperOpen and grasped:
                p.removeConstraint(constraintId)
                constraintId = None
                grasped = False
                print("[SUELTA] Objeto liberado.")

            p.stepSimulation()
            stepCount += 1

            # Mantiene el ritmo de simulación en SIM_HZ sin acumular retraso
            elapsed = time.time() - loopStart
            sleepTime = frameDuration - elapsed
            if sleepTime > 0:
                time.sleep(sleepTime)

    except KeyboardInterrupt:
        pass
    finally:
        if p.isConnected():
            p.disconnect(guiClient)


if __name__ == "__main__":
    main()


```

### 2.10 Código Arduino IDE

📄 Archivo: [`punto2/esp32_console/esp32_console.ino`](punto2/esp32_console/esp32_console.ino)

```cpp
/*
  Consola de mandos para Baxter (ESP32) — PyBullet
  =================================================

  Lee 3 potenciómetros (ejes X, Y, Z del efector final) y un pulsador
  (abrir/cerrar pinza), y envía por serial una línea con el formato:

        X,Y,Z,G\n

    X, Y, Z -> floats, posición objetivo dentro del espacio de trabajo
               alcanzable por Baxter (mismos rangos que el demo de
               PyBullet: X ~ [-0.3, 0.9], Y ~ [-0.8, 0.8], Z ~ [-0.3, 0.5])
    G       -> 1 = pinza abierta, 0 = pinza cerrada

  Debe coincidir con SERIAL_PORT / BAUD_RATE del script de Python
  (baxter_console_control.py). BAUD_RATE aquí = 115200.

  --------------------------------------------------------------------
  CONEXIONES (ajustables abajo en PIN_POT_X / PIN_POT_Y / PIN_POT_Z /
  PIN_BOTON):

    Potenciómetro X:  terminal central -> GPIO34 (ADC1_CH6)
    Potenciómetro Y:  terminal central -> GPIO35 (ADC1_CH7)
    Potenciómetro Z:  terminal central -> GPIO32 (ADC1_CH4)
    (los otros dos terminales de cada potenciómetro van a 3V3 y GND)

    Pulsador pinza:   un extremo -> GPIO25, el otro -> GND
                       (se usa INPUT_PULLUP, no requiere resistencia externa)

  NOTA: usar únicamente pines ADC1 (32-39) porque ADC2 no funciona si
  el ESP32 tiene el WiFi activo. Este firmware no usa WiFi.
*/

// ------------------- CONFIGURACIÓN DE PINES -------------------
const int PIN_POT_X = 34;
const int PIN_POT_Y = 35;
const int PIN_POT_Z = 32;
const int PIN_BOTON = 25;   // pulsador: presionado = pinza CERRADA

// ------------------- RANGOS DEL ESPACIO DE TRABAJO -------------------
// Ajustar estos límites según el alcance real que quieras permitirle
// al brazo en la simulación (coinciden con el demo original de PyBullet).
const float X_MIN = -0.30, X_MAX = 0.90;
const float Y_MIN = -0.80, Y_MAX = 0.80;
const float Z_MIN = -0.30, Z_MAX = 0.50;

// ------------------- SUAVIZADO (filtro exponencial) -------------------
// Además del filtrado que hace Python, se suaviza aquí la lectura cruda
// del ADC para eliminar ruido de los potenciómetros.
const float FILTRO_ALPHA = 0.25;

const unsigned long INTERVALO_ENVIO_MS = 30;  // ~33 Hz

float filtX = 0, filtY = 0, filtZ = 0;
bool primeraLectura = true;
unsigned long ultimoEnvio = 0;

float leerFiltrado(int pin, float &estadoFiltro) {
  int crudo = analogRead(pin);              // 0 - 4095 (ADC de 12 bits)
  float valor = crudo / 4095.0;             // normalizado 0.0 - 1.0
  if (primeraLectura) {
    estadoFiltro = valor;
  } else {
    estadoFiltro += FILTRO_ALPHA * (valor - estadoFiltro);
  }
  return estadoFiltro;
}

void setup() {
  Serial.begin(115200);
  analogReadResolution(12);          // 0-4095
  pinMode(PIN_BOTON, INPUT_PULLUP);  // presionado = LOW
  delay(500);
}

void loop() {
  float nx = leerFiltrado(PIN_POT_X, filtX);
  float ny = leerFiltrado(PIN_POT_Y, filtY);
  float nz = leerFiltrado(PIN_POT_Z, filtZ);
  primeraLectura = false;

  float x = X_MIN + nx * (X_MAX - X_MIN);
  float y = Y_MIN + ny * (Y_MAX - Y_MIN);
  float z = Z_MIN + nz * (Z_MAX - Z_MIN);

  // Pulsador presionado (LOW) -> pinza cerrada (G=0)
  int gripperOpen = (digitalRead(PIN_BOTON) == LOW) ? 0 : 1;

  unsigned long ahora = millis();
  if (ahora - ultimoEnvio >= INTERVALO_ENVIO_MS) {
    ultimoEnvio = ahora;
    Serial.print(x, 4);
    Serial.print(",");
    Serial.print(y, 4);
    Serial.print(",");
    Serial.print(z, 4);
    Serial.print(",");
    Serial.println(gripperOpen);
  }
}

```

**Explicación del código:**

- _Lectura de los potenciómetros (GPIO34, 35, 32):_
- _Lectura del pulsador de la pinza (GPIO25):_
- _Filtrado y envío por serial a 115200 baudios:_


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

### 3.9 Código Visual (Python)
```cpp
import argparse
import os
import time

import numpy as np
import pybullet as p
import pybullet_data
import serial

# =============================================================================
# Configuración general (ajusta estos valores mientras pruebas en el GUI)
# =============================================================================

BAXTER_URDF = "data/baxter_common/baxter_description/urdf/toms_baxter_ligero.urdf"
BAXTER_URDF_ORIGINAL = "data/baxter_common/baxter_description/urdf/toms_baxter.urdf"
BAXTER_BASE_POS = [0.5, -0.8, 0.0]
BAXTER_BASE_ORN = [0.0, 0.0, -1.0, -1.0]  # igual que baxter_ik_demo.py

# Nombres de los "endpoints" ya definidos en el URDF como referencia del
# efector final de cada brazo (joints fijos left_endpoint / right_endpoint).
ENDPOINT_JOINT_R = "right_endpoint"
ENDPOINT_JOINT_L = "left_endpoint"

RIGHT_ARM_JOINTS = ["right_s0", "right_s1", "right_e0", "right_e1",
                    "right_w0", "right_w1", "right_w2"]
LEFT_ARM_JOINTS = ["left_s0", "left_s1", "left_e0", "left_e1",
                    "left_w0", "left_w1", "left_w2"]

GRIPPER_R_JOINTS = ("r_gripper_l_finger_joint", "r_gripper_r_finger_joint")
GRIPPER_L_JOINTS = ("l_gripper_l_finger_joint", "l_gripper_r_finger_joint")
GRIPPER_OPEN_SIGN = {"r_gripper_l_finger_joint": 1, "r_gripper_r_finger_joint": -1,
                      "l_gripper_l_finger_joint": 1, "l_gripper_r_finger_joint": -1}
GRIPPER_MAX_TRAVEL = 0.020833  # límite real tomado del URDF

HEAD_PAN_JOINT = "head_pan"
HEAD_PAN_RANGE = (-1.3963, 1.3963)  # límites reales del URDF

# Posiciones de reposo ("home") de cada efector final, en coordenadas
# mundiales. Punto de partida tomado del target por defecto que ya funciona
# en baxter_ik_demo.py (0.2, 0.0, -0.1); se separan +/- en Y para cada brazo.
# AJUSTA ESTOS VALORES viendo el robot en el GUI si el punto no es alcanzable.
HOME_POS_R = np.array([0.2, -0.30, -0.10])
HOME_POS_L = np.array([0.2, 0.30, -0.10])

# Límites de trabajo (caja segura) para cada efector final, en coordenadas
# mundiales. Evita mandar objetivos claramente inalcanzables o que crucen
# de un lado al otro del robot.
WORKSPACE_X = (-0.15, 0.75)
WORKSPACE_Y_R = (-0.75, -0.05)
WORKSPACE_Y_L = (0.05, 0.75)
WORKSPACE_Z = (-0.35, 0.45)

XY_MAX_SPEED = 0.35      # m/s a máxima deflexión del joystick
CONTROL_HZ = 50.0
MAX_JOINT_VELOCITY = 2.0  # rad/s -> limita qué tan "brusco" se ve el movimiento
MAX_FORCE = 300.0

# =============================================================================
# Utilidades de PyBullet: mapear nombres de joints a índices
# =============================================================================


def build_joint_maps(body_id):
    """
    Devuelve:
      name_to_index: {nombre_joint: índice_joint}
      name_to_qindex: {nombre_joint: índice dentro del vector que devuelve
                        calculateInverseKinematics}
      num_dofs: número total de grados de libertad no fijos del cuerpo
    """
    name_to_index = {}
    name_to_qindex = {}
    num_dofs = 0

    for i in range(p.getNumJoints(body_id)):
        info = p.getJointInfo(body_id, i)
        joint_name = info[1].decode("utf-8")
        q_index = info[3]
        name_to_index[joint_name] = i
        if q_index > -1:
            # Mismo criterio usado en baxter_ik_demo.py: el vector de IK se
            # indexa como qIndex - 7 (la ESP32/pybullet reserva 7 slots para
            # una base flotante aunque la base esté fija).
            name_to_qindex[joint_name] = q_index - 7
            num_dofs += 1

    return name_to_index, name_to_qindex, num_dofs


def get_joint_ranges(body_id, num_dofs):
    """Límites amplios + poses de reposo en el origen, igual que el demo."""
    lower = [-2.0] * num_dofs
    upper = [2.0] * num_dofs
    ranges = [2.0] * num_dofs
    rest = [0.0] * num_dofs
    return lower, upper, ranges, rest


def solve_ik(body_id, endpoint_index, target_pos, lower, upper, ranges, rest):
    return p.calculateInverseKinematics(
        body_id, endpoint_index, list(target_pos),
        lowerLimits=lower, upperLimits=upper,
        jointRanges=ranges, restPoses=rest,
    )


def apply_arm_pose(body_id, name_to_index, name_to_qindex, joint_names, ik_result):
    """Aplica SOLO las articulaciones de un brazo (evita pisar el otro brazo)."""
    for name in joint_names:
        j_idx = name_to_index[name]
        q_idx = name_to_qindex[name]
        p.setJointMotorControl2(
            bodyIndex=body_id, jointIndex=j_idx,
            controlMode=p.POSITION_CONTROL,
            targetPosition=ik_result[q_idx],
            maxVelocity=MAX_JOINT_VELOCITY,
            force=MAX_FORCE,
        )


def set_gripper(body_id, name_to_index, joint_names, open_fraction):
    for name in joint_names:
        j_idx = name_to_index[name]
        sign = GRIPPER_OPEN_SIGN[name]
        target = sign * GRIPPER_MAX_TRAVEL * open_fraction
        p.setJointMotorControl2(
            bodyIndex=body_id, jointIndex=j_idx,
            controlMode=p.POSITION_CONTROL,
            targetPosition=target, force=50.0, maxVelocity=1.0,
        )


def set_head_pan(body_id, name_to_index, angle):
    j_idx = name_to_index[HEAD_PAN_JOINT]
    p.setJointMotorControl2(
        bodyIndex=body_id, jointIndex=j_idx,
        controlMode=p.POSITION_CONTROL,
        targetPosition=angle, force=100.0, maxVelocity=1.5,
    )


# =============================================================================
# Lectura de la consola (serial real, o sliders de PyBullet en modo --test)
# =============================================================================


class SerialConsole:
    """Lee la trama del ESP32 sin acumular retraso: si llegaron varias
    líneas desde la última lectura, se queda solo con la más reciente."""

    FIELDS = 12

    def __init__(self, port, baud=115200):
        self.ser = serial.Serial(port, baud, timeout=0)
        time.sleep(2.0)  # tiempo típico de reinicio de la ESP32 al abrir el puerto
        self.ser.reset_input_buffer()
        self._prev_grip_r = 0
        self._prev_grip_l = 0
        self._prev_home_r = 0
        self._prev_home_l = 0

    def read(self):
        """Devuelve un dict con la última trama válida, o None si no hay dato nuevo."""
        last_line = None
        while self.ser.in_waiting:
            raw = self.ser.readline()
            if raw:
                last_line = raw

        if last_line is None:
            return None

        try:
            text = last_line.decode("utf-8", errors="ignore").strip()
            parts = text.split(",")
            if len(parts) != self.FIELDS:
                return None
            vals = [float(x) for x in parts]
        except ValueError:
            return None

        return {
            "vxR": vals[0], "vyR": vals[1], "zR": vals[2],
            "vxL": vals[3], "vyL": vals[4], "zL": vals[5],
            "head": vals[6],
            "gripR": int(vals[7]), "gripL": int(vals[8]),
            "homeR": int(vals[9]), "homeL": int(vals[10]),
            "estop": bool(vals[11]),
        }


class DebugSliderConsole:
    """Sustituto de la ESP32 usando los sliders de depuración de PyBullet,
    para validar la parte de PyBullet antes de tener el hardware armado."""

    def __init__(self):
        self.sx_r = p.addUserDebugParameter("R vx", -1, 1, 0)
        self.sy_r = p.addUserDebugParameter("R vy", -1, 1, 0)
        self.sz_r = p.addUserDebugParameter("R potZ", 0, 1, 0.5)
        self.sx_l = p.addUserDebugParameter("L vx", -1, 1, 0)
        self.sy_l = p.addUserDebugParameter("L vy", -1, 1, 0)
        self.sz_l = p.addUserDebugParameter("L potZ", 0, 1, 0.5)
        self.head = p.addUserDebugParameter("head", 0, 1, 0.5)
        self.grip_r = p.addUserDebugParameter("Grip R (toggle)", 0, 1, 0)
        self.grip_l = p.addUserDebugParameter("Grip L (toggle)", 0, 1, 0)
        self.home_r = p.addUserDebugParameter("Home R (toggle)", 0, 1, 0)
        self.home_l = p.addUserDebugParameter("Home L (toggle)", 0, 1, 0)
        self.estop = p.addUserDebugParameter("ESTOP", 0, 1, 0)
        self._prev = {"gripR": 0, "gripL": 0, "homeR": 0, "homeL": 0}

    def _edge(self, key, raw_level):
        level = 1 if raw_level > 0.5 else 0
        edge = 1 if (level == 1 and self._prev[key] == 0) else 0
        self._prev[key] = level
        return edge

    def read(self):
        return {
            "vxR": p.readUserDebugParameter(self.sx_r),
            "vyR": p.readUserDebugParameter(self.sy_r),
            "zR": p.readUserDebugParameter(self.sz_r),
            "vxL": p.readUserDebugParameter(self.sx_l),
            "vyL": p.readUserDebugParameter(self.sy_l),
            "zL": p.readUserDebugParameter(self.sz_l),
            "head": p.readUserDebugParameter(self.head),
            "gripR": self._edge("gripR", p.readUserDebugParameter(self.grip_r)),
            "gripL": self._edge("gripL", p.readUserDebugParameter(self.grip_l)),
            "homeR": self._edge("homeR", p.readUserDebugParameter(self.home_r)),
            "homeL": self._edge("homeL", p.readUserDebugParameter(self.home_l)),
            "estop": p.readUserDebugParameter(self.estop) > 0.5,
        }


# =============================================================================
# Programa principal
# =============================================================================


def main():
    parser = argparse.ArgumentParser(description="Consola de mandos ESP32 para Baxter en PyBullet")
    parser.add_argument("--port", type=str, default=None, help="Puerto serie de la ESP32 (ej. COM5, /dev/ttyUSB0)")
    parser.add_argument("--baud", type=int, default=115200)
    parser.add_argument("--test", action="store_true", help="Usar sliders de PyBullet en vez de la ESP32")
    args = parser.parse_args()

    if not args.test and not args.port:
        parser.error("Debes indicar --port COMx (o usar --test para probar sin la ESP32)")

    if not os.path.exists(BAXTER_URDF):
        raise SystemExit(
            f"No se encontró '{BAXTER_URDF}'.\n"
            "Genera primero el URDF ligero (una sola vez):\n"
            "  cd baxter_common/baxter_description/urdf\n"
            "  python generar_urdf_ligero.py\n"
            "y vuelve a ejecutar este script desde la carpeta pybullet_robots."
        )

    # ---- Mundo PyBullet, con la ventana 3D aligerada para GPU integrada ----
    # Resolución reducida (menos píxeles = menos trabajo de la GPU).
    p.connect(p.GUI, options="--width=900 --height=560")
    p.setAdditionalSearchPath(pybullet_data.getDataPath())

    # Sombras y paneles laterales apagados: son el segundo mayor consumo de
    # GPU después de las mallas 3D (que ya quitamos con el URDF ligero).
    p.configureDebugVisualizer(p.COV_ENABLE_SHADOWS, 0)
    if not args.test:
        # En modo --test, este panel es el que muestra los sliders de
        # control; solo se apaga cuando ya se usa la ESP32 real.
        p.configureDebugVisualizer(p.COV_ENABLE_GUI, 0)
    p.configureDebugVisualizer(p.COV_ENABLE_RGB_BUFFER_PREVIEW, 0)
    p.configureDebugVisualizer(p.COV_ENABLE_DEPTH_BUFFER_PREVIEW, 0)
    p.configureDebugVisualizer(p.COV_ENABLE_SEGMENTATION_MARK_PREVIEW, 0)

    p.resetDebugVisualizerCamera(2.0, 180, 0.0, [0.52, 0.2, 0.3])
    p.setGravity(0, 0, -10)

    p.loadURDF("plane.urdf", [0, 0, -1], useFixedBase=True)
    body_id = p.loadURDF(BAXTER_URDF, useFixedBase=True)
    p.resetBasePositionAndOrientation(body_id, BAXTER_BASE_POS, BAXTER_BASE_ORN)

    for _ in range(100):
        p.stepSimulation()

    name_to_index, name_to_qindex, num_dofs = build_joint_maps(body_id)
    lower, upper, ranges, rest = get_joint_ranges(body_id, num_dofs)

    endpoint_r = name_to_index[ENDPOINT_JOINT_R]
    endpoint_l = name_to_index[ENDPOINT_JOINT_L]

    # ---- Consola de entrada ----
    console = DebugSliderConsole() if args.test else SerialConsole(args.port, args.baud)

    # ---- Estado de control ----
    target_r = HOME_POS_R.copy()
    target_l = HOME_POS_L.copy()
    grip_open_r = True
    grip_open_l = True

    dt = 1.0 / CONTROL_HZ
    print("Consola lista. Ctrl+C para salir.")

    try:
        while True:
            frame = console.read()

            if frame is not None:
                if frame["estop"]:
                    # Paro de emergencia: no se actualiza nada, los motores
                    # mantienen su última posición ya asignada.
                    pass
                else:
                    if frame["homeR"]:
                        target_r = HOME_POS_R.copy()
                    else:
                        target_r[0] += frame["vxR"] * XY_MAX_SPEED * dt
                        target_r[1] += frame["vyR"] * XY_MAX_SPEED * dt
                        target_r[2] = np.interp(frame["zR"], [0, 1], WORKSPACE_Z)
                        target_r[0] = float(np.clip(target_r[0], *WORKSPACE_X))
                        target_r[1] = float(np.clip(target_r[1], *WORKSPACE_Y_R))

                    if frame["homeL"]:
                        target_l = HOME_POS_L.copy()
                    else:
                        target_l[0] += frame["vxL"] * XY_MAX_SPEED * dt
                        target_l[1] += frame["vyL"] * XY_MAX_SPEED * dt
                        target_l[2] = np.interp(frame["zL"], [0, 1], WORKSPACE_Z)
                        target_l[0] = float(np.clip(target_l[0], *WORKSPACE_X))
                        target_l[1] = float(np.clip(target_l[1], *WORKSPACE_Y_L))

                    if frame["gripR"]:
                        grip_open_r = not grip_open_r
                    if frame["gripL"]:
                        grip_open_l = not grip_open_l

                    head_angle = np.interp(frame["head"], [0, 1], HEAD_PAN_RANGE)
                    set_head_pan(body_id, name_to_index, head_angle)
                    set_gripper(body_id, name_to_index, GRIPPER_R_JOINTS, 1.0 if grip_open_r else 0.0)
                    set_gripper(body_id, name_to_index, GRIPPER_L_JOINTS, 1.0 if grip_open_l else 0.0)

                    ik_r = solve_ik(body_id, endpoint_r, target_r, lower, upper, ranges, rest)
                    ik_l = solve_ik(body_id, endpoint_l, target_l, lower, upper, ranges, rest)
                    apply_arm_pose(body_id, name_to_index, name_to_qindex, RIGHT_ARM_JOINTS, ik_r)
                    apply_arm_pose(body_id, name_to_index, name_to_qindex, LEFT_ARM_JOINTS, ik_l)

            p.stepSimulation()
            time.sleep(dt)

    except KeyboardInterrupt:
        print("\nCerrando...")
    finally:
        p.disconnect()


if __name__ == "__main__":
    main()


```

### 3.10 Código Arduino IDE

📄 Archivo: [`punto3/esp32_baxter_console.ino`](punto3/esp32_baxter_console.ino)

```cpp
const int PIN_VRX_R    = 34;
const int PIN_VRY_R    = 35;
const int PIN_POTZ_R   = 32;
const int PIN_VRX_L    = 33;
const int PIN_VRY_L    = 36;
const int PIN_POTZ_L   = 39;
const int PIN_POT_HEAD = 25;

// ---------------------------- Pines digitales --------------------------------
const int PIN_BTN_HOME_R = 18;
const int PIN_BTN_GRIP_R = 19;
const int PIN_BTN_HOME_L = 21;
const int PIN_BTN_GRIP_L = 22;
const int PIN_BTN_ESTOP  = 23;

// ---------------------------- Parámetros de filtrado -------------------------
const float EMA_ALPHA          = 0.25f;  // 0<alpha<=1 ; más bajo = más suave/lento
const int   DEADZONE           = 150;    // zona muerta joystick (sobre 0-4095, centro ~2048)
const unsigned long SEND_PERIOD_MS = 20; // 50 Hz

// ---------------------------- Estado filtrado ---------------------------------
float emaVxR = 0, emaVyR = 0, emaZR = 0;
float emaVxL = 0, emaVyL = 0, emaZL = 0;
float emaHead = 0;
unsigned long lastSend = 0;

// ---------------------------- Manejo de botones con antirrebote --------------
struct Button {
  int pin;
  bool state;      // estado "toggle" ya confirmado (para los de flanco)
  int  lastRead;
  unsigned long lastChange;
};

Button btnHomeR = {PIN_BTN_HOME_R, false, HIGH, 0};
Button btnGripR = {PIN_BTN_GRIP_R, false, HIGH, 0};
Button btnHomeL = {PIN_BTN_HOME_L, false, HIGH, 0};
Button btnGripL = {PIN_BTN_GRIP_L, false, HIGH, 0};
Button btnEstop = {PIN_BTN_ESTOP,  false, HIGH, 0};

const unsigned long DEBOUNCE_MS = 30;

// Devuelve true SOLO en el instante en que el botón pasa de suelto a presionado
bool readButtonEdge(Button &b) {
  int reading = digitalRead(b.pin);
  bool pressedEdge = false;

  if (reading != b.lastRead) {
    b.lastChange = millis();
  }
  if ((millis() - b.lastChange) > DEBOUNCE_MS) {
    if (reading == LOW && b.state == false) {
      b.state = true;
      pressedEdge = true;
    } else if (reading == HIGH) {
      b.state = false;
    }
  }
  b.lastRead = reading;
  return pressedEdge;
}

// Devuelve el nivel actual estable (para el botón de mantener, ej. paro de emergencia)
bool readButtonLevel(Button &b) {
  int reading = digitalRead(b.pin);
  if (reading != b.lastRead) {
    b.lastChange = millis();
  }
  if ((millis() - b.lastChange) > DEBOUNCE_MS) {
    b.state = (reading == LOW);
  }
  b.lastRead = reading;
  return b.state;
}

// ---------------------------- Utilidades de señal -----------------------------
float applyDeadzoneAndNormalize(int raw) {
  const int center = 2048;
  const int maxDev = 2048;
  int dev = raw - center;
  if (abs(dev) < DEADZONE) {
    dev = 0;
  } else {
    dev = (dev > 0) ? (dev - DEADZONE) : (dev + DEADZONE);
  }
  float norm = (float)dev / (float)(maxDev - DEADZONE);
  if (norm > 1.0f)  norm = 1.0f;
  if (norm < -1.0f) norm = -1.0f;
  return norm;
}

float ema(float prev, float sample, float alpha) {
  return alpha * sample + (1.0f - alpha) * prev;
}

// ---------------------------- Setup / Loop -------------------------------------
void setup() {
  Serial.begin(115200);

  analogReadResolution(12);        // 0-4095
  analogSetAttenuation(ADC_11db);  // permite leer el rango completo 0-3.3V

  pinMode(PIN_BTN_HOME_R, INPUT_PULLUP);
  pinMode(PIN_BTN_GRIP_R, INPUT_PULLUP);
  pinMode(PIN_BTN_HOME_L, INPUT_PULLUP);
  pinMode(PIN_BTN_GRIP_L, INPUT_PULLUP);
  pinMode(PIN_BTN_ESTOP,  INPUT_PULLUP);

  delay(300); // estabilización de la fuente/ADC al arrancar
}

void loop() {
  // ---- Lectura cruda ----
  int rawVxR  = analogRead(PIN_VRX_R);
  int rawVyR  = analogRead(PIN_VRY_R);
  int rawZR   = analogRead(PIN_POTZ_R);
  int rawVxL  = analogRead(PIN_VRX_L);
  int rawVyL  = analogRead(PIN_VRY_L);
  int rawZL   = analogRead(PIN_POTZ_L);
  int rawHead = analogRead(PIN_POT_HEAD);

  // ---- Normalización + zona muerta en joysticks ----
  float nVxR = applyDeadzoneAndNormalize(rawVxR);
  float nVyR = applyDeadzoneAndNormalize(rawVyR);
  float nVxL = applyDeadzoneAndNormalize(rawVxL);
  float nVyL = applyDeadzoneAndNormalize(rawVyL);

  // ---- Potenciómetros: valor absoluto 0..1 ----
  float nZR   = rawZR   / 4095.0f;
  float nZL   = rawZL   / 4095.0f;
  float nHead = rawHead / 4095.0f;

  // ---- Filtro EMA: esto es lo que hace el movimiento FLUIDO ----
  emaVxR  = ema(emaVxR,  nVxR,  EMA_ALPHA);
  emaVyR  = ema(emaVyR,  nVyR,  EMA_ALPHA);
  emaZR   = ema(emaZR,   nZR,   EMA_ALPHA);
  emaVxL  = ema(emaVxL,  nVxL,  EMA_ALPHA);
  emaVyL  = ema(emaVyL,  nVyL,  EMA_ALPHA);
  emaZL   = ema(emaZL,   nZL,   EMA_ALPHA);
  emaHead = ema(emaHead, nHead, EMA_ALPHA);

  // ---- Botones ----
  bool homeR_edge = readButtonEdge(btnHomeR);
  bool gripR_edge = readButtonEdge(btnGripR);
  bool homeL_edge = readButtonEdge(btnHomeL);
  bool gripL_edge = readButtonEdge(btnGripL);
  bool estopLevel = readButtonLevel(btnEstop);

  // ---- Envío de trama a 50 Hz ----
  if (millis() - lastSend >= SEND_PERIOD_MS) {
    lastSend = millis();
    Serial.print(emaVxR, 3);            Serial.print(',');
    Serial.print(emaVyR, 3);            Serial.print(',');
    Serial.print(emaZR,  3);            Serial.print(',');
    Serial.print(emaVxL, 3);            Serial.print(',');
    Serial.print(emaVyL, 3);            Serial.print(',');
    Serial.print(emaZL,  3);            Serial.print(',');
    Serial.print(emaHead, 3);           Serial.print(',');
    Serial.print(gripR_edge ? 1 : 0);   Serial.print(',');
    Serial.print(gripL_edge ? 1 : 0);   Serial.print(',');
    Serial.print(homeR_edge ? 1 : 0);   Serial.print(',');
    Serial.print(homeL_edge ? 1 : 0);   Serial.print(',');
    Serial.println(estopLevel ? 1 : 0);
  }
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

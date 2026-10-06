# Plataforma experimental ARBOREON Mini para navegación agrícola autónoma
## Selección de base móvil, motores, controladores, LiDAR, cámara, ROS 2, micro-ROS y ArduPilot

**Versión:** 1.0  
**Fecha:** 6 de octubre de 2026  
**Estado:** Documento técnico de diseño para prototipo de investigación  
**Proyecto:** ARBOREON — plataforma robótica autónoma, modular y abierta para olivar ecológico

---

## Resumen

Este documento sintetiza y formaliza las decisiones de diseño discutidas para construir un primer prototipo móvil de **ARBOREON Mini**, destinado a validar navegación autónoma en exterior, seguimiento de misiones definidas en Mission Planner, percepción mediante LiDAR y cámara, control diferencial tipo *skid-steering*, integración con ROS 2 y futura transferencia tecnológica hacia una desbrozadora de orugas de mayor escala.

Se comparan distintas bases móviles, motores DC con encoder, controladores de potencia, el papel de una placa MicroROS basada en ESP32-S3, controladores de cuatro motores con STM32, Raspberry Pi 5 como ordenador embarcado y dos alternativas LiDAR: **Yahboom MS200** y **SLAMTEC RPLIDAR C1**. También se analiza la conveniencia de conectar el LiDAR directamente a ArduPilot o procesarlo primero en ROS 2 para añadir semántica visual.

La recomendación principal es utilizar una **base de orugas de tamaño medio con dos motores DC de 12 V con encoder**, controlada por un **driver Yahboom de cuatro canales con STM32F103** o, si la corriente lo exige, por un **driver de potencia externo tipo Cytron MDD10A**. La navegación global se delega a **Cube Orange/ArduPilot + GNSS**, mientras que una **Raspberry Pi 5 de 8 GB** ejecuta ROS 2, Nav2, SLAM, visión y fusión sensorial. Como LiDAR se recomienda el **RPLIDAR C1**, principalmente porque puede utilizarse tanto directamente con ArduPilot como con ROS 2, proporcionando mayor flexibilidad experimental.

---

# 1. Objetivo del prototipo

El prototipo no pretende reproducir todavía una desbrozadora agrícola completa. Su objetivo es validar, a escala reducida, las capas fundamentales de la arquitectura final:

1. navegación global mediante GNSS y misiones creadas en Mission Planner;
2. conducción diferencial mediante dos orugas;
3. evitación de obstáculos;
4. localización y SLAM mediante LiDAR;
5. percepción visual;
6. fusión LiDAR + cámara;
7. clasificación semántica de obstáculos;
8. futura integración de navegación entre filas de olivos;
9. transición desde un prototipo educativo a una plataforma agrícola real.

La meta inicial más importante es:

> **Crear una ruta en Mission Planner, transmitirla al robot y observar cómo el vehículo la ejecuta físicamente de forma autónoma.**

Posteriormente se añadirán percepción y comportamiento semántico.

---

# 2. Filosofía de diseño

La arquitectura debe separar claramente cuatro niveles.

```text
┌──────────────────────────────────────────────┐
│ NIVEL 4 — INTERACCIÓN Y SUPERVISIÓN         │
│ Mission Planner / RViz / interfaz futura    │
└────────────────────────┬─────────────────────┘
                         │
┌────────────────────────▼─────────────────────┐
│ NIVEL 3 — AUTONOMÍA DE ALTO NIVEL           │
│ Raspberry Pi 5                              │
│ ROS 2 / Nav2 / SLAM / visión / BT / IA     │
└────────────────────────┬─────────────────────┘
                         │
┌────────────────────────▼─────────────────────┐
│ NIVEL 2 — CONTROL DE VEHÍCULO                │
│ Cube Orange / ArduPilot Rover               │
│ GNSS / EKF / misiones / failsafes           │
└────────────────────────┬─────────────────────┘
                         │
┌────────────────────────▼─────────────────────┐
│ NIVEL 1 — ACTUACIÓN                          │
│ STM32 motor driver / MDD10A / motores       │
│ encoders / alimentación                      │
└──────────────────────────────────────────────┘
```

El propósito es que la lógica de percepción y navegación avanzada pueda evolucionar sin tener que modificar el control eléctrico de los motores.

---

# 3. Selección de la base móvil

## 3.1 Robot MicroROS original

El robot MicroROS de Yahboom es adecuado para aprender:

- ROS 2;
- micro-ROS;
- SLAM;
- control diferencial;
- sensores;
- visión;
- navegación básica.

Sin embargo, presenta limitaciones para un prototipo agrícola:

- poco espacio físico;
- pequeñas ruedas;
- escasa capacidad de carga;
- menor representatividad en terreno irregular;
- limitada capacidad para montar Cube, GNSS, RFD868x, Raspberry Pi, cámara y LiDAR simultáneamente.

## 3.2 Base 4WD amplia

Una base 4WD ofrece:

- más espacio;
- más carga;
- cuatro motores;
- plataforma superior para electrónica;
- facilidad de montaje.

Puede controlarse como un sistema diferencial:

```text
lado izquierdo  = motor 1 + motor 3
lado derecho    = motor 2 + motor 4
```

Esto aproxima la cinemática *skid-steering* y es muy útil en laboratorio, aunque no reproduce fielmente la interacción suelo-oruga.

## 3.3 Base de orugas

La base de orugas es la opción recomendada porque reproduce directamente la cinemática de la futura desbrozadora:

```text
motor izquierdo  → oruga izquierda
motor derecho    → oruga derecha
```

Ventajas:

- giro diferencial real;
- posibilidad de giro casi sobre el propio eje;
- deslizamiento lateral real;
- ensayo sobre hierba, tierra, grava y desniveles;
- comportamiento más próximo a una desbrozadora agrícola;
- permite estudiar *slip* y degradación de odometría por encoder.

### Recomendación

> **Base de orugas de aproximadamente 40–70 cm, con dos motores DC de 12 V con encoder y capacidad suficiente para transportar 3–10 kg de electrónica.**

---

# 4. Motores

Se han considerado motores DC con reductora y encoder Hall.

## 4.1 Motores Yahboom 520

Yahboom comercializa motores 520 con encoder en versiones de aproximadamente:

- 205 rpm;
- 333 rpm;
- 550 rpm.

Una variante alrededor de **333 rpm y reducción 1:30** resulta especialmente interesante para un robot de orugas.

El fabricante describe:

- encoder magnético;
- señal cuadrada acondicionada;
- *pull-up* integrado;
- engranajes metálicos;
- motor DC brushed.

## 4.2 Motores MG513 / MG540 de bases de orugas

En las bases estudiadas aparecen motores similares con:

- 12 V;
- reductora próxima a 1:30;
- encoder Hall AB;
- velocidades cercanas a 330 rpm tras reductora.

Una variante MG540 vista en los anuncios estudiados se anuncia alrededor de:

- 12 V;
- 15 W;
- 1,44 A nominal;
- ~330 rpm;
- reducción 1:30.

Estos datos proceden de anuncios de terceros y deben verificarse experimentalmente antes del diseño final.

---

# 5. Uso de encoders

Los encoders **sí son útiles**, pero no deben considerarse la fuente principal de posición global.

En agricultura pueden aparecer:

- patinamiento longitudinal;
- deslizamiento lateral durante giros;
- barro;
- hierba húmeda;
- grava;
- pendientes;
- irregularidades.

Por tanto:

```text
encoder ≠ posición absoluta fiable
```

Pero sí son muy útiles para:

```text
encoder
  ↓
velocidad real del motor
  ↓
PID
  ↓
PWM corregido
```

La localización global debe depender principalmente de:

```text
GNSS / RTK
+
IMU
+
LiDAR odometry / SLAM
+
visión
```

---

# 6. Placa MicroROS Yahboom con ESP32-S3

La placa MicroROS integra:

- ESP32-S3;
- micro-ROS;
- cuatro canales de motor con encoder;
- IMU de 6 ejes;
- dos servos PWM;
- interfaz LiDAR;
- Wi-Fi y Bluetooth;
- comunicación serie;
- alimentación para Raspberry Pi 5 de 5,1 V / 5 A.

Yahboom recomienda sus motores 310 y el LiDAR MS200, aunque indica que pueden utilizarse otros motores y LiDAR verificando el pinout.

### Cuándo es especialmente útil

```text
PC externo
   ↓ Wi-Fi/USB
ESP32 micro-ROS
   ↓
motores + encoders
```

También puede utilizarse con Raspberry Pi.

### Limitación

No se publica de forma suficientemente clara la corriente máxima por canal de la etapa de potencia. Esto limita su uso directo con motores más grandes.

---

# 7. Driver Yahboom de cuatro motores con STM32

Yahboom ofrece un **4-Channel Encoder Motor Drive Module** basado en:

```text
STM32F103RCT6
```

Características declaradas:

- cuatro motores DC independientes;
- cuatro encoders;
- procesamiento local de encoder;
- comunicación UART;
- comunicación I²C;
- compatibilidad con Raspberry Pi, Jetson, STM32, MSPM0, etc.;
- motores 520 con alimentación de 12,6 V;
- motores 310/TT con alimentación de 7,4 V.

## 7.1 Ventaja principal

Permite sustituir:

```text
Raspberry Pi
  ↓
ESP32 micro-ROS
  ↓
driver externo
  ↓
motores
```

por:

```text
Raspberry Pi
  ↓ UART/USB
STM32 motor controller
  ↓
motores + encoders
```

Esto reduce cableado y complejidad.

## 7.2 Comunicación con Raspberry Pi

Opción recomendada:

```text
Raspberry Pi USB
      │
      ▼
USB/UART
      │
STM32 Yahboom
```

o directamente UART:

```text
Pi TX → STM32 RX
Pi RX ← STM32 TX
GND   ↔ GND
```

En ROS 2 se implementaría:

```text
/cmd_vel
    ↓
arboreon_motor_driver
    ↓
conversión diferencial
    ↓
velocidad_L / velocidad_R
    ↓
UART
    ↓
STM32
```

## 7.3 Conversión diferencial

Para una separación efectiva entre orugas `W`:

```text
vL = v - ω·W/2
vR = v + ω·W/2
```

donde:

- `v` = velocidad lineal;
- `ω` = velocidad angular;
- `vL` = velocidad de la oruga izquierda;
- `vR` = velocidad de la oruga derecha.

---

# 8. ¿Es necesaria la placa MicroROS si usamos el driver STM32?

No es necesaria para mover el robot.

La Raspberry Pi ejecuta ROS 2 completo, por lo que puede controlar directamente el driver STM32 mediante UART.

Arquitectura mínima:

```text
Raspberry Pi 5
   │
   ├── ROS 2
   ├── Nav2
   ├── SLAM
   ├── visión
   │
   └── UART
         ↓
   STM32 motor driver
     ├── motor L + encoder
     └── motor R + encoder
```

Por tanto:

> **micro-ROS no es obligatorio para ejecutar ROS 2 en el robot.**

## 8.1 Cuándo conservar la placa MicroROS

Puede resultar interesante para:

- docencia;
- investigación micro-ROS;
- IMU secundaria;
- bumper;
- sensores ambientales;
- servos de cámara;
- iluminación;
- E-stop auxiliar;
- sensores de proximidad;
- actuadores auxiliares.

Arquitectura ampliada:

```text
                 Raspberry Pi 5
                /                             /                          USB micro-ROS         UART
             ↓                   ↓
          ESP32               STM32
       sensores/servos     motores/encoders
```

---

# 9. Driver externo Cytron MDD10A

Si los motores requieren más corriente que el controlador Yahboom, una alternativa robusta es **Cytron MDD10A**.

Características declaradas por el fabricante:

- 5–30 V;
- 10 A continuos por canal;
- hasta 30 A de pico;
- dos motores DC brushed;
- lógica compatible con 3,3/5 V;
- control PWM + DIR.

Arquitectura:

```text
ESP32 / STM32
   │
PWM + DIR
   ▼
MDD10A
 ├── motor L
 └── motor R
```

Los encoders se conectan al microcontrolador encargado del PID, no al MDD10A.

### Criterio de selección

**Driver Yahboom STM32**  
→ preferible si soporta holgadamente la corriente de los motores.

**MDD10A**  
→ preferible cuando necesitamos mayor margen eléctrico o motores más potentes.

---

# 10. Raspberry Pi 5

Para el prototipo se recomienda:

> **Raspberry Pi 5, 8 GB**

Especificaciones relevantes:

- Broadcom BCM2712;
- 4 × Cortex-A76 a 2,4 GHz;
- USB 3.0;
- Gigabit Ethernet;
- interfaces MIPI;
- PCIe 2.0 x1;
- variantes de hasta 16 GB.

La versión de 8 GB proporciona margen para ejecutar simultáneamente:

```text
ROS 2
Nav2
SLAM Toolbox
LiDAR
cámara
OpenCV
MAVLink/MAVROS
BehaviorTree
rosbag2
```

RViz y Mission Planner deberían ejecutarse preferentemente en un PC remoto.

---

# 11. Cámara

Para el prototipo interesa una cámara conectada directamente a Raspberry Pi:

```text
USB / MIPI
   ↓
Raspberry Pi
   ↓
ROS 2 / OpenCV / IA
```

Aplicaciones:

- detección de personas;
- detección de troncos;
- clasificación de piedra/vegetación;
- segmentación del terreno;
- seguimiento de filas;
- fusión con LiDAR.

En etapas posteriores puede sustituirse por una cámara RGB-D o cámara industrial PoE.

---

# 12. Comparación LiDAR: MS200 vs RPLIDAR C1

## 12.1 Yahboom MS200

Ventajas:

- compacto;
- integración sencilla con ecosistema Yahboom;
- drivers/tutoriales ROS;
- adecuado para SLAM educativo;
- buena opción si forma parte del kit.

Limitaciones:

- no se ha identificado soporte nativo equivalente en ArduPilot;
- queda más ligado a la capa ROS.

## 12.2 SLAMTEC RPLIDAR C1

Especificaciones oficiales:

- hasta 12 m en superficie clara;
- 360°;
- 5 kHz;
- 8–12 Hz de giro;
- UART TTL a 460800;
- ±30 mm;
- IP54;
- hasta 40 klux de luz ambiente.

Ventaja decisiva:

> **ArduPilot soporta directamente el RPLIDAR C1 como sensor de proximidad 360°.**

Configuración típica:

```text
SERIALx_PROTOCOL = 11   # Lidar360
SERIALx_BAUD     = 460800
PRX1_TYPE        = 5
PRX1_ORIENT      = 0
```

En Rover puede ser necesario:

```text
SCHED_LOOP_RATE = 400
```

## 12.3 Recomendación

> **RPLIDAR C1**

porque puede utilizarse en dos fases:

### Fase A — directamente con ArduPilot

```text
RPLIDAR C1
    ↓
Cube Orange
    ↓
BendyRuler
    ↓
avoidance
```

### Fase B — mediante ROS 2

```text
RPLIDAR C1
    ↓
Raspberry Pi
    ↓
SLAM / Nav2 / semántica
    ↓
Cube
```

---

# 13. ArduPilot y comportamiento ante obstáculos

ArduPilot soporta varias estrategias de avoidance.

Para Rover, **BendyRuler** es especialmente interesante:

- examina direcciones alrededor del vehículo;
- busca espacio libre;
- intenta mantener progreso hacia el destino;
- funciona en AUTO, GUIDED y RTL.

Configuración conceptual:

```text
OA_TYPE = 1
```

Parámetros principales:

```text
OA_BR_LOOKAHEAD
OA_MARGIN_MAX
```

También existe:

```text
OA_TYPE = 3
```

para combinar Dijkstra y BendyRuler.

---

# 14. Problema agrícola: hierba alta

Un LiDAR 2D no conoce la naturaleza del objeto.

Un retorno puede representar:

```text
hierba
piedra
persona
tronco
rama
poste
```

Si el C1 está conectado directamente al Cube:

```text
retorno LiDAR
     ↓
obstáculo geométrico
     ↓
ArduPilot
```

la hierba alta puede generar falsos obstáculos.

Esto es especialmente importante en una desbrozadora porque la vegetación que debe cortar no debería necesariamente tratarse como un obstáculo.

---

# 15. Añadir semántica mediante visión

La solución propuesta es fusionar:

```text
LiDAR → geometría
Cámara → semántica
```

Arquitectura:

```text
             RPLIDAR C1
                  │
                  ▼
             Raspberry Pi
            ┌─────┴─────┐
            │           │
          LiDAR       cámara
            │           │
       geometría      visión
            │           │
            └─────┬─────┘
                  ▼
             sensor fusion
                  │
          semantic obstacle
                  │
                  ▼
             Behavior Tree
```

Clases operativas recomendadas:

| Clase | Acción |
|---|---|
| GRASS / vegetación transitable | continuar |
| HIGH_GRASS | continuar lentamente |
| ROCK | evitar |
| TREE | aproximar / CircleTree |
| PERSON | STOP |
| ANIMAL | STOP |
| UNKNOWN | STOP o evitar |
| HARD_OBSTACLE | evitar |

La pregunta deja de ser:

> “¿Hay un objeto?”

y pasa a ser:

> **“¿Es ese objeto transitable/desbrozable?”**

---

# 16. Fusión cámara–LiDAR

Se recomienda calibrar ambos sensores mediante ROS TF2:

```text
base_link
├── lidar_link
└── camera_link
```

La cámara aporta:

```text
person
tree
rock
grass
```

El LiDAR aporta:

```text
ángulo
distancia
geometría
```

Ejemplo:

```text
camera → PERSON @ +18°
LiDAR  → 2.3 m @ +18°
```

Resultado:

```text
class: PERSON
distance: 2.3 m
bearing: +18°
action: STOP
```

Para vegetación resulta especialmente prometedora la **segmentación semántica**.

---

# 17. Arquitectura final recomendada

```text
                         PC REMOTO
                  Mission Planner / RViz
                           │
                        RFD868x
                           │
                           ▼
                       Cube Orange
                      /                             Here2 GPS      MAVLink
                                  │
                                  ▼
                          Raspberry Pi 5 8GB
                   ┌──────────┬───────────┐
                   │          │           │
              RPLIDAR C1    cámara      ROS 2
                   │          │           │
                   └──────┬───┘         Nav2
                          │               │
                       fusión          /cmd_vel
                          │               │
                    Behavior Tree         │
                                          ▼
                                arboreon_motor_driver
                                          │
                                         UART
                                          │
                                          ▼
                              Yahboom STM32 4CH driver
                                  /                                             Motor L             Motor R
                                ▲                   ▲
                           Encoder L            Encoder R
```

Opcionalmente:

```text
Raspberry Pi
     │
 micro-ROS
     │
ESP32 Yahboom
     │
sensores auxiliares
```

---

# 18. Alimentación

Para motores 12 V:

```text
Li-ion / LiPo 3S
11.1 V nominal
12.6 V cargada
```

Distribución:

```text
                  BATERÍA 3S
                      │
       ┌──────────────┼───────────────┐
       │              │               │
       ▼              ▼               ▼
 motor driver     electrónica     Cube power module
       │
   motores 12 V
```

Se recomienda:

- fusible cerca de batería;
- fusible independiente para locomoción;
- fusible independiente para electrónica;
- masa común controlada;
- distribución en estrella;
- cables de potencia separados de señal;
- condensadores cerca de drivers;
- protección TVS;
- power module dedicado para Cube.

---

# 19. Mejor base para pruebas agrícolas

| Plataforma | Campo | Espacio | Similitud con desbrozadora | Recomendación |
|---|---:|---:|---:|---|
| MicroROS pequeño | Baja | Baja | Media | Docencia |
| 4WD amplia | Media | Alta | Media | Laboratorio |
| **Orugas 12 V + encoder** | **Alta** | **Alta** | **Muy alta** | **Preferida** |

### Recomendación final

> **Base de orugas de tamaño medio con dos motores 12 V + encoder.**

Es la plataforma con mayor valor científico y tecnológico porque reproduce desde el principio los problemas reales del vehículo agrícola.

---

# 20. Secuencia experimental propuesta

## Experimento 1 — Locomoción

```text
Raspberry Pi
→ STM32 driver
→ motores
```

Objetivo: avanzar, retroceder, girar, pivotar y controlar velocidades L/R.

## Experimento 2 — Cube + Mission Planner

```text
Mission Planner
→ RFD868x
→ Cube
→ GNSS
→ robot
```

Objetivo: seguir una misión de waypoints.

## Experimento 3 — RPLIDAR C1 directamente en Cube

Objetivo: detectar obstáculos, activar BendyRuler, rodear y continuar misión.

## Experimento 4 — C1 en Raspberry Pi

Objetivo: `/scan`, SLAM, mapa y Nav2.

## Experimento 5 — Cámara

Objetivo: persona, tronco, piedra y vegetación.

## Experimento 6 — Fusión semántica

```text
grass → traversable
rock → avoid
person → stop
tree → CircleTree
unknown → stop
```

## Experimento 7 — Terreno agrícola

Medir:

- error lateral;
- porcentaje de misión completada;
- intervenciones manuales;
- distancia mínima al obstáculo;
- falsos positivos por vegetación;
- influencia del deslizamiento;
- estabilidad de velocidad;
- deriva SLAM;
- error encoder vs GNSS/LiDAR.

---

# 21. Hipótesis de investigación

**H1.** La navegación local basada en LiDAR + visión puede mantener una trayectoria agrícola útil incluso cuando la odometría por encoder está degradada por *slip*.

**H2.** La combinación de clasificación visual y geometría LiDAR reduce significativamente los falsos obstáculos causados por vegetación frente a la evitación puramente geométrica.

**H3.** Una arquitectura jerárquica:

```text
GNSS/ArduPilot → navegación global
ROS2/LiDAR/visión → navegación local
```

es más robusta que cualquiera de los dos sistemas por separado.

**H4.** Una plataforma de orugas a escala proporciona un banco experimental suficientemente representativo para transferir algoritmos a una desbrozadora agrícola real.

---

# 22. Decisión resumida de componentes

| Subsistema | Recomendación |
|---|---|
| Base | **Orugas, tamaño medio** |
| Motores | **DC 12 V + encoder Hall AB** |
| Driver inicial | **Yahboom 4CH STM32** |
| Driver alternativo | **Cytron MDD10A** |
| Control motor | STM32 + encoder |
| Ordenador | **Raspberry Pi 5 8 GB** |
| ROS | ROS 2 |
| micro-ROS | Opcional/experimental |
| Autopiloto | Cube Orange |
| Respaldo | Cube Black |
| GNSS inicial | Here2 |
| Telemetría | RFD868x |
| LiDAR | **SLAMTEC RPLIDAR C1** |
| Cámara | USB/MIPI 2 MP o superior |
| Local planner | Nav2 / ArduPilot BendyRuler |
| Percepción | cámara + LiDAR |
| Seguridad semántica | PERSON/UNKNOWN → STOP |

---

# 23. Conclusión

La evolución de la arquitectura permite simplificar considerablemente el prototipo.

El robot MicroROS completo deja de ser imprescindible. Una base de orugas más grande aporta más espacio, una cinemática más representativa y mejores condiciones de ensayo en campo.

El controlador Yahboom de cuatro motores basado en STM32 permite delegar locomoción y encoders a un microcontrolador dedicado. Raspberry Pi puede concentrarse en ROS 2, percepción, SLAM y planificación. La placa MicroROS ESP32 puede conservarse como plataforma experimental y controlador de sensores auxiliares, pero deja de ser obligatoria.

El **RPLIDAR C1** es la elección preferente porque proporciona una ruta incremental muy atractiva:

```text
C1 → Cube → avoidance
```

primero, y posteriormente:

```text
C1 + cámara → Raspberry Pi → ROS2 → navegación semántica
```

El mayor reto científico aparece precisamente en agricultura: un LiDAR 2D no distingue entre piedra, tronco y hierba alta. La combinación de visión y LiDAR permite evolucionar desde la simple evitación geométrica hacia una **clasificación semántica de transitabilidad**, capaz de decidir qué debe cortar el robot, qué debe rodear y ante qué debe detenerse.

---

# 24. Referencias técnicas y enlaces

## ArduPilot

1. **ArduPilot — RPLIDAR A1/A2/C1/S1 360° lidar**  
   https://ardupilot.org/copter/docs/common-rplidar-a2.html

2. **ArduPilot Rover — Object Avoidance**  
   https://ardupilot.org/rover/docs/common-object-avoidance-landing-page.html

3. **ArduPilot — BendyRuler Path Planning**  
   https://ardupilot.org/copter/docs/common-oa-bendyruler.html

4. **ArduPilot — Dijkstra + BendyRuler**  
   https://ardupilot.org/copter/docs/common-oa-dijkstrabendyruler.html

5. **ArduPilot Rover — Proximity Sensors**  
   https://ardupilot.org/rover/docs/common-proximity-landingpage.html

6. **ArduPilot Rover — Parameters**  
   https://ardupilot.org/rover/docs/parameters.html

7. **ArduPilot GitHub**  
   https://github.com/ArduPilot/ardupilot

8. **Mission Planner GitHub**  
   https://github.com/ArduPilot/MissionPlanner

## LiDAR

9. **SLAMTEC RPLIDAR C1 — Official**  
   https://www.slamtec.com/en/c1/

10. **SLAMTEC RPLIDAR C1 — Specifications**  
    https://www.slamtec.com/en/c1/spec

## Yahboom

11. **Yahboom MicroROS Control Board**  
    https://category.yahboom.net/products/microros-board

12. **Yahboom 4-Channel Encoder Motor Drive Module**  
    https://category.yahboom.net/products/quad-md-module

13. **Yahboom tutorials**  
    https://www.yahboom.net/

## ROS 2 / micro-ROS

14. **ROS 2**  
    https://docs.ros.org/

15. **micro-ROS**  
    https://micro.ros.org/

16. **Navigation2**  
    https://docs.nav2.org/

17. **Nav2 — Odometry / differential drive**  
    https://docs.nav2.org/rolling/configuration_and_development/first_time_robot_setup_guide/odom/setup_odom/

18. **BehaviorTree.CPP**  
    https://github.com/BehaviorTree/BehaviorTree.CPP

## Raspberry Pi

19. **Raspberry Pi 5**  
    https://www.raspberrypi.com/news/introducing-raspberry-pi-5/

20. **Raspberry Pi 5 Product Brief**  
    https://datasheets.raspberrypi.com/rpi5/raspberry-pi-5-product-brief.pdf

## Motor driver

21. **Cytron MDD10A**  
    https://www.cytron.io/c-dc-motor-drivers/p-10amp-5v-30v-dc-motor-driver-2-channels

## Cobertura agrícola

22. **Fields2Cover**  
    https://github.com/Fields2Cover/Fields2Cover

23. **OpenNav Coverage**  
    https://github.com/open-navigation/opennav_coverage

24. **Navigation2 GitHub**  
    https://github.com/ros-navigation/navigation2

---

# 25. Recursos de vídeo recomendados

- **ArduPilot YouTube**  
  https://www.youtube.com/@ArduPilot

- **Open Robotics YouTube**  
  https://www.youtube.com/@OpenRoboticsOrg

- Buscar demostraciones oficiales de:
  - Nav2;
  - SLAM Toolbox;
  - differential drive;
  - RPLIDAR C1;
  - Yahboom Raspberry Pi / ROS 2.

---

## Nota de ingeniería

Las especificaciones eléctricas procedentes de anuncios de terceros, especialmente motores MG513/MG540 y determinadas bases de orugas, deben verificarse con mediciones reales antes de seleccionar fusibles, drivers o límites térmicos. Las corrientes de bloqueo son especialmente importantes en vehículos de orugas, donde los giros pivotantes pueden producir corrientes cercanas al *stall*.

El prototipo debe ensayarse inicialmente a baja velocidad, sobre superficie despejada, con paro manual accesible y sin herramienta de corte instalada.

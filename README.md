# carrera-robots-autonomos

Carrera de robots autónomos.

## Objetivo del proyecto

Este proyecto es un **marco de trabajo para experimentar con código de conducción
autónoma**, en concreto con **algoritmos de navegación local y global**.

La idea es que cada participante escriba solo la "inteligencia" del robot: leer los
sensores, decidir y mover los actuadores. Todo lo demás (física, sensores, circuito
y visualización) lo pone el marco de trabajo. Así se pueden comparar algoritmos
distintos sobre el mismo robot y el mismo circuito en igualdad de condiciones.

## Arquitectura general

El sistema sigue un modelo **cliente-servidor**:

```
                        ┌──────────────────────────────┐
                        │           SERVIDOR           │
  ┌─────────────────┐   │  - Simulación física (2D)    │   ┌──────────────────┐
  │ Cliente control │◄──┤  - Simulación de sensores    ├──►│ Cliente web      │
  │ (algoritmo)     ├──►│  - Circuitos                 │   │ (espectador)     │
  └─────────────────┘   │  - Gestión de clientes       │   └──────────────────┘
  ┌─────────────────┐   │                              │   ┌──────────────────┐
  │ Cliente control │◄─►│                              ├──►│ Cliente web      │
  └─────────────────┘   └──────────────────────────────┘   └──────────────────┘

   telemetría ◄── servidor          servidor ──► estado de la carrera
   actuadores ──► servidor                       y telemetría (solo lectura)
```

### Servidor

- Es la **única fuente de verdad** de la simulación.
- Hace la **simulación de físicas** y el **comportamiento de los sensores** de
  todos los robots.
- Calcula las lecturas de cada sensor y las envía a su cliente **intentando
  llegar a la frecuencia objetivo de cada sensor** (cada tipo de sensor puede
  tener una frecuencia distinta).
- Aplica los valores de los actuadores que recibe de cada cliente en el siguiente
  paso de la simulación.
- Carga los circuitos y coloca los robots en la salida.

### Clientes de control

- Se conectan al servidor y **eligen qué modelo de robot van a usar**.
- Según el modelo elegido, el servidor les envía la **telemetría** de ese robot
  (las lecturas de sus sensores).
- El cliente ejecuta su algoritmo de navegación y devuelve los **valores de los
  actuadores** (motores, servos, etc.).
- El cliente no tiene acceso al estado interno de la simulación: solo "ve" lo que
  ven los sensores de su robot, igual que un robot real.

### Cliente web (espectador)

- Se conecta al servidor desde el navegador (la comunicación puede hacerse con
  **WebSockets**).
- Sirve para **ver lo que está pasando** en la simulación: circuito, robots y su
  posición.
- Puede **recibir los datos de los sensores** de los robots (por ejemplo, para
  mostrarlos en pantalla o para depurar).
- **No puede controlar** ningún robot: su conexión es de solo lectura.

## Motor de físicas

Puede usarse **cualquier motor de físicas**, ya que **basta con que sea 2D**: los
robots se mueven sobre un plano y los circuitos son dibujos en el suelo. Algunas
opciones válidas son Box2D, Chipmunk2D, Rapier (2D) o Matter.js, o un modelo
cinemático/dinámico propio si es suficiente.

## Modelos de robot

Los robots se ofrecen en orden de dificultad creciente. Cada modelo define sus
sensores (entradas del algoritmo) y sus actuadores (salidas del algoritmo).

### Robot 1: sigue líneas con 3 sensores infrarrojos

El robot más sencillo.

- **Sensores:** 3 sensores infrarrojos orientados al suelo, uno a la izquierda,
  otro en el centro y otro a la derecha.
- **Actuadores:** 2 motores independientes (rueda izquierda y rueda derecha),
  es decir, tracción diferencial.
- **Apoyo:** una rueda de bola trasera en el centro, sin tracción.

```
        vista superior (avance hacia arriba)

              I    C    D        ← sensores IR
              ●    ●    ●
          ┌─────────────────┐
          │                 │
       ███│                 │███  ← motores / ruedas
       ███│                 │███    izquierda y derecha
          │                 │
          │        ○        │   ← rueda de bola trasera
          └─────────────────┘
```

### Robot 2: sigue líneas con matriz de 5 sensores infrarrojos

Igual que el robot 1 (mismo chasis, mismos 2 motores y misma rueda de bola),
pero con una **matriz de 5 sensores infrarrojos** en la parte delantera. Al
tener más resolución lateral, permite estimar mejor cuánto se ha desviado el
robot de la línea y usar controles más finos (por ejemplo, PID).

```
           ●   ●   ●   ●   ●     ← matriz de 5 sensores IR
          ┌─────────────────┐
       ███│                 │███
       ███│                 │███
          │        ○        │
          └─────────────────┘
```

### Restricción de diseño de los sensores

**La distancia entre sensores infrarrojos contiguos debe ser menor que la anchura
de la línea que deben seguir.**

Así, la línea siempre queda bajo al menos un sensor mientras el robot esté sobre
ella, y nunca puede "colarse" entre dos sensores sin que ninguno la detecte. Por
ejemplo, con una línea de 19 mm de ancho, la separación entre sensores debería ser
de menos de 19 mm.

## Circuitos de ejemplo

### Circuito 1: óvalo ("O")

Circuito en **O** típico: **dos rectas y dos curvas**. Es el circuito de
iniciación para comprobar que el robot sigue la línea de forma estable.

```
      ╭──────────────────╮
     │                    │
     │                    │
      ╰──────────────────╯
```

### Circuito 2: ocho ("8")

Circuito en forma de **8**: **dos curvas y dos rectas que se cruzan entre sí**,
con un ángulo de cruce de **al menos 60°**.

El cruce es la principal dificultad: al pasar por él los sensores ven a la vez la
línea propia y la transversal, y el algoritmo debe seguir recto sin confundirse.
Un ángulo de cruce de 60° o más garantiza que las dos líneas se distinguen con
claridad en el punto de intersección.

```
      ╭───╮       ╭───╮
     │     ╲     ╱     │
     │      ╲   ╱      │
     │        ╳        │    ← cruce (≥ 60°)
     │      ╱   ╲      │
     │     ╱     ╲     │
      ╰───╯       ╰───╯
```

## Flujo de una sesión

1. El servidor arranca y carga un circuito.
2. Un cliente de control se conecta y elige un modelo de robot.
3. El servidor crea el robot en la salida y empieza a enviarle telemetría a la
   frecuencia objetivo de cada sensor.
4. El cliente responde con los valores de los actuadores.
5. El servidor aplica esos valores, avanza la simulación física y vuelve a
   calcular las lecturas de los sensores.
6. En paralelo, los clientes web espectadores reciben el estado de la carrera y,
   si lo desean, la telemetría de los robots.

## Licencia

Consulta el archivo [LICENSE](LICENSE).

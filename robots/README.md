# Definición de robots

Cada modelo de robot se define en un fichero **YAML** de este directorio. El
servidor los carga al arrancar, y la lista de modelos entre los que pueden
elegir los clientes sale de aquí. El motor de simulación usa los parámetros
físicos y la pasarela sirve las capas SVG al cliente web.

| Fichero | Modelo |
|---------|--------|
| [`sigue-lineas-3ir.yaml`](sigue-lineas-3ir.yaml) | Robot 1: sigue líneas con 3 sensores infrarrojos. |
| [`sigue-lineas-5ir.yaml`](sigue-lineas-5ir.yaml) | Robot 2: sigue líneas con una matriz de 5 sensores infrarrojos. |

## Unidades y coordenadas

- **Unidades del sistema internacional:** distancias en metros (m), velocidades
  en m/s, aceleraciones en m/s² y frecuencias en Hz.
- **Origen:** el **centro de gravedad** del robot es la coordenada `[0, 0]`.
- **Ejes:** `+x` apunta hacia el **frente** del robot y `+y` hacia su
  **izquierda**. Los ángulos, cuando se usen, crecen en sentido antihorario.

```
              +y (izquierda)
               ▲
               │
               │
     ──────────●──────────►  +x (frente)
               │ [0, 0] = centro de gravedad
               │
```

## Formato del fichero

| Campo | Obligatorio | Descripción |
|-------|-------------|-------------|
| `id` | Sí | Identificador único del modelo. Es el que usa el cliente para elegirlo. |
| `nombre` | Sí | Nombre legible del modelo. |
| `descripcion` | No | Descripción breve. |
| `cuerpo.radio_colision` | Sí | Radio (m) del círculo que envuelve al robot. Se usa para las colisiones entre robots y para la distancia de seguridad al entrar al mundo. |
| `actuadores` | Sí | Lista de actuadores (ver abajo). |
| `apoyos` | No | Elementos sin tracción, como una rueda de bola. No son entradas ni salidas del algoritmo; sirven para documentar el robot. |
| `sensores` | Sí | Lista de sensores (ver abajo). |
| `capas` | Sí | Lista de ficheros SVG que forman el dibujo del robot (ver [Capas SVG](#capas-svg)). |

### Actuadores de tipo `motor`

| Campo | Unidad | Descripción |
|-------|--------|-------------|
| `id` | | Identificador del actuador, único dentro del robot. Es la clave que usa el cliente para enviar su valor. |
| `tipo` | | `motor`. |
| `posicion` | m | `[x, y]` del punto de contacto de la rueda respecto al centro de gravedad. |
| `rango` | | Valores admitidos para el actuador. `[-1, 1]`: `1` es el máximo hacia delante, `-1` el máximo hacia atrás y `0` es parar. |
| `velocidad_max` | m/s | Velocidad lineal de la rueda con el actuador al máximo (`1`). |
| `aceleracion_max` | m/s² | Aceleración máxima para alcanzar una velocidad mayor. |
| `deceleracion_max` | m/s² | Deceleración máxima cuando se pide una velocidad menor (frenada activa). |
| `deceleracion_reposo` | m/s² | Deceleración cuando el motor está **desactivado**, por ejemplo tras perder la conexión: la rueda queda libre y frena solo por el rozamiento. |

La velocidad objetivo de la rueda es `valor × velocidad_max`. La simulación se
acerca a ella respetando `aceleracion_max` o `deceleracion_max`, lo que da la
inercia del motor.

### Sensores de tipo `infrarrojo`

| Campo | Unidad | Descripción |
|-------|--------|-------------|
| `id` | | Identificador del sensor, único dentro del robot. Es la clave con la que llega su valor al cliente. |
| `tipo` | | `infrarrojo`: sensor orientado al suelo que detecta la línea. |
| `posicion` | m | `[x, y]` del punto del suelo que mira el sensor, respecto al centro de gravedad. |
| `frecuencia_hz` | Hz | Frecuencia de muestreo. `10` para los infrarrojos de los robots de prácticas. |

## Capas SVG

El dibujo del robot en el frontal web se compone de **varias capas SVG
apiladas**, enlazadas en el campo `capas` con rutas relativas al fichero YAML.

- Se pintan **en orden, de la primera a la última**: las partes opacas de una
  capa tapan a las de las capas anteriores, y las transparentes las dejan ver.
- Así se puede **reutilizar un mismo cuerpo** y cambiar solo la capa de
  sensores, o la posición o composición de los sensores. Por ejemplo, los dos
  robots de prácticas comparten [`svg/cuerpo-diferencial.svg`](svg/cuerpo-diferencial.svg)
  y solo cambia la capa de sensores:
  [`svg/sensores-3ir.svg`](svg/sensores-3ir.svg) o
  [`svg/sensores-5ir.svg`](svg/sensores-5ir.svg).

Para que las capas encajen entre sí y con las coordenadas del YAML, todas siguen
el mismo convenio:

- **1 unidad del SVG = 1 mm.**
- El punto `(0, 0)` del SVG es el **centro de gravedad**.
- El frente del robot apunta a `+x`, es decir, **hacia la derecha** en el
  dibujo.
- Como en SVG el eje `y` crece hacia abajo, **la `y` del SVG es la `y` del robot
  cambiada de signo**: un sensor en `posicion: [0.070, 0.015]` (15 mm a la
  izquierda) se dibuja en `cx="70" cy="-15"`.
- Todas las capas usan el mismo `viewBox` (`-90 -90 180 180` en los robots de
  prácticas), para que se superpongan sin desplazamientos.

El frontal web coloca el conjunto de capas en la posición del robot y lo gira
según su orientación.

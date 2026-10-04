# Definición de robots

Cada modelo de robot se define en un fichero **YAML** de este directorio. La
pasarela los carga al arrancar. Los administradores eligen, al dar de alta cada
mundo, qué modelos se permiten en él, y los clientes eligen entre esos. La
pasarela sirve las capas SVG al visor web y entrega a cada motor de simulación
las definiciones de los robots permitidos en su mundo, que usa para la física.

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
| `cuerpo.radio_colision` | Sí | Radio (m) del círculo que envuelve al robot. Se usa para las colisiones entre robots y con las paredes, y para la distancia de seguridad al entrar al mundo. |
| `cuerpo.masa` | Sí | Masa del robot en kg. Decide quién empuja a quién en los choques. Los robots de prácticas tienen todos la misma (0,25 kg). |
| `cuerpo.momento_inercia` | No | Momento de inercia en kg·m² respecto al centro de gravedad. Si no se indica, se calcula como el de un disco uniforme: `masa × radio_colision² / 2`. |
| `cuerpo.rozamiento` | Sí | Coeficiente de rozamiento del cuerpo con otros robots y con las paredes. Hace que el robot se arrastre por una pared o que gire al rozarla. |
| `cuerpo.restitucion` | Sí | Coeficiente de restitución de los choques, de `0` (no rebota) a `1` (rebota sin perder energía). |
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
| `adherencia` | | Coeficiente de rozamiento de la rueda con el suelo. Limita la fuerza que puede transmitir la rueda antes de patinar, tanto hacia delante como de lado. |

La velocidad objetivo de la rueda es `valor × velocidad_max`. La simulación se
acerca a ella respetando `aceleracion_max` o `deceleracion_max`, lo que da la
inercia del motor.

La rueda empuja al robot para que el suelo, bajo ella, se mueva a esa velocidad,
y se resiste a deslizar de lado. Las dos fuerzas están limitadas por
`adherencia × peso sobre la rueda`. Mientras nada estorba, el robot se mueve
igual que con un modelo sin deslizamiento. Cuando otro robot lo empuja o choca
con una pared, la rueda puede **patinar**.

### Sensores de tipo `infrarrojo`

| Campo | Unidad | Descripción |
|-------|--------|-------------|
| `id` | | Identificador del sensor, único dentro del robot. Es la clave con la que llega su valor al cliente. |
| `tipo` | | `infrarrojo`: sensor orientado al suelo que detecta la línea. |
| `posicion` | m | `[x, y]` del punto del suelo que mira el sensor, respecto al centro de gravedad. |
| `frecuencia_hz` | Hz | Frecuencia de muestreo. `10` para los infrarrojos de los robots de prácticas. |

El sensor `infrarrojo` es **digital**: vale `1` si su punto de medida está sobre la línea (a menos de `ancho_linea / 2` del trazado) y `0` si no.

### Sensores de tipo `infrarrojo_promedio` (previsto)

> Todavía no está implementado. Se documenta para que los contratos y los clientes ya lo admitan.

Sensor **analógico** que imita mejor a un sensor real. Toma varias lecturas digitales repartidas por una pequeña zona de medida alrededor de su `posicion` y devuelve su **promedio**: un número entre `0` (ninguna lectura ve la línea) y `1` (todas la ven). Así se sabe, por ejemplo, si el sensor está justo en el borde de la línea, y los algoritmos pueden afinar más, por ejemplo con un PID que pondere por intensidad.

| Campo | Unidad | Descripción |
|-------|--------|-------------|
| `id`, `posicion`, `frecuencia_hz` | | Igual que en `infrarrojo`. |
| `tipo` | | `infrarrojo_promedio`. |
| `muestras` | | Número de lecturas que se promedian en cada muestreo. |
| `radio_medida` | m | Radio de la zona de medida donde se reparten las lecturas. |

Como los valores de los sensores siempre son números, un cliente escrito para el sensor digital funciona sin cambios con este, aunque no aproveche la información extra.

## Capas SVG

El dibujo del robot en el visor web se compone de **varias capas SVG
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

El visor web coloca el conjunto de capas en la posición del robot y lo gira
según su orientación.

# Definición de circuitos

Cada circuito se define en un fichero **YAML** de este directorio. La pasarela los carga al arrancar y los entrega:

- al **motor de simulación**, en la respuesta a su petición de configuración, para calcular las lecturas de los sensores;
- al **visor web** (`GET /circuitos/<id>`), para dibujar el circuito en la vista de administrador.

Cada motor indica su circuito con la variable `CIRCUITO`, que es el `id` del fichero.

| Fichero | Circuito |
|---------|----------|
| [`ovalo.yaml`](ovalo.yaml) | Circuito 1: óvalo, con dos rectas y dos curvas. |
| [`ocho.yaml`](ocho.yaml) | Circuito 2: ocho, con dos rectas que se cruzan a 70°. |

## Unidades y coordenadas

- **Distancias en metros** y **ángulos en grados**.
- **Origen:** la esquina inferior izquierda del mapa es `[0, 0]`.
- **Ejes:** `+x` hacia la derecha y `+y` hacia arriba. Los ángulos se miden desde `+x` y crecen en sentido antihorario, como en el [formato de los robots](../robots/README.md#unidades-y-coordenadas).

Al dibujar en SVG, donde el eje `y` crece hacia abajo, hay que cambiar el signo de la `y` (o usar `transform="scale(1, -1)"`), igual que con las capas de los robots.

## Formato del fichero

| Campo | Obligatorio | Descripción |
|-------|-------------|-------------|
| `id` | Sí | Identificador único del circuito. Es el valor de `CIRCUITO` en el motor. |
| `nombre` | Sí | Nombre legible. |
| `descripcion` | No | Descripción breve. |
| `dimensiones` | Sí | `[ancho, alto]` del mapa en metros. Sus bordes son paredes. |
| `ancho_linea` | Sí | Anchura de la línea en metros. `0.025` (25 mm) para los robots de prácticas: ver la [restricción de diseño de los sensores](../README.md#restricción-de-diseño-de-los-sensores). |
| `trazados` | Sí | Lista de trazados. Cada uno es una secuencia continua de tramos. |
| `paredes.rozamiento` | Sí | Coeficiente de rozamiento de las paredes. Con el del robot, decide cuánto se frena y cuánto gira al rozarlas. |
| `paredes.restitucion` | Sí | Coeficiente de restitución de las paredes, de `0` (no rebota) a `1`. |

### Paredes

Los cuatro bordes del mapa son **paredes** rígidas. Un robot que llega a una
no la atraviesa: según el ángulo y la velocidad con que llega, **rebota**, se
**arrastra** a lo largo de ella o **gira** por el rozamiento, como en un choque
real. En cada contacto se combinan los coeficientes de la pared y del robot
(ver [`robots/README.md`](../robots/README.md#formato-del-fichero)).

### Trazados

| Campo | Descripción |
|-------|-------------|
| `id` | Identificador del trazado dentro del circuito. |
| `cerrado` | `true` si el final del último tramo coincide con el principio del primero. |
| `tramos` | Lista de tramos, cada uno empezando donde acaba el anterior. |

### Tramos

| Tipo | Campos | Descripción |
|------|--------|-------------|
| `recta` | `desde`, `hasta` | Segmento entre dos puntos `[x, y]`. |
| `arco` | `centro`, `radio`, `inicio`, `fin` | Arco de circunferencia desde el ángulo `inicio` hasta el ángulo `fin`. Si `fin > inicio`, se recorre en sentido **antihorario**; si `fin < inicio`, en sentido **horario**. |

Con rectas y arcos, la distancia de un punto a la línea se calcula de forma exacta. Un sensor infrarrojo ve la línea cuando su punto de medida está a menos de `ancho_linea / 2` del trazado más cercano.

Los **cruces** no necesitan ningún campo especial: aparecen solos cuando dos tramos se cortan, como las dos rectas del ocho.

## Reglas para diseñar circuitos

- La separación entre dos partes del trazado que no se cruzan debe ser mucho mayor que la anchura del robot, para que sus sensores no vean dos líneas a la vez fuera de los cruces.
- Los cruces deben formar un ángulo de **al menos 60°**, para que las dos líneas se distingan con claridad.
- Debe quedar margen entre el trazado y el borde del mapa, para que los robots puedan salirse un poco de la línea y volver a ella.
- El radio de las curvas debe permitir que el robot las tome sin perder la línea a su velocidad máxima.

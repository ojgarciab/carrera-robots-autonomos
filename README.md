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
  │ (algoritmo)     ├──►│  - Circuitos                 │   │ (administrador)  │
  └─────────────────┘   │  - Autenticación y usuarios  │   └──────────────────┘
  ┌─────────────────┐   │  - API WebSocket y REST      │   ┌──────────────────┐
  │ Cliente control │◄─►│                              ├──►│ Cliente web      │
  └─────────────────┘   └──────────────────────────────┘   │ (usuario)        │
                                                           └──────────────────┘

   telemetría ◄── servidor          servidor ──► administrador: posición y
   actuadores ──► servidor                       orientación exactas de todos
                                                 usuario: sensores de su robot
                                                 (ambos solo lectura)
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
- Carga los circuitos y coloca los robots en el mundo (ver
  [Ciclo de vida del robot en el mundo](#ciclo-de-vida-del-robot-en-el-mundo)).
- Limita el **número máximo de robots** simultáneos con un parámetro de
  configuración (ver [Configuración](#configuración)).
- Internamente se divide en una pasarela de API y un motor de simulación (ver
  [Arquitectura del servidor](#arquitectura-del-servidor)).

### Clientes de control

- Se **autentican** en el servidor y se conectan a través de la
  [API de cliente](#api-de-cliente).
- **Eligen qué modelo de robot van a usar**.
- Según el modelo elegido, el servidor les envía la **telemetría** de ese robot
  (las lecturas de sus sensores).
- El cliente ejecuta su algoritmo de navegación y devuelve los **valores de los
  actuadores** (motores, servos, etc.).
- El cliente no tiene acceso al estado interno de la simulación: solo "ve" lo que
  ven los sensores de su robot, igual que un robot real.

### Cliente web

Se conecta al servidor desde el navegador (la comunicación puede hacerse con
**WebSockets**) y **no puede controlar** ningún robot: su conexión es de solo
lectura. Tiene **dos niveles de acceso**:

| Nivel | Qué puede ver | Uso típico |
|-------|---------------|------------|
| **Administrador** | El circuito y **todos** los robots con sus **coordenadas y orientación exactas**. Sobre cada robot se muestra una **etiqueta con el nombre del usuario** al que pertenece. | Proyectar la prueba o la carrera en televisiones o pantallas grandes. |
| **Usuario** | **Solo los valores de los sensores de su propio robot**. No ve la posición real ni los robots de los demás. | Depurar su algoritmo viendo lo mismo que "ve" su robot. |

Así se mantiene la regla de que un participante solo dispone de la información
que le dan los sensores de su robot, mientras que el administrador tiene la
vista completa de la carrera.

## API de cliente

Los clientes de control usan una API autenticada:

1. **Autenticación.** El cliente se identifica ante el servidor y obtiene una
   credencial de sesión. El cliente elige si la quiere como **token** (por
   ejemplo, en la cabecera `Authorization`) o como **cookie**.
2. **Datos en tiempo real.** Con esa credencial el cliente recibe la telemetría y
   envía los valores de los actuadores por uno de estos dos medios, a su
   elección:
   - **WebSocket:** una conexión persistente a un endpoint WebSocket. El servidor
     *empuja* las lecturas de los sensores en cuanto se generan y el cliente
     envía los valores de los actuadores por la misma conexión.
   - **Polling REST:** el cliente consulta periódicamente un endpoint REST para
     leer los últimos valores de los sensores y envía los valores de los
     actuadores con otra petición REST.

### Lectura de sensores por polling

- Por defecto, una petición de polling **responde inmediatamente con el último
  valor obtenido** de cada sensor, aunque el cliente ya lo hubiera leído.
- Si el cliente marca una **opción de espera** en la petición, la conexión
  **queda en espera hasta que se produzca el siguiente muestreo** del sensor y
  entonces responde con ese valor nuevo (*long polling*). Así el cliente puede
  sincronizar su bucle de control con los 10 Hz de los sensores sin tener que
  consultar más a menudo de lo necesario y sin leer valores repetidos.

### Marcas de tiempo y latencia

Cada dato de sensor se entrega al cliente con una **marca de tiempo** del
momento en que se tomó la muestra. La marca usa el reloj del servidor y tiene una
**precisión de al menos milisegundos**. Lo mismo vale para WebSocket y para
polling.

Con estas marcas el cliente puede:

- Calcular el **delta de tiempo** entre dos muestras seguidas (unos 100 ms para
  los infrarrojos) y usarlo en su control, por ejemplo en el término derivativo
  o integral de un PID.
- Detectar si se ha perdido o retrasado alguna muestra.
- Estimar la **latencia de la conexión**.

Para medir la latencia hay un **comando `ping`**, disponible tanto como
**endpoint REST** como **comando WebSocket**. El servidor responde al instante
con su **marca de tiempo actual**. Así el cliente puede:

1. Anotar su hora local `t0` al enviar el `ping` y `t1` al recibir la respuesta
   con la marca del servidor `ts`.
2. Obtener el **tiempo de ida y vuelta**: `rtt = t1 - t0`.
3. Estimar la **diferencia entre su reloj y el del servidor**:
   `desfase ≈ ts - (t0 + t1) / 2`.
4. Con ese desfase, calcular cuánto tarda en llegar cada dato de sensor: la hora
   local de llegada menos (marca del sensor - desfase).

El modo de conexión influye en cómo se detecta la desconexión del cliente (ver
[Desconexión](#desconexión)).

## Arquitectura del servidor

Cada robot supone carga de proceso para el servidor: física, cálculo de los
sensores, envío de telemetría y gestión de su conexión. Para que el servidor
escale y la latencia sea baja, se propone separarlo en dos componentes que se
comunican por un **bus de mensajes**.

```
  clientes de control                        clientes web
  (WebSocket / REST)                         (admin / usuario)
          │                                         │
          ▼                                         ▼
  ┌──────────────────────────────────────────────────────────┐
  │            PASARELA DE API (una o varias réplicas)       │
  │  - Autenticación (token / cookie)                        │
  │  - Conexiones WebSocket y endpoints REST                 │
  │  - Caché del último valor de cada sensor (polling)       │
  │  - Long polling, ping y filtrado por nivel de acceso     │
  │  - Sirve los ficheros del cliente web                    │◄──► BASE DE
  └───────────────┬─────────────────────────▲────────────────┘     DATOS
     actuadores,  │                         │  sensores,
     actividad,   │      BUS DE MENSAJES    │  estado del mundo,
     altas/bajas  ▼   (NATS / RabbitMQ …)   │  eventos
  ┌──────────────────────────────────────────────────────────┐
  │          MOTOR DE SIMULACIÓN (uno por mundo/circuito)    │
  │  - Bucle de física 2D a paso fijo                        │
  │  - Muestreo de sensores a su frecuencia (10 Hz IR)       │
  │  - Inercia, aceleración y deceleración de los motores    │
  │  - Ciclo de vida: entrada, temporizadores, salida        │
  │  - Límite de número máximo de robots                     │
  └──────────────────────────────────────────────────────────┘
```

### Reparto de responsabilidades

- **Pasarela de API.** Se encarga de todo lo que depende de la red y de los
  clientes: autenticación, conexiones, serialización, caché para el polling,
  long polling, `ping` y control de qué puede ver cada nivel de acceso. No
  simula nada, así que se pueden levantar varias réplicas detrás de un
  balanceador si hay muchos clientes.
- **Base de datos.** Guarda los **usuarios**, sus credenciales y su **rol**
  (administrador o usuario), además del historial de carreras. Solo la usa la
  pasarela y queda **fuera del camino de los datos en tiempo real**: los
  sensores y los actuadores nunca pasan por ella.
- **Motor de simulación.** Solo hace cálculo: avanza la física, muestrea los
  sensores y aplica los actuadores. No sabe nada de HTTP ni de WebSockets, por
  lo que su bucle no se ve afectado por clientes lentos o por picos de
  conexiones. Es la única fuente de verdad del mundo y también lleva los
  temporizadores del ciclo de vida (5 s y 5 minutos) y el límite de robots,
  para que no dependan de qué réplica de la pasarela atienda al cliente.

### Mensajes entre componentes

| Tema (ejemplo) | Sentido | Contenido |
|----------------|---------|-----------|
| `robot.<id>.sensores` | simulación → pasarela | Lecturas de sensores con su marca de tiempo. |
| `robot.<id>.actuadores` | pasarela → simulación | Nueva consigna de los motores. |
| `robot.<id>.actividad` | pasarela → simulación | Aviso de que el cliente sigue vivo (lectura de sensores o `ping` por polling, conexión o desconexión WebSocket). Se puede agrupar, por ejemplo uno por segundo como máximo. |
| `mundo.<id>.control` | pasarela → simulación | Peticiones de entrada y salida de robots, con respuesta (aceptado o lleno). |
| `mundo.<id>.estado` | simulación → pasarela | Posición y orientación exactas de todos los robots, solo para la vista de administrador. |
| `mundo.<id>.eventos` | simulación → pasarela | Robot que entra, sale o pierde los actuadores. |

Todos estos mensajes son de tipo **"vale el último"**: una lectura de sensor o
una consigna de motor antigua no sirve de nada si ya hay una más nueva. Por eso:

- **No hace falta persistencia ni confirmaciones** (*acks*). Los mensajes pueden
  ser transitorios y en memoria.
- Las colas deben ser **cortas** (por ejemplo, de longitud 1 o con un tiempo de
  vida pequeño) y **descartar los mensajes viejos**, en vez de acumularlos y
  entregarlos con retraso.
- Conviene un **formato binario compacto** (MessagePack, Protocol Buffers o
  FlatBuffers) en lugar de JSON entre componentes internos.

### Elección del bus de mensajes

RabbitMQ sirve, pero está pensado sobre todo para entregar mensajes de forma
fiable, con persistencia, confirmaciones y enrutado complejo. Aquí no hace
falta nada de eso y añade un salto por el broker. Para minimizar la latencia
son preferibles alternativas más ligeras:

| Opción | Ventajas | Inconvenientes |
|--------|----------|----------------|
| **NATS** (recomendada) | Publicación/suscripción muy ligera, latencias típicamente por debajo de 1 ms en red local, temas jerárquicos (`robot.*.sensores`), petición-respuesta incorporada. | Sin persistencia en el modo básico, que aquí no hace falta. |
| **ZeroMQ** | Sin broker: los componentes se conectan directamente, con la menor latencia posible. | Hay que gestionar a mano el descubrimiento de servicios y las reconexiones. |
| **Redis Pub/Sub** | Sencillo, y Redis puede servir además como caché del último valor de cada sensor. | Si un suscriptor va lento, Redis puede acabar desconectándolo. |
| **RabbitMQ** | Muy conocido, con buenas herramientas de gestión. | Más pesado. Para no penalizar la latencia, hay que usar mensajes no persistentes, colas no durables, `auto-ack` y colas con longitud máxima. |

### Presupuesto de latencia

Los sensores infrarrojos se muestrean cada 100 ms. Por tanto, lo importante es
que el tiempo añadido por el servidor sea pequeño comparado con ese periodo:

- **Paso de física:** el motor avanza a paso fijo, por defecto a **60 Hz**
  (unos 16,7 ms por paso), independientemente de la frecuencia de los sensores.
  Una consigna de motor se aplica como mucho un paso después de llegar, es decir,
  con menos de 17 ms de retraso. Además, 60 Hz encaja con los sensores: cada
  muestra de los infrarrojos (100 ms) cae exactamente cada 6 pasos, y el estado
  del mundo para el administrador (30 Hz) cada 2 pasos. Si la máquina lo
  permite, se puede subir la frecuencia para ganar precisión.
- **Bus de mensajes:** con NATS o ZeroMQ en la misma máquina o red local, cada
  salto añade típicamente bastante menos de 1 ms. Con RabbitMQ bien configurado
  es algo más, pero sigue siendo pequeño frente a 100 ms.
- **Red hasta el cliente:** suele ser la mayor parte de la latencia total y no
  depende de la arquitectura interna.

### Relojes y marcas de tiempo

La **marca de tiempo de cada muestra la pone el motor de simulación** en el
momento de tomarla. El `ping`, en cambio, lo responde la pasarela. Para que el
cliente pueda combinar ambos valores, todas las máquinas del servidor deben
tener los **relojes sincronizados** (NTP o, si se quiere más precisión, PTP).
En el despliegue local con Docker no hace falta hacer nada: todos los
contenedores comparten el reloj de la máquina anfitriona.

### Evolución por fases

La parte de servidor se diseña desde el principio para correr en **Docker**,
con cada componente en su propio contenedor (ver
[Despliegue local con Docker](#despliegue-local-con-docker)):

1. **Docker local:** un contenedor de pasarela, uno de simulación por mundo,
   el bus NATS y la base de datos, todos en la misma máquina. Es el despliegue
   de referencia.
2. **Varias máquinas:** varias réplicas de la pasarela detrás de un balanceador
   y motores de simulación repartidos entre máquinas.

La clave es definir la interfaz entre la pasarela y la simulación como
**mensajes**, sin depender del transporte. Así las pruebas automáticas pueden
usar un transporte en memoria, sin NATS, y cambiar de bus más adelante no obliga
a reescribir nada.

## Configuración

Parámetros principales del servidor:

| Parámetro | Descripción |
|-----------|-------------|
| Número máximo de robots | Límite de robots simultáneos en el mundo; **4 por defecto**. Cada robot consume CPU (física, sensores y telemetría), así que este valor debe ajustarse a la capacidad de la máquina. Si se alcanza, las nuevas entradas se rechazan. |
| Frecuencia de la física | Pasos de simulación por segundo; **60 Hz por defecto**. Cuanto más alta, más precisa es la simulación y más CPU consume. |

Los valores por defecto (60 Hz y 4 robots) están pensados para **no sobrecargar
la máquina** y bastan para los robots de prácticas, que son lentos (20 cm/s).
Para **robots veloces**, por ejemplo en competiciones, se puede **subir la
frecuencia de la física** para mantener la precisión, siempre que haya CPU de
sobra.

En Docker estos parámetros se pasan como variables de entorno; la lista
completa está en [Despliegue local con Docker](#despliegue-local-con-docker).

## Despliegue local con Docker

El repositorio incluye un fichero de referencia,
[`compose.yaml`](compose.yaml), para levantar **toda la parte de servidor** en
Docker local.

**Los clientes no forman parte del despliegue:**

- El **cliente web** es siempre un navegador. La pasarela sirve sus ficheros
  y el navegador se conecta a ella.
- Los **clientes de control** corren de forma independiente, en la máquina y el
  lenguaje que quiera cada participante, y se conectan a la pasarela por
  WebSocket o REST.

### Servicios

| Servicio | Imagen | Puertos expuestos | Función |
|----------|--------|-------------------|---------|
| `pasarela` | Se construye desde `./pasarela` | `8080` (configurable) | API REST, WebSocket, `ping`, autenticación y cliente web. Es el único servicio accesible desde fuera. |
| `simulacion-ovalo` | Se construye desde `./simulacion` | Ninguno | Motor de simulación del mundo con el circuito en O. Tiene CPU reservada para que el bucle de física no compita con el resto de servicios. |
| `simulacion-ocho` | Se construye desde `./simulacion` | Ninguno | Segundo mundo con el circuito en 8. Viene comentado; se activa descomentándolo. |
| `bus` | `nats:2-alpine` | Ninguno (`8222` para monitorización, comentado) | Bus de mensajes sin persistencia entre la pasarela y la simulación. |
| `bd` | `postgres:17-alpine` | Ninguno | Base de datos de usuarios, roles e historial, con un volumen persistente (`datos-bd`). |

Todos los servicios comparten una red interna. Solo la pasarela publica un
puerto en la máquina anfitriona. El directorio [`robots/`](robots/) se monta en
modo de solo lectura en la pasarela, que sirve las capas SVG y la lista de
modelos, y en la simulación, que usa los parámetros físicos. Así se pueden
añadir o cambiar modelos sin reconstruir las imágenes. La pasarela y la simulación esperan a que el
bus y la base de datos estén sanos (*healthcheck*) antes de arrancar.

Los directorios `./pasarela` y `./simulacion`, con su `Dockerfile`, se crearán
al implementar cada componente. Mientras no existan, se pueden levantar solo
la infraestructura: `docker compose up -d bus bd`.

### Puesta en marcha

```sh
cp .env.example .env          # ajustar al menos POSTGRES_PASSWORD
docker compose up -d --build  # construye y arranca todos los servicios
docker compose logs -f        # ver los registros
docker compose down           # parar (añadir -v para borrar la base de datos)
```

Después, abrir `http://localhost:8080` en el navegador y conectar los clientes
de control a esa misma dirección.

### Variables de entorno

Se definen en el fichero `.env`; [`.env.example`](.env.example) sirve de
plantilla.

| Variable | Por defecto | Servicio | Descripción |
|----------|-------------|----------|-------------|
| `POSTGRES_USER` | `carrera` | `bd`, `pasarela` | Usuario de la base de datos. |
| `POSTGRES_PASSWORD` | *(obligatoria)* | `bd`, `pasarela` | Contraseña de la base de datos. |
| `POSTGRES_DB` | `carrera` | `bd`, `pasarela` | Nombre de la base de datos. |
| `PASARELA_PUERTO` | `8080` | `pasarela` | Puerto de la máquina anfitriona donde se publica la pasarela. |
| `LONG_POLLING_TIMEOUT_MS` | `1000` | `pasarela` | Espera máxima de una petición de long polling. |
| `SESION_TTL` | `12h` | `pasarela` | Duración de los tokens de sesión. |
| `MAX_ROBOTS` | `4` | simulación | Número máximo de robots simultáneos en cada mundo. |
| `PASO_FISICA_HZ` | `60` | simulación | Frecuencia del paso fijo de la física. |
| `ESTADO_MUNDO_HZ` | `30` | simulación | Frecuencia con la que se publica el estado del mundo para la vista de administrador. |
| `PARADA_MOTORES_S` | `5` | simulación | Segundos sin instrucciones antes de desconectar los motores (polling). |
| `SALIDA_MUNDO_S` | `300` | simulación | Segundos sin actividad antes de que el robot salga del mundo. |

## Motor de físicas

Puede usarse **cualquier motor de físicas**, ya que **basta con que sea 2D**: los
robots se mueven sobre un plano y los circuitos son dibujos en el suelo. Algunas
opciones válidas son Box2D, Chipmunk2D, Rapier (2D) o Matter.js, o un modelo
cinemático/dinámico propio si es suficiente.

Los robots de prácticas son **lentos** (20 cm/s como máximo), así que
prácticamente **no derrapan ni pierden el control** aunque giren a máxima
velocidad. Para ellos basta un **modelo cinemático de tracción diferencial**
sin deslizamiento de las ruedas, más los límites de aceleración y deceleración
de cada motor. Los robots veloces, por ejemplo de competición, pueden necesitar
un modelo dinámico con rozamiento y derrape, además de una frecuencia de física
mayor.

## Modelos de robot

Los robots se ofrecen en orden de dificultad creciente. Cada modelo define sus
sensores (entradas del algoritmo) y sus actuadores (salidas del algoritmo).

Cada modelo se describe en un **fichero YAML** del directorio
[`robots/`](robots/) (ver [Definición de los robots](#definición-de-los-robots)).
Los dos robots de prácticas tienen estas características comunes:

| Característica | Valor |
|----------------|-------|
| Velocidad máxima | **20 cm/s** (0,20 m/s) con los actuadores al máximo (`1`). |
| Rango de los actuadores | De `-1` (máximo hacia atrás) a `1` (máximo hacia delante). |
| Frecuencia de los sensores IR | 10 Hz. |
| Separación entre sensores IR | 15 mm (3 sensores) y 12 mm (5 sensores). |

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
ella, y nunca puede "colarse" entre dos sensores sin que ninguno la detecte.

Hay una segunda condición: **lo que avanza el robot entre dos lecturas de los
sensores también debe ser menor que la anchura de la línea**. Si no, al cruzar
la línea de frente podría atravesarla entre dos lecturas sin verla:

```
velocidad máxima × periodo de muestreo < anchura de la línea
```

Para los robots de prácticas, 0,20 m/s × 0,1 s = **20 mm** por lectura. Por eso
se propone una **línea de 25 mm de ancho**, que cumple las dos condiciones:

| Robot | Separación entre sensores | Avance por lectura | Anchura de la línea |
|-------|---------------------------|--------------------|---------------------|
| 3 sensores IR | 15 mm | 20 mm | 25 mm |
| 5 sensores IR | 12 mm | 20 mm | 25 mm |

### Comportamiento de sensores y actuadores

- **Sensores infrarrojos:** cada sensor genera **un valor cada 0,1 s (10 lecturas
  por segundo)**. El servidor intenta mantener esa frecuencia al enviar la
  telemetría.
- **Motores:** el servidor aplica la nueva consigna de los motores **en cuanto
  la recibe**, sin esperar al siguiente ciclo de sensores. Pero la velocidad
  real de la rueda no cambia al instante: la simulación tiene en cuenta la
  **inercia** y limita la **aceleración** y la **deceleración** del motor. Por
  eso el robot tarda un tiempo en alcanzar la velocidad pedida o en frenar, y
  el algoritmo de control tiene que tenerlo en cuenta.
- **Valor de los actuadores:** cada motor recibe un valor de `-1` a `1`. La
  velocidad que se busca para la rueda es ese valor por la velocidad máxima del
  motor: 20 cm/s con el valor `1` en los robots de prácticas.
- **Motores desactivados:** cuando se desconectan los actuadores, la rueda
  queda libre y frena con la **deceleración en reposo** del motor, más suave que
  la frenada activa.

### Definición de los robots

Los parámetros de cada robot están en un fichero YAML del directorio
[`robots/`](robots/). El formato completo se explica en
[`robots/README.md`](robots/README.md). Incluye, entre otros:

- Las **coordenadas de cada motor y de cada sensor** respecto al **centro de
  gravedad**, que es la coordenada `[0, 0]`. El eje `+x` apunta al frente del
  robot y el `+y` a su izquierda.
- La **velocidad máxima** (m/s), la **aceleración y la deceleración máximas**
  (m/s²) y la **deceleración en reposo** (m/s²) de cada motor.
- La frecuencia de muestreo de cada sensor.
- El **dibujo del robot** como una lista de **capas SVG** enlazadas desde el
  YAML. El frontal web las pinta **de la primera a la última**, así que las
  partes opacas de la última capa tapan a las de debajo. Esto permite
  **compartir el cuerpo del robot** y cambiar solo la capa con la posición o la
  composición de los sensores. Los dos robots de prácticas usan el mismo
  `cuerpo-diferencial.svg`, con `sensores-3ir.svg` o con `sensores-5ir.svg`
  encima.

```yaml
# Extracto de robots/sigue-lineas-3ir.yaml
actuadores:
  - id: motor_izquierdo
    tipo: motor
    posicion: [0.020, 0.055]     # m, respecto al centro de gravedad
    rango: [-1, 1]
    velocidad_max: 0.20          # m/s con el actuador a 1
    aceleracion_max: 0.5         # m/s²
    deceleracion_max: 0.8        # m/s²
    deceleracion_reposo: 0.3     # m/s², con el motor desactivado
sensores:
  - id: ir_central
    tipo: infrarrojo
    posicion: [0.070, 0.0]
    frecuencia_hz: 10
capas:                           # de abajo arriba
  - svg/cuerpo-diferencial.svg
  - svg/sensores-3ir.svg
```

## Ciclo de vida del robot en el mundo

### Entrada al mundo

Cuando un cliente se conecta con su robot, este **entra al mundo** así:

- Solo entra si queda sitio: si ya está el **número máximo de robots**
  configurado, el servidor rechaza la entrada con un error que lo indica, y el
  cliente puede volver a intentarlo más tarde.

- Aparece en un **punto aleatorio del mapa** que **no esté ocupado por otro
  robot**, respetando una **distancia de seguridad** con todos los demás.
- Aparece **orientado hacia el centro del mapa**. De este modo, si el robot
  avanza en línea recta, **siempre acabará cruzándose con la línea** en algún
  punto, y el algoritmo tiene que encontrarla y engancharse a ella.

### Desconexión

Si el cliente deja de comunicarse con el servidor, los **actuadores se
desconectan**: los motores dejan de recibir consigna y el robot **va decelerando
hasta parar**, según la deceleración simulada. Si el cliente no vuelve en
**5 minutos**, el robot **sale del mundo**.

Cómo se detecta la desconexión depende del modo de conexión:

| Modo | Se desconectan los actuadores… | Sale del mundo… |
|------|--------------------------------|-----------------|
| **WebSocket** | En cuanto se cierra o se pierde la conexión WebSocket. | A los **5 minutos** sin reconectar. |
| **Polling REST** | Tras **5 segundos** sin recibir instrucciones para los motores. | A los **5 minutos** sin ninguna petición a la API (ni instrucciones de motores, ni lecturas de sensores, ni `ping`). |

Con polling, la parada de los motores y la vida del robot se cuentan por
separado. Los motores solo se mantienen vivos con instrucciones de motores. La
vida del robot **se extiende con cualquier petición a la API**: basta con
consultar los datos de sus sensores o hacer un `ping`. Por eso un cliente puede tener el robot
parado y seguir leyendo sus sensores sin que salga del mundo.

Si el cliente vuelve antes de que pasen los 5 minutos, **recupera el control del
robot allí donde esté en ese momento**: puede seguir frenando por la inercia o
estar ya parado.

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
2. Un cliente de control se autentica (con token o cookie) y elige un modelo de
   robot.
3. Si no se ha alcanzado el número máximo de robots, el servidor coloca el robot
   en un punto libre y aleatorio del mapa, orientado hacia el centro. Si se ha
   alcanzado, rechaza la entrada.
4. El servidor genera la telemetría a la frecuencia objetivo de cada sensor
   (10 Hz para los infrarrojos), con una marca de tiempo de precisión de
   milisegundos en cada muestra. La envía por WebSocket o la deja disponible
   para polling REST, de forma inmediata o esperando al siguiente muestreo. El
   cliente puede usar el comando `ping` para medir la latencia.
5. El cliente responde con los valores de los actuadores, que el servidor aplica
   en cuanto los recibe, con la inercia, la aceleración y la deceleración de los
   motores.
6. El servidor avanza la simulación física y vuelve a calcular las lecturas de
   los sensores.
7. A la vez, la pantalla de administración muestra todos los robots con su
   posición, su orientación y el nombre de su usuario, y cada usuario ve en su
   cliente web los sensores de su propio robot.
8. Si el cliente se desconecta, o deja de mandar instrucciones de motores durante
   5 s por polling, el robot frena hasta parar. Si pasan 5 minutos sin volver, o
   sin ninguna petición a la API en el caso de polling, sale del mundo. Si
   vuelve antes, recupera el control allí donde esté el robot.

## Licencia

Consulta el archivo [LICENSE](LICENSE).

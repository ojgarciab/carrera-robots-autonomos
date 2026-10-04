# carrera-robots-autonomos

Carrera de robots autónomos.

## Objetivo del proyecto

Este proyecto es un **marco de trabajo para experimentar con código de conducción
autónoma**, en concreto con **algoritmos de navegación local y global**.

La idea es que cada participante escriba solo la "inteligencia" del robot: leer los
sensores, decidir y mover los actuadores. Todo lo demás (física, sensores, circuito
y visualización) lo pone el marco de trabajo. Así se pueden comparar algoritmos
distintos sobre el mismo robot y el mismo circuito en igualdad de condiciones.

## Repositorios

El código de cada componente vive en su propio repositorio:

| Repositorio | Componente |
|-------------|------------|
| [`carrera-robots-autonomos`](https://github.com/ojgarciab/carrera-robots-autonomos) (este) | Repositorio común: documentación, contratos entre componentes, definiciones de robots y circuitos, despliegue de referencia (`compose.yaml`) y configuración de NATS. |
| [`robot-2d-pasarela`](https://github.com/ghCreaR/robot-2d-pasarela) | [Pasarela de API](#reparto-de-responsabilidades) en tiempo real: REST, WebSocket, validación de tokens y autenticación de los motores en NATS. |
| [`robot-2d-motor-fisicas`](https://github.com/ghCreaR/robot-2d-motor-fisicas) | [Motor de simulación](#reparto-de-responsabilidades) de cada mundo: física 2D, sensores y actuadores. |
| [`robot-2d-interfaz-web`](https://github.com/ghCreaR/robot-2d-interfaz-web) | [Interfaz de gestión](#interfaz-de-gestión): usuarios, mundos, accesos y tokens de API. Es la dueña del esquema de la base de datos. |
| [`robot-2d-visor-web`](https://github.com/ghCreaR/robot-2d-visor-web) | [Visor web](#visor-web): ficheros estáticos para ver la carrera y los sensores de los robots. |
| [`robot-2d-cliente-referencia-python`](https://github.com/ghCreaR/robot-2d-cliente-referencia-python) | Cliente de control de referencia en Python para los sigue líneas. |

## Arquitectura general

El sistema sigue un modelo **cliente-servidor**:

```
                        ┌──────────────────────────────┐
                        │           SERVIDOR           │
  ┌─────────────────┐   │  - Simulación física (2D)    │   ┌──────────────────┐
  │ Cliente control │◄──┤  - Simulación de sensores    ├──►│ Visor web        │
  │ (algoritmo)     ├──►│  - Circuitos                 │   │ (administrador)  │
  └─────────────────┘   │  - Autenticación y usuarios  │   └──────────────────┘
  ┌─────────────────┐   │  - API WebSocket y REST      │   ┌──────────────────┐
  │ Cliente control │◄─►│                              ├──►│ Visor web        │
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
- Internamente se divide en una pasarela de API y uno o varios motores de
  simulación, uno por **mundo** (ver
  [Arquitectura del servidor](#arquitectura-del-servidor)). Los mundos los dan de
  alta los administradores (ver [Registro de mundos](#registro-de-mundos)).

### Clientes de control

- Se conectan a través de la [API de cliente](#api-de-cliente) con un **token
  de API de lectura-escritura** que el usuario genera en la interfaz de gestión (ver
  [Tokens de API](#tokens-de-api)).
- **Eligen el mundo**, de entre los servidores activos, y **qué modelo de robot
  van a usar**, de entre los permitidos en ese mundo.
- Según el modelo elegido, el servidor les envía la **telemetría** de ese robot
  (las lecturas de sus sensores).
- El cliente ejecuta su algoritmo de navegación y devuelve los **valores de los
  actuadores** (motores, servos, etc.).
- El cliente no tiene acceso al estado interno de la simulación: solo "ve" lo que
  ven los sensores de su robot, igual que un robot real.

### Visor web

El visor web ([`robot-2d-visor-web`](https://github.com/ghCreaR/robot-2d-visor-web))
son **solo ficheros estáticos** (HTML, CSS y JavaScript): se pueden servir desde
cualquier servidor web y se conectan desde el navegador a la pasarela, que puede
estar en **otro origen** (ver [CORS y orígenes permitidos](#cors-y-orígenes-permitidos)).
La dirección de la pasarela es configurable.

Se conecta por **WebSocket** y **no puede controlar** ningún robot: su conexión
es de solo lectura. Tiene **dos vistas**:

| Vista | Qué muestra | Uso típico |
|-------|-------------|------------|
| **Administrador** | El circuito y **todos** los robots con sus **coordenadas y orientación exactas**, dibujados con sus [capas SVG](robots/README.md#capas-svg). Sobre cada robot se muestra una **etiqueta con el nombre del usuario** al que pertenece. | Proyectar la prueba o la carrera en televisiones o pantallas grandes. |
| **Usuario** | **Solo los valores de los sensores de un robot**. No muestra la posición real ni los robots de los demás. | Que el dueño depure su algoritmo viendo lo mismo que "ve" su robot, o que terceros sigan ese robot con el token de solo lectura. |

Así se mantiene la regla de que un participante solo dispone de la información
que le dan los sensores de su robot, mientras que el administrador tiene la
vista completa de la carrera.

El visor se autentica con un **token de API de solo lectura**. La vista de
administrador solo está disponible si el token pertenece a un usuario con **rol
de administrador**.

**Robot fuera del mundo.** Si el robot que se está viendo no está en el mundo,
por ejemplo porque su dueño se desconectó hace más de 5 minutos o salió de forma
voluntaria, el visor muestra el mensaje **"El robot no está actualmente en el
mundo"**. No da la conexión por perdida: en cuanto el robot vuelve a entrar,
recupera automáticamente la comunicación y sigue mostrando sus sensores.

### Interfaz de gestión

La interfaz de gestión
([`robot-2d-interfaz-web`](https://github.com/ghCreaR/robot-2d-interfaz-web)) es
la aplicación web donde se administra el sistema. El usuario **inicia sesión con
su cuenta**; esa sesión solo sirve para esta aplicación, porque los clientes de
control y los visores usan tokens de API. Permite:

- **Ver el listado de mundos** a los que el usuario tiene acceso, con su
  nombre, los robots permitidos y si están **activos** en ese momento.
- **Generar y revocar los tokens de API** del usuario (ver
  [Tokens de API](#tokens-de-api)).
- A los **administradores**: **dar de alta usuarios y mundos**, generar el
  testigo de acceso de cada mundo (ver [Registro de mundos](#registro-de-mundos))
  y **gestionar a qué mundos tiene acceso cada usuario** (ver
  [Acceso de los usuarios a los mundos](#acceso-de-los-usuarios-a-los-mundos)).

Es la **dueña del esquema de la base de datos** (y de sus migraciones). No
participa en los datos en tiempo real.

## API de cliente

Los clientes de control y los visores usan una API autenticada con **tokens de
API**:

1. **Autenticación.** Cada petición lleva un token de API que el usuario ha
   generado antes en la interfaz de gestión. No hay endpoint de inicio de
   sesión en la API.
2. **Datos en tiempo real.** Con ese token el cliente recibe la telemetría y,
   si el token es de lectura-escritura, envía los valores de los actuadores. Hay
   dos medios, a elección del cliente:
   - **WebSocket:** una conexión persistente a un endpoint WebSocket. El servidor
     *empuja* las lecturas de los sensores en cuanto se generan y el cliente
     envía los valores de los actuadores por la misma conexión.
   - **Polling REST:** el cliente consulta periódicamente un endpoint REST para
     leer los últimos valores de los sensores y envía los valores de los
     actuadores con otra petición REST.

### Acceso de los usuarios a los mundos

- **Los administradores deciden a qué mundos tiene acceso cada usuario.** Un
  usuario recién dado de alta **no tiene acceso a ningún mundo** hasta que un
  administrador se lo concede.
- Un usuario solo ve en el listado, y solo puede entrar en, los mundos a los que
  tiene acceso. La pasarela ofrece ese listado, con los mundos activos, en un
  endpoint de la API (`GET /mundos`), para que los clientes de control y el
  visor puedan elegir mundo.
- **Un robot por usuario en cada mundo.** Un usuario puede tener a la vez un
  robot en cada uno de los mundos a los que tiene acceso, pero nunca dos en el
  mismo mundo. Si intenta entrar de nuevo en un mundo donde ya tiene robot,
  recupera ese robot (o entra desde cero si había hecho una salida voluntaria).
- Cada robot se identifica por el par **(mundo, usuario)**. Por eso las
  peticiones de la API indican siempre el mundo, por ejemplo con rutas como
  `/mundos/<uuid>/…`.
- Si un administrador **retira el acceso** a un mundo donde el usuario tiene un
  robot, el robot sale del mundo en ese momento.

### Tokens de API

Cada usuario genera sus tokens desde la interfaz de gestión. **Los tokens valen para
todos los mundos** a los que el usuario tiene acceso: con un mismo token se
controla, o se observa, su robot en cada uno de ellos. Hay dos tipos:

| | Lectura-escritura (`rw`) | Solo lectura (`ro`) |
|---|---|---|
| Uso previsto | Su propio cliente de control. | Dejar que **terceros vean los sensores** de su robot sin poder controlarlo, por ejemplo en un visor web. |
| Leer los sensores de su robot | ✅ | ✅ |
| `ping` | ✅ | ✅ |
| Enviar valores de los actuadores | ✅ | ❌ |
| Entrar al mundo y salir de forma voluntaria | ✅ | ❌ |
| **Alargar la vida del robot en el mundo** | ✅ | **❌** |

Reglas:

- **Como máximo uno de cada tipo por usuario.** Al generar un token nuevo se
  revoca el anterior del mismo tipo, que deja de funcionar en ese momento.
- **Caducidad de 7 días como máximo.** El usuario puede elegir una más corta y
  puede revocar sus tokens en cualquier momento.
- **Se muestran una sola vez**, al generarlos. El servidor solo guarda su
  *hash*.
- Son **opacos y aleatorios**, con un prefijo que indica su tipo (por ejemplo
  `crt_rw_…` y `crt_ro_…`), lo que ayuda a reconocerlos y a detectarlos si se
  publican por error. Se prefieren a los JWT porque un token revocado debe dejar
  de funcionar al instante.
- La pasarela guarda en memoria durante unos segundos los tokens ya validados,
  para no consultar la base de datos en cada petición de polling. Cuando la
  interfaz de gestión revoca un token, lo avisa con una notificación de
  PostgreSQL (`NOTIFY`), y todas las réplicas de la pasarela, que escuchan con
  `LISTEN`, lo olvidan al instante.

El token se envía de una de estas formas, a elección del cliente:

- **Cabecera `Authorization: Bearer <token>`.** Es lo natural para los clientes
  de control y para las peticiones REST del visor web.
- **Primer mensaje del WebSocket.** Los navegadores no permiten poner la
  cabecera `Authorization` al abrir un WebSocket, así que el visor web envía el
  token en el primer mensaje de la conexión. La pasarela no acepta ningún otro
  mensaje hasta recibirlo y cierra la conexión si no llega en unos segundos.
- **Cookie.** Solo cuando el cliente y la pasarela comparten sitio (mismo
  dominio). La cookie es `HttpOnly`, `Secure` y `SameSite=Strict`, por lo que
  **no sirve para un visor alojado en otro dominio**: en ese caso se usan las
  dos formas anteriores.

El token **nunca va en la URL**, porque acabaría en los registros.

### CORS y orígenes permitidos

El visor web son ficheros estáticos que pueden estar alojados en un origen
distinto al de la pasarela (otro dominio o puerto). La pasarela tiene una
**lista de orígenes permitidos** configurable (`CORS_ORIGENES`) y la aplica así:

- **Peticiones REST** (por ejemplo, `GET /mundos` o el polling de sensores):
  - Responde a las peticiones previas (*preflight*) `OPTIONS`.
  - Devuelve `Access-Control-Allow-Origin` con el **origen concreto** que hizo
    la petición, si está en la lista, nunca `*`, y añade `Vary: Origin`.
  - `Access-Control-Allow-Methods: GET, POST, OPTIONS`.
  - `Access-Control-Allow-Headers: Authorization, Content-Type`.
  - `Access-Control-Max-Age` para que el navegador recuerde el resultado de la
    petición previa y no la repita en cada consulta.
  - No envía `Access-Control-Allow-Credentials`, porque desde otro origen se
    usa el token `Bearer` y no cookies.
- **WebSocket:** el navegador no aplica CORS a los WebSocket, así que la
  pasarela **comprueba ella misma la cabecera `Origin`** al abrir la conexión y
  la rechaza si no está en la lista. Los clientes que no son navegadores, como
  los clientes de control, no envían `Origin` y se aceptan; para ellos la
  protección es el token.

Los ficheros SVG de los robots que sirve la pasarela llevan las mismas
cabeceras, para que el visor pueda cargarlos desde otro origen.

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
  clientes de control                        visor web
  (WebSocket / REST)                         (admin / usuario)
          │                                         │
          ▼                                         ▼
  ┌──────────────────────────────────────────────────────────┐
  │            PASARELA DE API (una o varias réplicas)       │
  │  - Autenticación (token / cookie)                        │
  │  - Conexiones WebSocket y endpoints REST                 │
  │  - Caché del último valor de cada sensor (polling)       │
  │  - Long polling, ping y filtrado por nivel de acceso     │
  │  - Capas SVG de los robots, CORS                         │◄──► BASE DE ◄──► INTERFAZ
  └───────────────┬─────────────────────────▲────────────────┘     DATOS        DE GESTIÓN
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
  (administrador o usuario), los **tokens de API** (solo su *hash*), el
  **registro de mundos**, **a qué mundos tiene acceso cada usuario** y el
  historial de carreras. La usan dos componentes:
  - La **interfaz de gestión**, que es la dueña del esquema y la que da de alta,
    modifica y revoca.
  - La **pasarela**, que la consulta para validar tokens, accesos y testigos, y
    solo escribe en ella la hora del último latido de cada mundo.

  Los cambios que afectan a la pasarela (un token revocado o un acceso
  retirado) se avisan con `NOTIFY` de PostgreSQL. La base de datos queda **fuera
  del camino de los datos en tiempo real**: los sensores y los actuadores nunca
  pasan por ella.
- **Interfaz de gestión.** Aplicación web de administración (usuarios, mundos,
  accesos y tokens). No se conecta al bus ni a los motores.
- **Motor de simulación.** Solo hace cálculo: avanza la física, muestrea los
  sensores y aplica los actuadores. No sabe nada de HTTP ni de WebSockets, por
  lo que su bucle no se ve afectado por clientes lentos o por picos de
  conexiones. Es la única fuente de verdad del mundo y también lleva los
  temporizadores del ciclo de vida (5 s y 5 minutos) y el límite de robots,
  para que no dependan de qué réplica de la pasarela atienda al cliente. No
  tiene acceso a la base de datos: se identifica ante el bus con el testigo de
  su mundo.

### Registro de mundos

Cada motor de simulación es un **mundo** que los **administradores dan de alta**
desde la interfaz de gestión. La base de datos guarda de cada mundo:

| Campo | Descripción |
|-------|-------------|
| Identificador interno | UUID aleatorio generado al darlo de alta. Se usa en los temas del bus (`mundo.<uuid>.…`). |
| Nombre visible | El que ven los usuarios en el listado de servidores. |
| Robots permitidos | Modelos de robot (los `id` de [`robots/`](robots/)) que se pueden usar en ese mundo. |
| Testigo de acceso | Secreto aleatorio de 256 bits que **solo pueden generar los administradores**. Se muestra una sola vez y se guarda solo su *hash*. Se puede regenerar o revocar. |

#### Cómo se conecta un motor de simulación

El motor de simulación se conecta **directamente al bus**, sin pasar por la
pasarela, para no añadir latencia. Para autenticarlo se usa el mecanismo de
**autenticación delegada de NATS** (*auth callout*, disponible desde NATS 2.10):

```
  motor de simulación             NATS                    pasarela        base de datos
          │  conectar con          │                          │                 │
          │  uuid + testigo        │                          │                 │
          ├───────────────────────►│  ¿es válido?             │                 │
          │                        ├─────────────────────────►│  comprobar hash │
          │                        │                          ├────────────────►│
          │                        │  sí: permisos solo para  │◄────────────────┤
          │                        │  mundo.<uuid>.>          │                 │
          │  conectado             │◄─────────────────────────┤                 │
          │◄───────────────────────┤                          │                 │
```

1. El motor se conecta a NATS presentando el UUID y el testigo de su mundo.
2. NATS no decide por sí mismo: pregunta a la pasarela, que tiene una conexión
   especial autorizada para responder a estas consultas.
3. La pasarela comprueba el testigo contra la base de datos y responde con unos
   permisos **limitados a los temas de ese mundo** (`mundo.<uuid>.>`). Un motor
   no puede leer ni escribir en los temas de otro mundo.
4. La respuesta va firmada con una clave (*nkey*) que solo conoce la pasarela;
   NATS solo acepta respuestas con esa firma.

Así la base de datos es la única fuente de verdad, dar de alta o revocar un
mundo no obliga a reconfigurar ni a reiniciar NATS, y los datos en tiempo real
siguen yendo directos por el bus.

**NATS nunca se expone fuera del clúster.** Los motores de simulación corren en
distintas máquinas de un clúster Docker Swarm o Kubernetes y llegan a NATS solo
por la **red interna** del clúster.

Para proteger el tráfico entre máquinas **se recomienda activar TLS en NATS**,
o bien cifrar la red interna: red *overlay* cifrada en Swarm, o políticas de red
y cifrado del CNI en Kubernetes.

Como alternativa, si la red está gestionada y es privada, se pueden conectar los
nodos del clúster por una **VLAN dedicada**, aislada del resto de la red, de
modo que el tráfico de NATS no salga de ella. Aun en ese caso, **se recomienda
mantener TLS** como defensa en profundidad: protege el tráfico aunque algún otro
equipo llegue a tener acceso a la VLAN.

#### Servidores activos

Estar **registrado** (dato fijo en la base de datos) no es lo mismo que estar
**activo** (en marcha ahora mismo). Cada motor publica un **latido** cada pocos
segundos en `mundo.<uuid>.latido`, con su circuito, los robots que hay y el
máximo admitido. Con esos latidos la pasarela:

- Ofrece el **listado de mundos activos** en su API (`GET /mundos`), para los
  clientes de control y el visor.
- Guarda en la base de datos la **hora del último latido** de cada mundo, como
  mucho cada pocos segundos, para que la interfaz de gestión muestre si un mundo
  está activo sin tener que conectarse al bus.

Un mundo sin latidos durante un tiempo se da por parado.

Al conectarse, el motor pide a la pasarela la **configuración de su mundo**: los
robots permitidos y sus definiciones YAML. Así no necesita tener una copia del
directorio `robots/`.

### Mensajes entre componentes

| Tema (ejemplo) | Sentido | Contenido |
|----------------|---------|-----------|
| `mundo.<uuid>.robot.<id>.sensores` | simulación → pasarela | Lecturas de sensores con su marca de tiempo. |
| `mundo.<uuid>.robot.<id>.actuadores` | pasarela → simulación | Nueva consigna de los motores. |
| `mundo.<uuid>.robot.<id>.actividad` | pasarela → simulación | Aviso de que el cliente sigue vivo (solo con token de lectura-escritura: lectura de sensores o `ping` por polling, conexión o desconexión WebSocket). Se puede agrupar, por ejemplo uno por segundo como máximo. |
| `mundo.<uuid>.control` | pasarela → simulación | Peticiones de entrada y de salida voluntaria de robots, con respuesta (aceptado o lleno). |
| `mundo.<uuid>.estado` | simulación → pasarela | Posición y orientación exactas de todos los robots, solo para la vista de administrador. |
| `mundo.<uuid>.eventos` | simulación → pasarela | Robot que entra, sale o pierde los actuadores. |
| `mundo.<uuid>.latido` | simulación → pasarela | El mundo está activo: circuito, robots presentes y máximo admitido. |
| `mundo.<uuid>.configuracion` | simulación → pasarela | Petición de la configuración del mundo al conectarse (robots permitidos y sus definiciones). |

Todos los temas de un mundo empiezan por `mundo.<uuid>.`, de modo que un único
permiso (`mundo.<uuid>.>`) basta para aislar cada motor de simulación de los
demás.

Hay dos tipos de mensajes, y cada uno se trata de forma distinta.

**Datos en tiempo real** (`sensores`, `actuadores`, `actividad`, `estado` y
`latido`).
Son de tipo **"vale el último"**: una lectura de sensor o una consigna de motor
antigua no sirve de nada si ya hay una más nueva. Por eso:

- **No hace falta persistencia ni confirmaciones** (*acks*). Los mensajes pueden
  ser transitorios y en memoria.
- Las colas deben ser **cortas** (por ejemplo, de longitud 1 o con un tiempo de
  vida pequeño) y **descartar los mensajes viejos**, en vez de acumularlos y
  entregarlos con retraso.

**Operaciones de ciclo de vida** (`control`, `eventos` y `configuracion`). Cambian quién controla
cada robot, así que **no se pueden descartar**:

- Se hacen como **petición-respuesta**, con un **identificador de petición**
  para emparejar cada respuesta con su petición.
- Son **idempotentes**: si una petición de entrada o salida se repite, por
  ejemplo porque se perdió la respuesta, el resultado es el mismo. Un usuario
  que reintenta entrar recupera su robot y no crea uno nuevo.
- Si la pasarela se reinicia, pide a la simulación la lista de robots y de sus
  dueños para volver a sincronizarse. La simulación sigue siendo la única fuente
  de verdad y sus temporizadores paran o retiran los robots aunque se pierda
  algún mensaje.

En los dos casos conviene un **formato binario compacto** (MessagePack, Protocol
Buffers o FlatBuffers) en lugar de JSON entre componentes internos.

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
2. **Clúster (Docker Swarm o Kubernetes):** varias réplicas de la pasarela
   detrás de un balanceador, y motores de simulación repartidos entre las
   máquinas del clúster. Todos se comunican por la red interna; desde fuera
   solo es accesible la pasarela. Los relojes de todas las máquinas deben estar
   sincronizados por NTP (ver [Relojes y marcas de tiempo](#relojes-y-marcas-de-tiempo)).

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

Cada componente se **construye directamente desde su repositorio** (ver
[Repositorios](#repositorios)), por lo que cada uno debe incluir su
`Dockerfile`.

**Los clientes de control no forman parte del despliegue:** corren de forma
independiente, en la máquina y el lenguaje que quiera cada participante, y se
conectan a la pasarela por WebSocket o REST. El visor web sí se incluye, servido
como ficheros estáticos, para tenerlo a mano en local; en otros despliegues se
puede alojar en cualquier servidor web.

### Servicios

| Servicio | Imagen | Puertos expuestos | Función |
|----------|--------|-------------------|---------|
| `pasarela` | Desde `robot-2d-pasarela` | `8080` (configurable) | API REST y WebSocket, `ping`, validación de tokens, CORS, capas SVG de los robots y autenticación de los motores en NATS (*auth callout*). |
| `interfaz` | Desde `robot-2d-interfaz-web` | `8082` (configurable) | Interfaz de gestión de usuarios, mundos, accesos y tokens. Dueña del esquema de la base de datos. |
| `visor` | Desde `robot-2d-visor-web` | `8081` (configurable) | Servidor web de los ficheros estáticos del visor. El navegador se conecta desde ahí a la pasarela, que es otro origen. |
| `simulacion-ovalo` | Desde `robot-2d-motor-fisicas` | Ninguno | Motor de simulación de un mundo con el circuito en O. Está en el perfil `mundos` porque necesita el UUID y el testigo del mundo. Tiene CPU reservada para que el bucle de física no compita con el resto de servicios. |
| `simulacion-ocho` | Desde `robot-2d-motor-fisicas` | Ninguno | Segundo mundo con el circuito en 8. Viene comentado; se activa descomentándolo y dándolo de alta como otro mundo. |
| `bus` | `nats:2-alpine` | Ninguno (`8222` para monitorización, comentado y solo en `127.0.0.1`) | Bus de mensajes sin persistencia entre la pasarela y los motores. Su configuración está en [`nats/nats.conf`](nats/nats.conf). |
| `bd` | `postgres:17-alpine` | Ninguno | Base de datos de usuarios, roles, tokens de API, mundos e historial, con un volumen persistente (`datos-bd`). |

Todos los servicios comparten una red interna. Solo la pasarela, la interfaz
de gestión y el visor publican puertos en la máquina anfitriona; el bus y la
base de datos no son accesibles desde fuera. El directorio [`robots/`](robots/)
se monta en modo de solo lectura en la pasarela, que sirve las capas SVG al
visor y entrega a cada mundo las definiciones de sus robots permitidos, y en la
interfaz de gestión, que los ofrece al configurar los mundos. Así se pueden
añadir o cambiar modelos sin reconstruir las imágenes. La pasarela espera a que
el bus y la base de datos estén sanos (*healthcheck*) antes de arrancar, la
interfaz espera a la base de datos y los motores esperan al bus y a la pasarela.

Como el visor (`http://localhost:8081`) y la pasarela (`http://localhost:8080`)
son orígenes distintos, `CORS_ORIGENES` incluye por defecto el origen del visor
(ver [CORS y orígenes permitidos](#cors-y-orígenes-permitidos)). Si se cambia el
puerto o el dominio del visor, hay que actualizar esa variable.

Mientras los repositorios de los componentes no tengan su `Dockerfile`, se
puede levantar solo la infraestructura: `docker compose up -d bus bd`.

**Autenticación en NATS.** El usuario `pasarela` entra con contraseña. Cualquier
otra conexión, como la de un motor de simulación, pasa por el *auth callout*:
NATS espera a que la pasarela la apruebe y, si no lo hace, la rechaza. Las
respuestas de la pasarela van firmadas con una clave *nkey* que se genera una
vez y se guarda en `.env`.

### Puesta en marcha

1. Preparar el fichero `.env`:

   ```sh
   cp .env.example .env
   # Ajustar POSTGRES_PASSWORD y NATS_PASARELA_PASSWORD.
   # Generar el par de claves del auth callout y copiarlas en
   # NATS_CALLOUT_SEED (línea que empieza por "SA") y NATS_CALLOUT_ISSUER
   # (línea que empieza por "A"):
   docker run --rm natsio/nats-box nk -gen account -pubout
   ```

2. Arrancar el bus, la base de datos, la pasarela, la interfaz de gestión y el
   visor:

   ```sh
   docker compose up -d --build
   ```

3. Abrir la interfaz de gestión en `http://localhost:8082`, entrar como
   administrador, **dar de alta el mundo** y copiar su UUID y su testigo en
   `MUNDO_OVALO_UUID` y `MUNDO_OVALO_TESTIGO` del `.env`. Dar acceso a ese mundo
   a los usuarios que lo vayan a usar.

4. Arrancar los motores de simulación:

   ```sh
   docker compose --profile mundos up -d --build
   ```

5. Cada usuario genera sus tokens de API en la interfaz de gestión y conecta
   sus clientes de control a la pasarela en `http://localhost:8080`. El visor
   está en `http://localhost:8081`.

Otros comandos útiles:

```sh
docker compose logs -f                  # ver los registros
docker compose --profile mundos down    # parar todo (añadir -v para borrar la base de datos)
```

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
| `NATS_PASARELA_PASSWORD` | *(obligatoria)* | `bus`, `pasarela` | Contraseña del usuario `pasarela` en NATS. |
| `NATS_CALLOUT_SEED` | *(obligatoria)* | `pasarela` | Clave privada con la que la pasarela firma las respuestas del *auth callout*. |
| `NATS_CALLOUT_ISSUER` | *(obligatoria)* | `bus` | Clave pública correspondiente; NATS solo acepta respuestas firmadas con ella. |
| `CORS_ORIGENES` | `http://localhost:8081` | `pasarela` | Orígenes permitidos para CORS y para WebSocket desde navegador, separados por comas. |
| `INTERFAZ_PUERTO` | `8082` | `interfaz` | Puerto de la máquina anfitriona donde se publica la interfaz de gestión. |
| `SESION_WEB_TTL` | `12h` | `interfaz` | Duración de la sesión en la interfaz de gestión. No afecta a los tokens de API. |
| `TOKEN_API_MAX_DIAS` | `7` | `interfaz` | Caducidad máxima de los tokens de API. |
| `VISOR_PUERTO` | `8081` | `visor` | Puerto de la máquina anfitriona donde se publica el visor. |
| `PASARELA_URL_PUBLICA` | `http://localhost:8080` | `visor` | Dirección de la pasarela tal y como la ve el navegador; el visor la usa para conectarse. |
| `MUNDO_OVALO_UUID` | *(vacía)* | simulación | UUID del mundo, obtenido al darlo de alta en la interfaz de gestión. |
| `MUNDO_OVALO_TESTIGO` | *(vacía)* | simulación | Testigo de acceso del mundo, obtenido al darlo de alta. |
| `LATIDO_S` | `2` | simulación | Segundos entre latidos que indican que el mundo está activo. |
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
  YAML. El visor web las pinta **de la primera a la última**, así que las
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

Cuando un cliente se conecta con su robot (con un token de lectura-escritura),
este **entra al mundo** así:

- El usuario debe **tener acceso a ese mundo** y **no tener ya otro robot en
  él**. Si ya lo tiene, recupera ese robot en lugar de crear uno nuevo.
- Solo puede usar un modelo de robot **permitido en ese mundo**.
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
| **Polling REST** | Tras **5 segundos** sin recibir instrucciones para los motores. | A los **5 minutos** sin ninguna petición a la API con el token de lectura-escritura (ni instrucciones de motores, ni lecturas de sensores, ni `ping`). |

Con polling, la parada de los motores y la vida del robot se cuentan por
separado. Los motores solo se mantienen vivos con instrucciones de motores. La
vida del robot **se extiende con cualquier petición a la API hecha con el token
de lectura-escritura**: basta con consultar los datos de sus sensores o hacer un
`ping`. Por eso un cliente puede tener el robot parado y seguir leyendo sus
sensores sin que salga del mundo.

Las peticiones con el **token de solo lectura no alargan la vida del robot**.
Así, un tercero que solo está mirando no puede mantener en el mundo el robot de
alguien que ya se ha ido.

Si el cliente vuelve antes de que pasen los 5 minutos, **recupera el control del
robot allí donde esté en ese momento**: puede seguir frenando por la inercia o
estar ya parado.

### Salida voluntaria

El cliente puede enviar una **señal de salida** (comando `salir`, disponible
como endpoint REST y como comando WebSocket, solo con el token de
lectura-escritura). El robot **sale del mundo en ese momento**, sin esperar a los
5 minutos.

La siguiente conexión de ese cliente **entra desde cero**: se busca de nuevo un
punto libre y aleatorio del mapa, orientado hacia el centro, como en la primera
entrada. Es útil, por ejemplo, para reiniciar una prueba sin esperar a que
caduque el robot anterior.

### Visores con token de solo lectura

Un visor (por ejemplo, el visor web abierto por un tercero) indica el mundo que quiere
ver. Como el token vale para todos los mundos del usuario, con el mismo token de
solo lectura se puede seguir su robot en cualquiera de ellos. El visor sigue al
robot aunque este no esté en el mundo:

- Mientras el robot no está, el visor muestra **"El robot no está actualmente en
  el mundo"**.
- Cuando el robot vuelve a entrar, ya sea recuperando su posición o desde cero
  tras una salida voluntaria, la pasarela **reanuda automáticamente** el envío
  de sus sensores al visor, sin que este tenga que reconectarse.

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

1. Un administrador da de alta el mundo en la interfaz de gestión. El motor de
   simulación arranca con su UUID y su testigo, se autentica en el bus, pide la
   configuración de su mundo, carga su circuito y empieza a publicar latidos.
   Desde ese momento aparece en el listado de servidores activos.
2. El usuario genera en la interfaz de gestión su token de API de lectura-escritura. Su
   cliente de control se conecta con él (como `Bearer` o como cookie), elige un
   mundo activo al que un administrador le haya dado acceso y un modelo de robot
   permitido en ese mundo. El mismo token le sirve para tener otro robot en
   cada uno de sus demás mundos, pero solo uno por mundo.
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
7. A la vez, la vista de administrador del visor web muestra todos los robots con su
   posición, su orientación y el nombre de su usuario, y cada usuario ve en su
   visor web los sensores de su propio robot. Con el token de solo lectura,
   terceros pueden ver también esos sensores, sin poder controlar el robot ni
   alargar su vida en el mundo.
8. Si el cliente se desconecta, o deja de mandar instrucciones de motores durante
   5 s por polling, el robot frena hasta parar. Si pasan 5 minutos sin volver, o
   sin ninguna petición a la API con el token de lectura-escritura en el caso de
   polling, sale del mundo. Si vuelve antes, recupera el control allí donde esté
   el robot.
9. El cliente también puede enviar el comando `salir`: el robot sale del mundo
   en ese momento y la siguiente conexión entra desde cero, en un punto nuevo
   orientado hacia el centro. Mientras tanto, los visores muestran "El robot no
   está actualmente en el mundo" y se reanudan solos cuando el robot vuelve.

## Licencia

Consulta el archivo [LICENSE](LICENSE).

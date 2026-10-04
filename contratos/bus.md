# Contrato: mensajes del bus

Interfaz entre la **pasarela** y los **motores de simulación**, a través de **NATS**. Los mensajes van codificados en **MessagePack**. Los nombres de los campos son los mismos que en este documento.

La justificación del diseño (por qué NATS, mensajes "vale el último" frente a operaciones de ciclo de vida, presupuesto de latencia…) está en el [README principal](../README.md#mensajes-entre-componentes).

## Identidad y permisos del motor

- El motor se conecta con el **UUID de su mundo como usuario** y el **testigo como contraseña**. NATS pregunta a la pasarela por el *auth callout* y esta comprueba el testigo contra la base de datos.
- Si es válido, la pasarela le concede:
  - publicar y suscribirse en `mundo.<uuid>.>`;
  - **responder** a las peticiones que reciba (`allow_responses`), para contestar a las de `control`.
- Las respuestas a las peticiones **del motor** (por ejemplo, `configuracion`) le llegan a su **buzón** `mundo.<uuid>.buzon.>`, que queda dentro de sus permisos. El motor debe configurar ese prefijo de buzón (`inbox_prefix`) en su cliente NATS en lugar del `_INBOX` por defecto.

Un motor no puede leer ni escribir en los temas de otro mundo.

## Temas

`<usuario>` es el UUID del dueño del robot. Todas las marcas `ts` siguen la [convención común](README.md#convenciones-comunes).

### Datos en tiempo real ("vale el último")

Sin persistencia ni confirmaciones. Quien va retrasado descarta los mensajes viejos.

| Tema | Sentido | Contenido |
|------|---------|-----------|
| `mundo.<uuid>.robot.<usuario>.sensores` | motor → pasarela | `{ts, seq, valores: {<id_sensor>: número}}` |
| `mundo.<uuid>.robot.<usuario>.actuadores` | pasarela → motor | `{ts, valores: {<id_actuador>: número}}` |
| `mundo.<uuid>.robot.<usuario>.actividad` | pasarela → motor | `{ts, motivo}` (ver abajo) |
| `mundo.<uuid>.estado` | motor → pasarela | `{ts, robots: [{usuario, modelo, x, y, theta, motores_activos}]}` |
| `mundo.<uuid>.latido` | motor → pasarela | `{ts, circuito, robots: [usuario…], max_robots}` |

Valores de `motivo` en `actividad`. Solo se envían para tokens de lectura-escritura:

| Motivo | Cuándo | Efecto en el motor |
|--------|--------|--------------------|
| `polling` | Cualquier petición REST del dueño (sensores, `ping`…). Se agrupan: como mucho uno por segundo. | Reinicia la cuenta de salida del mundo (5 min). |
| `ws_abierto` | Se abre un WebSocket del dueño que ha entrado o sigue el mundo. | Modo WebSocket: los motores siguen activos mientras haya conexiones abiertas. |
| `ws_cerrado` | Se cierra ese WebSocket. | Si no queda ninguna conexión, desactiva los motores y empieza la cuenta de salida. |

Las consignas de `actuadores` también cuentan como actividad.

### Operaciones de ciclo de vida (petición-respuesta)

No se pueden descartar. Llevan un `id_peticion` y son **idempotentes**: repetir una petición da el mismo resultado.

| Tema | Sentido | Petición | Respuesta |
|------|---------|----------|-----------|
| `mundo.<uuid>.control` | pasarela → motor | `{id_peticion, op, …}` (ver abajo) | `{id_peticion, resultado, …}` |
| `mundo.<uuid>.configuracion` | motor → pasarela | `{circuito}` | `{nombre, robots: {<id>: definición}, circuito: definición}` |

Operaciones de `control`:

| `op` | Campos | Resultados |
|------|--------|------------|
| `entrar` | `usuario`, `nombre`, `modelo` | `dentro` (nuevo), `recuperado` (ya estaba), `lleno`, `modelo_no_permitido` |
| `salir` | `usuario` | `fuera` (también si ya estaba fuera) |
| `expulsar` | `usuario`, `motivo` | `fuera`. Se usa cuando un administrador retira el acceso al mundo o desactiva al usuario. |
| `listar` | — | `ok`, con `robots: [{usuario, nombre, modelo}]`. La pasarela lo usa al arrancar para resincronizarse. |

Las definiciones de `robots` y `circuito` en `configuracion` tienen la misma estructura que sus ficheros YAML ([robots](../robots/README.md), [circuitos](../circuitos/README.md)), ya convertida a mapa.

### Eventos

| Tema | Sentido | Contenido |
|------|---------|-----------|
| `mundo.<uuid>.eventos` | motor → pasarela | `{ts, evento, usuario, motivo}` |

| `evento` | `motivo` |
|----------|----------|
| `entra` | `nuevo`, `recuperado` |
| `sale` | `voluntaria`, `inactividad`, `expulsado` |
| `motores_off` | `sin_instrucciones`, `desconexion` |
| `motores_on` | — |

Los eventos son informativos: si se pierde alguno, la pasarela se resincroniza con los latidos y con `listar`.

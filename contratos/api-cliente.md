# Contrato: API de cliente

Interfaz entre la **pasarela** y sus clientes: los **clientes de control** de los participantes y el **visor web**. Todos los mensajes van en **JSON**.

Las reglas generales (tipos de token, caducidad, CORS, desconexión…) están en el [README principal](../README.md#api-de-cliente). Este documento fija las rutas, los mensajes y los códigos.

## Autenticación

El token de API se envía de una de estas formas:

| Forma | Uso |
|-------|-----|
| Cabecera `Authorization: Bearer <token>` | Peticiones REST y WebSocket desde clientes que no son navegadores. |
| Primer mensaje del WebSocket: `{"tipo": "autenticar", "token": "…"}` | Navegadores, que no pueden poner cabeceras al abrir un WebSocket. Si no llega en 5 s, la pasarela cierra la conexión con el código `4401`. |
| Cookie `crt_token` (`HttpOnly`, `Secure`, `SameSite=Strict`) | Solo si el cliente comparte sitio con la pasarela. Se obtiene con `POST /sesion`. |

El token **nunca** va en la URL.

## Errores

Las respuestas de error REST tienen este formato, con el código HTTP correspondiente:

```json
{"error": "mundo_lleno", "mensaje": "El mundo ya tiene el número máximo de robots."}
```

| Código | HTTP | Significado |
|--------|------|-------------|
| `token_ausente` | 401 | No se ha enviado token. |
| `token_invalido` | 401 | Token desconocido, caducado o revocado. |
| `solo_lectura` | 403 | La operación necesita un token de lectura-escritura. |
| `sin_acceso` | 403 | El usuario no tiene acceso a ese mundo. |
| `no_admin` | 403 | La operación es solo para administradores. |
| `mundo_desconocido` | 404 | No existe ese mundo. |
| `robot_fuera` | 404 | El robot del usuario no está en el mundo. |
| `mundo_lleno` | 409 | Se ha alcanzado el número máximo de robots. |
| `modelo_no_permitido` | 422 | El modelo no está permitido en ese mundo. |
| `actuador_invalido` | 422 | Un `id` de actuador no existe en el modelo o su valor está fuera de rango. |
| `mundo_inactivo` | 503 | El motor de simulación del mundo no está en marcha. |

## REST

| Método y ruta | Token | Descripción |
|---------------|-------|-------------|
| `GET /salud` | — | Comprobación de vida. Responde `{"estado": "ok"}`. |
| `POST /sesion` | `ro`/`rw` (`Bearer`) | Devuelve la cookie `crt_token` con el mismo token, para clientes del mismo sitio. `DELETE /sesion` la borra. |
| `GET /ping` | `ro`/`rw` | `{"ts": 1767225600123.45}`. Con un token `rw` y `?mundo=<uuid>`, alarga la vida del robot en ese mundo. |
| `GET /yo` | `ro`/`rw` | `{"usuario": "<uuid>", "nombre": "…", "rol": "admin" \| "usuario", "token": "rw" \| "ro"}`. |
| `GET /mundos` | `ro`/`rw` | Mundos a los que el usuario tiene acceso (ver abajo). |
| `GET /robots/<id>` | `ro`/`rw` | Definición del modelo de robot, en JSON, con la misma estructura que su YAML. Las capas son rutas a `/robots/svg/…`. |
| `GET /robots/svg/<fichero>` | — | Capa SVG del robot, con cabeceras CORS. |
| `GET /circuitos/<id>` | `ro`/`rw` | Definición del circuito, en JSON, con la misma estructura que su YAML. |
| `POST /mundos/<uuid>/entrar` | `rw` | Cuerpo `{"modelo": "<id>"}`. Respuesta `{"resultado": "dentro" \| "recuperado", "modelo": "<id>"}`. |
| `POST /mundos/<uuid>/salir` | `rw` | Salida voluntaria. Respuesta `{"resultado": "fuera"}`, también si ya estaba fuera. |
| `GET /mundos/<uuid>/robot` | `ro`/`rw` | `{"en_mundo": true, "modelo": "<id>"}` o `{"en_mundo": false}`. |
| `GET /mundos/<uuid>/sensores` | `ro`/`rw` | Últimos valores de los sensores del robot del usuario (ver abajo). |
| `POST /mundos/<uuid>/actuadores` | `rw` | Cuerpo `{"valores": {"motor_izquierdo": 0.5, "motor_derecho": 0.4}}`. Respuesta `204`. |

### `GET /mundos`

```json
[
  {
    "uuid": "6f1c…",
    "nombre": "Óvalo de prácticas",
    "activo": true,
    "circuito": "ovalo",
    "robots_permitidos": ["sigue-lineas-3ir", "sigue-lineas-5ir"],
    "robots": 2,
    "max_robots": 4
  }
]
```

Si el mundo no está activo, `circuito`, `robots` y `max_robots` valen `null`.

### `GET /mundos/<uuid>/sensores`

```json
{"ts": 1767225600100.0, "seq": 4182, "valores": {"ir_izquierdo": 0, "ir_central": 1, "ir_derecho": 0}}
```

| Parámetro | Efecto |
|-----------|--------|
| *(ninguno)* | Responde al instante con la última muestra, aunque ya se hubiera leído. |
| `esperar=1` | *Long polling*: espera a la siguiente muestra, o como mucho `LONG_POLLING_TIMEOUT_MS`. Si se agota el tiempo, responde con la última muestra. |
| `desde=<seq>` | Con `esperar=1`: si ya hay una muestra con `seq` mayor, responde al instante con ella. Así el cliente no se pierde muestras entre dos peticiones. |

`seq` crece de uno en uno con cada muestra del robot; un salto indica muestras perdidas. Vuelve a empezar cuando el robot entra al mundo desde cero.

## WebSocket

Endpoint `GET /ws`. Cada mensaje es un objeto JSON con un campo `tipo`. Si el cliente añade un campo `id`, la respuesta a ese mensaje lo repite.

### Del cliente a la pasarela

| Mensaje | Token | Descripción |
|---------|-------|-------------|
| `{"tipo": "autenticar", "token": "…"}` | — | Primer mensaje, si no se usó cabecera ni cookie. Respuesta `{"tipo": "autenticado", "usuario", "rol", "token"}`. |
| `{"tipo": "entrar", "mundo": "<uuid>", "modelo": "<id>"}` | `rw` | Entra al mundo y se suscribe a los sensores de su robot. Respuesta `{"tipo": "entrado", "mundo", "resultado"}`. |
| `{"tipo": "seguir", "mundo": "<uuid>"}` | `ro`/`rw` | Se suscribe a los sensores del robot del usuario sin entrar al mundo. Lo usan el visor y los terceros. |
| `{"tipo": "observar", "mundo": "<uuid>"}` | admin | Vista de administrador: estado de todos los robots. |
| `{"tipo": "dejar", "mundo": "<uuid>"}` | `ro`/`rw` | Cancela `seguir` u `observar`. |
| `{"tipo": "actuadores", "mundo": "<uuid>", "valores": {…}}` | `rw` | Nueva consigna de los motores. Sin respuesta, salvo error. |
| `{"tipo": "salir", "mundo": "<uuid>"}` | `rw` | Salida voluntaria. Respuesta `{"tipo": "salido", "mundo"}`. |
| `{"tipo": "ping"}` | `ro`/`rw` | Respuesta `{"tipo": "pong", "ts": …}`. |

Una conexión puede seguir u observar varios mundos a la vez.

### De la pasarela al cliente

| Mensaje | Descripción |
|---------|-------------|
| `{"tipo": "sensores", "mundo", "ts", "seq", "valores"}` | Cada muestra de los sensores, en cuanto llega del motor. |
| `{"tipo": "robot", "mundo", "en_mundo": true \| false}` | El robot entra o sale del mundo. Se envía al suscribirse y en cada cambio. Con `en_mundo: false`, el visor muestra "El robot no está actualmente en el mundo". |
| `{"tipo": "motores", "mundo", "activos": true \| false}` | Los motores se desactivan (sin instrucciones o por desconexión) o se reactivan. |
| `{"tipo": "estado", "mundo", "ts", "robots": [{"usuario", "nombre", "modelo", "x", "y", "theta"}]}` | Solo con `observar`: posición (m) y orientación (rad) exactas de todos los robots, a 30 Hz. |
| `{"tipo": "error", "error": "<código>", "mensaje"}` | Error de un comando, con los mismos códigos que REST. |

Si un cliente no lee lo bastante rápido, la pasarela **descarta las muestras viejas** en lugar de acumularlas, porque los datos en tiempo real son de tipo "vale el último".

### Cierre

| Código | Motivo |
|--------|--------|
| `1000` | Cierre normal. |
| `4401` | No se autenticó a tiempo, o el token no es válido. |
| `4403` | El token se ha revocado durante la conexión. |
| `4429` | Demasiadas conexiones de ese usuario. |

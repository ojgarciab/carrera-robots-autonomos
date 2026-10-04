# Contrato: base de datos

Esquema de **PostgreSQL** compartido entre la **interfaz de gestión**, que es su dueña y lo crea con sus migraciones, y la **pasarela**, que lo lee.

Las tablas y columnas de este documento son el contrato: la interfaz puede añadir otras para su uso interno, pero no cambiar estas sin acordarlo con la pasarela.

## Tablas

### `usuarios`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | `uuid` PK | Identificador del usuario. Es el `<usuario>` de los temas del bus. |
| `nombre_usuario` | `text` único | Nombre para iniciar sesión en la interfaz. |
| `nombre_visible` | `text` | Etiqueta que se muestra sobre su robot en la vista de administrador. |
| `password` | `text` | *Hash* de la contraseña (Argon2). Solo lo usa la interfaz. |
| `rol` | `text` | `admin` o `usuario`. |
| `activo` | `boolean` | Un usuario desactivado no puede usar sus tokens. |
| `creado` | `timestamptz` | |

### `tokens_api`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | `uuid` PK | |
| `usuario_id` | `uuid` → `usuarios.id` | Dueño del token. |
| `tipo` | `text` | `rw` (lectura-escritura) o `ro` (solo lectura). |
| `hash` | `text` único | SHA-256 en hexadecimal del token completo, prefijo incluido. |
| `pista` | `text` | Últimos 4 caracteres, para que el usuario lo reconozca. |
| `caduca` | `timestamptz` | Como mucho `TOKEN_API_MAX_DIAS` después de crearlo. |
| `revocado` | `boolean` | |
| `creado` | `timestamptz` | |

Como mucho hay un token no revocado de cada tipo por usuario: índice único parcial sobre (`usuario_id`, `tipo`) `WHERE NOT revocado`.

Los tokens tienen la forma `crt_rw_<aleatorio>` o `crt_ro_<aleatorio>`, con 256 bits aleatorios en base64 URL. Como son largos y aleatorios, basta con un SHA-256 para guardarlos, y la pasarela puede validarlos rápido.

### `mundos`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `id` | `uuid` PK | Identificador interno del mundo. Es el `<uuid>` de los temas del bus. |
| `nombre` | `text` | Nombre visible. |
| `robots_permitidos` | `text[]` | `id` de los modelos de robot permitidos. |
| `testigo_hash` | `text` | SHA-256 del testigo de acceso, o `NULL` si está revocado. |
| `testigo_creado` | `timestamptz` | |
| `ultimo_latido` | `timestamptz` | **La escribe la pasarela**, como mucho cada 5 s por mundo. |
| `creado` | `timestamptz` | |

### `accesos`

| Columna | Tipo | Descripción |
|---------|------|-------------|
| `usuario_id` | `uuid` → `usuarios.id` | |
| `mundo_id` | `uuid` → `mundos.id` | |
| `concedido_por` | `uuid` → `usuarios.id` | Administrador que lo concedió. |
| `creado` | `timestamptz` | |

Clave primaria: (`usuario_id`, `mundo_id`).

## Permisos

Se recomienda que la pasarela use un **usuario de PostgreSQL propio** con permiso de lectura sobre estas tablas y de escritura solo sobre la columna `mundos.ultimo_latido`.

## Avisos (`NOTIFY`)

La interfaz avisa de los cambios que afectan a la pasarela con `NOTIFY`, emitido por **triggers** de la base de datos para que el aviso salga sea cual sea el origen del cambio. Como `NOTIFY` solo se entrega al confirmar la transacción, la pasarela nunca recibe el aviso de un cambio que luego se deshace.

| Canal | Carga (JSON) | Cuándo |
|-------|--------------|--------|
| `token_revocado` | `{"hash": "…"}` | Un token pasa a revocado (también al generar otro del mismo tipo) o se borra. |
| `acceso_retirado` | `{"usuario": "<uuid>", "mundo": "<uuid>"}` | Se borra una fila de `accesos`. |
| `usuario_desactivado` | `{"usuario": "<uuid>"}` | Un usuario pasa a inactivo o cambia de rol. |
| `testigo_revocado` | `{"mundo": "<uuid>"}` | Cambia o se anula el `testigo_hash` de un mundo. |

Al recibirlos, la pasarela olvida lo que tuviera en caché. Con `acceso_retirado` y `usuario_desactivado`, además, pide al motor que expulse al robot (`control` `expulsar`). Si la pasarela pierde la conexión con la base de datos, al reconectar vacía toda su caché, porque puede haberse perdido algún aviso.

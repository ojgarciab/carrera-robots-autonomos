# Contratos entre componentes

Cada componente vive en su propio repositorio, así que las interfaces entre ellos se definen aquí, en un único sitio. Cualquier cambio en un contrato debe acordarse entre los componentes afectados y actualizarse en este directorio **antes** de cambiar el código.

| Contrato | Entre | Fichero |
|----------|-------|---------|
| API de cliente: REST y WebSocket (JSON) | Clientes de control y visor web ↔ pasarela | [`api-cliente.md`](api-cliente.md) |
| Mensajes del bus (MessagePack sobre NATS) | Pasarela ↔ motores de simulación | [`bus.md`](bus.md) |
| Esquema de la base de datos y avisos `NOTIFY` | Interfaz de gestión (dueña) → pasarela | [`base-de-datos.md`](base-de-datos.md) |
| Definición de los robots (YAML) | Repositorio común → pasarela, motores, visor, interfaz | [`../robots/README.md`](../robots/README.md) |
| Definición de los circuitos (YAML) | Repositorio común → pasarela, motores, visor | [`../circuitos/README.md`](../circuitos/README.md) |

## Convenciones comunes

- **Marcas de tiempo:** milisegundos desde el 1 de enero de 1970 (UTC), como número con decimales para tener precisión por debajo del milisegundo. Las de los sensores las pone el motor de simulación; la del `ping`, la pasarela.
- **Identificadores:** los mundos y los usuarios se identifican con UUID. Cada robot se identifica por el par (mundo, usuario).
- **Unidades:** las del sistema internacional (m, m/s, m/s², Hz), salvo los ángulos de los circuitos, que van en grados para que sean fáciles de escribir.
- **Valores de los sensores:** números. Los sensores infrarrojos actuales son **digitales** y valen `0` o `1`. Los clientes deben tratarlos como números y no como booleanos, para admitir sin cambios los sensores analógicos previstos (ver [tipos de sensor](../robots/README.md#sensores-de-tipo-infrarrojo)).
- **Versiones:** mientras el sistema esté en desarrollo, los contratos no llevan versión. Antes de la primera carrera se fijará la versión 1 y los cambios incompatibles irán en una versión nueva.

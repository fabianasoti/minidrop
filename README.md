# MiniDrop

Sincronizador de archivos cliente-servidor sobre TCP, escrito en Python (solo biblioteca estándar). Un servidor central guarda la copia oficial y varios clientes mantienen carpetas locales sincronizadas, detectando cambios, borrados y conflictos.

Proyecto de portfolio para practicar computación distribuida en red: protocolo propio, concurrencia con `asyncio`, consistencia y resolución de conflictos.

## Uso rápido

```bash
export MINIDROP_TOKEN=un-secreto

# Terminal 1: servidor
python -m minidrop server --root ./datos --port 8765

# Terminal 2 y 3: dos clientes (pueden estar en máquinas distintas)
python -m minidrop client --folder ~/A --host 127.0.0.1 --id portatil
python -m minidrop client --folder ~/B --host 127.0.0.1 --id sobremesa
```

Crea un archivo en `~/A` y aparecerá en `~/B` en pocos segundos.

Tests: `python -m unittest discover -s tests -v`

## Arquitectura

```
 Cliente A ──┐                         ┌─ files/   (copia oficial)
             ├── TCP (protocolo propio) ── Servidor asyncio ─┤
 Cliente B ──┘                         └─ index.json (versión/rev/sha256 por ruta)
```

- **protocol.py**: mensajes con cabecera JSON + payload binario, con prefijo de longitud.
- **server.py**: `Store` (archivos e índice persistente, validación de rutas) y `Server` (autenticación por token, un manejador por cliente).
- **client.py**: ciclo de sincronización periódico y estado local en `.minidrop_state.json`.

### Mensajes

| Tipo | Dirección | Función |
|---|---|---|
| `hello` | C→S | Autenticación (token) |
| `changes {since}` | C→S | Cambios con `rev` mayor que el indicado |
| `pull {path}` | C→S | Descargar un archivo |
| `push {path, base_version, sha256}` + datos | C→S | Subir; falla con `conflict` si `base_version` no es la actual |
| `delete {path, base_version}` | C→S | Borrar (deja una *lápida* con versión) |

### Decisiones de diseño

- **Versionado optimista**: cada archivo tiene una `version`. Un cliente solo puede subir si parte de la versión actual del servidor; si no, hay conflicto. Es el mismo principio que el control de concurrencia optimista de las bases de datos.
- **Revisión global (`rev`)**: permite pedir al servidor "lo que cambió desde N", así la sincronización es incremental.
- **Lápidas de borrado**: el borrado también es un cambio versionado, para que otros clientes lo repliquen y para detectar conflictos borrado/edición.
- **Conflictos**: gana la primera subida; la copia del otro cliente se conserva como `nombre (conflicto de <cliente>).ext` y se sincroniza como archivo normal. Nunca se pierde información. Si un archivo se edita en un cliente y se borra en otro, gana la edición.
- **Integridad**: SHA-256 comprobado al subir y al bajar; escrituras atómicas (archivo temporal + `os.replace`).
- **Seguridad básica**: token compartido (comparación en tiempo constante) y rechazo de rutas con `..` o absolutas.

## Limitaciones actuales (y hoja de ruta)

Cosas que el MVP aún no hace, ordenadas por valor para el portfolio:

1. **Notificaciones push**: ahora los clientes consultan cada pocos segundos. Añadir una conexión de suscripción para que el servidor avise al instante (WebSockets o `asyncio`).
2. **Transferencia por bloques y deltas**: los archivos se envían enteros y en memoria. Dividirlos en bloques con hash (como rsync o Dropbox) para subir solo lo modificado y soportar archivos grandes con reanudación.
3. **Cifrado TLS** y autenticación por usuario (contraseñas con hash, tokens por dispositivo) en lugar de un token compartido.
4. **Vigilancia del sistema de archivos** con `watchdog` en vez de escanear la carpeta.
5. **Múltiples usuarios y carpetas** con permisos, y metadatos en SQLite o MySQL en lugar de `index.json`.
6. **Servidor replicado**: varios nodos con replicación y elección de líder (el paso hacia sistemas distribuidos de verdad).
7. **Dockerización**, panel web de estado y empaquetado (`pyproject.toml`).
8. Directorios vacíos y renombrados (hoy un renombrado es borrar + crear).

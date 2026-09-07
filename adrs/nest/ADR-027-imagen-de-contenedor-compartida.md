# ADR-027: Una Imagen de Contenedor para Todos los Servicios NestJS

## Estado

Aceptada (implementada)
**Scope:** `nest`

## Fecha

2026-09-07. Publicado en `@ahincho/nova-nestjs-toolchain` 0.12.0.

## Contexto

El generador dejaba un servicio que compila, se prueba y arranca, y **no dejaba forma de
empaquetarlo**. El README lo decía explícitamente: no genera `Dockerfile`, porque la imagen base,
el usuario y el puerto dependen de dónde se despliegue y uno inventado sería peor que ninguno.

Esa razón era buena para no adivinar y mala como estado final. Los once servicios NestJS de A303
se despliegan en ECS, todos como imagen, todos con la misma forma: Node, pnpm, `tsc`, `node
dist/main`. La variación real entre ellos es la línea de Node -los BFF corren `node:26-alpine` y
los ACL `node:24-alpine`- y nada más.

Un Dockerfile copiado en once repositorios tiene el mismo problema que tenía el `main.ts` copiado,
sólo que peor: **una imagen vieja no falla, sigue construyendo.** El día que haya que cambiar de
usuario, agregar una etapa o mover el punto de entrada, hay once pull requests y ninguna forma de
saber cuáles se hicieron.

## Decisión

**Un Dockerfile en el toolchain, y `nova docker` que lo usa con `-f`.**

Es la misma decisión que hizo que los scripts dejaran de nombrar herramientas: lo que evoluciona
vive en un solo lugar, y el servicio invoca un comando estable.

```bash
nova docker                                # etiqueta <nombre>:<version>
nova docker --build-arg NODE_VERSION=26
```

Lo que varía por servicio va como `ARG` -`NODE_VERSION`, `PNPM_VERSION`-, nunca como una edición
local.

### Cuatro etapas, y ninguna herramienta en la imagen final

`base` instala pnpm, `deps` resuelve el árbol contra el lockfile, `build` compila y hace
`pnpm prune --prod`, y `runtime` parte de la imagen limpia y copia sólo `node_modules` y `dist`.
No queda pnpm, ni código fuente, ni dependencias de desarrollo.

Corre como el usuario `node`, que la imagen oficial ya trae.

### El token del registry entra como secreto de BuildKit

`@ahincho/*` se resuelve contra GitHub Packages, así que el build necesita una credencial. Va
montada, no copiada: **un `ARG` queda en el historial de la imagen y un `COPY` queda en una capa.**

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    pnpm install --frozen-lockfile
```

`nova docker` monta el `~/.npmrc` de la máquina si existe; en CI lo pasa el llamador. El `.npmrc`
del repositorio sí se copia, porque no lleva credencial: sólo apunta el scope al registry.

### pnpm se instala con npm, no con corepack

corepack está marcado como obsoleto y no está garantizado en todas las líneas de Node. Atarse a él
haría que subir la imagen base rompa el build por un motivo que no tiene nada que ver con el
servicio.

### El `.dockerignore` sí se genera por servicio

No por inconsistencia: **Docker lo lee desde la raíz del contexto**, así que no puede vivir en el
toolchain. Es la misma división que con `.gitignore` -la lógica se comparte, lo que describe a
este repositorio se queda en él-.

Sin él se copian los binarios nativos compilados para Windows dentro de una imagen Linux, y el
fallo aparece recién al arrancar el contenedor.

## Consecuencias

### Un pipeline que exige el archivo en la raíz tiene salida

`nova docker --eject` escribe el Dockerfile en el repositorio, con un encabezado diciendo de dónde
salió y que editarlo lo desincroniza. Existe para el caso concreto de A303: el reusable corporativo
construye la imagen con un Dockerfile en la raíz y no acepta un `-f`.

El encabezado va **debajo** de la directiva `# syntax=`, que sólo cuenta si es la primera línea del
archivo. Empujarla hacia abajo deja el build en el parser viejo, donde `--mount=type=secret` no
existe y el token tendría que entrar por un `ARG`.

### `nova verify` no lo incluye

Construir una imagen no dice nada sobre si el código está bien, y tarda como si lo dijera.

### CI construye la imagen y la levanta

El chequeo del servicio generado, después de `nova verify`, corre `nova docker` y arranca el
contenedor hasta que `/health/live` contesta. Los dos sabores, en Node 24 y en 26. Que construya no
prueba que arranque, así que se prueban las dos cosas.

Sale gratis en credenciales: en ese chequeo las dependencias llegan por tarball, así que no hace
falta ningún token.

Tres cosas se descubrieron construyéndola, y ninguna se habría visto de otra forma:

- **La receta clásica no servía.** Copiar sólo los manifiestos, instalar y después copiar el código
  se apoya en la caché de capas, y deja fuera de la etapa a un servicio cuyas dependencias son
  archivos de su propio repositorio. Ahora entra el contexto entero y la caché la da el montaje del
  store de pnpm, que con el store caliente relinkea en vez de descargar.
- **`pnpm prune` corta sin TTY**, pidiendo confirmación para borrar `node_modules`. La etapa declara
  `CI=true`, que es un hecho y no una preferencia.
- **`@scarf/scarf` rompía el install de todo consumidor que ya existía.** Lo encontró el chequeo del
  servicio de ejemplo, no una revisión.

## Referencias

- `packages/toolchain/docker/Dockerfile` y `packages/toolchain/bin/nova.mjs`
- ADR-016, por la línea de Node
- ADR-026, por el generador que deja el `.dockerignore`

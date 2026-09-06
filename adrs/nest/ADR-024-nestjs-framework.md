# ADR-024: NestJS 12 como Framework Backend

## Estado

Aceptada (implementada)
**Scope:** `nest`
Reemplaza el placeholder que decía «NestJS 10.x» y estaba sin redactar.

## Fecha

2026-09-06. Publicado en `@ahincho/nova-nestjs` 0.8.0, con el arreglo de 0.8.1.

## Contexto

NestJS era la elección de facto del stack -los cuatro arquetipos de backend de A303 ya lo
usaban- pero nunca se escribió por qué, ni en qué versión. Este ADR hace las dos cosas, y la
versión es la parte con consecuencias.

**NestJS 12 se publicó el 2026-08-28 y su núcleo pasa a ser ESM puro.** `@nestjs/common`,
`core`, `platform-express`, `config`, `terminus` y `testing` declaran `"type": "module"` y su
`exports` apunta a un único `./index.js`: **no hay build de CommonJS**.

Eso convierte «subir de versión» en una decisión de arquitectura, porque abre dos caminos:

1. **Migrar la plataforma y los servicios a ESM.**
2. **Seguir en CommonJS y consumir el ESM con `require(esm)`**, que Node soporta sin bandera
   desde 22.12.

## Decisión

**NestJS 12, y la plataforma sigue siendo CommonJS.**

Es lo que el propio NestJS asume: **`nest upgrade` no migra un proyecto a ESM**, y el esquema
de proyecto CommonJS sigue existiendo en 12. La opción 1 arrastraría `__dirname`, `require`, la
forma del build y el punto de entrada del contenedor, en los tres paquetes y en cada servicio,
a cambio de nada que se note en ejecución.

Comprobado antes de tocar el repositorio, no deducido: un paquete TypeScript con
`"type": "commonjs"` y `module: NodeNext` compila contra `@nestjs/common` 12 con `tsc` sin
errores, emite `require`, y levanta un módulo con inyección por constructor. Después, los 288
tests de `core` pasaron sin tocar un solo import.

### Lo que se adopta de 12

| Qué                                              | Por qué                                                                                      |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `routeConflictPolicy: { duplicate: 'error', shadow: 'warn' }` | una ruta duplicada es siempre un error; corta el arranque en vez de la primera petición |
| `return503OnClosing: true`                       | la mitad del apagado ordenado que los hooks no cubren                                          |
| `errorCode` en el filtro global                  | un código de dominio sin escribir una excepción por cada uno                                   |

Las tres van dentro de `bootstrap()`, que es donde viven las decisiones que todos los servicios
tomaban igual. `routeConflictPolicy` se puede relajar con la opción `routeConflicts`.

### Lo que se deja fuera, y por qué

- **`@nestjs/observe`.** Es una plataforma alojada: manda telemetría a `observe.nestjs.com` con
  una app key y un secreto, no tiene exportador OTel y no se puede alojar uno mismo. La
  dirección es **OpenTelemetry contra el sidecar de OpenSearch propio**, así que no aplica. Lo
  que sí sirve de 12 es el gancho genérico `instrument: { instanceDecorator }`, que es público y
  cualquier envoltorio de OTel puede implementar; queda anotado y sin usar hasta que exista esa
  capa.
- **`ConsoleLogger` con `structuredParams` / `flattenParams`.** Se había anotado como útil para
  la forma del documento en OpenSearch, y **es un error**: la plataforma loguea por `pino`
  (`nestjs-pino`), y esas opciones sólo afectan al `ConsoleLogger` de NestJS. No tocan la
  ingesta.
- **Rspack** como bundler de monorepo. La plataforma compila con `tsc` y no es un monorepo de
  aplicaciones.
- **`routeResolutionStrategy: 'specificity'`.** Cambia el orden de registro de rutas; con el
  diagnóstico de conflictos encendido, primero conviene ver si hay algo que ordenar.

## Consecuencias

### Cambio incompatible: `@nestjs/config` cambia Joi por Standard Schema

`validationSchema` pasa a esperar un esquema
[Standard Schema](https://standardschema.dev/) -Zod, Arktype, valibot-. **Un servicio que traía
un esquema de Joi tiene que cambiarlo.**

La plataforma no depende de ninguno. Quien no quiera sumar una librería tiene dos salidas:
omitirlo y validar dentro de sus namespaces, o pasarle `validate` a `ConfigModule`, que es una
función `(config) => config` y no necesita nada instalado.

`NovaConfigModuleOptions.validationSchema` deja de ser `unknown` y toma el tipo que declara
`@nestjs/config`, derivado de su propia interfaz con
`NonNullable<ConfigModuleOptions['validationSchema']>` para no agregar una dependencia por un
tipo.

### El choque de peers que sólo se ve desde un servicio

`@nestjs/cli` 12 trae `chokidar` 5 y Angular DevKit 22; los schematics declaraban DevKit 20,
cuyo peer es `chokidar` ^4. **En el monorepo no aparece**, porque cada paquete resuelve su
propio árbol; en un servicio los dos caen en el mismo y el install corta con
`unmet peer chokidar`.

Es la lección operativa de esta migración: **`pnpm peers check` sobre el monorepo no alcanza**.
Lo encontró instalar el paquete publicado en el servicio de ejemplo, que es la única prueba que
ve el árbol aplanado que ve un consumidor.

### El piso de Node sube a `>=24.15`

No por NestJS, que pide `>= 20`, sino por Angular DevKit 22, que declara
`^22.22.3 || ^24.15.0 || >=26.0.0`. Mismo criterio que fijó
[ADR-016](ADR-016-node-version-objetivo.md): el número tiene que poder justificarse contra el
`engines` de alguna dependencia.

### Los tests tardan más en arrancar

El grafo de módulos es ESM y pesa más de cargar. El primer test de cada archivo lo paga: 740 ms
con la máquina libre, y visto cruzar los 5 s de límite por defecto de Vitest con el build y el
lint corriendo antes en la misma pasada. **El preset sube el límite a 20 s**, ajustable con
`timeoutMs`. Lo que se evita no es un test lento sino un fallo intermitente que se lee como un
defecto del código.

### Lo que no cambió

Los hooks de ciclo de vida ahora se invocan por jerarquía de componentes. **La plataforma no
implementa ninguno** -el apagado ordenado lo maneja terminus por dentro, con
`gracefulShutdownTimeoutMs`-, así que el cambio no la toca. Se revisó explícitamente.

## Referencias

- [ADR-016: Node.js 24 como Versión Objetivo](ADR-016-node-version-objetivo.md)
- [ADR-021: Framework de Testing](ADR-021-jest-testing.md), cuya urgencia venía de este ESM
- [ADR-025: Tres Paquetes NestJS en Lugar de Once](ADR-025-tres-paquetes-en-lugar-de-once.md)
- `ahincho/nova-nestjs` 0.8.0 y 0.8.1, la implementación
- Tipos leídos de `@nestjs/common` 12.0.1 y `@nestjs/config` 12.0.0 instalados, no de anuncios

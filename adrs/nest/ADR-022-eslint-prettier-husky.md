# ADR-022: Linter y Formateador

## Estado

**Parcialmente aceptada.** El linter está decidido e implementado: oxlint, publicado en
`@ahincho/nova-nestjs` 0.7.0. **El formateador se queda en Prettier**, con fecha de revisión
atada a que oxfmt publique su 1.0; hasta entonces no hay nada que ejecutar.

**Scope:** `nest`

> El archivo conserva el nombre `ADR-022-eslint-prettier-husky.md` porque es el destino de los
> enlaces que le apuntan desde [ADR-021](ADR-021-jest-testing.md) y desde el índice. Un ADR se
> identifica por su número, no por su nombre de archivo.

## Fecha

Medición: 2026-09-04 y 2026-09-06. Linter decidido e implementado: 2026-09-06. Formateador:
resuelto el 2026-09-06 como «Prettier por ahora», a revisar cuando oxfmt publique 1.0.

## Contexto

Los servicios usaban **ESLint con `typescript-eslint` y Prettier**, que es lo que trae el
arquetipo de NestJS. Frontend ya migró a Biome, así que la pregunta natural era si backend
debía seguirlo. Medido en frío sobre un servicio real de 70 archivos:

| Etapa          | Hoy            | Alternativa              | Diferencia |
| -------------- | -------------- | ------------------------ | ---------- |
| lint con tipos | ESLint 14.4 s  | oxlint + tsgolint 0.75 s | 19x        |
| formato        | Prettier 2.7 s | oxfmt 0.37 s             | 7x         |

**Biome quedó descartado para backend**, y por corrección y no por velocidad:

- No tiene la familia `no-unsafe-*`. Usa inferencia de tipos propia y parcial, así que no puede
  reemplazarlas. En un BFF esas reglas son el gate que impide que un `any` viaje desde la
  respuesta de un upstream hasta el controlador sin que nadie lo note.
- Rechaza los decoradores de parámetro -`@Res()`, `@Inject()`- salvo que se active
  `unsafeParameterDecoratorsEnabled`, y eso es todo NestJS.

**oxlint sí las tiene**, porque su motor con tipos (`oxlint-tsgolint`) está construido sobre
TypeScript 7. Sobre un archivo de prueba dio **los mismos 10 hallazgos** que ESLint. Además
`typescript-eslint` declara que no soporta TypeScript 7, así que el camino actual tenía fecha de
vencimiento propia.

## Decisión

### Linter: oxlint, implementado

El toolchain publica `oxlint/oxlintrc.json` en vez del preset plano de ESLint, y ya no instala
`eslint` ni `typescript-eslint`.

```json
// .oxlintrc.json del servicio
{ "extends": ["./node_modules/@ahincho/nova-nestjs-toolchain/oxlint/oxlintrc.json"] }
```

```json
// package.json
{ "scripts": { "lint": "oxlint --type-aware" } }
```

### Formateador: Prettier, y se vuelve a mirar cuando oxfmt llegue a 1.0

**Decidido que no se cambia todavía.** No es que oxfmt sea peor -midiéndolo no le encontramos
ningún defecto-, es que la ganancia no paga el costo de migrar dos veces.

Medido sobre el repositorio de la plataforma, 146 archivos versionados, las dos herramientas con
el mismo método de reloj y tres corridas cada una:

|                | Tiempo         | Difieren de Prettier            | Markdown           | YAML   |
| -------------- | -------------- | ------------------------------- | ------------------ | ------ |
| Prettier 3.9.6 | 1426 - 1492 ms | -                               | sí                 | sí     |
| oxfmt 0.66.0   | 931 - 1006 ms  | **1** (un tipo unión largo)     | sí, byte a byte    | sí     |
| Biome 2.5.12   | ~880 ms        | 5 (unión, arreglos JSON juntos) | **no, los ignora** | **no** |

El tiempo de Biome viene de la medición anterior, tomada con otro método, así que no es
comparable renglón a renglón con los otros dos. Da igual: Biome está descartado por lo que no
formatea, no por lo que tarda.

La única diferencia de oxfmt está en `packages/core/src/auth/tokens.ts`, un tipo unión largo que
parte en líneas con el pipe adelante y Prettier deja en una. Es cosmético, y discutiblemente más
legible el de oxfmt.

#### oxfmt no tiene el hueco de cobertura que tiene Biome

Es la comprobación que decide entre los dos, y hay que hacerla con archivos **deliberadamente
desordenados**: sobre un árbol ya formateado por Prettier, «no hay diferencias» no distingue
entre «formatea igual» y «no lo tocó».

Hecha así, con un archivo desordenado de cada uno de los seis tipos que hay en el repositorio
-Markdown, YAML, JSON, `.mjs`, `.d.mts` y `.ts`-, oxfmt produce **la misma salida byte a byte**
que Prettier en los seis. También conserva los comentarios del `.oxlintrc.json`.

**Biome, en ese mismo test, deja el Markdown y el YAML intactos**: los reporta como «rutas
provistas pero ignoradas». Para un repositorio cuya documentación son catorce archivos Markdown,
eso solo ya lo descarta, y por corrección y no por velocidad.

#### Por qué se espera igual

**oxfmt está en 0.66.0, publicada el 2026-09-01, con minors semanales y ningún 1.0 anunciado.**
Ese es todo el argumento. Un formateador existe para producir bytes estables; uno pre-1.0 puede
cambiar su salida entre versiones menores, y cada cambio de salida reescribe archivos en todo el
repositorio y ensucia el `blame`. Ese costo se paga una vez, así que conviene pagarlo cuando la
salida ya no se mueva.

**La ganancia son cinco décimas de segundo.** Comparar con el linter, que pasó de 14.4 s a
0.75 s: ahí el número justificaba solo la migración, acá no.

**Esperar no cuesta nada.** Prettier no está en riesgo de quedar sin mantenimiento y NestJS 12 no
trae formateador por defecto, así que quedarse no desalinea con nada.

#### Cuándo volver a mirarlo, y qué hacer entonces

El disparador es **oxfmt 1.0.0 en `latest`**. Cuando salga, el motivo para migrar no va a ser la
velocidad sino la consolidación: un solo proveedor para lint y formato, mismo proyecto Oxc, mismo
estilo de configuración, mismo `--migrate`. Es el mismo argumento que ya rindió con oxlint.

La migración ya está probada y es barata: `oxfmt --migrate=prettier` leyó el `.prettierrc`,
arrastró los patrones del `.prettierignore` y no inventó opciones. Nada que ver con
`biome migrate prettier`, que puso `semicolons: "asNeeded"` y borró todos los punto y coma de 76
archivos.

Al migrar hay que decidir una sola cosa: si se acepta el corte de oxfmt en los tipos unión largos
-que reformatea ese archivo- o si se busca la opción que lo evite. Se acepta, previsiblemente: el
punto de un formateador es no discutirle.

### Husky no se adopta

Un hook que comprueba tipos falla en un runner por razones que no tienen que ver con el código
-ya pasó en frontend, con `TS2307` porque los tipos de los módulos hermanos sólo existen con los
servidores de desarrollo levantados- y termina apagándose con una variable de entorno. El gate va
en CI, que es donde no se puede saltar con `--no-verify`.

## Consecuencias

### `--type-aware` no es opcional, y su ausencia no avisa

Las 23 reglas que necesitan tipos **sólo corren con `oxlint --type-aware`**, que a su vez
necesita `oxlint-tsgolint` instalado. Sin la bandera oxlint no falla ni advierte: simplemente no
las evalúa, y el reporte sale verde con la mitad del análisis sin hacer.

Es la consecuencia más peligrosa de esta decisión, porque el modo degradado es indistinguible del
modo correcto. Por eso el script se llama `oxlint --type-aware` y no `oxlint` a secas, y por eso
esconderlo detrás de un comando propio -`nova lint`- pasa de ser comodidad a ser una defensa.

### `@oxlint/migrate` descarta las reglas con tipos en silencio

La conversión automática produce una configuración que parece completa y no lo es: las 23 reglas
que importan hay que escribirlas a mano. Están listadas una por una en el preset, bajo su propio
comentario, para que se note si alguna se cae.

Es la misma familia de trampa que `biome migrate prettier` poniendo `semicolons: "asNeeded"` y
borrando todos los punto y coma. **Toda migración automática de configuración hay que diffearla.**

### `extends` de oxlint resuelve rutas, no paquetes

No se puede escribir `"extends": ["@ahincho/nova-nestjs-toolchain/oxlint"]`. La ruta va relativa
al archivo de configuración y entra a `node_modules`. Funciona de forma estable porque el
toolchain es una dependencia directa del servicio y pnpm le deja un enlace real en la raíz;
verificado instalando el paquete empaquetado, no en el monorepo.

Es una diferencia con ESLint y con Vitest, que sí resuelven especificadores de paquete, y es la
razón por la que la línea del consumidor se ve más fea de lo que uno esperaría.

### El monorepo ahora se lintea a sí mismo

Antes no lo hacía: publicaba un preset de linter que nunca corría sobre su propio código. La
primera pasada sobre 82 archivos tardó 1.1 s y encontró **siete hallazgos reales**, todos
corregidos:

- Un `Array.isArray` sobre un `readonly string[]` en `NovaConfigModule.forRoot`. Estrecha a
  `any[]`, así que la rama que parecía la segura era justo la que metía un `any` en el
  `envFilePath` que se le pasa a `@nestjs/config`. Es el que justifica el ejercicio.
- Cinco aserciones de tipo que no cambiaban nada.
- Un `async` sin `await` en un test de concurrencia, que al arreglarlo dejó a la vista que el
  `Promise.all` mezclaba una promesa con un valor sincrónico.

También hizo falta publicar `vitest/index.d.mts`: sin declaración de tipos, el `vitest.config.mjs`
de un consumidor importa un valor sin tipar y `no-unsafe-call` lo marca. Un preset sin tipos deja
de ser sólo incómodo cuando el linter con tipos lo mira.

### Positivas

- El lint deja de ser el paso lento de CI: 14.4 s a 0.75 s en un servicio, 1.1 s en el monorepo.
- Las reglas con tipos sobreviven a TypeScript 7.
- La plataforma corre sobre sí misma lo que le exige a sus consumidores.

### Negativas

- oxlint tiene menos plugins de terceros que ESLint. Hoy no usamos ninguno, pero acota el futuro.
- Cada servicio de A303 que ya tenga ESLint tiene que borrar su `eslint.config.mjs`, escribir el
  `.oxlintrc.json` y cambiar dos entradas de su `publicHoistPattern`.

## Referencias

- [ADR-021: Framework de Testing](ADR-021-jest-testing.md), la otra mitad del toolchain
- Medición sobre `A303-Nova_A303-14-backend-bff-courses`, 2026-09-04, ejecuciones en frío
- Medición del formateador sobre `ahincho/nova-nestjs`, 143 archivos, 2026-09-06
- `ahincho/nova-nestjs` 0.7.0, la implementación del linter
- `@utpxpedition/orbit`: el toolchain equivalente de frontend, sobre Biome

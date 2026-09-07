# ADR-019: TypeScript en Modo Estricto

## Estado

Aceptada (implementada)
**Scope:** `nest`
Reemplaza el placeholder que decía «TypeScript 5.x» y estaba sin redactar.

## Fecha

2026-09-07. En uso desde el primer commit; se escribe ahora junto con las banderas que se
agregaron después.

## Contexto

Un meta-framework publica tipos. Si sus tipos mienten, el error no aparece en el framework:
aparece en el servicio que confió en ellos, meses después, y ahí nadie sospecha del paquete.

`strict: true` es la línea de base obvia. Lo que no es obvio es qué más se enciende, y qué se
deja apagado a propósito.

## Decisión

**`strict: true` más cuatro banderas, en el preset `tsconfig/base.json` del toolchain**, que es
lo que extienden los tres paquetes y todo servicio generado.

| Bandera                       | Por qué                                                              |
| ----------------------------- | -------------------------------------------------------------------- |
| `noUncheckedIndexedAccess`    | `array[0]` es `T \| undefined`, que es la verdad                     |
| `noImplicitOverride`          | un `override` olvidado sobreescribe en silencio                      |
| `noFallthroughCasesInSwitch`  | el `break` que falta no se ve leyendo                                 |
| `useUnknownInCatchVariables`  | lo que se atrapa no es un `Error` hasta que alguien lo comprueba      |

Y dos que quedan **apagadas**, que es la parte que merece explicación:

**`exactOptionalPropertyTypes: false`.** Distinguir «ausente» de «presente y `undefined`» es
correcto, pero casi ninguna librería del ecosistema declara sus tipos así. Encendida, obliga a
escribir `...(x === undefined ? {} : { x })` en cada llamada a código de terceros, y eso es ruido
que esconde los errores de verdad en vez de mostrarlos.

**`strictPropertyInitialization: false`, sólo en el preset `nestjs`.** Un DTO se llena por
`class-transformer` y una entidad por el ORM: el compilador no puede ver esa inicialización, y la
alternativa es un `!` en cada campo, que no aporta nada y se copia sin pensar.

### `module: NodeNext`, no `CommonJS`

Es lo que hace que TypeScript resuelva como resuelve Node, con `exports` y condiciones incluidos.
Sin eso, un paquete que sólo declara la condición `import` -`dependency-cruiser`, por ejemplo-
compila acá y falla en ejecución.

### `isolatedModules: true`

Cada archivo tiene que poder transpilarse solo. Es obligatorio para Vitest, que transpila con Oxc
sin hacer chequeo de tipos, y sirve de recordatorio de que **el transpilador y el chequeador son
dos cosas distintas**: `nova verify` corre `typecheck` aparte por eso.

## Consecuencias

### Habilita las 23 reglas de lint con tipos

`typescript/no-unsafe-argument`, `no-floating-promises`, `no-misused-promises` y compañía sólo
pueden decir algo si los tipos son confiables. En un proyecto sin `strict` producen ruido; con
`strict` encuentran errores reales. Están en ADR-022.

### El `tsBuildInfoFile` va **dentro** del `outDir`

Suena a detalle y es una trampa cara. Por defecto el archivo de estado incremental queda al lado
del `tsconfig`, o sea fuera de `dist`. Entonces cualquier cosa que borre `dist` -el `deleteOutDir`
de nest-cli, un `rimraf`, alguien a mano- deja el estado diciendo que ya está todo compilado.

**El build siguiente no emite nada y termina con éxito.** El fallo aparece recién en el
contenedor, como un `MODULE_NOT_FOUND` sobre `dist/main.js`, donde nadie va a buscar un archivo
de caché.

```jsonc
"tsBuildInfoFile": "${configDir}/dist/tsconfig.tsbuildinfo"
```

Adentro se borran juntos y no pueden contradecirse. `${configDir}` lo resuelve el tsconfig que
extiende y existe desde TypeScript 5.5.

### Dos tsconfig en cada servicio, no uno

`tsconfig.json` incluye `src/` y `test/` con `rootDir: "."`, para que el typecheck vea los specs.
`tsconfig.build.json` estrecha a `./src`. Con uno solo, `rootDir: "./src"` más un `include` que
alcanza `test/` corta con TS6059, y `rootDir: "."` a secas emite el `dist` con una carpeta de más.

## Referencias

- `packages/toolchain/tsconfig/{base,nestjs}.json`
- ADR-022, por las reglas de lint que esto habilita
- ADR-021, por el runner que transpila sin chequear

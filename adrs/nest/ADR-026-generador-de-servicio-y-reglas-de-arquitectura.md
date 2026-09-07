# ADR-026: Generador de Servicio y Reglas de Arquitectura Ejecutables

## Estado

Aceptada (implementada)
**Scope:** `nest`

## Fecha

2026-09-07. Publicado en `@ahincho/nova-nestjs-schematics` 0.11.0, con el arreglo de 0.11.2.

## Contexto

Un servicio nuevo se creaba copiando el más parecido. Eso produce dos problemas distintos que se
suelen confundir.

El primero es el arranque: el `main.ts`, los dos `tsconfig`, el `vitest.config.mjs`, el
`nest-cli.json`. Se copian una vez y quedan viejos en silencio.

El segundo es más caro. **La arquitectura hexagonal de los ACL y de los BFF vive hoy en la cabeza
de quien revisa.** Que el `service` no importe el `adapter`, que un DTO del sistema externo no se
escape del suyo, que dos contextos acotados no se importen entre sí: son las reglas que hacen que
un ACL sea un ACL, y lo único que las sostiene es una revisión atenta un martes a las seis.

## Decisión

**Un schematic `service` que genera el servicio entero, con sus reglas de arquitectura como
archivo ejecutable.**

```bash
pnpm exec schematics @ahincho/nova-nestjs-schematics:service academic-acl --style acl
```

Deja 18 archivos y un servicio que pasa `nova verify` en limpio, en dos sabores: `acl` -hexagonal
por contexto acotado- y `bff` -features con los adaptadores de salida compartidos en
`src/upstream/`-.

### Las reglas son dependency-cruiser, no una convención

`.dependency-cruiser.js` es el equivalente de ArchUnit en Java, y `nova lint:arch` lo corre dentro
de `nova verify`. Nueve reglas en el ACL, cinco en el BFF, cada una con su motivo escrito.

**oxlint no puede expresarlas.** Tiene `no-restricted-imports`, pero filtra por el especificador
importado y no por dónde vive el archivo que importa, así que no sabe decir «el service no importa
el adapter, pero el module sí». Esa es la razón concreta de que haya una segunda herramienta.

### Ninguna regla enumera contextos

Es la decisión con más consecuencias del archivo:

```js
const CONTEXT = '[^/]+';
{
  name: 'service-must-not-import-adapter',
  from: { path: `^src/(${CONTEXT})/service/` },
  to: { path: '^src/$1/adapter/' },
}
```

Una regla que lista los contextos a mano **sigue en verde cuando aparece el octavo**, y nadie se
entera de que dejó de mirarlo. Ya pasó: en `19-backend-acl-app` dos reglas transversales enumeran
tres contextos y el repositorio tiene siete.

### El `name` en inglés, el `comment` en español

El `name` es un identificador: aparece en la salida y es la clave con la que un baseline de
`--ignore-known` referencia la regla. El `comment` lo lee una persona cuando la regla salta, así
que va en español, con sus tildes.

## Consecuencias

### Lo que el generador **no** genera es el argumento

No hay `src/common/` ni `src/core/`. El filtro global, el interceptor del sobre, las sondas, el
cliente HTTP, la configuración, el contexto de petición y el logger llegan dentro de
`@ahincho/nova-nestjs`.

En los templates de los que sale esta forma, esas dos carpetas eran **entre el 40 % y el 50 % de
`src`**. Un servicio nace con la mitad de los archivos que antes había que copiar y después
mantener sincronizados.

### El motor de plantillas emite CRLF en Windows

Aunque la plantilla en disco esté en LF. El generador aplica una regla `forEach` que normaliza a
LF antes de escribir, porque si no, el servicio recién generado **falla su propio
`format:check`**.

Eso arregla lo que se emite y no lo que pasa al clonar después, así que el servicio también nace
con un `.gitattributes` con `eol=lf`. Sin él, el mismo falso negativo vuelve en la primera
máquina Windows que lo clone.

### Un servicio generado se verifica en CI, y tiene que generarse fuera del repositorio

CI genera los dos sabores en limpio, les instala los tarballs empaquetados y les corre
`nova verify` entero. Es lo único que prueba el generador contra un árbol de dependencias real, y
lo único que ejercita `lint:arch`: el servicio de ejemplo está escrito a mano y no tiene reglas de
arquitectura.

Dos trampas encontradas al montarlo, las dos con la misma forma -pasar en verde sin hacer nada-:

- **La CLI de schematics hace dry-run sin decirlo** cuando cree que la colección es local.
  Imprime los `CREATE`, no escribe nada y sale con 0. Su prueba de «local» es de ruta POSIX, así
  que no se dispara en Windows y sí en un runner Linux. Va con `--no-dry-run` explícito.
- **El `.gitignore` de la plataforma alcanzaba al servicio generado dentro del repositorio.**
  oxlint sube el árbol buscando archivos de ignore, y `nova lint` terminaba con «No files found to
  lint». Se genera en `RUNNER_TEMP`, que además es donde vive un servicio de verdad.

### El generador todavía deja el esqueleto vacío

Un ACL nace sin ningún contexto acotado y un BFF sin ninguna feature. `nova generate feature`
llena cada uno, pero nada encadena los dos pasos. Es la deuda conocida de esta decisión.

## Referencias

- `packages/schematics/src/service/` y `packages/schematics/README.md`
- `.github/actions/generated-service-check/`
- ADR-012, por el estándar de calidad que `nova verify` hace cumplir

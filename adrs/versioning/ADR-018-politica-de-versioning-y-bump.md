# ADR-018: Politica de Versioning y Bump para Nova Platform

## Estado
Aceptada (implementada)
**Scope:** `versioning` (Java exclusivamente; NestJS tendra su propio ADR)

## Fecha
2026-07-09

## Contexto
Con la adopcion de Conventional Commits (ADR-006) y release-please (ADR-007) como herramientas de automatizacion, faltaba formalizar la **politica** de cuando y como bumpear versiones. Las decisiones estaban dispersas en el doc 06, en commits, y en conversaciones. Necesitamos un documento normativo que defina:

1. Version inicial para nuevos repos.
2. Cuando bumpear major/minor/patch.
3. Que estrategia de pre-release usar (si alguna).
4. Que hacer cuando un bug se descubre post-release.
5. Como manejar breaking changes en un ecosistema multi-repo con BOM coordinador.

## Decision

### 1. Version inicial: `1.0.0`

Nova Platform NO usa pre-1.0 (`0.x.y`). El primer release de cada repo es **`1.0.0`**, indicando que la API es considerada publica y estable desde el inicio.

**Excepcion unica:** una lib experimental que aun no tiene consumidores puede partir de `0.1.0` si se declara explicitamente en el README y en `.release-please-manifest.json`. Esta excepcion requiere aprobacion explicita en el PR que crea el repo.

**Justificacion:** en SemVer canonico, `0.x.y` significa "API inestable, cualquier cambio puede romper en el minor". Para Nova Platform, donde las libs se consumen desde otros repos del mismo ecosistema, la inestabilidad implicita de `0.x.y` genera confusion innecesaria. Preferimos `1.0.0` con breaking changes gestionados via `!` en commits (que fuerzan major bump).

### 2. Mapa de commits a bumps

| Commit type | Bump | Ejemplo |
|---|---|---|
| `feat` | minor | `1.0.0` -> `1.1.0` |
| `fix` | patch | `1.0.0` -> `1.0.1` |
| `perf` | patch | (no es un feature nuevo, es una mejora) |
| `refactor` | patch | (no cambia la API publica) |
| `docs`, `test`, `chore`, `ci`, `build`, `style` | **ninguno** | No dispara release |
| Cualquier tipo con `!` (breaking) | **major** | `1.0.0` -> `2.0.0` |
| Footer `BREAKING CHANGE:` | **major** | (alternativa al `!`) |

**Nota:** `release-please` detecta automaticamente el tipo de bump basandose en los commits desde el ultimo tag. NO requiere intervencion manual.

### 3. Estrategia de pre-release

Nova Platform **NO genera pre-releases** (`-RC1`, `-BETA`, `-SNAPSHOT`). El flujo es:

```
push feat: commit -> release-please abre PR -> review humano -> merge -> tag vX.Y.Z -> publish
```

Si un artefacto necesita testing antes de release, se usa:
- **Branch feature/preview** con CI que corre build + tests (sin publish).
- **`dry-run: true`** en los publish workflows para validar sin publicar.
- Nunca se publica un `-SNAPSHOT` o `-RC` a GitHub Packages.

**Justificacion:** Maven Central es inmutable. Publicar `1.0.0-RC1` y luego `1.0.0` genera confusion para consumidores que pinean a `1.0.0-RC1` (que es "mayor" que `1.0.0` en Maven version ordering). Evitar pre-releases simplifica el grafo de dependencias.

### 4. Bug descubierto post-release

Escenario: se publica `1.2.0` y se descubre un bug critico.

Flujo:
1. Developer crea branch `fix/critical-bug` desde `main`.
2. Push commit: `fix(mask-utils): fix null pointer in Peru strategy`.
3. PR a main, CI pasa.
4. Merge a main.
5. `release-please` detecta el `fix:` commit y abre PR con bump `1.2.0` -> `1.2.1`.
6. Reviewer aprueba y mergea.
7. Tag `v1.2.1` se pushea, `publish-on-tag.yml` publica.

**No se hacen hotfix branches** (no `release/1.2.x`). El trunk (`main`) es siempre la fuente de verdad. Si se necesita un patch a una version antigua (ej: `1.0.x` cuando ya existe `1.2.x`), se crea un branch `support/1.0.x` — pero esto no es esperado en Nova Platform por ahora.

### 5. Breaking changes con BOM coordinador

Escenario: `nova-java-mask-utils` va de `1.x` a `2.0.0` (breaking change).

Flujo de propagacion:
1. `mask-utils` publica `2.0.0` con `BREAKING CHANGE:` en commit.
2. `nova-bom` actualiza `<nova.mask.version>2.0.0</nova.mask.version>` en su POM.
3. `nova-bom` bumpea a la siguiente minor o major (decision del maintainer).
4. Consumidores que importan el BOM reciben `mask-utils:2.0.0` transitivamente.
5. Si el breaking change rompe a un consumidor directo, ese consumidor debe actualizar su codigo antes de actualizar el BOM.

**Regla:** el BOM **no skipea versiones breaking**. Si `mask-utils` sube a `2.0.0`, el BOM lo refleja. Los consumidores son responsables de adaptarse. El CHANGELOG de `mask-utils` documenta la migracion.

### 6. Configuracion en `.release-please-manifest.json`

Cada repo inicializa su manifest con la version actual (no la version "siguiente"). Ejemplo para un repo nuevo:

```json
{
  ".": "1.0.0"
}
```

Despues del primer release, `release-please` actualiza automaticamente este archivo al mergear el PR de release.

### 7. Tags

- Formato: `vX.Y.Z` (con prefijo `v`). Ejemplo: `v1.0.0`, `v2.3.1`.
- Los tags son inmutables. Nunca se mueve un tag existente.
- Los tags los crea `release-please` al mergear el PR de release.
- NO se crean tags manuales (excepto para bootstrap de repos sin historial).

## Consecuencias

### Positivas
- Flujo determinista: commit type -> bump type -> tag -> publish.
- Sin ambiguedad sobre que version es "estable" (todas lo son, no hay pre-releases).
- BOM se mantiene sincronizado con las libs (propagacion explicita).
- `release-please` automatiza todo el flujo de release sin intervencion manual.

### Negativas
- No hay mecanismo de "canary release" para testing incremental.
- Si se necesita soporte de versiones antiguas (backports), requiere crear branches `support/*` que no estan automatizados.
- Los consumidores deben estar preparados para breaking changes al actualizar el BOM.

## Alternativas consideradas

| Alternativa | Razon de rechazo |
|---|---|
| Pre-releases con `-RC` | Confunde el ordering de Maven; agrega complejidad sin beneficio para un meta-framework interno |
| Trunk-based con `0.x.y` perpetuo | No comunica estabilidad; cada minor podria romper |
| CalVer (`YYYY.MM.DD`) | No indica compatibilidad; rompe expectativas de Maven ecosystem |
| Manual bumps (sin automation) | Error-prone; inconsistente entre repos; causa del estado inicial de Nova (no-versioning) |

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.5 (flujo end-to-end)
- `docs/java/06-semantic-versioning-en-java.md` Seccion 11.8.3 (decision 1.0.0)
- `docs/java/06-semantic-versioning-en-java.md` Seccion 15.1 Decision #16 (version inicial)
- ADR-006 (Conventional Commits)
- ADR-007 (release-please)
- NOVA-SEMVER-18

# ADR-006: Conventional Commits y Semantic Versioning

## Estado
Aceptada (implementada)
**Scope:** `shared` (Java + NestJS)

## Fecha
2026-07-08

## Contexto
Java no tiene bump automatico nativo como `npm version`. Los commit messages eran libres, imposibilitando automatizar changelogs y version bumps. Necesitamos una convencion de commits universal que funcione en los 15 repos Java y en el resto del stack.

## Decision
Adoptar **Conventional Commits** (https://www.conventionalcommits.org/) como estandar en los 15 repos Java + multi-stack.

Formato: `<type>(<scope>): <description>`

Tipos y su efecto en semver:

| Tipo | Bump | Ejemplo |
|---|---|---|
| `feat` | minor | `feat(mask-utils): add Peru credit card strategy` |
| `fix` | patch | `fix(date-utils): fix DST bug in relative format` |
| `perf` | minor | `perf(api-standard): reduce wrapping overhead` |
| `refactor` | patch | `refactor(observability): simplify filter` |
| `docs` | none | `docs(readme): update install instructions` |
| `test` | none | `test(mask-utils): add property test` |
| `chore` | none | `chore(deps): bump spring-boot` |

Breaking change: `!` despues del scope o `BREAKING CHANGE:` en footer → bump **major**.

### Enforcement

- **Local:** `lefthook` (reemplazo multi-lenguaje de husky) ejecuta `commitlint --edit` en hook `commit-msg`.
- **CI:** `reusable-commitlint.yml` valida commits en PRs.

### Archivos creados en cada repo

- `commitlint.config.js` — extends `@commitlint/config-conventional`
- `lefthook.yml` — hook `commit-msg`
- `package.json` — devDependencies: `@commitlint/cli`, `@commitlint/config-conventional`

Version inicial: `0.1.0-SNAPSHOT` en todos los repos (reset desde `1.0.0` — no era production-ready, madurez 3.2/10).

## Consecuencias
### Positivas
- Changelogs auto-generados a partir del historial de commits.
- Bumps automaticos de version basados en el tipo de commit.
- Historial legible y consistente en todos los repos.
- Compatible con release-please para automatizacion de releases.

### Negativas
- Curva de aprendizaje para developers que no conocen la convencion.
- Node.js requerido como dev dependency para commitlint (incluso en repos Java).

## Referencias
- `docs/java/06-semantic-versioning-en-java.md` Seccion 7
- `docs/java/06-semantic-versioning-en-java.md` Seccion 8.7
- NOVA-SEMVER-01
- NOVA-SEMVER-02

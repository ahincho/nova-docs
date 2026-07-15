# 11. Plan de limpieza de pipelines y securización de tokens

> **Estado:** Pendiente de ejecución (Fase 1 sin riesgo, listo para implementar).
> **Fecha:** 2026-07-15.
> **Aplica a:** Equipo Nova, mantenedores de `nova-devops`, security audit de la org `ahincho`.

---

## 11.1 Contexto y diagnóstico

Auditoría realizada el 2026-07-15 sobre el repo `ahincho/nova-devops` y los 29 repos Nova
publicados en la org `ahincho`. Se identificaron **3 problemas concretos** + 1 backlog
(framework NestJS) que se detallan en este plan.

### 11.1.1 Universo auditado

| Categoría | Repos |
|---|---|
| Repos Nova publicados en `ahincho` | **29** (19 Java + 4 NestJS + 6 varios) |
| Workflows reusables en `nova-devops` | 18 reusables + 5 single-purpose |
| Composite actions en `nova-devops` | 6 (`nova-setup-{java,node,gpg}`, `nova-validate-build`, `nova-gather-facts`, `nova-publish-aggregator`) |
| Repos con secretos Nova configurados | 23 / 29 |

### 11.1.2 Problemas detectados (resumen ejecutivo)

| # | Problema | Severidad | Acción |
|---|---|---|---|
| 1 | 6 workflows `reusable-publish-{gradle,maven}-{maven-central,nexus,multi-registry}.yml` con **0 consumidores** | Media (deuda técnica) | Borrar |
| 2 | `nova-java-api-standard-quarkus-extension` tiene **ambos tokens** (`NOVA_PACKAGES_READ_TOKEN` + `NOVA_RELEASE_PAT`) — el segundo es redundante | Baja (redundancia) | Borrar uno |
| 3 | `NOVA_RELEASE_PAT` se usa **solo como fallback de read** en 19 repos, no para release operations (esas usan `GH_TOKEN`) | Media (segregación de privilegios) | Plan de migración gradual |
| 4 | 4 repos NestJS sin secretos ni pipelines | Backlog (fuera de alcance) | Trabajos futuros (§11.6) |

---

## 11.2 Problema 1: 6 workflows muertos (publish multi-registry)

### 11.2.1 Estado actual

Workflows **publicados en `ahincho/nova-devops/.github/workflows/`** pero **no consumidos por ningún repo Nova** (verificado el 2026-07-15 con búsqueda exhaustiva en los 29 repos):

| Workflow | Tamaño | Consumidores |
|---|---|---|
| `reusable-publish-gradle-maven-central.yml` | 2.5 KB | **0** |
| `reusable-publish-gradle-multi-registry.yml` | 4.1 KB | **0** |
| `reusable-publish-gradle-nexus.yml` | 3.0 KB | **0** |
| `reusable-publish-maven-maven-central.yml` | 3.1 KB | **0** |
| `reusable-publish-maven-multi-registry.yml` | 4.1 KB | **0** |
| `reusable-publish-maven-nexus.yml` | 3.5 KB | **0** |
| **TOTAL** | **20.2 KB** | **0** |

### 11.2.2 Por qué son deuda

- **YAGNI aplicado al revés**: se anticiparon 3 registries × 2 herramientas × escenarios
  futuros que **nunca se materializaron** (ningún repo Nova publica a Maven Central ni a
  Nexus hoy).
- **Costo de mantenimiento**: cada workflow tiene un comentario largo explicando scopes,
  secretos y permisos. Si el código cambia (p. ej. el patrón de secretos), hay que
  actualizar 6 archivos en lugar de 1.
- **Riesgo de seguridad por confusión**: un mantenedor podría asumir que estos workflows
  están "activos" y basar decisiones de seguridad en un modelo que no se usa.
- **Costo de revisión de PRs**: cuando se toca `nova-devops`, hay que validar 6 archivos
  innecesarios.

### 11.2.3 Acción concreta (Fase 1)

1. Commit en `ahincho/nova-devops` con mensaje:
   ```
   chore(workflows): remove 6 unused publish-multi-registry reusables

   - reusable-publish-gradle-maven-central.yml
   - reusable-publish-gradle-multi-registry.yml
   - reusable-publish-gradle-nexus.yml
   - reusable-publish-maven-maven-central.yml
   - reusable-publish-maven-multi-registry.yml
   - reusable-publish-maven-nexus.yml

   Zero consumers across all 29 Nova repos (audited 2026-07-15).
   Re-create on demand when a real consumer emerges.
   ```
2. Push a `main` en `nova-devops`.
3. Validar que ningún `release-please` PR de los 18 repos Gradle se rompa (no debería,
   porque ninguno referencia estos workflows).

### 11.2.4 Riesgo de la acción

**Cero**. Verificado:
- 0 referencias en `.github/workflows/` de `nova-devops`.
- 0 referencias en `.github/workflows/` de los 18 repos Java con Gradle.
- 0 referencias en `.github/workflows/` de los 4 repos NestJS (que ni siquiera tienen CI).
- 0 referencias en `.github/workflows/` de `nova-bom`, `nova-devops`, `nova-docs`,
  `nova-infrastructure`, `nova-java-example`, `nova-java-quarkus-example`,
  `nova-java-spring-boot-archetype`, `nova-java-spring-boot-parent`.

---

## 11.3 Problema 2: Token redundante en `nova-java-api-standard-quarkus-extension`

### 11.3.1 Estado actual

Único repo Nova con **ambos tokens** configurados:

```
Secret                                    Estado
─────────────────────────────────────────────────────────
NOVA_PACKAGES_READ_TOKEN                  ✅ configurado
NOVA_RELEASE_PAT                          ✅ configurado
```

### 11.3.2 Por qué quedó así (historia)

| Fecha | Evento |
|---|---|
| 2026-06 | Se introduce `NOVA_RELEASE_PAT` (PAT classic, scope `repo`) como único token para resolver deps cross-repo + tag push. |
| 2026-07-08 | Se introduce `NOVA_PACKAGES_READ_TOKEN` (scope solo `packages:read`) como alternativa read-only. Se configura en `nova-java-example` y `nova-java-quarkus-example` (instances). |
| 2026-07-12 | Durante debug del **ghost-publish** del extension Quarkus, se agrega `NOVA_PACKAGES_READ_TOKEN` al extension para verificar que el cross-repo read funcionara con un token read-only. |
| 2026-07-12 | Se resuelve el ghost-publish, pero `NOVA_PACKAGES_READ_TOKEN` queda configurado **sin remover** `NOVA_RELEASE_PAT`. |

Resultado: el extension tiene ambos, y como el fallback chain prefiere `NOVA_PACKAGES_READ_TOKEN`, es este el que realmente se usa. El `NOVA_RELEASE_PAT` está **muerto** en este repo.

### 11.3.3 Acción concreta (Fase 2)

Borrar `NOVA_RELEASE_PAT` del extension (es el legacy, no el canónico):

```bash
gh secret delete NOVA_RELEASE_PAT --repo ahincho/nova-java-api-standard-quarkus-extension
```

**Justificación de la elección**:
- `NOVA_PACKAGES_READ_TOKEN` es el canónico para read (menor blast radius).
- `NOVA_RELEASE_PAT` no se necesita porque el extension **publica a GitHub Packages
  usando `GH_TOKEN`** (que es el `GITHUB_TOKEN` auto del runner con permisos correctos
  vía el bloque `permissions:`).
- `NOVA_RELEASE_PAT` solo se justifica cuando se publique a Maven Central o Nexus
  (cross-org), y eso no está planeado.

### 11.3.4 Validación previa

Antes del borrado, validar que el extension funciona solo con `NOVA_PACKAGES_READ_TOKEN`:

1. Trigger manual de `reusable-build-gradle.yml` con un commit vacío.
2. Confirmar que el build pasa (debe leer `nova-notifications` desde otro repo).
3. Trigger manual de `reusable-publish-gradle.yml` simulando un tag.
4. Confirmar que el JAR se publica correctamente (usa `GH_TOKEN`, no `NOVA_RELEASE_PAT`).

Si ambos pasan → proceder con el borrado.

### 11.3.5 Riesgo de la acción

**Bajo**. El único riesgo es si el extension tiene alguna operación que requiera
`repo` scope (push de commits cross-repo, force-push, etc.). Verificado el 2026-07-15:
ninguna operación del extension requiere scope `repo` — solo `contents:read`,
`packages:write` y `packages:read` (cross-repo), todo cubierto por `GH_TOKEN` +
`NOVA_PACKAGES_READ_TOKEN`.

---

## 11.4 Problema 3: Separación de responsabilidades de tokens (análisis detallado, modelo estricto)

### 11.4.0 Regla del modelo estricto

**Solo 2 tokens de GitHub configurados**. Cero fallbacks. Si algo no entra en estos 2
tokens, se consulta al usuario antes de añadir nada.

| Token | Scope mínimo | Responsabilidad única |
|---|---|---|
| `NOVA_PACKAGES_READ_TOKEN` | `packages:read` | Resolver deps Nova publicadas en otros repos (read-only cross-repo). |
| `NOVA_RELEASE_PAT` | `repo` (PAT classic) **o** fine-grained `contents:write` + `pull-requests:write` + `packages:write` | Publicar releases y todo lo que requiera push/tag/PR en nombre de Nova. |

**Tokens externos permitidos (documentados, no son GitHub tokens)**:
- `NVD_API_KEY`: API key de NIST NVD para OWASP dependency check.
- `NOVA_SONAR_TOKEN`: token de SonarCloud (renombrado desde `SONAR_TOKEN` el 2026-07-15).
  **DORMANT** — no configurado en ningún repo, pero los reusables de Sonar ya lo
  referencian con warning si falta. Se activará cuando se integre SonarCloud.

Cualquier otro secret que se quiera añadir debe:
- Ser **una API key de un servicio externo** justificado, o
- **Ser notificado al usuario** antes de configurar.

### 11.4.1 Mapa actual de uso de tokens (con fallbacks)

Búsqueda exhaustiva el 2026-07-15 en todos los workflows de `nova-devops`:

**`NOVA_RELEASE_PAT`** se usa en **5 reusables**, siempre como fallback de read:

```
reusable-build-gradle.yml      (líneas 51, 54, 58, 73) — fallback TOKEN_B
reusable-build-matrix.yml      (líneas 52, 69)         — fallback packages-read-token
reusable-build-maven.yml       (línea 33)              — fallback packages-read-token
reusable-owasp-check.yml       (líneas 88, 232)        — fallback packages-read-token
reusable-sbom.yml              (líneas 58, 67)         — fallback packages-read-token
```

**`NOVA_PACKAGES_READ_TOKEN`** se usa en los **mismos 5 reusables**, como token primario:

```
reusable-build-gradle.yml      (líneas 50, 54, 74)     — token primario (TOKEN_A)
reusable-build-matrix.yml      (líneas 52, 69)         — token primario
reusable-build-maven.yml       (línea 33)              — token primario
reusable-owasp-check.yml       (líneas 88, 232)        — token primario
reusable-sbom.yml              (líneas 58, 67)         — token primario
```

**`GH_TOKEN`** (input `secrets.GH_TOKEN` en workflow_call, pasado por el caller) se usa en
**6 reusables + 1 single** para publish/release:

```
reusable-publish-gradle.yml        (L24 input, L69 use)         — publicación a GH Packages
reusable-publish-maven.yml         (L24 input, L67 use)         — publicación a GH Packages
reusable-release-please.yml        (L37 input, L53+L63 use)     — push PR de release + tag push
reusable-release-publish.yml       (L35 input, L113+L120 use)   — publicación post-release
reusable-version-bump-gradle.yml  (L30 use)                    — bump de versión
reusable-version-bump-maven.yml   (L37 use)                    — bump de versión
reusable-commitlint.yml            (L26 input)                  — push del commit de fix
nvd-mirror-update.yml              (L60, L93 use github.token)  — publica el mirror NVD a nova-devops
```

> **Nota importante**: `reusable-version-bump-{gradle,maven}.yml` recibe el secret con
> nombre `GH_PAT` (no `GH_TOKEN`) — ver L30/L37. Esto es una inconsistencia histórica
> a corregir en la migración.

### 11.4.2 Naturaleza del `NOVA_RELEASE_PAT` actual: **PAT personal**

Verificado el 2026-07-15 vía `GET https://api.github.com/user` con el propio token:

| Atributo | Valor |
|---|---|
| Tipo de token | **PAT classic** (no fine-grained) |
| Owner del token | `ahincho` (login personal = `Angel Eduardo Hincho Jove`) |
| Tipo de cuenta | `User` (no organización, no GitHub App) |
| Scopes otorgados | `repo`, `write:packages`, `delete:packages` |
| Scope `repo` cubre implícitamente | `repo:status`, `repo_deployment`, `public_repo`, `repo:invite`, `security_events`, `read:packages` |

**Esto significa que `NOVA_RELEASE_PAT`:**
1. **Es un PAT personal tuyo**, no un token dedicado ni una service account.
2. **Tiene scope `repo` completo** = acceso total a TODOS los repos donde está
   configurado (push, merge, force-push, eliminar refs, etc.), no solo release.
3. **Si tu cuenta personal se compromete**, todos los repos Nova quedan expuestos.
4. **Si rotás tu password / salís de la org**, los 19 builds que dependen de este token
   se rompen hasta que se configure uno nuevo.

### 11.4.3 Hallazgo clave (versión estricta)

**`NOVA_RELEASE_PAT` se usa HOY en 2 roles distintos** (modelo actual violá tu regla):

1. **Como fallback de read** (5 reusables) — el `NOVA_PACKAGES_READ_TOKEN` lo prefiere
   pero el `NOVA_RELEASE_PAT` está en el chain. **Esto es un fallback**, no permitido.
2. **Como identidad para publish** — pero los publish workflows reciben `GH_TOKEN` (que
   el caller pasa como `${{ secrets.GITHUB_TOKEN }}` o `${{ secrets.NOVA_RELEASE_PAT }}`
   según el repo).

**Doble falla del modelo actual**:

- **Falla de segregación**: el mismo PAT cubre read y release (con scopes diferentes
  necesarios). Un leak de la read surface expone también la write surface.
- **Falla de fallback**: si configuras `NOVA_RELEASE_PAT` solo (sin READ), el build
  funciona pero usa el PAT personal con scope `repo` para una operación de read.

### 11.4.4 Modelo objetivo estricto (2 tokens, sin fallbacks, sin GH_TOKEN)

Cada token tiene **una sola responsabilidad y una identidad dedicada, NO personal**:

| Operación | Token | Justificación |
|---|---|---|
| Resolver deps Nova cross-repo (starters → libs) | `NOVA_PACKAGES_READ_TOKEN` | Read-only. Sin este token, el build **falla explícitamente** (no fallback). |
| Push tag / crear PR / crear GitHub Release | `NOVA_RELEASE_PAT` | Scope `contents:write` + `pull-requests:write`. |
| Publicar JAR a GitHub Packages (mismo repo) | `NOVA_RELEASE_PAT` | Scope `packages:write`. |
| Publicar a **Maven Central** (futuro, cross-org) | `NOVA_RELEASE_PAT` | Scope `repo` completo + credenciales de Sonatype. |
| Publicar a **Nexus privado** (futuro, cross-org) | `NOVA_RELEASE_PAT` | Idem. |
| Version bump / commitlint fix / NVD mirror update | `NOVA_RELEASE_PAT` | Scope `contents:write` + `pull-requests:write`. |

**`GITHUB_TOKEN` auto del runner**: solo para system ops (checkout, setup-java, upload
artifacts, `gh` CLI). **NO** se pasa como `secrets.GH_TOKEN` en workflow_call para
release operations.

**`GH_PAT`** (input inconsistente en `reusable-version-bump-*`): renombrar a
`NOVA_RELEASE_PAT` en la migración.

### 11.4.5 Cambios concretos en workflows

#### Cambio A: Eliminar TODOS los fallbacks de read (5 reusables)

**Antes** (en cada uno de los 5 reusables):
```yaml
NOVA_PACKAGES_READ_TOKEN || NOVA_RELEASE_PAT || GITHUB_TOKEN
```

**Después** (sin fallback, falla explícita):
```yaml
NOVA_PACKAGES_READ_TOKEN  # REQUIRED. Sin esto el build falla con ::error::
```

Implementación práctica en `reusable-build-gradle.yml` (línea 50-66 → eliminar TOKEN_B
y TOKEN_C):

```yaml
- name: Validate NOVA_PACKAGES_READ_TOKEN
  id: validate_read_token
  env:
    TOKEN: ${{ secrets.NOVA_PACKAGES_READ_TOKEN }}
  run: |
    if [ -z "${TOKEN}" ]; then
      echo "::error::NOVA_PACKAGES_READ_TOKEN is required to resolve Nova cross-repo dependencies. Configure it in repo/org secrets. See docs/java/11 §11.4.4."
      exit 1
    fi
    echo "value=${TOKEN}" >> "$GITHUB_OUTPUT"
```

> **Nota**: Cambio A hace que los 18 repos con solo `NOVA_RELEASE_PAT` fallen
> inmediatamente. La migración debe ser: agregar READ primero, luego eliminar RELEASE
> (después de validar). Cambio A NO se aplica hasta que los 18 repos estén migrados.

#### Cambio B: Reemplazar `GH_TOKEN` (caller-provided) por `NOVA_RELEASE_PAT`

Workflows afectados: 6 reusables + 1 single-purpose.

**Estrategia**: el caller sigue pasando `secrets.GH_TOKEN` al reusable (por la
restricción de nombres de GitHub Actions), pero el valor que pasa debe ser
`secrets.NOVA_RELEASE_PAT`, no `secrets.GITHUB_TOKEN`. Esto renombra la responsabilidad
sin tocar el contrato del workflow_call (no se puede usar `NOVA_RELEASE_PAT` como nombre
de `workflow_call.secrets` porque colisionaría con el secret real al mapear).

Workflows a modificar:

```
reusable-publish-gradle.yml          → caller pasa GH_TOKEN: ${{ secrets.NOVA_RELEASE_PAT }}
reusable-publish-maven.yml           → caller pasa GH_TOKEN: ${{ secrets.NOVA_RELEASE_PAT }}
reusable-release-please.yml          → caller pasa GH_TOKEN: ${{ secrets.NOVA_RELEASE_PAT }}
reusable-release-publish.yml         → caller pasa GH_TOKEN: ${{ secrets.NOVA_RELEASE_PAT }}
reusable-commitlint.yml              → caller pasa GH_TOKEN: ${{ secrets.NOVA_RELEASE_PAT }}
reusable-version-bump-gradle.yml     → caller pasa GH_PAT: ${{ secrets.NOVA_RELEASE_PAT }}  (renombra GH_PAT → GH_TOKEN input, consistente)
reusable-version-bump-maven.yml      → caller pasa GH_PAT: ${{ secrets.NOVA_RELEASE_PAT }}
nvd-mirror-update.yml                → usar secrets.NOVA_RELEASE_PAT directo (single-purpose, no input restriction)
```

> **Decisión pendiente D7**: ¿es OK seguir usando el nombre `GH_TOKEN` como input del
> workflow_call (por la limitación de GitHub Actions) aunque el valor real venga de
> `NOVA_RELEASE_PAT`? La alternativa es forkear la action externa de release-please para
> que acepte un nombre de input distinto — mucho trabajo para un cambio cosmético.

#### Cambio C: Migrar `NOVA_RELEASE_PAT` a identidad dedicada (machine user / GitHub App)

**Bloqueante** para considerar el modelo "production-ready":

1. Crear **GitHub App** `nova-bot` (recomendado sobre machine user porque es más seguro:
   permisos granulares por repo, expiración de tokens, audit log separado).
2. Instalar la App en cada repo donde publique.
3. Otorgar a la App solo los permisos necesarios: `contents:write`, `pull-requests:write`,
   `packages:write`, `metadata:read`.
4. Generar el **private key** de la App y usarlo para autenticar en lugar de un PAT.
5. Documentar la rotación: el private key se regenera cada N meses sin invalidar builds
   (porque la App sigue existiendo).

> **Decisión pendiente D8**: ¿GitHub App o machine user?
> - **GitHub App**: más seguro (permisos granulares, no atado a persona), requiere
>   generar private key, integración con workflows via `actions/create-github-app-token@v1`.
> - **Machine user**: más simple (es un PAT de un user nuevo), pero sigue siendo PAT
>   classic con scope `repo`.

### 11.4.6 Inventario completo de configured secrets en los 29 repos

| Secret | # repos | Categoría | Acción |
|---|---|---|---|
| `NOVA_RELEASE_PAT` | 19 | GitHub token (PAT classic personal) | Mantener, migrar a identidad dedicada (Cambio C) |
| `NOVA_PACKAGES_READ_TOKEN` | 3 | GitHub token (read-only) | Mantener, propagar a los 18 restantes (Fase 3) |
| `NVD_API_KEY` | 14 | API key externa (NIST NVD) | Mantener (no es GitHub token) |
| `NOVA_SONAR_TOKEN` | **0** (dormant) | API key externa (SonarCloud) | Renombrado 2026-07-15. Dormant hasta integración de Sonar. Ver §11.10. |

**Sobre los externos**: el usuario confirmó el 2026-07-15 que Sonar cuenta como "token
extra" (3er configured secret después de los 2 GitHub). Por consistencia con la naming
convention, se renombró `SONAR_TOKEN` → `NOVA_SONAR_TOKEN` en
`reusable-sonarcloud-{gradle,maven}.yml`.

### 11.4.7 Estado actual vs objetivo (matriz repo por repo)

| Repos | `NOVA_RELEASE_PAT` actual | `NOVA_PACKAGES_READ_TOKEN` actual | Estado objetivo |
|---|---|---|---|
| 18 Java libs + devops + 3 demos | ✅ (PAT personal) | ❌ | Agregar READ, **luego** borrar RELEASE |
| `nova-java-api-standard-quarkus-extension` | ✅ (PAT personal) | ✅ | Borrar RELEASE (Fase 2) |
| `nova-java-example` | ❌ | ✅ | Ya correcto |
| `nova-java-quarkus-example` | ❌ | ✅ | Ya correcto |
| `nova-bom` (Maven) | ❌ | ❌ | No necesita |
| `nova-java-spring-boot-archetype` (Maven) | ❌ | ❌ | No necesita |
| `nova-infrastructure` | ❌ | ❌ | No es Java |
| `nova-docs` | ❌ | ❌ | No es Java |

### 11.4.8 Acción concreta (Fase 3 — modelo estricto, requiere decisión D3+D4)

**Pre-condición**: tener `NOVA_PACKAGES_READ_TOKEN` ya generado (D6).

1. **Configurar `NOVA_PACKAGES_READ_TOKEN`** en los 18 repos restantes (D6).
2. **Validar builds** repo por repo (commit vacío dispara CI; verifica resolución
   cross-repo via READ).
3. **Borrar `NOVA_RELEASE_PAT`** de los 18 repos (después de validar que ningún workflow
   de release/publish los necesita directamente — ver §11.4.5 Cambio B).
4. **Aplicar Cambio A** en los 5 reusables (eliminar fallback, falla explícita).
5. **Aplicar Cambio B** en consumer workflows de los 18 repos (cambiar `GH_TOKEN:
   ${{ secrets.GITHUB_TOKEN }}` → `GH_TOKEN: ${{ secrets.NOVA_RELEASE_PAT }}`).
6. **Aplicar Cambio C** (migrar RELEASE a identidad dedicada — bloqueante si se quiere
   salir del PAT personal).

### 11.4.9 Riesgos de la Fase 3 estricta

**Alto**. Si algún consumer pasa `GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}` y se aplica
Cambio A simultáneamente, el build falla inmediatamente porque `GH_TOKEN` (auto) no
tiene `packages:read` cross-repo. Mitigación:
- Hacer la migración repo por repo (no masiva).
- Rollback: restaurar `NOVA_RELEASE_PAT` en el repo que falla y aplicar Cambio A en
  orden inverso.
- Validar con un "dry-run": aplicar Cambio A pero temporalmente mantener el chain
  `NOVA_PACKAGES_READ_TOKEN || NOVA_RELEASE_PAT` (sin `GITHUB_TOKEN`), ver qué falla.

---

## 11.5 Plan de ejecución por fases

| Fase | Acción | Riesgo | Esfuerzo | Bloqueante |
|---|---|---|---|---|
| **1** | Borrar 6 workflows muertos | Cero | 5 min | No |
| **1.5** | Rename `SONAR_TOKEN` → `NOVA_SONAR_TOKEN` en `reusable-sonarcloud-{gradle,maven}.yml` | Cero (no hay repos con SONAR_TOKEN configurado) | 5 min | No |
| **2** | Borrar `NOVA_RELEASE_PAT` del extension Quarkus | Bajo | 10 min (incluye validación previa §11.3.4) | No |
| **3** | Migración de 18 repos a `NOVA_PACKAGES_READ_TOKEN` + invertir fallback (Cambio A) | Medio | 4-6 h (1 PAT + 18 configs + 5 cambios en reusables + validación) | Sí (decisión estratégica D3+D4) |
| **4** | Crear machine user / GitHub App `nova-bot` y migrar `NOVA_RELEASE_PAT` a identidad dedicada (Cambio C) | Bajo | 2-3 h | Sí (decisión estratégica D5) |
| **5** | Generar `NOVA_PACKAGES_READ_TOKEN` desde identidad dedicada (no personal) | Bajo | 1-2 h | Recomendado hacerlo junto con Fase 3 o 4 |
| **6** | Backlog NestJS (ver §11.6) | — | — | — |
| **7** | Activar `NOVA_SONAR_TOKEN` cuando se integre SonarCloud (ver §11.10) | — | — | Backlog |

**Recomendación**: ejecutar Fases 1, 1.5 y 2 esta misma sesión. Dejar Fases 3-5 como
**decisión estratégica separada** con plan documentado (§11.4.7).

---

## 11.6 Trabajos futuros: NestJS y otros repos sin CI/CD

### 11.6.1 Estado actual de los 4 repos NestJS

| Repo | Último push | Secrets configurados | Workflows |
|---|---|---|---|
| `nova-nestjs-commons` | 2026-07-08 | 0 | 0 |
| `nova-nestjs-observability-starter` | 2026-07-08 | 0 | 0 |
| `nova-nestjs-parent` | 2026-07-08 | 0 | 0 |
| `nova-nestjs-starter` | 2026-07-08 | 0 | 0 |

Los 4 repos NestJS están en `D:\Galaxy\Projects\nest\` pero **no se han tocado desde
el 2026-07-08**. No tienen CI/CD ni secrets. No son parte del plan actual.

### 11.6.2 Decisión pendiente sobre NestJS

Cuando se reactive el trabajo en NestJS (fecha indefinida), evaluar:

1. **¿Vale la pena migrar NestJS al mismo modelo de reusables que Java?** Hoy Java tiene
   18 reusables en `nova-devops`; NestJS no usa nada de eso.
2. **¿Necesita `NOVA_RELEASE_PAT` o `NOVA_PACKAGES_READ_TOKEN`?** Hoy no, porque no
   resuelve deps cross-repo desde CI.
3. **¿Publica a GitHub Packages?** Verificar: ¿hay un `package.json` con `publishConfig`
   apuntando a GitHub Packages?
4. **¿Hay un bus de eventos entre Java y NestJS?** Los 4 NestJS son `commons`,
   `observability`, `parent`, `starter` — su rol es paralelo al stack Java.

### 11.6.3 Otros repos no auditados

| Repo | Razón de exclusión |
|---|---|
| `D:\Galaxy\Projects\notification-parent\` | Repo de OscarBarahona (`github.com/OscarBarahona/notification-parent`), no parte de Nova. Es una referencia externa, no consumir. |
| `D:\Galaxy\Projects\jira\` | Tickets / docs internos, no código. |
| `D:\Galaxy\Projects\examples\archetypes\java-projects\quarkus-hexagonal-archetype\` | Referencia para Fase 1 de doc 08, no repo Nova. |

---

## 11.7 Validación previa al plan

Comandos para revalidar el estado antes de ejecutar cualquier fase:

```bash
# Validar workflows muertos (debe dar 0 referencias)
$token = "ghp_..."
$headers = @{ Authorization = "Bearer $token" }
Set-Location "D:\Galaxy\Projects\java"
$dirs = Get-ChildItem -Directory -Name
$matches = 0
foreach ($d in $dirs) {
  $wfFiles = Get-ChildItem -File $d/.github/workflows/*.yml -ErrorAction SilentlyContinue
  foreach ($f in $wfFiles) {
    $hits = Select-String -Path $f.FullName -Pattern "reusable-publish-(gradle|maven)-(maven-central|nexus|multi-registry)"
    $matches += $hits.Count
  }
}
"matches: $matches (debe ser 0)"

# Validar tokens por repo (19 RELEASE, 3 READ, 1 ambos)
foreach ($r in @("nova-bom", ..., "nova-nestjs-starter")) {
  $secrets = Invoke-RestMethod -Uri "https://api.github.com/repos/ahincho/$r/actions/secrets?per_page=100" -Headers $headers
  $has = $secrets.secrets | ForEach-Object { $_.name }
  "{0,-50} RELEASE={1} READ={2}" -f $r, ($has -contains "NOVA_RELEASE_PAT"), ($has -contains "NOVA_PACKAGES_READ_TOKEN")
}
```

---

## 11.8 Decisiones pendientes del usuario

| # | Decisión | Impacto si se ejecuta |
|---|---|---|
| **D1** | ¿Ejecutar Fase 1 (borrar 6 workflows muertos) esta misma sesión? | Riesgo cero. 5 min. |
| **D1.5** | ¿Ejecutar Fase 1.5 (rename `SONAR_TOKEN` → `NOVA_SONAR_TOKEN`)? | Riesgo cero. 5 min. |
| **D2** | ¿Ejecutar Fase 2 (borrar `NOVA_RELEASE_PAT` del extension Quarkus) esta misma sesión? | Riesgo bajo. 10 min (incluye validación previa §11.3.4). |
| **D3** | ¿Abrir Fase 3 (migración 18 repos a `NOVA_PACKAGES_READ_TOKEN`) como trabajo separado? | Riesgo medio. 4-6 h. |
| **D4** | Si D3=SI, ¿Cambio A opción A (masiva) o B (conservadora, gradual)? | Determina el modelo de riesgo de Fase 3. |
| **D5** | ¿Migrar `NOVA_RELEASE_PAT` a identidad dedicada (machine user / GitHub App `nova-bot`)? | Trabajo independiente. 2-3 h. Resuelve el riesgo de PAT personal con scope `repo`. |
| **D6** | ¿Generar el `NOVA_PACKAGES_READ_TOKEN` también desde identidad dedicada (no personal)? | Trabajo independiente. 1-2 h. Recomendado: SIEMPRE (no atar CI a identidad personal). |
| **D7** | ¿OK mantener `GH_TOKEN` como nombre del input del workflow_call (por limitación de GitHub Actions), aunque el valor venga de `NOVA_RELEASE_PAT`? | Cosmético. Si NO, requiere fork de release-please-action. |
| **D8** | Si D5=SI, ¿GitHub App o machine user? | Determina el approach técnico. |
| **D9** ✅ Resuelto 2026-07-15 | Sonar cuenta como 3er token externo. Renombrado a `NOVA_SONAR_TOKEN`. Dormant hasta integración futura (ver §11.10). | — |

**Recomendación**:
- **Esta sesión**: D1=SI, D1.5=SI, D2=SI (impacto inmediato, bajo riesgo).
- **Próxima sesión**: D3+D4+D5+D6+D7+D8 juntos como "Plan de hardening de tokens" (4-8 h).
- **Backlog**: NestJS (§11.6), Sonar integración futura (§11.10).

---

## 11.10 Activación futura de SonarCloud (`NOVA_SONAR_TOKEN`)

### 11.10.1 Estado actual (dormant)

El 2026-07-15 se renombró `SONAR_TOKEN` → `NOVA_SONAR_TOKEN` en los 2 reusables de
SonarCloud (`reusable-sonarcloud-gradle.yml`, `reusable-sonarcloud-maven.yml`). El
comportamiento sigue siendo: si el secret no está configurado, el step de análisis
emite un `::warning::` y continúa sin fallar.

**Ningún repo Nova tiene `NOVA_SONAR_TOKEN` configurado al 2026-07-15** (verificado en
el audit de los 29 repos). El reusables de Sonar:
- 13 repos consumen `reusable-sonarcloud-gradle.yml` (todos sin token, todos skip).
- 1 repo consume `reusable-sonarcloud-maven.yml` (`nova-java-notifications`, sin token,
  skip).

### 11.10.2 Plan de activación (cuando vos decidas)

Cuando se reactive la integración con SonarCloud:

1. **Crear cuenta de SonarCloud** para la org `ahincho` (gratis para proyectos públicos).
2. **Crear un proyecto SonarCloud** por cada repo Nova que publique (≈ 19 proyectos
   para los 15 Java libs + 3 demos + devops; las 4 NestJS no, salvo que también publiquen).
3. **Generar un token de SonarCloud** (no es un token de GitHub — está fuera del modelo
   de 2 tokens).
4. **Configurar `NOVA_SONAR_TOKEN`** en cada repo (secret a nivel de repo o de org):
   ```bash
   gh secret set NOVA_SONAR_TOKEN --repo ahincho/<repo> --body "<sonar-token>"
   # O a nivel org:
   gh secret set NOVA_SONAR_TOKEN --org ahincho --visibility all --body "<sonar-token>"
   ```
5. **Validar que el step de Sonar corre** (quitar el `::warning::`, ver análisis en
   SonarCloud dashboard).
6. **Documentar Quality Gates** en `docs/java/06-semantic-versioning-en-java.md` (o un
   doc nuevo): coverage mínimo, blocker issues, etc.

### 11.10.3 Decisiones pendientes para cuando se reactive

| # | Pregunta |
|---|---|
| ¿Sonar a nivel repo o nivel org? | Org reduce config (1 secret para todos) pero requiere que los proyectos SonarCloud existan antes. Repo da granularidad. |
| ¿Coverage mínimo como Quality Gate? | Si se establece, builds fallarán si coverage < X%. Hoy no hay enforcement. |
| ¿Aplicar también a las 4 NestJS? | Si se quiere calidad uniforme. Requiere crear proyecto SonarCloud para Node. |
| ¿Migrar a SonarQube self-hosted en lugar de SonarCloud? | Mayor control, pero requiere infra. |

### 11.10.4 Por qué NO se activó ahora

- **No había prisa**: el usuario (2026-07-15) confirmó que la integración es para
  "próximamente", no inmediata.
- **Costo cognitivo**: añadir un 4º configured secret antes de cerrar la migración de
  los 2 tokens principales habría desordenado el plan.
- **Workflows listos**: cuando se decida activar, solo se necesita generar el token y
  configurarlo en cada repo. No hay código nuevo que escribir.

---

## 11.9 Referencias

- `ahincho/nova-devops/.github/workflows/`: 23 workflows auditados.
- `ahincho/nova-devops/.github/actions/`: 6 composite actions.
- `docs/java/06-semantic-versioning-en-java.md` §11.9: historial de bugs y fixes de CI/CD.
- `docs/java/07-quarkus-analisis-adopcion.md` §13.4: cierre Fase 0 Quarkus.
- Auditoría de secrets: ejecutada vía `GET https://api.github.com/repos/ahincho/{repo}/actions/secrets`
  con PAT classic scope `repo` el 2026-07-15.
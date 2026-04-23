# Análisis Técnico: Remediación Arquitectónica Integral Post-Ratings

> Documento maestro de remediación derivado del análisis arquitectónico integral
> del proyecto Platziflix tras la entrega de la feature de Ratings.
>
> **Autor:** Agente Architect
> **Fecha:** 2026-04-23
> **Estado:** Propuesta formal — lista para ejecución
> **Audiencia:** Equipo Backend, Frontend Web, Mobile (Android / iOS), QA y Seguridad

---

## Problema

La entrega de la feature de Ratings introdujo valor de producto pero dejó al
sistema en un estado arquitectónico inconsistente. El análisis integral posterior
identificó cinco clases de defectos que comprometen la seguridad, la corrección
funcional, la paridad multiplataforma y la escalabilidad del backend:

1. **Vulnerabilidad de autenticación (HIGH):** los endpoints de ratings aceptan
   operaciones de escritura/lectura sin verificar identidad del llamante, lo que
   permite suplantación de usuarios, envenenamiento de la distribución de
   calificaciones y exfiltración de datos asociados a otros `user_id`.
2. **Semántica HTTP incorrecta (`204 vs 404`):** el endpoint
   `get_user_course_rating` devuelve `204 No Content` cuando el recurso no
   existe, en lugar de `404 Not Found`. Esto rompe el contrato REST, confunde a
   los clientes (Android/iOS interpretan ausencia de recurso como éxito vacío) y
   bloquea el flujo UX de "aún no has calificado".
3. **Paridad móvil rota:** Android consume la API de ratings con DTOs y flujos
   que divergen de los del cliente web; iOS aún no integra ratings y carece de
   specs. No existe un contrato compartido, por lo que cada cliente infiere el
   esquema.
4. **N+1 en `get_all_courses`:** el listado de cursos ejecuta una consulta por
   curso para obtener `avg_rating` y `rating_count`, lo que degrada el endpoint
   desde O(1) consultas a O(N). Con catálogo creciente, el p95 se deteriora de
   forma no lineal.
5. **Violación de capa en `get_class_by_id` y ausencia de cliente OpenAPI
   compartido:** la capa API accede directamente a ORM/repositorio saltándose el
   servicio; los tres clientes (web, Android, iOS) mantienen modelos manuales
   que ya divergen entre sí y del backend.

Estos defectos deben resolverse en un orden que minimice riesgo (seguridad
primero) y maximice reutilización (contrato compartido antes de nuevas
integraciones móviles).

---

## Impacto Arquitectural

### Backend (FastAPI + SQLAlchemy)
- **Nuevos/ajustes en capa API:** middleware/dependencia de autenticación
  aplicada a routers de ratings; cambio de status code en
  `get_user_course_rating`; eliminación del acceso directo a repositorio desde
  `get_class_by_id`.
- **Capa Service:** nuevo `RatingService` consolidando reglas (un rating por
  `(user_id, course_id)`), y `ClassService.get_by_id` delegado correctamente.
- **Capa Repository:** nuevo método agregado
  `list_courses_with_rating_stats()` que resuelve `avg_rating` y `rating_count`
  con una sola consulta (`LEFT JOIN ... GROUP BY` o subconsulta agregada).
- **Contrato OpenAPI:** enriquecimiento de `operation_id`, `response_model` y
  `responses` para que el cliente generado tenga nombres estables.

### Frontend Web (Next.js + TypeScript)
- Reemplazo progresivo de tipos manuales por tipos generados desde OpenAPI.
- Adaptación del hook `useUserCourseRating` al nuevo `404` (diferenciar
  "sin rating previo" de error real).
- Envío de token de autenticación en requests de ratings.

### Mobile — Android (Kotlin)
- Adopción del cliente generado desde OpenAPI (Kotlin client o Retrofit +
  modelos generados).
- Alineación de DTOs: renombrado de campos a snake_case esperado por backend o
  uso de `@SerializedName` consistente.
- Manejo explícito de `404` y `401` en el flujo de "mi rating".

### Mobile — iOS (Swift)
- Especificación formal del módulo de ratings (documento de features + mapping
  a endpoints).
- Generación de cliente Swift desde OpenAPI (Swift OpenAPI Generator o similar).
- Integración inicial: listado con `avg_rating`, envío/edición de rating,
  lectura de "mi rating".

### Base de Datos (PostgreSQL)
- Sin cambios de esquema destructivos. Se requieren:
  - Índice compuesto `idx_ratings_course_user (course_id, user_id)` si no
    existe, para soportar tanto la agregación como el lookup por usuario.
  - Índice `idx_ratings_course (course_id)` para acelerar el `GROUP BY` en la
    agregación masiva.
- Validación de cardinalidad: constraint único sobre `(user_id, course_id)`
  verificado (debería existir; si no, añadir migración).

---

## Seguridad

- **Modelo de amenazas cubierto:** suplantación (spoofing), repudiación,
  elevación de privilegios y exposición de datos ajenos en endpoints de
  ratings.
- **Controles a introducir:**
  - Dependencia `get_current_user` obligatoria en todos los endpoints de
    ratings (lectura `mi_rating`, escritura, actualización, borrado).
  - Autorización de recurso: el `user_id` del rating debe derivarse del token,
    nunca del body del request.
  - Rate limiting por usuario en creación/actualización de ratings (protección
    contra rating farming).
  - Logs de auditoría en creación/actualización/borrado (sin PII sensible).
- **Tests de seguridad obligatorios** (ver sección de tests de cada paso).

---

## Contratos de API (cambios normativos)

| Endpoint | Antes | Después |
|---|---|---|
| `GET /courses/{slug}/ratings/me` | `204 No Content` si no existe | `404 Not Found` con `{ "detail": "rating_not_found" }` |
| `POST /courses/{slug}/ratings` | Sin auth, `user_id` en body | Requiere `Authorization: Bearer`, `user_id` ignorado si viene en body |
| `PUT /courses/{slug}/ratings/me` | Sin auth | Requiere `Authorization: Bearer` |
| `DELETE /courses/{slug}/ratings/me` | Sin auth | Requiere `Authorization: Bearer` |
| `GET /courses` | N+1 (sin `avg_rating`/`rating_count` o con N+1) | Respuesta incluye `avg_rating: float \| null` y `rating_count: int` resueltos en una sola consulta |
| `GET /classes/{id}` | API accede a repo | API delega a `ClassService.get_by_id` |

Se publicará `openapi.json` versionado en el repositorio y se generarán
clientes para Web (TS), Android (Kotlin) e iOS (Swift) a partir de él.

---

## Propuesta de Solución

La solución sigue Clean Architecture estricta: **API → Service → Repository →
DB**, con un contrato OpenAPI canónico como fuente de verdad que alimenta a los
tres clientes. La remediación se ejecuta en cinco pasos priorizados por riesgo
y dependencia.

---

## Plan de Implementación

### Paso 1 — Cerrar vulnerabilidad de autenticación en endpoints de Ratings (HIGH)

**Justificación.** Es el único hallazgo clasificado HIGH. Mientras esté abierto,
cualquier otra corrección se despliega sobre una superficie insegura. Debe ir
primero porque: (a) bloquea el uso productivo de la feature, (b) los pasos 2 y
3 modifican los mismos handlers y conviene hacerlo sobre código ya securizado,
(c) evita reescribir tests dos veces.

**Archivos afectados (paths absolutos):**
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/api/routers/ratings.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/api/dependencies/auth.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/services/rating_service.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/schemas/rating.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/tests/api/test_ratings_auth.py`

**Pasos concretos:**
1. Definir/confirmar la dependencia `get_current_user` en
   `app/api/dependencies/auth.py` (extrae y valida JWT, devuelve `User`).
2. Añadir `Depends(get_current_user)` a **todas** las rutas de
   `ratings.py`: `POST`, `PUT`, `DELETE`, y `GET /me`.
3. En el service, forzar `rating.user_id = current_user.id` ignorando cualquier
   `user_id` que venga en el body (defensa en profundidad).
4. Actualizar `RatingCreate` y `RatingUpdate` para **no** aceptar `user_id`
   desde el cliente (removerlo del schema Pydantic).
5. Añadir rate limit por `user_id` (ej. 30 writes/min) vía middleware.
6. Registrar evento de auditoría estructurado en cada mutación.

**Criterios de aceptación:**
- Request sin `Authorization` a cualquier endpoint de ratings devuelve `401`.
- Request con token válido pero intentando escribir con `user_id` distinto al
  del token **termina persistiendo con el `user_id` del token** (no del body).
- Documentación OpenAPI muestra el requisito de auth (`security` scheme) en
  los endpoints afectados.

**Tests requeridos:**
- `test_rating_post_without_token_returns_401`
- `test_rating_post_with_spoofed_user_id_uses_token_user`
- `test_rating_put_other_user_rating_returns_403`
- `test_rating_delete_requires_owner`
- Test de carga ligero que verifique el rate limit.

---

### Paso 2 — Corregir semántica `204 ↔ 404` en `get_user_course_rating`

**Justificación.** Bug de contrato REST visible para los tres clientes.
Bloquea el flujo de UX "aún no calificaste este curso" en móvil (Android trata
`204` como éxito y no muestra el CTA). Debe ejecutarse después del Paso 1 porque
el endpoint ya estará protegido por auth y el test puede aislar el caso
"autenticado, sin rating previo".

**Archivos afectados:**
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/api/routers/ratings.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/services/rating_service.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/tests/api/test_ratings_me.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/frontend/src/features/ratings/hooks/useUserCourseRating.ts`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/frontend/src/features/ratings/api/ratings.client.ts`

**Pasos concretos:**
1. En `rating_service.get_user_course_rating`, lanzar
   `RatingNotFoundError` cuando no exista registro.
2. En el router, mapear esa excepción a
   `HTTPException(status_code=404, detail="rating_not_found")`.
3. Documentar el `404` en el decorador del endpoint
   (`responses={404: {"model": ErrorResponse}}`).
4. Actualizar `useUserCourseRating` en el frontend web para tratar `404` como
   "estado válido: sin rating", devolviendo `null` en lugar de lanzar.
5. Propagar el cambio al paso 3 (documentar contract-change para móvil).

**Criterios de aceptación:**
- `GET /courses/{slug}/ratings/me` con usuario sin rating devuelve `404` con
  body `{"detail": "rating_not_found"}`.
- Con rating existente devuelve `200` y el objeto `Rating`.
- El hook del frontend renderiza CTA "Calificar curso" sin mostrar error.

**Tests requeridos:**
- `test_get_my_rating_returns_404_when_absent`
- `test_get_my_rating_returns_200_with_payload_when_present`
- Test de frontend (RTL) que verifica render del CTA ante `404`.

---

### Paso 3 — Specs móviles + integración de Ratings en Android/iOS

**Justificación.** Restaura paridad multiplataforma y evita que cada equipo
móvil reinvente contratos. Va después de los pasos 1 y 2 porque las specs
deben referenciar el contrato ya corregido (auth obligatorio, `404` correcto).
Ir antes duplicaría trabajo.

**Archivos afectados:**
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/spec/mobile/android/01_ratings_integration.md` (nuevo)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/spec/mobile/ios/01_ratings_integration.md` (nuevo)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/mobile/android/app/src/main/java/com/platziflix/ratings/` (refactor)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/mobile/ios/Platziflix/Features/Ratings/` (nuevo módulo)

**Pasos concretos:**
1. Redactar spec Android con: pantallas afectadas (detalle de curso), flujos
   (crear, editar, borrar, ver mi rating), manejo de `401/404`, componentes UI,
   estados de carga/error, métricas.
2. Redactar spec iOS equivalente con las mismas secciones y mapeo a
   SwiftUI/UIKit según convención del repo.
3. Android: migrar DTOs actuales al modelo generado (Paso 5) — o, si el Paso 5
   aún no está listo, alinear manualmente los nombres de campo al contrato
   backend canónico.
4. iOS: implementar capa `RatingsRepository` + `RatingsViewModel` + vistas.
5. Ambos clientes deben inyectar el token en el `Authorization` header.

**Criterios de aceptación:**
- Android e iOS muestran `avg_rating` y `rating_count` en el detalle de curso.
- Ambos permiten crear/editar/borrar rating propio con UX equivalente al web.
- Ambos diferencian correctamente "sin rating" (`404`) de "error real".
- Specs aprobadas por Product y Arquitectura antes de mergear código.

**Tests requeridos:**
- Android: tests unitarios de `RatingsRepository` + test de UI
  (Espresso/Compose) para los estados vacío/con-rating/error.
- iOS: tests XCTest del ViewModel + snapshot tests de las vistas.
- Contract tests comparando el fixture de respuesta de backend con el parser
  de cada cliente móvil.

---

### Paso 4 — Refactor N+1 en `get_all_courses` a agregación SQL

**Justificación.** Problema de performance que crece con el catálogo. Se hace
tras los pasos 1–3 porque: (a) no es un defecto de corrección funcional sino de
eficiencia, (b) los clientes móviles ya consumirán el endpoint corregido y no
hay beneficio en optimizar antes de que el contrato esté alineado, (c) permite
medir el impacto con clientes reales integrados.

**Archivos afectados:**
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/repositories/course_repository.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/services/course_service.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/schemas/course.py`
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/alembic/versions/xxxx_rating_indexes.py` (nuevo)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/tests/repositories/test_course_repository_perf.py`

**Pasos concretos:**
1. Crear migración Alembic que añada:
   - `idx_ratings_course_user` sobre `ratings(course_id, user_id)` si no existe.
   - `idx_ratings_course` sobre `ratings(course_id)`.
2. Implementar `CourseRepository.list_with_rating_stats()` usando una sola
   consulta, equivalente a:
   ```sql
   SELECT c.*,
          COALESCE(AVG(r.score), NULL) AS avg_rating,
          COUNT(r.id) AS rating_count
   FROM courses c
   LEFT JOIN ratings r ON r.course_id = c.id
   GROUP BY c.id
   ORDER BY c.created_at DESC;
   ```
   Implementación SQLAlchemy con `func.avg`, `func.count`, `outerjoin`,
   `group_by(Course.id)`.
3. Reemplazar el uso del método anterior en `CourseService.get_all_courses`.
4. Extender el `CourseRead` schema con `avg_rating: float | None` y
   `rating_count: int`.
5. Añadir benchmark reproducible que compare el antes/después en un dataset
   seed de N=500 cursos.

**Criterios de aceptación:**
- `GET /courses` ejecuta exactamente **1** consulta principal
  (verificable con `SQLAlchemy echo` o middleware de conteo).
- p95 del endpoint mejora al menos 60% en el dataset de benchmark.
- La respuesta incluye `avg_rating` y `rating_count` con semántica clara
  (`avg_rating=null` cuando `rating_count=0`).

**Tests requeridos:**
- `test_list_courses_runs_single_query` (usa listener de eventos SQLAlchemy
  para contar queries).
- `test_list_courses_returns_null_avg_when_no_ratings`.
- `test_list_courses_returns_correct_aggregates`.
- Benchmark en CI opcional (umbral duro p95 < 200ms con 500 cursos).

---

### Paso 5 — Cliente OpenAPI compartido (Web + Android + iOS)

**Justificación.** Cierra la causa raíz del mismatch DTOs: tener tres
representaciones manuales del mismo contrato. Va último porque requiere que el
contrato ya esté estabilizado (pasos 1, 2 y 4 lo modifican) y que existan las
specs móviles (paso 3) para saber qué superficies del cliente generar.

**Archivos afectados:**
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/scripts/export_openapi.py` (nuevo)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/contracts/openapi.json` (nuevo, artefacto versionado)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/frontend/src/api-client/` (generado)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/mobile/android/app/src/main/java/com/platziflix/apiclient/` (generado)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/mobile/ios/Platziflix/APIClient/` (generado)
- `/home/ja54312/PROYECTOS/CURSOS/claude-code/.github/workflows/openapi-contract.yml` (nuevo)

**Pasos concretos:**
1. Añadir `operation_id` explícito y `response_model` a todos los endpoints
   del backend para obtener nombres estables en el schema generado.
2. Crear `scripts/export_openapi.py` que serialice `app.openapi()` a
   `contracts/openapi.json`.
3. Integrar generadores:
   - Web: `openapi-typescript` o `openapi-typescript-codegen`.
   - Android: `openapi-generator-cli` con template `kotlin` + `retrofit2`.
   - iOS: `swift-openapi-generator` (oficial de Apple).
4. Reemplazar progresivamente los tipos/clientes manuales en cada plataforma.
5. Añadir job CI `openapi-contract` que:
   - Regenera `openapi.json` desde el código.
   - Falla el build si difiere del versionado (forzando PR consciente).
   - Regenera clientes y corre tests de los tres proyectos.

**Criterios de aceptación:**
- `contracts/openapi.json` existe, está versionado y actualizado.
- Ninguno de los tres clientes mantiene DTOs manuales para endpoints cubiertos
  por la generación.
- PR que cambia un endpoint sin actualizar `openapi.json` es rechazado por CI.

**Tests requeridos:**
- Contract test en CI: `openapi diff` entre artefacto versionado y generado.
- Smoke test por plataforma consumiendo un endpoint real con el cliente
  generado.
- Test de tipos TS (`tsc --noEmit`) y Kotlin/Swift build verde.

---

## Paso transversal — Violación de capa en `get_class_by_id`

Aunque no estaba entre los 5 priorizados como encabezado, es un pre-requisito
de higiene arquitectónica que debe ejecutarse **junto con el Paso 1** (mismo
PR o inmediatamente posterior) porque es un cambio trivial que evita perpetuar
el antipatrón cada vez que se toque el router:

- Archivo: `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/api/routers/classes.py`.
- Mover la lógica al service
  `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/app/services/class_service.py`.
- Tests en
  `/home/ja54312/PROYECTOS/CURSOS/claude-code/backend/tests/services/test_class_service.py`.

---

## Orden de Ejecución y Dependencias

```
[1] Auth en Ratings (HIGH)  ──►  [2] 204→404 en mi-rating  ──►  [3] Specs + Integración móvil
         │                                │                              │
         └──────────────┬─────────────────┘                              │
                        ▼                                                 │
                [4] Refactor N+1 en /courses  ◄──────────────────────────┘
                        │
                        ▼
                [5] OpenAPI client compartido
```

**Racional del orden:**

1. **Seguridad primero (Paso 1).** Cualquier otra mejora desplegada sobre
   endpoints inseguros amplifica el blast radius de la vulnerabilidad.
2. **Corrección funcional de contrato (Paso 2).** Con auth ya aplicada, el
   `404` tiene significado unívoco ("autenticado, sin recurso") y no se
   confunde con el caso "no autorizado". Hacerlo antes del Paso 1 habría
   requerido rehacer los tests.
3. **Paridad móvil (Paso 3).** Las specs y la integración deben apuntar al
   contrato ya corregido (auth + `404`). Invertir 2 y 3 obligaría a Android/iOS
   a implementar dos veces.
4. **Performance (Paso 4).** No bloquea funcionalidad pero sí experiencia.
   Se posterga hasta que el contrato esté estable para evitar tocar el mismo
   endpoint múltiples veces y para medir con clientes reales integrados.
5. **Contrato compartido (Paso 5).** Requiere que los pasos 1, 2 y 4 hayan
   modificado el schema a su forma final; de lo contrario, los clientes
   generados quedarían obsoletos al día siguiente y habría que regenerarlos
   múltiples veces, erosionando la confianza del equipo en el flujo de
   generación.

El paso transversal (violación de capa en `get_class_by_id`) se ejecuta en el
mismo PR que el Paso 1 por ser mecánico y estar en el mismo dominio de revisión.

---

## Criterios de Aceptación Globales

- Cero endpoints de ratings accesibles sin autenticación.
- Ningún endpoint del dominio ratings devuelve `204` para ausencia de recurso.
- `GET /courses` ejecuta 1 sola query principal y expone `avg_rating` y
  `rating_count`.
- Android e iOS consumen ratings con specs aprobadas y con paridad funcional
  respecto al cliente web.
- `contracts/openapi.json` está versionado y es la única fuente de verdad de
  DTOs en los tres clientes.
- Todos los routers respetan la regla API → Service → Repository (ningún
  acceso directo a repositorio desde la capa API).
- Cobertura de tests para cada paso >= 85% sobre el código tocado.

---

## Riesgos

| # | Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|---|
| R1 | Clientes en producción rompen al cambiar `204`→`404` antes de que actualicen | Media | Alto | Feature flag temporal `ratings_404_enabled` y release coordinado; anuncio a consumidores. |
| R2 | Introducir auth rompe a usuarios anónimos que hoy leen `avg_rating` | Baja | Medio | Auth obligatoria solo para endpoints de ratings por-usuario; lectura agregada sigue pública. |
| R3 | La agregación SQL degrada otro query por bloqueo/índice | Baja | Medio | Benchmark antes/después; índices creados en la misma migración; rollback plan documentado. |
| R4 | Generadores OpenAPI producen código idiomáticamente pobre en Swift/Kotlin | Media | Medio | PoC previo con un endpoint piloto antes de migración masiva. |
| R5 | Drift entre `openapi.json` versionado y el código si el CI check falla en fin de sprint | Media | Bajo | Gate en CI es bloqueante; runbook de regeneración en el README del repo. |
| R6 | Specs móviles demoran la feature paridad | Media | Medio | Time-box de 3 días para specs; revisión asíncrona con template estándar. |
| R7 | Rate limit afecta a usuarios legítimos con multi-dispositivo | Baja | Bajo | Umbral conservador (30 writes/min) + métrica de `429` por usuario. |

---

## Métricas de Éxito

- **Seguridad:** 0 incidencias de suplantación reportadas post-deploy; 100% de
  requests a endpoints de ratings con `Authorization` válido.
- **Contrato:** 0 tests de contrato rojos durante 30 días consecutivos.
- **Performance:** `p95(GET /courses) < 200ms` con 500 cursos seed;
  `queries/request == 1` en el endpoint.
- **Paridad:** ambos clientes móviles liberados con feature completa en
  la misma ventana de release.
- **Reutilización:** 0 DTOs manuales en los tres clientes para endpoints
  cubiertos por OpenAPI.

---

## Entregables

- Migración Alembic con índices de ratings.
- `RatingService` y `ClassService` con lógica completa.
- `CourseRepository.list_with_rating_stats`.
- Specs móviles (`spec/mobile/android/01_ratings_integration.md`,
  `spec/mobile/ios/01_ratings_integration.md`).
- `contracts/openapi.json` versionado + pipelines de generación.
- Test suites por paso según se detalla arriba.
- Runbook de despliegue coordinado (R1).

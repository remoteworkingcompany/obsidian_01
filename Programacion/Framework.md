# links
[[LegalCorpus]], [[Knowledge BC]], [[Tabular Ingestion]]
# Arquitectura de Referencia — Framework Laravel DDD

**Versión:** 1.0
**Fecha:** Abril 2026

Documento de arquitectura para nuevos proyectos. Define las decisiones de infraestructura de software, los patrones de comportamiento del sistema y las convenciones de frontend. La IA y los desarrolladores usan este documento como contrato: todo lo que se construye respeta estas reglas.

---

## Índice

0. [Fase 0 — Definición del producto antes de escribir código](#fase-0--definición-del-producto-antes-de-escribir-código)
1. [Capa 0 — Lenguaje y Framework](#capa-0--lenguaje-y-framework)
2. [Capa 1 — Scaffolding de aplicación](#capa-1--scaffolding-de-aplicación)
3. [Capa 2 — Librerías frontend transversales](#capa-2--librerías-frontend-transversales)
4. [Capa 3 — Arquitectura DDD + Hexagonal](#capa-3--arquitectura-ddd--hexagonal)
   - [3.1 Estructura de carpetas](#31-estructura-de-carpetas)
   - [3.2 Domain](#32-domain)
   - [3.3 Application: handlers atómicos + orquestadores](#33-application-handlers-atómicos--orquestadores)
   - [3.4 Infrastructure](#34-infrastructure)
   - [3.5 Bindings](#35-bindings)
5. [Capa 3B — Patrones de comportamiento del sistema](#capa-3b--patrones-de-comportamiento-del-sistema)
   - [Patrón 1 — Toda operación es simple, pipeline o BPMN](#patrón-1--toda-operación-es-simple-pipeline-o-bpmn)
   - [Patrón 2 — EventBus como columna vertebral](#patrón-2--eventbus-como-columna-vertebral)
   - [Patrón 3 — Broadcasting como consecuencia](#patrón-3--broadcasting-como-consecuencia)
   - [Patrón 4 — Audit en tres capas](#patrón-4--audit-en-tres-capas)
   - [Patrón 5 — El controller es tonto](#patrón-5--el-controller-es-tonto)
   - [Patrón 6 — Los jobs son puntos de entrada, no lógica](#patrón-6--los-jobs-son-puntos-de-entrada-no-lógica)
   - [Patrón 14 — Servidor MCP como puerto de entrada para LLMs](#patrón-14--servidor-mcp-como-puerto-de-entrada-para-llms)
   - [Patrón 15 — LlmChat como puerto de salida para conversaciones con tool calling](#patrón-15--llmchat-como-puerto-de-salida-para-conversaciones-con-tool-calling)
   - [Patrón 16 — Reglas de negocio como objetos de dominio testeables](#patrón-16--reglas-de-negocio-como-objetos-de-dominio-testeables)
   - [Patrón 17 — Asistente conversacional in-app con confirmación humana de side effects](#patrón-17--asistente-conversacional-in-app-con-confirmación-humana-de-side-effects)
   - [Patrón 18 — Observabilidad técnica como puerto directo (fuera del EventBus)](#patrón-18--observabilidad-técnica-como-puerto-directo-fuera-del-eventbus)
   - [Patrón 19 — Strategy + Registry para variabilidad intra-dominio](#patrón-19--strategy--registry-para-variabilidad-intra-dominio)
6. [Capa 4 — Frontend](#capa-4--frontend)
   - [4.1 Estructura de archivos](#41-estructura-de-archivos)
   - [Patrón 7 — API Layer separado](#patrón-7--api-layer-separado)
   - [Patrón 8 — Composables como puente entre backend y UI](#patrón-8--composables-como-puente-entre-backend-y-ui)
   - [Patrón 9 — Container / Presenter](#patrón-9--container--presenter)
   - [Patrón 10 — Carga de datos de las páginas](#patrón-10--carga-de-datos-de-las-páginas)
   - [Patrón 11 — Estado compartido con Pinia](#patrón-11--estado-compartido-con-pinia)
   - [Patrón 12 — Componentes transversales](#patrón-12--componentes-transversales)
   - [Patrón 13 — Página reactiva por suscripción](#patrón-13--página-reactiva-por-suscripción)
7. [Apéndice A — Riesgos y decisiones conscientes](#apéndice-a--riesgos-y-decisiones-conscientes)
8. [Apéndice B — Convenciones de nombrado](#apéndice-b--convenciones-de-nombrado)

---

## Fase 0 — Definición del producto antes de escribir código

**Nada de lo que viene después sirve si no se completa esta fase.** Este framework es una cáscara vacía sin una definición previa del producto. Antes de crear una carpeta, antes de instalar un paquete, antes de tocar una línea de código, hay que responder a estas preguntas. Las respuestas son el input que alimenta toda la arquitectura.

### 0.1 Qué problema resuelve el producto

Una descripción en lenguaje de negocio, no técnico, de qué hace el sistema y para quién. Si no cabe en un párrafo, no está claro. Ejemplos reales:

- "Monitoriza la actividad de LinkedIn de empresas objetivo y detecta señales de compra para comerciales."
- "Gestiona clusters de landing pages con deploy automático y formularios de captación de leads."
- "Procesa emails de cuentas conectadas, los clasifica con IA y extrae datos financieros."

De aquí salen los bounded contexts. Cada verbo principal del párrafo es un candidato a contexto.

### 0.2 Bounded contexts

Identificar los contextos del dominio. Cada contexto es un área de negocio con su propio lenguaje, sus propias entidades y sus propias reglas. No son módulos técnicos, son áreas de significado.

Para cada contexto, definir:

| Campo | Qué es | Ejemplo (LinkedIn) |
|---|---|---|
| Nombre | El área de negocio | Company, Contact, Signal, Post |
| Responsabilidad | Qué gestiona este contexto, en una frase | "Gestión del ciclo de vida de empresas monitorizadas" |
| Entidades | Los objetos con identidad propia | Company, CompanyCluster |
| Value Objects | Objetos sin identidad, definidos por sus atributos | CompanyId, ContactTier, SignalStrength |
| Enums | Estados y tipos finitos | CompanyStatus, ContractStatus |
| Dependencias externas | Qué servicios externos necesita | API LinkedIn, LLM, Storage |

**Cómo identificar bounded contexts:** si dos conceptos usan la misma palabra con significado diferente, están en contextos distintos. Si un cambio en un concepto no afecta a otro, están en contextos distintos. Si no se puede explicar uno sin explicar el otro, están en el mismo contexto.

**Contexto Shared:** siempre existe. Contiene lo transversal: EventBusPort, DomainEvent base, contracts compartidos (ActivityLoggableEvent, NotifiableEvent).

### 0.3 Entidades y sus relaciones

Para cada entidad, definir:

| Campo | Qué es |
|---|---|
| Nombre | Nombre de la entidad |
| Contexto | A qué bounded context pertenece |
| Identidad | Qué la identifica (id numérico, UUID, slug...) |
| Atributos principales | Los campos que la definen (no todos, los importantes) |
| Estados | Si tiene ciclo de vida, cuáles son los estados (enum) |
| Relaciones | Con qué otras entidades se relaciona y cómo (1:N, N:M, pertenece a) |
| Eventos que emite | Qué pasa cuando se crea, modifica, elimina (nombres de DomainEvent) |

Este ejercicio fuerza a pensar en el dominio antes de pensar en tablas. La tabla es una consecuencia de la entidad, no al revés. Si alguien empieza diseñando tablas en vez de entidades, está haciendo CRUD con carpetas bonitas, no DDD.

### 0.4 Ports: dependencias externas

Listar todas las dependencias externas del sistema. Cada una será un Port (interface en Domain) con un Adapter (implementación en Infrastructure):

| Dependencia | Port | Adapter inicial | Adapter futuro posible |
|---|---|---|---|
| Base de datos | `{Entidad}RepositoryPort` | `Eloquent{Entidad}Repository` | — |
| API LinkedIn | `LinkedInProviderPort` | `HarvestApiAdapter` | Scraping propio, otro proveedor |
| LLM | `LlmServicePort` | `ReplicateLlmAdapter` | OpenAI, Anthropic, local |
| Storage de imágenes | `ImageStoragePort` | `CloudflareImageAdapter` | S3, local |
| Colas | (Laravel Queue, no necesita port propio) | `driver=database` | Redis, SQS |
| Broadcasting | (Laravel Broadcasting, no necesita port propio) | Pusher | Ably, Soketi, websockets propios |

Las colas y el broadcasting son infraestructura de Laravel que se cambia por configuración (`.env`), no necesitan port propio. Los ports son para dependencias donde la interfaz de negocio es específica del dominio (qué métodos necesita el negocio para hablar con LinkedIn, con el LLM, con el storage).

### 0.5 Operaciones del sistema

Listar las operaciones principales que el usuario puede hacer. Para cada una, clasificar:

| Operación | Tipo | Pasos | Necesita tracking visible |
|---|---|---|---|
| Crear formulario | Simple (< 2s) | 1 | No |
| Subir archivo | Simple (< 2s) si es pequeño, Pipeline si es pesado | 1-2 | Sí si es pesado |
| Importar 100 empresas de CSV | Pipeline | 8 pasos por empresa | Sí |
| Deploy de cluster | Pipeline | N pasos (1 por landing) | Sí |
| Refresh diario de empresa | Orquestador | 3-5 pasos según config | Sí |
| Generar imagen con IA | Cadena de 2 jobs | 2 | Sí |
| Evaluación de materialidad CSRD | BPMN | Gateways semánticos, human gates, bucles | Sí |

De aquí sale directamente la arquitectura de Application:
- Operaciones simples → handler atómico
- Pipelines → PipelineConfig + PipelineState + Executor + handlers por paso
- Orquestadores → componen handlers atómicos con lógica de qué pasos ejecutar
- BPMN → BpmnDefinition + BpmnRun + DirectorAgent + tools por dominio

### 0.6 Roles y permisos

Definir quién puede hacer qué antes de implementar nada:

| Rol | Descripción | Acceso |
|---|---|---|
| super_admin | Bypass total | Todo, sin comprobaciones |
| admin | Administrador | Todos los permisos explícitos |
| editor | Creador de contenido | Crear/editar, sin gestión de usuarios ni operaciones destructivas |
| viewer | Solo lectura | Ver datos, sin modificar |

Los permisos siguen el formato `familia.accion` (ej: `landings.ver`, `clusters.desplegar-preview`). Se listan todas las familias de permisos y sus acciones posibles. Esto alimenta el `config/authorization.php` y el seeder.

### 0.7 Checklist antes de escribir código

Antes de pasar a la Capa 0, verificar que se tiene:

- [ ] Descripción del producto en un párrafo
- [ ] Lista de bounded contexts con sus responsabilidades
- [ ] Entidades con atributos, estados, relaciones y eventos
- [ ] Ports identificados con adapter inicial
- [ ] Operaciones clasificadas (simple / pipeline / orquestador)
- [ ] Roles y permisos definidos
- [ ] Diagrama de relaciones entre entidades (puede ser informal, en papel)

Si falta alguno de estos puntos, no se empieza a desarrollar. Se vuelve a esta fase. Todo lo que viene después — la estructura de carpetas, los patrones de comportamiento, el frontend — se deriva mecánicamente de estas respuestas.

---

## Capa 0 — Lenguaje y Framework

PHP + Laravel (última versión estable). Todo se construye encima de Laravel.

---

## Capa 1 — Scaffolding de aplicación

**Jetstream (Inertia + Teams)** como base. Resuelve autenticación, registro, 2FA, gestión de perfil y la base de multi-tenancy con Teams. No se usa Breeze. No se hace auth custom.

Si el proyecto necesita login social (Google, GitHub, etc.), se añade **Socialite** encima de Jetstream, no en lugar de.

**Vue 3** como framework de frontend. **Inertia** como puente entre Laravel y Vue. Sin router Vue propio — se usa el router de Laravel.

---

## Capa 2 — Librerías frontend transversales

Decisiones de producto que se toman antes de escribir una línea de negocio. Si no se estandarizan, cada desarrollador (o cada sesión de IA) elige una librería diferente.

| Librería | Paquete | Propósito |
|---|---|---|
| Chart.js | `vue-chartjs` + `chart.js` | Gráficos y visualización de datos |
| DataTable | Componente propio | Tabla con paginación, skeleton, slots por celda |
| KpiCard | Componente propio | Tarjeta de métrica (título, valor, icono, color, loading) |
| StatusBadge | Componente propio | Badge de estado multiuso con colores por estado |
| Laravel Echo | `laravel-echo` + `pusher-js` | Escucha de broadcasting en cliente |
| Markdown | `markdown-it` + `markdown-it-texmath` | Renderizado markdown con fórmulas |
| Fórmulas | `katex` | Renderizado LaTeX en markdown |
| Diagramas | `mermaid` | Renderizado de diagramas en markdown |
| Sanitización | `dompurify` | Sanitización HTML del markdown renderizado |
| Artifacts | Sistema propio | Detección, renderizado y panel lateral de contenido generado por LLM |
| Toasts | Componente propio | Notificaciones efímeras (error, success, warning) |

---

## Capa 3 — Arquitectura DDD + Hexagonal

### 3.1 Estructura de carpetas

```
app/Domain/              → PHP puro, cero dependencias de Laravel
app/Application/         → Handlers atómicos, orquestadores, pipelines
app/Infrastructure/      → Adapters (Eloquent, APIs, Pusher, Jobs...)
app/Http/Controllers/    → Presentación HTTP (infraestructura, pero Laravel los pone aquí)
app/Providers/           → Bindings port → adapter
```

**Regla de dependencia absoluta:** Domain no importa nada de Laravel ni de ningún paquete externo. Application solo depende de Domain. Infrastructure implementa lo que Domain define. Controllers y Jobs son infraestructura.

### 3.2 Domain

Cada bounded context tiene su carpeta bajo `app/Domain/`:

```
app/Domain/{Context}/
├── Entities/          → POPOs (Plain Old PHP Objects), sin Eloquent
├── ValueObjects/      → final readonly class, validación en constructor
├── Enums/             → string-backed, siempre con label() para UI
├── Events/            → Clases que extienden DomainEvent
├── Ports/             → Interfaces que define el dominio
└── Contracts/         → Interfaces opcionales (ActivityLoggable, Notifiable...)
```

Un contexto `Shared` contiene lo transversal: `EventBusPort`, clase base `DomainEvent`, contracts compartidos, value objects comunes.

**Entities:**
- POPO con trait `RaisesDomainEvents` para acumular eventos.
- Factory method estático `create()` que registra evento de creación.
- Métodos de mutación que registran eventos de cambio.
- `fromEloquent()` estático para reconstruir desde el modelo de persistencia.
- El modelo Eloquent correspondiente expone `toDomainEntity()` para la conversión inversa.

**Enums:** siempre `string`-backed, siempre con método `label()` para UI. Los enums se castean directamente en el modelo Eloquent.

**Events:** extienden la clase base `DomainEvent` con `aggregateType`, `aggregateId`, `actorId`. Implementan `eventName()` (formato `contexto.accion`, ej: `landing.created`) y `payload()`. Opcionalmente implementan `ActivityLoggableEvent` y/o `NotifiableEvent` para activar comportamientos automáticos.

**Ports:** interfaces con `declare(strict_types=1)`, tipos estrictos, PHPDoc para arrays complejos.

Todo en Domain usa `declare(strict_types=1)`, clases `final` donde aplique.

### 3.3 Application: handlers atómicos + orquestadores

La capa Application tiene dos niveles de responsabilidad:

**Handlers atómicos** — una operación de dominio, reutilizables, sin efectos de progreso:

```
app/Application/{Context}/
├── ScanCompanyProfileHandler.php       → una operación concreta
├── ScanCompanyPostsHandler.php         → otra operación concreta
├── EnrichCompanyProfileHandler.php     → otra más
└── ...
```

Reglas del handler atómico:
- `final class`
- Recibe primitivos o enums de dominio en sus métodos, no config objects de un pipeline
- Inyecta ports en constructor (nunca servicios concretos)
- Emite DomainEvents vía `EventBusPort`
- No emite eventos de progreso (eso es cosa del orquestador)
- Es reutilizable en cualquier contexto (pipeline de import, refresh individual, backfill masivo)

**Orquestadores** — componen handlers para flujos complejos:

```
app/Application/Run{Context}{Action}/       → orquestador individual
├── {Context}RefreshSteps.php               → qué pasos ejecutar
└── Run{Context}{Action}Handler.php         → compone handlers atómicos

app/Application/{Context}Pipeline/          → orquestador de pipeline multi-paso
├── {Pipeline}Config.php                    → DTO readonly con parámetros
├── {Pipeline}State.php                     → estado serializado (pipeline.json)
├── {Pipeline}Executor.php                  → crea run, recorre pasos, emite progreso
└── {StepSpecific}Handler.php               → handlers propios del pipeline (si los hay)
```

Reglas del orquestador:
- Compone handlers atómicos, no duplica su lógica
- Decide qué pasos ejecutar según configuración
- Emite progreso (conecta con broadcasting)
- Gestiona estado de pipeline si es un flujo multi-paso
- Puede ejecutarse síncrono (comando artisan con `--sync`) o asíncrono (jobs encadenados)

**Servicios de agregación (solo lectura):** para dashboards y reportes que solo leen datos, se permite un patrón alternativo que usa Eloquent directo, sin ports ni eventos. Vive en `app/Application/Admin/` o similar. No es un handler DDD, es aceptable porque leer datos no tiene efectos secundarios.

### 3.4 Infrastructure

```
app/Infrastructure/{Context}/
├── Eloquent{Entity}Repository.php    → implementa el port de persistencia
├── {Nombre}Adapter.php               → implementa ports de servicios externos
└── ...

app/Infrastructure/EventBus/
├── LaravelEventBus.php               → implementa EventBusPort
├── Subscribers/                      → AuditTrail, ActivityLog, Notification, JobFlow
└── Broadcasting/Events/              → eventos ShouldBroadcastNow

app/Infrastructure/Queue/
├── TrackableJob.php                  → clase base para jobs con tracking
├── Traits/TracksProgress.php         → tracking + broadcasting + isCancelled()
├── TrackedJobOutcomeWriter.php       → escritor de resultado final
└── Jobs/                             → jobs concretos del proyecto

app/Infrastructure/Persistence/Models/ → modelos Eloquent con toDomainEntity()
```

### 3.5 Bindings

Un solo `DomainServiceProvider` con métodos privados por contexto:

```php
class DomainServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->registerMediaPorts();
        $this->registerAuthorizationPorts();
        // ...
    }

    private function registerMediaPorts(): void
    {
        $this->app->singleton(MediaManagementPort::class, MediaManagementAdapter::class);
    }
}
```

Si el proyecto crece a 10+ bounded contexts, se extraen a providers separados. El default es uno centralizado.

---

## Capa 3B — Patrones de comportamiento del sistema

Estos patrones son ortogonales al negocio. Se aplican a cualquier proyecto. Son las reglas de cómo el sistema se comporta, no de qué hace.

### Patrón 1 — Toda operación es simple, pipeline o BPMN

Cualquier operación de negocio cae en una de tres categorías:

**Operación simple** — un handler atómico, se ejecuta en el request HTTP, dura menos de 2 segundos. El controller llama al handler, el handler hace su trabajo, emite un DomainEvent, devuelve resultado.

**Pipeline** — múltiples pasos con flujo predefinido, se ejecuta asíncrono. Siempre tiene la misma anatomía:

| Componente | Responsabilidad |
|---|---|
| `PipelineConfig` | DTO `readonly` con los parámetros del pipeline. Se serializa en pipeline.json. |
| `PipelineState` | Estado de cada paso (pending, running, completed, failed), timestamps, errores, metadatos. Driver-based: File o Database según infraestructura. |
| Handlers atómicos | Uno por paso. Reutilizables fuera del pipeline. |
| Executor | Crea el run, recorre pasos, llama a handlers, actualiza PipelineState, emite progreso. |

El directorio de cada run es autocontenido:

```
storage/app/{pipeline_name}/run_{timestamp}/
├── pipeline.json            → estado completo del run
├── step_1_output.json       → artefactos del paso 1 (si aplica)
└── ...
```

Los pasos se definen como constante con prefijo numérico que fuerza orden:

```php
private const STEPS = [
    '1_collect_signals',
    '2_normalize_topics',
    '3_score_and_rank',
    '4_validate_intent',
    '5_generate_reports',
    '6_finalize',
];
```

**Reanudabilidad:** si el paso 3 falla, se relanza con `--from-step=3 --run=run_xxx`. Los pasos 1 y 2 ya están completed en pipeline.json y no se re-ejecutan.

**Umbral:** si la operación tiene 2 o más pasos que pueden fallar independientemente, es pipeline formal con Config + State. Con 2 pasos y sin PipelineState, si falla el segundo hay que repetir el primero. Eso es exactamente lo que el pipeline resuelve. Si es un solo paso, es handler simple.

**PipelineState es driver-based.** La decisión se toma por infraestructura, no por entorno:

| Infraestructura | Driver | Razón |
|---|---|---|
| Un servidor | File (`storage/app/pipelines/`) | Más simple, debuggeable visualmente (abres el directorio y lees el JSON) |
| Múltiples workers sin disco compartido | Database (tabla `pipeline_runs` + `pipeline_steps`) | Los workers necesitan estado compartido |

No hay regla automática "producción = database". Si producción es un solo servidor, File funciona perfectamente.

**Concurrencia:** el executor verifica que no haya otro run activo del mismo tipo + entidad antes de arrancar.

**Fallos parciales intra-paso:** el paso es la unidad atómica. Si un paso procesa N items, el handler es tolerante a fallos (try/catch por item, registra fallos, continúa). La reanudabilidad es por paso, no por item.

**BPMN** — proceso complejo cuyo flujo viene definido por un documento BPMN externo, no por constantes en código. Un agente LLM (director) interpreta el BPMN en runtime, recorre los nodos, llama a ejecutores y toma decisiones en gateways basándose en el contenido semántico del payload.

| Criterio | Pipeline | BPMN |
|---|---|---|
| Flujo | Lineal o con ramas simples | Gateways con decisiones complejas, bucles, subprocesos |
| Definición de pasos | Constantes en código | Documento BPMN externo, dinámico |
| Decisiones entre pasos | Condicionales programadas | Inferidas por un LLM basándose en el estado del payload |
| Intervención humana | No contemplada | Human gates nativos: el proceso se pausa y reanuda |
| Adaptabilidad | Cambiar flujo = cambiar código | Cambiar flujo = cambiar BPMN, sin tocar el engine |

**Umbral:** si el flujo tiene más de 3 gateways con decisiones que dependen del contenido semántico de las respuestas (no de valores booleanos simples), es candidato a BPMN. Si las decisiones son binarias y predecibles, pipeline clásico.

**Anatomía del engine BPMN:**

El engine es un bounded context propio (`BpmnEngine`) que no sabe nada del dominio que ejecuta. Solo sabe interpretar BPMNs, recorrerlos y delegar trabajo.

| Componente | Responsabilidad |
|---|---|
| `BpmnDefinition` | El BPMN en sí: nodos, gateways, conexiones, subprocesos. Incluye descripciones semánticas de cada nodo para que el director LLM las interprete. |
| `BpmnRun` | Una ejecución concreta: nodo actual, payload, historial de nodos visitados, decisiones tomadas, respuestas de ejecutores. |
| `DirectorAgentPort` | El agente LLM que recorre el BPMN. Es un port — el adapter decide qué modelo usar. |
| `ToolExecutorPort` | Los ejecutores que hacen el trabajo en cada nodo. No saben del BPMN ni del recorrido. |

**El director (agente LLM como orquestador):**

El director es el equivalente del `PipelineExecutor` pero con inteligencia para interpretar, no solo iterar:

1. Recibe un `BpmnDefinition` y un payload inicial. Crea un `BpmnRun`.
2. En cada nodo **task**: lee la descripción semántica, extrae del payload los datos necesarios (por inferencia), llama al ejecutor vía `ToolExecutorPort`, evalúa la respuesta, actualiza el `PayloadState`.
3. En cada nodo **gateway**: evalúa la condición (descrita en lenguaje natural en el BPMN) contra el estado actual del payload. Infiere qué camino tomar.
4. En cada nodo **human_gate**: pausa el run, emite evento `BpmnHumanGateReached`, espera input externo para reanudar.
5. Al llegar al nodo **end**: marca completado, emite `BpmnRunCompleted`.

**Loop interno del director:** dentro de un nodo task, el director no siempre se conforma con la primera respuesta. Puede reformular la consulta y volver a llamar (al mismo ejecutor o a otro). Tiene un límite de reintentos por nodo (configurable en el `BpmnDefinition`). Este loop es lo que diferencia al engine de un pipeline clásico: el director tiene criterio para replantear, no solo para iterar.

**Ejecutores (tools):**

Los ejecutores son ports y adapters desacoplados. No saben del BPMN. Reciben input, hacen su trabajo, devuelven output.

| Tipo | Qué hace |
|---|---|
| API | Llama a un servicio externo y devuelve datos crudos |
| RAG | Consulta a capa de conocimiento vectorial, devuelve contexto relevante |
| Clasificador LLM | Modelo barato que categoriza, puntúa o filtra |
| Compuesto | Cadena de API → RAG → Clasificador dentro de un step (el director orquesta, no el ejecutor) |

Las tools no son del engine. Son del dominio que las necesita. Cada bounded context registra sus tools en un `BpmnToolServiceProvider`. El engine las consume vía `ToolExecutorPort` sin conocer la implementación.

**Human gates:**

Un human gate es un nodo donde el proceso se pausa y espera intervención humana (aprobación, revisión, input). Mecánica:

1. Director llega al nodo `human_gate`, cambia `RunStatus` a `paused_at_human_gate`.
2. Emite `BpmnHumanGateReached` vía EventBus → los subscribers de notificación y broadcasting reaccionan como con cualquier otro evento.
3. El usuario revisa y responde desde la UI.
4. Un controller (tonto) recibe la respuesta, llama a `ResumeBpmnRunHandler`.
5. El director retoma la ejecución desde el nodo siguiente.

**Costes y límites:**

El engine usa modelos LLM. Sin controles, un director en loop consume presupuesto indefinidamente.

| Control | Valor por defecto | Configurable en |
|---|---|---|
| Reintentos por nodo | 3 | `BpmnDefinition` (por nodo) |
| Reintentos totales por run | 20 | `BpmnDefinition` (global) |
| Timeout por nodo | 120 segundos | `BpmnDefinition` (por nodo) |
| Timeout total del run | 30 minutos | `BpmnDefinition` (global) |
| Tokens máximos por run | Configurable | `BpmnDefinition` (global) |

Cuando se alcanza un límite, el director marca el nodo o el run como fallido con el motivo.

**Estructura de carpetas:**

```
app/Domain/BpmnEngine/
├── Entities/
│   ├── BpmnDefinition.php
│   └── BpmnRun.php
├── ValueObjects/
│   ├── NodeId.php
│   ├── NodeType.php
│   ├── PayloadState.php
│   └── StepResult.php
├── Enums/
│   └── RunStatus.php
├── Events/
│   ├── BpmnRunStarted.php
│   ├── BpmnNodeEntered.php
│   ├── BpmnNodeCompleted.php
│   ├── BpmnGatewayEvaluated.php
│   ├── BpmnHumanGateReached.php
│   ├── BpmnRunCompleted.php
│   └── BpmnRunFailed.php
├── Ports/
│   ├── BpmnDefinitionRepositoryPort.php
│   ├── BpmnRunRepositoryPort.php
│   ├── DirectorAgentPort.php
│   └── ToolExecutorPort.php
└── Contracts/
    └── BpmnExecutableNode.php

app/Application/BpmnEngine/
├── StartBpmnRunHandler.php
├── ResumeBpmnRunHandler.php
├── CancelBpmnRunHandler.php
└── GetBpmnRunStatusHandler.php

app/Infrastructure/BpmnEngine/
├── AnthropicDirectorAdapter.php
├── EloquentBpmnDefinitionRepository.php
├── EloquentBpmnRunRepository.php
├── ToolRegistry.php
└── Jobs/
    └── ExecuteBpmnRunJob.php        → TrackableJob

app/Providers/
└── BpmnToolServiceProvider.php      → Registra tools de cada dominio
```

**Integración con el framework:** el engine BPMN no introduce patrones nuevos de comportamiento. Se monta encima de los existentes. Los eventos del engine pasan por el EventBus, los subscribers de audit/notification/broadcasting reaccionan automáticamente, la ejecución se lanza como TrackableJob, los controllers son tontos. Si el engine cumple las reglas del framework, funciona sin fricción.

### Patrón 2 — EventBus como columna vertebral

Todo lo que ocurre en el sistema pasa por un punto central: el EventBus. Es el mecanismo por el cual los patrones de comportamiento se enganchan al negocio sin que el negocio los conozca.

Un handler emite un DomainEvent vía `EventBusPort`:

```php
$this->eventBus->dispatch(new CompanyProfileScanned($companyId, $actorId));
```

El handler no sabe qué pasa después. Del otro lado, subscribers registrados reaccionan:

| Subscriber | Escucha | Hace |
|---|---|---|
| `AuditTrailSubscriber` | Todo `DomainEvent` | Persiste en tabla de audit (siempre) |
| `ActivityLogSubscriber` | Solo `ActivityLoggableEvent` | Persiste en activity log de negocio (selectivo) |
| `NotificationSubscriber` | Solo `NotifiableEvent` | Resuelve destinatarios y delega en los canales declarados por el evento (in-app, mail, telegram...) |
| `JobFlowSubscriber` | Eventos de jobs/saga | Gestiona flujo padre-hijo |

Añadir observabilidad a cualquier operación nueva es opt-in por interfaz:
- Audit: gratis, todo DomainEvent se audita automáticamente.
- Dashboard de negocio: implementar `ActivityLoggableEvent` en el evento.
- Notificación al usuario: implementar `NotifiableEvent` y declarar canales (`['in_app', 'mail']`).
- Ambas: implementar las dos interfaces. El audit ya está gratis.

**Notificaciones multi-canal (extensión del Patrón 2):** Una notificación es un mensaje dirigido; el canal es solo cómo se entrega. Por eso `NotificationSubscriber` no envía nada por sí mismo: resuelve destinatarios una vez (cluster → companies → users) y delega en una serie de canales intercambiables. Cada canal es un Port con su Adapter:

| Pieza | Tipo | Responsabilidad |
|---|---|---|
| `NotifiableEvent::notificationChannels()` | Contract | El evento declara por qué canales quiere entregarse: `['in_app']`, `['in_app','mail']`, etc. |
| `NotificationEnvelope` | Value Object | Sobre inmutable con destinatario + meta del evento. Es el contrato entre subscriber y canales. |
| `NotificationChannelPort` | Port | Una operación: `send(NotificationEnvelope)`. Una implementación por medio (in-app, mail, telegram, sms…). |
| `NotificationChannelRegistryPort` | Port | Indexa canales por nombre. Se construye desde `config/notifications.php`. |
| `NotificationSubscriber` | Subscriber | No conoce ningún canal concreto. Itera los nombres declarados, los resuelve en el registry y delega. |

Añadir un canal nuevo (Telegram, SMS, Slack, webhook…) **no toca el subscriber**: se crea un adapter en `app/Infrastructure/Notification/Channels/`, se registra en `config/notifications.php`, y los eventos que lo necesiten lo añaden a su array. Cada canal corre dentro de su propio try/catch para que un fallo en uno no impida los demás.

El canal mail despacha un job simple (no `TrackableJob`, ver Patrón 6) que resuelve email del usuario y envía el `Mailable`. La plantilla Blade se resuelve por convención `emails.notifications.{notifiable_snake}.{step_action}` con fallback a `emails.notifications.default`.

El EventBus es un port de Domain (`EventBusPort`), implementado como adapter en Infrastructure (`LaravelEventBus`). Los subscribers son infraestructura.

**Riesgos gestionados:**

- **Payload como contrato:** el payload del evento tiene estructura definida. Los tests del subscriber verifican que procesan el payload correctamente. No es un array libre.
- **Aislamiento de subscribers:** cada subscriber tiene su propio try/catch. Un fallo en audit no impide notificaciones.
- **Subscribers ligeros:** los subscribers hacen operaciones ligeras (un INSERT). Si necesitan algo pesado, despachan un job.
- **Dependencias ocultas:** la tabla de subscribers (arriba) se mantiene actualizada en la documentación del proyecto. Tests de integración verifican la cadena completa.

**Excepción documentada — instrumentación técnica de alto volumen:**

La instrumentación técnica (tracing, métricas, logging estructurado) **puede ir por port directo en lugar de EventBus** si pasar por bus duplicaría las escrituras de la propia capa de observabilidad. El criterio:

- Si el subscriber de audit añade valor por evento, va por bus.
- Si solo añade ruido proporcional al volumen, va directo.

Esta excepción tiene su propio patrón formal — ver **Patrón 18 — Observabilidad técnica como puerto directo** — donde se describe la materialización completa (bounded context `Observability`, `TracerPort`, glue objects con `LlmAudit`, idempotencia y subscribers de cierre de traza).

### Patrón 3 — Broadcasting como consecuencia

El broadcasting no es algo que el negocio haga. Es una consecuencia de algo que pasó:

1. Handler emite DomainEvent vía EventBus
2. Un subscriber reacciona al evento
3. El subscriber llama a `broadcast()` para notificar al frontend

El handler nunca llama a `broadcast()`. El frontend nunca recibe datos por el canal de broadcast. Recibe señales: "el job X cambió de estado", "hay una nueva entrada de audit", "tienes una notificación". Los datos se fetchean por HTTP cuando llega la señal.

**Excepción controlada:** broadcasts de progreso de job pueden incluir payload mínimo (`percent`, `message`) porque fetchear por HTTP en cada tick de progreso sería excesivo. Cambios de estado (completed, failed, cancelled) son solo señal.

Los eventos broadcast son siempre `ShouldBroadcastNow`, nunca `ShouldBroadcast` (que encolaría el broadcast y causaría doble encolado desde workers de cola).

**Degradación:** si el broadcaster no está disponible (desarrollo local sin keys, caída del servicio), el broadcast falla silenciosamente (se loguea, no rompe la operación). El frontend degrada a polling HTTP.

**Canales estándar:**

| Canal | Autorización | Uso |
|---|---|---|
| `private-user.{id}` | Solo el usuario propietario | Notificaciones dirigidas, jobs completados del usuario |
| `private-jobs` | Cualquier autenticado | Estado y progreso de todos los jobs |
| `private-audit` | Cualquier autenticado | Feed de actividad del sistema |

### Patrón 4 — Audit en tres capas

Tres niveles de observabilidad que sirven para cosas distintas. No es redundancia, es granularidad.

**Capa 1 — Audit trail exhaustivo.** Todo `DomainEvent` queda registrado vía `AuditTrailSubscriber`. Sin intervención manual. Es forense: quién hizo qué, cuándo, con qué payload. Candidato a migrar fuera de la BD principal (Elasticsearch, servicio externo).

**Capa 2 — Activity log de negocio.** Solo eventos que implementan `ActivityLoggableEvent`, vía `ActivityLogSubscriber`. Registra por entidad de negocio. Es lo que consume el dashboard para "actividad reciente".

**Capa 3 — Job tracking.** Tabla propia con estado del job (pending, processing, completed, failed, cancelled), progreso, duración, errores, relación padre-hijo. Es lo que consume el panel admin y el frontend para barras de progreso.

Las tres capas se alimentan del mismo EventBus. Cada subscriber filtra diferente.

**Relación con Spatie:** `spatie/laravel-activitylog` se puede usar como implementación de la capa 1 en proyectos que empiezan y no tienen EventBus maduro. Cuando el proyecto tiene DomainEvents, el `AuditTrailSubscriber` cubre la capa 1 y el `ActivityLogSubscriber` cubre la capa 2. Spatie se mantiene como fallback para modelos legacy que aún no emiten DomainEvents (Strangler Fig). **Nunca coexisten sobre el mismo modelo:** si el bounded context emite DomainEvents, Spatie se desactiva para esos modelos.

**Volumen:** política de retención desde el día 1 para la capa 1. Definir días de retención, particionado por fecha, y/o migración a cold storage al deployar.

### Patrón 5 — El controller es tonto

El controller solo hace tres cosas:

1. Valida el request (formato y tipos, vía Form Request de Laravel)
2. Llama un handler (síncrono) o despacha un job (asíncrono)
3. Devuelve respuesta HTTP

Nunca escribe en base de datos. Nunca emite eventos. Nunca instancia servicios de infraestructura. Nunca tiene lógica de negocio. Si un controller crece, falta un handler.

Para operaciones asíncronas, el controller despacha el job y devuelve el `jobId`:

```php
$job = new MiJob($entityId, auth()->id());
dispatch($job);
return response()->json(['jobId' => $job->getTrackedJobId()]);
```

**Validación de negocio** (unicidad, permisos de negocio, reglas complejas) va en el handler, no en el controller. El controller valida que el request es bien formado. El handler valida que la operación tiene sentido.

**Controllers de lectura:** para lectura pura (listados, dashboards, búsquedas) se permite el patrón de servicio de agregación (Eloquent directo, sin ports ni eventos). No es un handler DDD. Es aceptable porque leer datos no tiene efectos secundarios.

### Patrón 6 — Los jobs son puntos de entrada, no lógica

Un job de Laravel es equivalente a un controller pero para colas. Misma regla: no tiene lógica de negocio, solo deserializa parámetros y llama a un handler o executor.

**TrackableJob** — clase base para jobs que el usuario necesita supervisar. Da automáticamente:

- Registro en tabla de tracking al encolarse
- Actualización de estado (processing, completed, failed)
- Reporte de progreso: `$this->updateProgress(percent, message)`
- Comprobación de cancelación: `$this->isCancelled()`
- Broadcast de cambios de estado por canal de broadcasting
- Soporte de relación padre-hijo (saga)

**Patrón saga:** un job padre despacha N hijos, cada hijo tiene `parentTrackedJobId`. Cuando un hijo completa, notifica al padre. El `JobFlowSubscriber` agrega resultados. Cuando todos los hijos terminan, finaliza al padre. Si se cancela al padre, se cancelan todos los hijos activos en cascada.

**Cuándo usar TrackableJob vs job simple de Laravel:**
- TrackableJob: el usuario ve el resultado del job en la UI (progress bar, panel admin, notificación de completado).
- Job simple: housekeeping interno que no necesita tracking visible (enviar email, limpiar caché, purgar datos).

**Cancelación en pipelines asíncronos:** se usa el patrón saga con `parentTrackedJobId`, no `Bus::chain()`. La saga permite cancelar jobs pendientes porque cada job comprueba `isCancelled()` al arrancar. `Bus::chain()` no soporta cancelación de jobs ya encolados.

### Patrón 14 — Servidor MCP como puerto de entrada para LLMs

El **Model Context Protocol (MCP)** es un estándar para que clientes con LLM (Cursor, Claude Desktop, agentes propios) descubran y ejecuten capacidades expuestas por una aplicación. En el framework lo tratamos como **un transport más** del sistema, equivalente a HTTP o a colas: una nueva forma de llamar a operaciones del dominio, no un patrón nuevo de comportamiento.

**Encaje arquitectónico:**

| Pieza | Equivalente HTTP | Equivalente MCP |
|---|---|---|
| Punto de entrada | `Controller` | `McpTool` |
| Validación de formato | `FormRequest` | JSON Schema declarado en `definition()` |
| Lógica de negocio | `App\Application\…\*Handler` | El mismo handler |
| Auth | Middleware Laravel | Usuario inyectado por el comando `mcp:serve` |

Una `McpTool` es a un controller lo que un job es a un controller: **un punto de entrada tonto** que valida input y delega en handlers atómicos de `app/Application/`. Nunca contiene lógica de negocio, nunca persiste directamente.

**Bounded context `Mcp` (técnico):**

```
app/Domain/Mcp/
├── Contracts/
│   ├── McpTool.php                 → toda tool implementa este contrato
│   └── McpResource.php             → todo recurso de lectura idempotente
├── ValueObjects/
│   ├── McpToolDefinition.php       → name + description + JSON Schema input
│   ├── McpToolResult.php           → bloques de contenido + isError
│   └── McpResourceDefinition.php   → uri + name + description + mimeType
└── Ports/
    ├── McpToolRegistryPort.php     → catálogo de tools
    └── McpResourceRegistryPort.php → catálogo de resources

app/Application/Mcp/
├── HandleMcpToolCall.php           → resuelve tool por nombre y la ejecuta
└── HandleMcpResourceRead.php       → resuelve resource por uri y la lee

app/Infrastructure/Mcp/
├── InMemoryMcpToolRegistry.php     → registry construido desde config/mcp.php
├── InMemoryMcpResourceRegistry.php
├── Protocol/
│   ├── McpServer.php               → JSON-RPC 2.0 + dispatch del subset MCP
│   └── McpProtocolException.php    → errores con código JSON-RPC estándar
├── Transport/
│   └── StdioTransport.php          → loop bloqueante sobre stdin/stdout
├── Tools/{Bounded}/                → tools concretas por bounded context
│   └── Forms/
│       ├── ListFieldTypesTool.php
│       ├── ListFormArchetypesTool.php
│       ├── ValidateFormDraftTool.php
│       └── CreateContactFormTool.php
└── Resources/{Bounded}/            → resources concretas por bounded context
    └── Forms/
        └── FormGuidelinesResource.php

app/Console/Commands/Mcp/
└── McpServeCommand.php             → arranca el servidor en stdio
```

**Reglas de una `McpTool`:**

1. `final class`, en `app/Infrastructure/Mcp/Tools/{Bounded}/`.
2. `definition()` devuelve un `McpToolDefinition` con:
   - **Nombre canónico** en formato `{contexto}.{accion_snake_case}` (ej. `forms.create_contact_form`, `taxonomy.list_categories`).
   - **Descripción para el LLM**: explica qué hace, cuándo usarla, qué efectos tiene (lectura vs escritura) y dependencias entre tools (ej. "llamar a `forms.list_field_types` antes que esta").
   - **JSON Schema del input** declarativo, no validación procedural.
3. `execute(array $arguments)` delega en handlers atómicos de `app/Application/`. Nunca persiste directamente, nunca emite DomainEvents (eso lo hace el handler).
4. Devuelve `McpToolResult::json(...)` para payload estructurado, `McpToolResult::text(...)` para texto plano, `McpToolResult::error(...)` para errores que el LLM debe ver y poder reparar.

**Seguridad — el cluster_id NUNCA viene del LLM:**

Las tools que escriben recursos con scoping por cluster (Forms, Landings, Media, etc.) **deben resolver `cluster_id` desde `auth()->user()->resolveActiveCluster()`**, igual que los controllers. Aceptar `cluster_id` como argumento de la tool es una vía abierta a prompt injection: un LLM comprometido podría escribir en clusters que no son del usuario. La regla es la misma del Patrón 5 (controller tonto): la auth y el contexto activo no se pasan, se resuelven.

**Auth en transport stdio:**

El comando `mcp:serve` se invoca con `--user=email@dominio` (o `MCP_DEFAULT_USER_EMAIL`). El comando hace `Auth::setUser($user)` antes de arrancar el loop, así todas las tools ejecutadas en ese proceso operan como ese usuario. Para cliente HTTP futuro (cuando se exponga MCP por HTTP/SSE) se usaría Sanctum tokens y un middleware de autenticación.

**Validación antes de escribir:**

Toda tool de escritura debe ejecutar **primero** la validación de dominio (estructural + reglas de negocio) y **devolver los errores al LLM como `isError=true`** si hay infracciones. El LLM ve el JSON con la lista de violaciones, ajusta el draft y reintenta. Esto es lo que hace `forms.create_contact_form`: llama a `ValidateFormDraft` antes de `CreateContactForm`. Sin esto, un LLM puede crear formularios que incumplen RGPD aunque la guía se lo prohíba.

**Resources vs tools:**

| | Tool | Resource |
|---|---|---|
| Side effects | Sí (puede tenerlos) | No, lectura idempotente |
| Identifica por | Nombre (`{contexto}.{accion}`) | URI (`sicaland://{ruta}`) |
| LLM la invoca | Cuando decide ejecutar algo | Como contexto inicial / referencia |
| Ejemplo | `forms.create_contact_form` | `sicaland://forms/guidelines` |

Las **guidelines** (texto narrativo con reglas, recordatorios legales y buenas prácticas) van como Resource. Las **operaciones** van como Tool.

**Añadir una tool/resource nueva:**

1. Crear la clase en `app/Infrastructure/Mcp/Tools/{Bounded}/` o `app/Infrastructure/Mcp/Resources/{Bounded}/`.
2. Implementar `McpTool` o `McpResource`.
3. Añadir el FQCN a la lista en `config/mcp.php` (`tools` o `resources`).

No hay que tocar el servidor, el registry, el transport ni el comando. Punto.

**Wire protocol:**

`McpServer` implementa el subset JSON-RPC 2.0 de la spec MCP suficiente para Cursor/Claude Desktop:
- `initialize`, `notifications/initialized`, `ping`
- `tools/list`, `tools/call`
- `resources/list`, `resources/read`

Sin dependencias externas (ningún paquete `php-mcp/*`). Si en el futuro queremos transport HTTP/SSE, basta añadir `app/Infrastructure/Mcp/Transport/HttpTransport.php` reutilizando el mismo `McpServer`. Si queremos servirlo desde Node, el contrato `McpTool` puede portarse, pero la implementación PHP queda canónica porque vive donde está la lógica de dominio.

**Riesgos gestionados:**

| Riesgo | Decisión |
|---|---|
| Tool que pasa cluster_id desde el LLM | Prohibido. Se resuelve siempre con `resolveActiveCluster()`. |
| LLM crea recursos no válidos saltándose la guía | Toda tool de escritura ejecuta `ValidateFormDraft` (o equivalente) antes de persistir y devuelve `isError=true` si falla. |
| Logging a stdout rompe protocolo MCP | StdioTransport NUNCA escribe en stdout fuera del JSON-RPC. Logs van a `Log::*` (fichero) o a stderr. |
| Tool revienta y mata el proceso | `HandleMcpToolCall` envuelve la ejecución en try/catch y devuelve `McpToolResult::error(...)`. |
| Catálogo de tools desincronizado con la realidad del sistema | Tools dinámicas: `list_field_types` consulta `FormFieldStrategyRegistry` en runtime, no hay catálogo duplicado. |
| Versión del protocolo MCP cambia | `McpServer::PROTOCOL_VERSION` centralizado. La spec MCP es retrocompatible en estos métodos básicos. |

### Patrón 15 — LlmChat como puerto de salida para conversaciones con tool calling

El bounded context `LlmChat` añade un puerto de salida específico para mantener **conversaciones con un LLM con soporte de function calling**. Es complementario a `AiServicePort` (orientado a SEO / generación de imágenes en tareas one-shot): aquí lo que importa es el ciclo turno → tool call → resultado → siguiente turno.

**Por qué es un bounded context aparte y no un método más en `AiServicePort`:**

- El contrato es radicalmente distinto: `chat(messages[], tools[], options) → ChatResponse` con control de flujo conversacional.
- Hay value objects propios (`ChatMessage`, `ChatTool`, `ChatToolCall`, `ChatResponse`, `ChatRole`).
- El cambio de proveedor (Anthropic, OpenAI, Bedrock, modelo local) es ortogonal a SEO/imágenes.
- Permite tener varios adapters configurados por entorno sin liarla en el SEO existente.

**Estructura:**

```
app/Domain/LlmChat/
├── ValueObjects/
│   ├── ChatRole.php          → system | user | assistant | tool
│   ├── ChatMessage.php       → un turno (con factories ::system/::user/::assistant/::tool)
│   ├── ChatTool.php          → tool declarada (name + description + inputSchema)
│   ├── ChatToolCall.php      → invocación que el assistant propone
│   └── ChatResponse.php      → content + toolCalls + stopReason + tokens
└── Ports/
    └── LlmChatPort.php

app/Infrastructure/LlmChat/
└── Replicate/
    └── ReplicateClaudeHaikuChatAdapter.php   → Claude 4.5 Haiku vía Replicate
```

**Loop conversacional (responsabilidad del orquestador en Application):**

```
1. messages = [system: guidelines, user: "quiero un formulario de contacto"]
2. response = $llm->chat($messages, $tools)
3. Si response->wantsToolUse():
       Para cada toolCall:
           result = $mcp->handleToolCall($call->toolName, $call->arguments)
           messages[] = ChatMessage::tool($call->id, $call->toolName, $result->json)
       messages[] = ChatMessage::assistant(response->content, response->toolCalls)
       goto 2
   Si no:
       devolver response->content al usuario
```

El loop vive en un orquestador (cuando se construya el chat in-app) o, en el caso del MCP estándar (Cursor / Claude Desktop), lo hace el cliente externo y nosotros solo exponemos las tools.

**Adapter Replicate / Claude Haiku 4.5:**

- Usa el endpoint `POST /v1/models/anthropic/claude-4.5-haiku/predictions` con `Prefer: wait`.
- Renderiza la conversación a un único `prompt` + `system_prompt` (formato simple que admite Replicate hoy).
- Declara las tools en el system prompt como JSON, junto con un protocolo de invocación con bloques `<tool_call name="..." id="...">{...}</tool_call>` que el adapter parsea de la respuesta.
- Si Replicate expone tool_use nativo en el futuro para este modelo, basta cambiar `parseResponse()` y `renderConversation()` sin tocar nada más.

**Cambiar el modelo:**

Editar `LLM_CHAT_ADAPTER` en `.env` apuntando a otro adapter que implemente `LlmChatPort`. El binding está en `DomainServiceProvider::registerLlmChatPorts()`, lee de `config/llm.php`. No se toca nada del dominio.

### Patrón 16 — Reglas de negocio como objetos de dominio testeables

Cuando una entidad tiene reglas de negocio que combinan **legalidad** (RGPD, RD-Ley 13/2012, sectoriales) y **UX** (recomendaciones de buenas prácticas), la solución más mantenible es:

**Cada regla → una clase de dominio que implementa un contrato común:**

```
app/Domain/{Context}/Rules/
├── {Context}BusinessRule.php          → contrato: id() + evaluate(array $draft): list<Violation>
├── ReglaUno.php
├── ReglaDos.php
└── ...

app/Domain/{Context}/ValueObjects/
├── {Context}RuleSeverity.php          → enum: error | warning
├── {Context}RuleViolation.php         → ruleId + severity + message + suggestion
└── {Context}RulesResult.php           → agregado de violations con helpers
```

**Application orquesta:**

```
app/Application/{Context}/
└── Evaluate{Context}BusinessRules.php  → recibe lista de rules e itera
```

**Reglas obligatorias para que esto funcione:**

1. **Una regla por archivo**, sin "RuleSet" gigantes con varios checks dentro. Cada regla se testea independientemente.
2. **Severidad explícita**: `error` bloquea persistencia, `warning` solo informa. Eso lo decide la regla, no quien la consume.
3. **Sugerencia ejecutable**: cada `Violation` puede llevar una `suggestion` con la estructura concreta que repararía la infracción (un campo completo, un valor, etc.). Esto permite a un LLM aplicar la corrección sin inventarse nada y permite a la UI ofrecer un botón de "arreglar".
4. **`ruleId` estable** en formato `{categoria}.{regla_snake_case}` (ej. `rgpd.terms_required_with_personal_data`). Es la identidad de la regla — se usa en tests, en filtrado de severidad por config y en la audit trail.
5. **Bindings declarativos**: las reglas activas se inyectan en el handler `Evaluate{Context}BusinessRules` desde `DomainServiceProvider`. Añadir una regla nueva = una línea en el provider.

**Ejemplo concreto en Forms:**

`app/Domain/Forms/Rules/` contiene `AtLeastOneInputField`, `UniqueFieldNames`, `TermsRequiredWithPersonalData`, `MarketingConsentNotPreChecked`, `SuccessMessageRecommended`. El handler `EvaluateFormBusinessRules` las itera. El handler superior `ValidateFormDraft` agrega validación estructural (Laravel Validator) + reglas de negocio en un único `FormRulesResult`. Tanto el `ContactFormController` como las tools MCP (`forms.validate_draft`, `forms.create_contact_form`) consumen el mismo `ValidateFormDraft`. Una sola fuente de verdad para qué es un formulario válido en Sicaland.

**Por qué en Domain y no en Application o Infrastructure:**

Las reglas son **políticas de negocio**, no procedimientos ni infraestructura. Son lo que define qué significa "un formulario válido en este sistema". Por eso van en Domain, sin dependencias de Laravel. Se pueden testear con `new ReglaX(); $regla->evaluate($draft);` sin booteo de la app.

### Patrón 17 — Asistente conversacional in-app con confirmación humana de side effects

Cuando una funcionalidad de IA conversacional **se expone dentro de la propia aplicación** (no a través de un cliente MCP externo como Cursor), aplicamos un patrón distinto al MCP puro: el **orquestador del loop conversacional vive en el backend**, no en el cliente. El frontend solo dibuja UI y envía/recibe turnos.

**Diferencias clave con el flujo MCP estándar:**

| Aspecto | Cliente MCP externo (Cursor) | Asistente in-app (Sicaland) |
|---|---|---|
| Quién mantiene el historial | Cliente | Frontend (memoria de la sesión) |
| Quién ejecuta el loop tool-call | Cliente | Backend (orquestador en `Application/LlmChat`) |
| Tools de escritura | Disponibles directamente | NO se exponen al LLM. La acción de persistencia la hace el usuario humano explícitamente. |
| Validación previa a persistir | Recomendada | Obligatoria, antes de habilitar el botón de confirmación |
| Auth | Identidad inyectada por `mcp:serve` | `auth()->user()` HTTP normal |

**Estructura de capas:**

```
app/Application/LlmChat/
├── Conduct{Caso}Conversation.php   → orquestador específico del caso de uso
└── ConversationTurnResult.php       → VO de salida del orquestador

app/Http/Controllers/{Bounded}/
└── {Entidad}AiDesignerController.php → controller tonto (index + chat endpoint)

resources/js/Pages/{Bounded}/
└── AiDesigner.vue                   → página split-view con chat + preview

resources/js/Components/AiDesigner/
├── ChatPanel.vue                    → contenedor del chat con scroll + input
├── ChatMessage.vue                  → bubble individual (user / assistant / tool)
├── FormDraftPreview.vue             → preview del recurso que se está construyendo
└── {Recurso}Preview.vue             → render visual del recurso en cuestión
```

**Reglas del orquestador (`Conduct{Caso}Conversation`):**

1. **Construye el system prompt** combinando el `Resource` MCP de guidelines (la misma fuente de verdad que ven los clientes MCP externos) con instrucciones específicas del flujo in-app: "no intentes crear nada, eso lo confirma el usuario".
2. **Restringe las tools que expone al LLM** a una lista blanca explícita (constante `ALLOWED_TOOL_NAMES`). Las tools de escritura (`*.create_*`, `*.delete_*`, `*.update_*`) **no se incluyen**. Si el LLM intenta invocar una tool fuera de la whitelist, el orquestador devuelve un mensaje tool con error y la conversación continúa — el LLM aprende que no puede.
3. **Ejecuta el loop con un máximo de iteraciones** (`MAX_TOOL_ROUNDS`, p.ej. 6). Esto evita runaways si el LLM se queda en bucle pidiendo tools.
4. **Cada tool call se delega en `HandleMcpToolCall`** (mismo handler que usa el servidor MCP). Una sola implementación de cada tool sirve a ambos caminos.
5. **Al terminar el turno extrae el "draft actual"** del historial: recorre los mensajes hacia atrás buscando el último `forms.validate_draft` (o equivalente) y devuelve sus argumentos + el resultado de validación. El frontend usa esto para preview en vivo y para habilitar el botón de confirmación.
6. **Devuelve un VO `ConversationTurnResult`** con `newMessages`, `currentDraft`, `validation`, `finalContent`. NO devuelve la lista completa actualizada del historial — eso es estado del frontend.

**Reglas del controller HTTP:**

- `index()` renderiza la SPA con catálogos iniciales (tipos de campo, arquetipos, mensaje de bienvenida del assistant).
- `chat()` valida payload (`message` + `messages[]` historial), reconstruye `ChatMessage` instances con `ChatMessage::fromArray()`, llama al orquestador, devuelve JSON con `new_messages` (para concatenar al historial del frontend), `current_draft`, `validation`.
- **No persiste nada**. La persistencia se hace cuando el usuario pulsa "Crear" y el frontend invoca el endpoint estándar de creación (`contact-forms.store`, etc.) con el draft validado.

**Reglas del frontend Vue:**

- **El historial vive en `ref([])` en la página principal.** Cada turno: añade el mensaje del usuario, manda la lista al backend, concatena `new_messages` recibidos. No se persiste entre navegaciones — es una sesión efímera.
- **El draft y la validación viven en refs separados.** Se actualizan solo si el backend los devuelve no nulos (cuando el LLM ha llamado a `validate_draft` en este turno).
- **Botón "Crear" deshabilitado** mientras `validation.errors.length > 0` o no haya draft. Cuando se pulsa, hace `router.post(route('{recurso}.store'), draft)` reusando el flujo HTTP normal.
- **Preview visual del recurso** usando los mismos componentes que ya rendericen el recurso en otras pantallas (consistencia y reutilización).

**Por qué el orquestador NO vive en el frontend:**

1. **Secretos**: el API token del proveedor LLM (Replicate, etc.) no puede salir del backend.
2. **Whitelisting de tools**: la decisión de qué puede invocar el LLM es una decisión de seguridad — debe estar bajo control del backend, no negociable desde el navegador.
3. **Reuso del registry MCP**: las mismas tools que sirven al servidor MCP externo sirven al asistente in-app. Una sola implementación.
4. **Auditoría**: cada turno pasa por un punto de entrada HTTP autenticado y permisado, fácil de loggear/audit.

**Por qué el LLM no puede ejecutar la tool de creación final (aunque la confirmación humana sea redundante con la validación):**

Defensa en profundidad. La validación de negocio puede tener bugs o cubrir solo lo conocido. La revisión humana es la última barrera contra:
- Prompt injection en datos del usuario (si se incluyera contexto de fuera).
- Casos edge de UX que las reglas no detectan ("el formulario es válido pero el LLM ha entendido mal lo que quería el usuario").
- Cambios silenciosos no esperados durante una conversación larga.

El coste es un click extra. Para flujos de **escritura**, vale la pena siempre.

**Cuándo NO aplicar este patrón:**

- Para clientes MCP externos (Cursor, Claude Desktop). Ahí el cliente ya orquesta y el usuario ya está leyendo cada tool call en su UI — exponemos las tools de escritura directamente con sus validaciones.
- Para flujos puramente de lectura ("explícame cómo está configurado X"). Si no hay side effects, no hace falta confirmación humana.

### Patrón 18 — Observabilidad técnica como puerto directo (fuera del EventBus)

Cuando una operación de negocio produce telemetría técnica de alto volumen (spans de tracing, métricas, logs estructurados), aplicamos un patrón paralelo al EventBus: un puerto dedicado (`TracerPort`) que persiste directamente en su propio storage, sin pasar por bus. Es la materialización rigurosa de la excepción descrita en Patrón 2.

**Por qué un patrón propio y no "una excepción más":**

1. **La instrumentación NO es un hecho de dominio.** Es la equivalencia técnica de `Log::info()` — el sistema NO afirma "ha sucedido X cosa interesante para el negocio", afirma "esto duró 73 ms". `Log::info()` no pasa por bus; tracing tampoco debe.
2. **El volumen es proporcional a la actividad técnica, no a los hechos de negocio.** ~10 spans por turno de chat, decenas por ingesta documental. Forzarlos por `AuditTrailSubscriber` los duplicaría 1:1 sobre `audit_logs` — porque ya tienes la tabla `spans` que registra exactamente lo mismo.
3. **Solo los hitos del ciclo de vida** (1 traza arrancada, 1 traza completada) tienen valor narrativo cross-bounded-context. Esos sí pasan por bus.

**Estructura del bounded context `Observability`:**

```
app/Domain/Observability/
├── Entities/
│   ├── Trace.php                  → ciclo de vida de una operación observable
│   └── Span.php                   → operación medida dentro de una traza
├── ValueObjects/
│   ├── TraceId.php / SpanId.php   → UUID v7
│   ├── TraceKind.php              → enum extensible: chat_standard | chat_rag | knowledge_ingestion | ...
│   ├── SpanKind.php               → enum técnico: pipeline | step | llm_call | http_call | embedding | ...
│   ├── TraceStatus.php            → pending | running | completed | failed | rejected
│   └── TraceSubject.php           → polimórfico (type, id) — sin FK formal
├── Events/
│   ├── TraceStarted.php           → SÍ pasa por bus (1 por traza)
│   └── TraceCompleted.php         → SÍ pasa por bus (1 por traza)
└── Ports/
    ├── TracerPort.php             → API genérica (startTrace / startSpan / span / completeTrace / failTrace ...)
    ├── TraceRepositoryPort.php
    └── SpanRepositoryPort.php

app/Application/Observability/
├── Services/
│   └── TracedLlmCall.php          → glue object Observability ⇄ LlmAudit
├── Subscribers/
│   └── On{Bounded}TerminalCloseTrace.php → traduce eventos terminales del BC dueño de la saga
└── Handlers/
    ├── GetTraceWaterfallHandler.php
    └── RecordClientSpansHandler.php → ingestión de spans medidos en el frontend

app/Infrastructure/Observability/
├── DatabaseTracer.php             → adapter principal del TracerPort
└── Persistence/
    ├── EloquentTraceRepository.php
    ├── EloquentSpanRepository.php
    └── Models/{Trace,Span}.php
```

**Reglas del `TracerPort`:**

1. **API genérica, agnóstica del BC.** Se invoca con `TraceKind` (enum extensible) + `TraceSubject` (VO polimórfico `type+id`). Cualquier bounded context puede tracear sin tocar el puerto: solo añade un case al enum y se suscribe a sus propios eventos terminales.
2. **`TraceSubject` sin FK formal.** El acoplamiento entre la traza y la entidad de negocio es solo lógico (`subject_type` + `subject_id`). Si la entidad se borra, la traza histórica sobrevive con el `display_name` snapshot-eado para que la UI siga rindiendo algo legible. Mantener FK acoplaría observabilidad a la existencia perpetua de la entidad — exactamente lo contrario de lo que queremos.
3. **`Span` con flag `isPersisted`.** El tracer SIEMPRE devuelve un `Span`, incluso cuando está en modo OFF (debug-off en chat) o no hay traza activa (handler corriendo fuera del flujo de procesado). En esos casos el span lleva `isPersisted = false` y los métodos de cierre lo descartan. Esto evita el ruido de `?Span` por todas partes y previene FK violations en tablas que cruzan (ej. `llm_call_records.trace_span_id`).
4. **`startTrace` es idempotente respecto al `traceId`.** Si la traza ya existe en BD, se reusa sin re-emitir `TraceStarted`. Imprescindible cuando una operación de alto nivel se descompone en varios requests HTTP que comparten cabecera `X-Trace-Id`.
5. **Dos modos de operación dentro del mismo adapter:**
   - **Stack en memoria** (caso request HTTP único, ej. chat): los spans se cuelgan automáticamente del top of stack. Toggle por header (`X-Trace-Enabled`) para subset de URLs.
   - **Spans hermanos** (caso pipeline en jobs separados): cada job es su propio proceso PHP, sin memoria compartida. Los handlers pasan `$trace` explícito vía `resolveTraceForSubject(...)` — el adapter NO empuja al stack en este caso. Siempre activo (no requiere opt-in).

**Eventos que SÍ pasan por bus (volumen 1+1 por traza):**

- `TraceStarted` — punto de partida del ciclo de vida.
- `TraceCompleted` — terminal (sea `Completed`, `Failed` o `Rejected`).

Ambos llevan información agregada (`status`, `total_ms`, `span_count`, `subject_type`, `subject_id`) que permite a subscribers downstream cerrar `AccountAction`s, calcular métricas o alertar — sin tener que leer la tabla de spans.

**Cierre de trazas distribuidas — subscribers en `Application/Observability`:**

Cuando una saga eager (caso típico: ingesta documental con `ingest → embed → theme`) atraviesa varios jobs y bounded contexts, la traza se cierra desde un subscriber de Observability que escucha los eventos terminales del BC dueño del flujo:

```php
final class OnKnowledgeProcessingTerminalCloseTrace
{
    public function onIngested(KnowledgeIngested $event): void { /* completed */ }
    public function onRejected(KnowledgeItemRejected $event): void { /* rejected */ }
    public function onEmbeddingFailed(KnowledgeItemEmbeddingFailed $event): void { /* failed */ }
    public function onTopicAssignmentFailed(KnowledgeItemTopicAssignmentFailed $event): void { /* failed */ }
}
```

Esto preserva la dirección correcta del acoplamiento: **Observability conoce a Knowledge, Knowledge no sabe nada de Observability**. Cada subscriber:

- Es **idempotente** — si la traza ya está cerrada (re-procesado manual, evento duplicado), hace `Log::info` y vuelve.
- **Nunca lanza** (`try/catch` global con `Log::warning`) para no bloquear la propagación a otros subscribers ni romper la saga.

**Glue objects para cross-context: `TracedLlmCall`:**

Cuando una operación cruza varios bounded contexts de cross-cutting (Observability + LlmAudit + Usage), creamos un glue object en `Application/Observability/Services/`. Los handlers no instancian VOs ni emiten eventos de cada contexto: llaman al glue y se acabó.

```php
$rewritten = $this->tracedLlmCall->run(
    userId:       $user->id,
    spanName:     'rewrite_query',
    purpose:      LlmCallPurpose::QueryRewrite,
    provider:     'mlx_fast',
    model:        'lfm2-8b-a1b',
    input:        $input,
    callable:     fn () => $agent->prompt($prompt),
    extractOutput: fn ($r) => LlmCallOutput::fromResponse($r),
);
```

El glue:

1. Abre un span (`SpanKind::LlmCall`) — vía puerto directo.
2. Ejecuta el callable.
3. Registra un `LlmCallRecord` (vía `RecordLlmCallHandler` — sí emite `LlmCallCompleted` por bus).
4. Cierra el span con métricas finales (tokens, latencia, bytes).
5. En caso de excepción, marca span y record como fallidos antes de re-lanzar.

**Simetría con `LlmAudit`:**

El bounded context paralelo `LlmAudit` aplica la regla inversa: cada `LlmCallCompleted` SÍ pasa por bus, porque:

- Es un hecho de dominio ("el sistema invocó al LLM con este prompt").
- Volumen acotado (3-5 por turno).
- Tiene valor narrativo en `audit_logs` para auditoría de calidad del producto.

La regla de decisión es siempre la misma: **¿el subscriber de audit añadiría valor por evento, o solo ruido proporcional al volumen?**

**Cuándo aplicar este patrón:**

- Tracing técnico (spans + waterfall): siempre.
- Métricas internas de alta cardinalidad: sí, mismo patrón con un `MetricsPort` paralelo.
- Logs estructurados de aplicación: ya están fuera de bus por defecto (`Log::info`) — coincide.
- Cualquier telemetría donde `volumen × valor-por-evento` favorezca el acceso directo al storage.

**Cuándo NO aplicar este patrón:**

- Hechos de dominio aunque sean frecuentes (cada `MessageSent`, cada `LlmCallCompleted`): van por bus. La frecuencia no es el criterio único — lo es el valor narrativo del hecho.
- Eventos que necesitan reaccionar otros bounded contexts: van por bus, sino el destinatario tendría que poll-ear la tabla técnica.

**Riesgos gestionados:**

| Riesgo | Decisión |
|---|---|
| Forzar todos los spans por bus duplica `audit_logs` | Spans por puerto directo. Solo `TraceStarted`/`TraceCompleted` por bus. |
| FK violations cuando el tracing está OFF o no hay traza activa | `Span::isPersisted = false`. Los call-sites usan `$span->isPersisted ? $span->id : null`. |
| Acoplar la traza a la existencia de la entidad de negocio | `TraceSubject` polimórfico sin FK formal. La entidad se puede borrar; la traza sobrevive con `display_name` snapshot. |
| Misma traza en múltiples requests HTTP la duplica | `startTrace` idempotente: `findById` antes de crear; si existe, se reusa sin re-emitir `TraceStarted`. |
| Subscriber de cierre de traza bloquea la saga al fallar | `try/catch` global + `Log::warning`. Nunca lanza ni propaga. |
| Cada BC reinventa su propio tracer | Puerto único `TracerPort` con `TraceKind` enum extensible. Añadir un BC = añadir un case + suscribirse a sus eventos terminales. |
| Mezclar tracing + audit-de-LLM en cada handler | Glue object `TracedLlmCall` único. Los handlers llaman a `run(...)` y obtienen los dos contextos cosidos sin tocar VOs. |

---

### Patrón 19 — Strategy + Registry para variabilidad intra-dominio

Cuando un bounded context tiene una decisión "cómo se hace X" que admite múltiples implementaciones según el caso (tipología de documento, estrategia de agrupación, perfil de cliente), modelamos esa variabilidad como un **Port (la estrategia)** + un **Registry** que resuelve qué adapter aplica para una instancia concreta.

Es un patrón de la GoF (Strategy) + un nivel de indirección (Registry) que se ha consagrado en este proyecto en tres sitios distintos:

| BC | Port | Registry | Adapters |
|---|---|---|---|
| Knowledge / Lentes | `GroupingStrategy` | `GroupingStrategyRegistry` (lee `config/knowledge.php → lenses`) | `EmergentTopicsGroupingStrategy` ('topics'), futuros `ByDepartment`, `ByDate` |
| Knowledge / Procesado documental | `DocumentProcessorPort` | `DocumentProcessorRegistry` (lee `config/knowledge.php → processors`) | `LlmDrivenProcessor`, `StructuralPyMuPdfProcessor`, futuros `ScannedFullOcr`, `LegalNormAware` |
| Shared / AI SDK | drivers Prism | `PrismManager` (lee `config/ai.php → providers`) | `OllamaWithThink`, `VllmProvider`, OpenAI, Anthropic… |

**Estructura canónica del patrón:**

```php
// 1. Domain — Port y VOs
interface XStrategyPort {
    public function name(): XStrategyName;        // VO slug validado
    public function canHandle(...$signals): bool; // si aplica priority+canHandle
    public function priority(): int;
    public function execute(...): XResult;        // contrato del trabajo
}

interface XStrategyRegistry {
    public function names(): array;
    public function get(XStrategyName): XStrategyPort;
    public function has(XStrategyName): bool;
    public function resolve(...): XStrategyPort;  // canHandle + priority
}

// 2. Application — adapters concretos
final class FooBarStrategy implements XStrategyPort { ... }

// 3. Infrastructure — registry concreto
final class ConfigXStrategyRegistry implements XStrategyRegistry { ... }
// lee config/<bc>.php → 'x_strategies' y resuelve via container.

// 4. Service Provider — bindings
$this->app->singleton(XStrategyRegistry::class, ConfigXStrategyRegistry::class);

// 5. Config — registro declarativo
return [
    'x_strategies' => [
        'foo' => ['strategy' => FooBarStrategy::class, 'priority' => 100],
        ...
    ],
];
```

**Por qué es un patrón propio y no "código abierto a extensión":**

1. **DDD-aligned:** la variabilidad es una afirmación del dominio ("hay distintas tipologías de documento que requieren distintas estrategias de procesado"), no una preferencia técnica.
2. **Migración no destructiva:** se puede introducir un adapter nuevo SIN tocar el existente. El nuevo va a producción primero como adapter alternativo (priority=0 o canHandle restrictivo), se valida, se promociona subiendo priority. Reduce el riesgo del big-bang refactor.
3. **A/B testing nativo:** un comando admin (`php artisan knowledge:process-with --processor=X --shadow`) puede ejecutar un adapter alternativo y guardar su salida en una tabla espejo (`*_shadow`) para comparar lado a lado sin afectar producción. Capability permanente, no andamio temporal.
4. **Configuración declarativa:** el registro vive en `config/`. Un PR nuevo añade un adapter modificando 2 ficheros (la clase + 1 línea en el array). Sin tocar bindings ni magia.
5. **Tests aislados:** los adapters son testables por separado (mock del Port + del Registry); el resolver del Registry tiene su propio test (verifica que `canHandle + priority` elige correctamente).

**Cuándo aplicar este patrón:**

- Hay >1 forma legítima y permanente de hacer algo dentro de un BC (no solo "vieja vs nueva", sino "esta vs aquella según dato del input").
- La decisión de cuál usar depende de propiedades del input (signals) o del invocador (context), no de configuración global.
- Las implementaciones son de complejidad similar y se beneficiarían de testing/observabilidad uniformes.

**Cuándo NO aplicar este patrón:**

- Solo hay UNA forma de hacer algo y la abstracción no aporta. La indirección añade peso muerto sin valor.
- La decisión "cuál usar" no depende del input — es config global. Entonces basta un binding directo en el ServiceProvider.
- Es código en proceso de migración con un sustituto único previsto. Aplazar el patrón al refactor — evita inflar el alcance.

**Lectura cruzada:**

- En el BC Knowledge se aplica con dimensión `processor_name` añadida a `chunker_benchmark_runs` para corte por adapter — deja constancia operativa de qué strategy procesó cada documento.
- Cross-cutting concerns (UI events, audit, tracing) NO se duplican en cada adapter: el adapter declara *qué pasó* (`ProcessingResult.traceAttributes`, `progressMilestones`), el orquestador traduce a las tres capas. El adapter desconoce el sistema de tracing y el de UI.

---

## Capa 4 — Frontend

### 4.1 Estructura de archivos

```
resources/js/
├── api/                        → Capa HTTP
│   ├── httpClient.js           → fetch wrapper con CSRF + manejo errores Laravel
│   └── {dominio}Api.js         → funciones por dominio de negocio
├── composables/                → Lógica con estado y ciclo de vida
│   ├── useJobTracker.js        → suscripción a canal de jobs
│   ├── useAuditFeed.js         → suscripción a canal de audit
│   ├── useNotifications.js     → suscripción a canal de usuario
│   ├── useStreaming.js          → consumo de SSE para LLM
│   ├── useScrollAnchor.js      → auto-scroll inteligente
│   ├── useAutoResize.js        → textarea auto-grow
│   └── useDataTable.js         → paginación, búsqueda, ordenación
├── stores/                     → Pinia (solo estado que cruza componentes)
│   └── toast.js                → stack de notificaciones efímeras
├── Pages/
│   └── {Dominio}/
│       ├── Index.vue           → Container (orquesta)
│       └── Partials/           → Presenters (renderizan, emiten eventos)
├── Components/                 → Componentes reutilizables transversales
│   ├── DataTable.vue
│   ├── KpiCard.vue
│   ├── StatusBadge.vue
│   ├── NotificationBell.vue
│   ├── ToastStack.vue
│   ├── JobTrackerPanel.vue
│   ├── MarkdownMessage.vue
│   └── ArtifactPanel.vue
└── utils/                      → Funciones puras
    ├── markdown.js             → markdown-it + KaTeX + DOMPurify + artifacts
    └── mermaidRunner.js        → inicialización y ejecución de Mermaid
```

### Patrón 7 — API Layer separado

Toda comunicación HTTP pasa por una capa dedicada. Nunca se llama a `axios.get()` o `fetch()` directamente desde un componente.

**`api/httpClient.js`** — wrapper de fetch:
- CSRF automático: lee el token del meta tag y lo inyecta.
- Respuestas HTML inesperadas: parsea con `res.text()` + `JSON.parse()` en vez de `res.json()` (Laravel devuelve HTML en errores 419/500/redirect).
- Exporta `fetchJson(url, options)` y `fetchStream(url, options)`.

**`api/{dominio}Api.js`** — funciones por dominio de negocio. Cada una corresponde a un endpoint. Son funciones puras async, sin estado, sin refs de Vue:

```javascript
export async function listCompanies(params) {
    return fetchJson('/api/companies', { params });
}
export async function importCompany(data) {
    return fetchJson('/api/companies/import', { method: 'POST', body: data });
}
```

Si cambia un endpoint, se cambia en un sitio. Los componentes no conocen URLs ni métodos HTTP.

### Patrón 8 — Composables como puente entre backend y UI

Toda lógica con estado reactivo y ciclo de vida se extrae a un `useXxx()`. Retorna refs y funciones. Si dos componentes necesitan la misma lógica, es composable. Si es lógica de un solo componente, va dentro del componente.

**Composables de sistema** — conectan con los patrones de comportamiento del backend:

| Composable | Canal que escucha | Conecta con |
|---|---|---|
| `useJobTracker(userId)` | `private-jobs` + `private-user.{id}` | Patrón 6 (jobs) |
| `useAuditFeed(maxEntries)` | `private-audit` | Patrón 4 (audit) |
| `useNotifications(userId)` | `private-user.{id}` | Patrón 2 (EventBus → NotificationSubscriber) |
| `useStreaming({ onDelta, onComplete, onError })` | — (SSE directo) | LLM streaming |

**Composables de UX** — resuelven problemas de interfaz:

| Composable | Propósito |
|---|---|
| `useScrollAnchor()` | Auto-scroll inteligente para chat. Detecta scroll manual del usuario. |
| `useAutoResize(model, { maxHeight })` | Textarea que crece con el contenido. |
| `useDataTable(apiEndpoint)` | Paginación, búsqueda, ordenación contra API. |

### Patrón 9 — Container / Presenter

Las páginas se dividen en dos roles:

**Container** (`Pages/{Dominio}/Index.vue`):
- Recibe props de Inertia
- Inicializa composables y stores
- Conecta las piezas: cuando un Presenter emite un evento, el Container decide qué hacer
- No tiene HTML significativo propio, es un layout que coloca Presenters

**Presenters** (`Pages/{Dominio}/Partials/*.vue`):
- Reciben props, muestran UI, emiten eventos
- No acceden a stores, no llaman a APIs, no tienen lógica de negocio
- Son tontos y testeables

El Container es el equivalente frontend del controller: orquesta sin tener lógica propia. Los Presenters son renderizadores puros.

### Patrón 10 — Carga de datos de las páginas

**Inertia con props** — el controller pasa datos al renderizar. La página tiene datos inmediatos. Se usa cuando los datos son ligeros y se necesitan para el primer render (listado simple, formulario con datos precargados).

**Inertia vacío + API** — el controller solo hace `Inertia::render('Page')` sin datos. El componente fetchea en `onMounted` vía API Layer. Skeleton loading mientras carga. Se usa cuando los datos son pesados, paginados, o vienen de queries de agregación (dashboards, paneles admin, reportes).

**Regla:** si el query para obtener los datos tarda menos de 200ms, props de Inertia. Si tarda más o es paginado, patrón vacío + API. No se mezclan para los mismos datos en la misma página.

**Shared props** — datos que todas las páginas necesitan. Se comparten en `HandleInertiaRequests`:

```php
'auth' => [
    'user' => $user,
    'is_super_admin' => ...,
    'active_company' => ...,
],
'unread_notifications_count' => fn () => ..., // lazy
```

### Patrón 11 — Estado compartido con Pinia

Pinia se usa **solo** cuando hay estado que cruza componentes que no son padre-hijo directo. Siempre con sintaxis Composition API (setup store).

**Sí Pinia:** estado que comparten componentes sin relación directa (chat store, busy accounts, toast stack). Estado que necesita sobrevivir a la navegación dentro de la misma SPA.

**No Pinia:** estado de una sola página (refs locales en el Container). Estado efímero de un componente (abierto/cerrado, hover). Estado de ciclo de vida corto (composable).

### Patrón 12 — Componentes transversales

Componentes de sistema que todo proyecto tiene. No son de negocio. Cada uno sigue la regla de Presenter: recibe props, emite eventos, no accede a stores ni APIs.

**Datos:**
- `DataTable` — tabla con paginación, skeleton, slots por celda (`#cell-{key}`). Se conecta con `useDataTable`.
- `KpiCard` — tarjeta de métrica (título, valor, subtítulo, icono, color, loading).
- `StatusBadge` — badge de estado con colores por estado.

**Feedback:**
- `NotificationBell` — icono con badge de no leídas, conecta con `useNotifications`.
- `ToastStack` — notificaciones efímeras posicionadas en esquina.
- `JobTrackerPanel` — progreso de jobs, conecta con `useJobTracker`.

**Contenido:**
- `MarkdownMessage` — renderizado markdown con pipeline completo (KaTeX, Mermaid, DOMPurify, artifacts).
- `ArtifactPanel` — panel lateral para contenido generado por LLM (preview/source por tipo).

### Patrón 13 — Página reactiva por suscripción

En un sistema donde todo es asíncrono y pasa por colas, la mayoría de las páginas necesitan reaccionar a eventos del backend sin que el usuario recargue. No es una excepción, es la norma.

**Flujo completo:**

1. La página carga datos iniciales (por props de Inertia o por fetch en `onMounted`)
2. En `onMounted`, se suscribe a uno o más canales vía Echo
3. Cuando llega un evento, la página reacciona: actualiza un ref, fetchea datos frescos, muestra un toast, mueve un item de estado
4. En `onUnmounted`, se desuscribe de los canales

Si una página muestra datos que pueden cambiar mientras está abierta y no se suscribe a un canal, la página está muerta — muestra datos estáticos en un sistema dinámico.

**La suscripción siempre vive en un composable, nunca en el componente directamente.** El composable encapsula el canal, los eventos que escucha, y la lógica de reacción. El componente solo consume refs reactivos:

```javascript
// composables/useJobTracker.js
export function useJobTracker(userId) {
    const activeJobs = ref([]);

    onMounted(() => {
        fetchActiveJobs().then(jobs => activeJobs.value = jobs);

        if (window.Echo) {
            window.Echo.private('jobs')
                .listen('.job.status-changed', (e) => {
                    updateJobInList(activeJobs, e);
                })
                .listen('.job.progress-updated', (e) => {
                    updateJobProgress(activeJobs, e);
                });
        } else {
            pollInterval = setInterval(() => {
                fetchActiveJobs().then(jobs => activeJobs.value = jobs);
            }, 5000);
        }
    });

    onUnmounted(() => {
        if (window.Echo) window.Echo.leave('jobs');
        if (pollInterval) clearInterval(pollInterval);
    });

    return { activeJobs };
}
```

**Estrategia de reacción al evento:**

| Situación | Estrategia | Ejemplo |
|---|---|---|
| Lo que cambió cabe en 3-4 campos primitivos | Actualización directa del ref | Progreso de job (percent, message), cambio de status, contador |
| Se necesita un objeto completo o listado | Signal + fetch HTTP | Dashboard con KPIs recalculados, listado paginado que cambió |

**Degradación sin broadcaster:** si `window.Echo` no existe (desarrollo local sin keys, caída del servicio), los composables degradan a polling HTTP con `setInterval`. Frecuencia según caso: 5s para jobs activos, 30s para notificaciones, 60s para audit feed. El sistema funciona sin broadcaster, más lento pero funciona. El broadcaster es una optimización de latencia, no un requisito.

**Múltiples suscripciones en una página:** una página puede consumir varios composables de suscripción. Cada composable gestiona su propio canal y desuscripción. No colisionan.

**Ciclo completo del sistema:**

```
Usuario acción → Controller → Handler → EventBus → Subscribers
                                                      ├→ AuditTrailSubscriber → BD + broadcast audit
                                                      ├→ NotificationSubscriber → BD + broadcast user.{id}
                                                      └→ JobFlowSubscriber → BD + broadcast jobs

Frontend escucha → Composable reacciona → Ref se actualiza → Vue re-renderiza
```

El usuario hace una acción y la consecuencia le llega de vuelta sin que nadie haya programado ese camino explícitamente. El backend emite eventos, los subscribers reaccionan, los broadcasts llegan al frontend, los composables actualizan los refs, Vue re-renderiza. Cada pieza hace lo suyo sin conocer a las demás.

---

## Apéndice A — Riesgos y decisiones conscientes

| Riesgo | Decisión |
|---|---|
| Cada nuevo medio de notificación crearía un subscriber duplicado | Notificación = mensaje + N canales. Un solo `NotificationSubscriber` orquesta canales registrados (`NotificationChannelPort`) indexados en `NotificationChannelRegistryPort`. Añadir telegram/sms/slack = adapter + entrada en `config/notifications.php`, sin tocar el subscriber |
| PipelineState en File no es compartido entre workers | Driver-based: File para un servidor, Database cuando hay múltiples workers sin disco compartido. La decisión se toma por infraestructura, no por entorno |
| Volumen de audit trail exhaustivo | Política de retención desde día 1: días de retención, particionado, cold storage |
| Spatie + EventBus subscribers duplicando audit | No coexisten sobre el mismo modelo. Si el contexto emite DomainEvents, Spatie se desactiva para esos modelos |
| Subscribers ocultan dependencias | Tabla de subscribers en documentación del proyecto. Tests de integración de cadena completa |
| Sobreingeniería de pipeline para operaciones simples | Umbral: 2+ pasos con fallo independiente = pipeline formal. 1 paso = handler simple |
| TrackableJob para jobs triviales | TrackableJob si el usuario ve el resultado en UI. Job simple de Laravel para housekeeping interno |
| Cancelación en Bus::chain no funciona | Usar patrón saga con parentTrackedJobId. No Bus::chain para flujos que necesitan cancelación |
| Pipelines concurrentes del mismo tipo | El executor verifica que no haya otro run activo del mismo tipo + entidad |
| Fallos parciales dentro de un paso | El paso es la unidad atómica. Handler tolerante a fallos por item (try/catch, registra, continúa) |
| Broadcasting de progreso con doble viaje HTTP | Excepción controlada: payload mínimo (percent, message) permitido en broadcasts de progreso |
| Broadcaster no disponible | Falla silenciosamente. Frontend degrada a polling HTTP. El sistema funciona sin broadcaster |
| Orden de ejecución de subscribers | Cada subscriber tiene su propio try/catch. Un fallo en audit no impide notificaciones |
| Subscribers que hacen operaciones pesadas | Los subscribers hacen operaciones ligeras (un INSERT). Si necesitan algo pesado, despachan un job |
| Validación de negocio en controller vs handler | Formato y tipos en controller (Form Request). Lógica de negocio (unicidad, reglas complejas) en handler |
| MCP tool con `cluster_id` desde el LLM (prompt injection) | Prohibido. El `cluster_id` se resuelve siempre con `auth()->user()->resolveActiveCluster()`. La auth se inyecta en el comando `mcp:serve`, no se acepta del input |
| LLM crea recursos saltándose reglas legales (RGPD) | Toda tool de escritura ejecuta `Validate{Entity}Draft` antes de persistir. Si hay errores, devuelve `isError=true` con la lista para que el LLM repare el draft |
| Logging a stdout rompe el protocolo MCP en stdio | `StdioTransport` no escribe en stdout fuera del JSON-RPC. Logs van a `Log::*` (fichero) o a stderr |
| Catálogo de tools desincronizado con el dominio | Tools dinámicas: leen catálogos vivos en runtime (`FormFieldStrategyRegistry`, `ListFormArchetypes`...). Sin catálogos duplicados |
| Reglas de negocio dispersas entre controller, handler, model | Una clase por regla en `app/Domain/{Context}/Rules/` implementando `{Context}BusinessRule`. Una sola fuente de verdad consumida por controllers, MCP tools y commands |
| Cambio de proveedor LLM acoplado al dominio | `LlmChatPort` aislado en `app/Domain/LlmChat/Ports/`. El adapter se selecciona por `config/llm.php` (`LLM_CHAT_ADAPTER`) sin tocar nada del dominio |
| LLM in-app crea recursos sin revisión humana | El orquestador `Conduct{Caso}Conversation` NO expone tools de escritura al LLM (whitelist explícita). La persistencia la dispara el usuario humano con un botón confirmar en el frontend. El botón solo se habilita si la última `validate_draft` no tiene errores |
| LLM in-app entra en bucle infinito de tool calls | `MAX_TOOL_ROUNDS` (constante en el orquestador) limita a 6 iteraciones por turno. Si el LLM no termina, devuelve lo último que dijo |
| Frontend del asistente envía historial manipulado | El backend valida el payload del historial (roles permitidos, content presente) y reconstruye los `ChatMessage` con `fromArray()`. El system prompt se inyecta siempre desde el servidor, no se acepta del cliente |
| Tracing técnico de alto volumen multiplica `audit_logs` si pasa por EventBus | Patrón 18: el bounded context `Observability` usa `TracerPort` directo (no `DomainEvent`) para los spans. Equivale al logging técnico — `Log::info()` tampoco pasa por bus. Solo `TraceStarted` y `TraceCompleted` (1+1 por traza) son `DomainEvent`s. El bounded context paralelo `LlmAudit` sí pasa por bus (cada `LlmCallCompleted` es un hecho de dominio con volumen acotado). |
| Tracing acoplado al bounded context que tracea | `TracerPort` genérico con `TraceKind` enum extensible y `TraceSubject` polimórfico sin FK. Chat, Knowledge y LegalCorpus comparten el mismo puerto. Cierre de trazas distribuidas mediante subscribers `On{Bounded}TerminalCloseTrace` en `Application/Observability/`. |
| LlmAudit y Observability se mezclan en cada handler | Glue object `TracedLlmCall` (`Application/Observability/Services/`) único punto de cruce. Los handlers llaman a `run(...)` y obtienen span + LlmCallRecord + métricas sin tocar VOs ni dispatch de eventos. |

---

## Apéndice B — Convenciones de nombrado

| Elemento | Convención | Ejemplo |
|---|---|---|
| Port | `{Nombre}Port` | `ImageStoragePort`, `EventBusPort` |
| Adapter | `{Nombre}Adapter` | `CloudflareImageAdapter`, `ReplicateLlmAdapter` |
| Repository | `Eloquent{Entidad}Repository` | `EloquentCompanyRepository` |
| Handler atómico | verbo + sustantivo + `Handler` | `ScanCompanyProfileHandler`, `EnrichContactPositionHandler` |
| Orquestador | `Run{Context}{Action}Handler` | `RunCompanyUpdateHandler` |
| Pipeline executor | `{Pipeline}Executor` | `CompanyImportPipelineExecutor` |
| Pipeline config | `{Pipeline}Config` | `ClusterAnalysisPipelineConfig` |
| Pipeline state | `{Pipeline}State` | `ClusterAnalysisPipelineState` |
| Domain Event | sustantivo + participio pasado | `LandingCreated`, `FileUploaded`, `CompanyProfileScanned` |
| Enum | sustantivo + tipo | `LandingStatus`, `JobStatus`, `MediaFileType` |
| Value Object | sustantivo | `Slug`, `ContactTier`, `SignalStrength` |
| Job (trackable) | verbo + sustantivo + `Job` | `DeployClusterPreviewJob`, `GenerateAiImageJob` |
| BPMN Definition | sustantivo descriptivo + `Bpmn` | `IrosAssessmentBpmn`, `MaterialityScreeningBpmn` |
| BPMN Run | `BpmnRun` | (entidad genérica, no una por dominio) |
| BPMN tool | snake_case descriptivo | `score_impact_materiality`, `enrich_sector_context` |
| BPMN event | `Bpmn` + sustantivo + participio | `BpmnRunStarted`, `BpmnNodeCompleted`, `BpmnHumanGateReached` |
| BPMN handler | verbo + `BpmnRun` + `Handler` | `StartBpmnRunHandler`, `ResumeBpmnRunHandler` |
| BPMN job | `ExecuteBpmnRunJob` | (único, genérico) |
| Composable | `use{Nombre}` | `useJobTracker`, `useStreaming`, `useScrollAnchor` |
| API function | verbo + sustantivo | `listCompanies`, `importCompany`, `openMessageStream` |
| Store (Pinia) | `use{Nombre}Store` | `useChatStore`, `useToastStore` |
| Container (Vue) | `{Dominio}/Index.vue` | `Pages/Chat/Index.vue` |
| Presenter (Vue) | `{Dominio}/Partials/{Nombre}.vue` | `Partials/ChatSidebar.vue` |
| Event broadcast | `dominio.accion` | `job.status-changed`, `audit.entry-created` |
| Event name (domain) | `contexto.accion` | `landing.created`, `forms.submitted` |
| Permiso | `familia.accion` (español) | `landings.ver`, `clusters.desplegar-preview` |
| Notification channel | `{Nombre}NotificationChannel` | `InAppNotificationChannel`, `MailNotificationChannel`, `TelegramNotificationChannel` |
| Notification channel name (string) | snake_case corto | `'in_app'`, `'mail'`, `'telegram'`, `'sms'` |
| Notification email template | `emails.notifications.{notifiable_snake}.{step_action}` | `emails.notifications.contact_form_lead.lead_created` |
| MCP tool (clase) | `{Verbo}{Sustantivo}Tool` | `ListFieldTypesTool`, `CreateContactFormTool`, `ValidateFormDraftTool` |
| MCP tool (nombre canónico) | `{contexto}.{accion_snake_case}` | `forms.list_field_types`, `forms.create_contact_form`, `taxonomy.list_categories` |
| MCP resource (clase) | `{Sustantivo}Resource` | `FormGuidelinesResource` |
| MCP resource URI | `sicaland://{contexto}/{recurso-kebab}` | `sicaland://forms/guidelines`, `sicaland://forms/field-types` |
| Business rule (clase) | enunciado afirmativo en PascalCase | `TermsRequiredWithPersonalData`, `MarketingConsentNotPreChecked`, `UniqueFieldNames` |
| Business rule id (string) | `{categoria}.{regla_snake_case}` | `'rgpd.terms_required_with_personal_data'`, `'ux.success_message_recommended'` |
| LLM chat adapter | `{Proveedor}{Modelo}ChatAdapter` | `ReplicateClaudeHaikuChatAdapter`, `OpenAiGpt4MiniChatAdapter` |
| Orquestador conversacional in-app | `Conduct{Caso}Conversation` | `ConductFormDesignConversation`, `ConductLandingDesignConversation` |
| Página Vue de asistente in-app | `{Recurso}/AiDesigner.vue` | `ContactForms/AiDesigner.vue` |
| Componentes Vue compartidos del asistente | `Components/AiDesigner/{Funcion}.vue` | `ChatPanel.vue`, `ChatMessage.vue`, `FormDraftPreview.vue`, `FieldPreview.vue` |
| Tracer port | `TracerPort` (genérico, agnóstico del BC) | `App\Domain\Observability\Ports\TracerPort` |
| Trace kind (enum) | PascalCase + value snake_case | `ChatStandard => 'chat_standard'`, `KnowledgeIngestion => 'knowledge_ingestion'`, `LegalDiscovery => 'legal_discovery'` |
| Span kind (enum) | PascalCase + value snake_case | `Pipeline`, `Step`, `LlmCall => 'llm_call'`, `Embedding`, `Clustering`, `Frontend` |
| Trace subject type (string) | snake_case | `'conversation'`, `'knowledge_item'`, `'legal_document_version'` |
| Glue object trazado + audit | `Traced{Operacion}` en `Application/Observability/Services/` | `TracedLlmCall` |
| Subscriber de cierre de traza distribuida | `On{Bounded}{Action}CloseTrace` | `OnKnowledgeProcessingTerminalCloseTrace` |
| Eventos de Observability por bus | sustantivo + participio (solo lifecycle) | `TraceStarted`, `TraceCompleted` |

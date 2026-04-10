---
name: "Laravel Side Effects"
description: "Analiza minuciosamente los efectos colaterales de cambios propuestos en proyectos Laravel antes de ejecutarlos. Revisa model events, observers, resources, middleware, jobs y mas."
globs:
  - "app/**/*.php"
  - "routes/*.php"
  - "database/migrations/*.php"
  - "config/*.php"
---

# Impact Review - Analisis de Efectos Colaterales (Laravel)

Antes de ejecutar cualquier cambio o plan, realiza un analisis exhaustivo de impacto enfocado en el ecosistema Laravel. No procedas a implementar hasta completar este analisis.

## Cuando activar

- Al crear o ejecutar un plan de implementacion
- Al proponer cambios en modelos, controladores, servicios, o estructura de datos
- Cuando el usuario pida revisar efectos colaterales de una propuesta

## Paso 0: Leer configuracion del proyecto

**ANTES de hacer cualquier otra cosa**, lee el archivo de configuracion:

```
Read: {skill_base_dir}/config.json
```

Donde `{skill_base_dir}` es el directorio base de este skill (proporcionado como "Base directory for this skill" cuando se invoca).

El archivo `config.json` contiene:

- **project**: nombre, path al repositorio, y framework
- **directories**: rutas relativas a los directorios clave del proyecto (models, observers, resources, etc.)
- **criticalModels**: modelos que requieren revision extra con sus resources y observers asociados
- **criticalTables**: tablas de BD que son criticas y su nivel de seguridad
- **customRules**: reglas especificas del proyecto que siempre deben verificarse
- **ignorePaths**: directorios a ignorar en el analisis
- **notifications**: flags para activar/desactivar advertencias especificas

**Si `config.json` no existe**: Informa al usuario que debe crear su configuracion personal. Muestra el contenido de `config.example.json` como referencia y pide que lo copie como `config.json` con sus datos. No continues hasta que exista.

Usa los valores de `config.json` en todos los pasos siguientes. En estas instrucciones, las referencias como `config.project.path` significan el valor de esa clave en el JSON.

## Proceso de analisis

### 1. Identificar el alcance del cambio

- Que modelos, controladores, servicios, o clases se modifican directamente
- Que tablas o columnas de base de datos se afectan
- Que rutas o endpoints cambian
- **Verificar contra `config.criticalModels`**: si alguno de los modelos afectados esta en la lista, marcarlo inmediatamente como revision prioritaria
- **Verificar contra `config.criticalTables`**: si alguna tabla afectada esta en la lista, elevar la severidad del analisis

### 2. Cadena de efectos en Eloquent

Este es el punto mas critico. Un simple `save()`, `update()`, `delete()` o `create()` en un modelo puede desencadenar una cadena invisible de efectos.

#### Model Events y Observers
- Buscar si el modelo tiene `boot()` o `booted()` con closures registradas (`creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restoring`, `restored`)
- Buscar si existe un Observer registrado para el modelo en `config.directories.observers`
- Buscar en `EventServiceProvider` o `AppServiceProvider` si hay observers registrados con `Model::observe()`
- Si el modelo esta en `config.criticalModels`, verificar sus `relatedObservers` listados
- **Pregunta clave**: Si el cambio introduce un nuevo `save()` o `update()`, se van a disparar estos eventos en un contexto donde no se esperaba? (ej: un observer que envia un email cada vez que se actualiza un modelo)
- **Pregunta clave**: Si se modifica la logica de un observer, que otros flujos que hacen `save()` en ese modelo se ven afectados?
- Si `config.notifications.warnOnNewSave` es true: alertar siempre que se introduzca un nuevo `save()`, `update()`, `create()` o `delete()` en cualquier modelo critico

#### Model Events via `$dispatchesEvents`
- Verificar si el modelo tiene `$dispatchesEvents` que mapea eventos del modelo a clases Event custom
- Rastrear los listeners de esos eventos en `EventServiceProvider`

#### Traits del modelo
- Revisar traits que el modelo usa — muchos traits registran eventos en su `boot` (ej: `SoftDeletes`, traits custom de audit, sluggable, etc.)
- Un cambio en el modelo puede activar logica oculta en traits

### 3. API Resources y Response Contracts

#### JsonResource / ResourceCollection
- Si se modifica un modelo (agregar/renombrar/eliminar atributo, cambiar relacion, modificar accessor/mutator):
  - Buscar todos los `JsonResource` que transforman ese modelo en `config.directories.resources`
  - Si el modelo esta en `config.criticalModels`, verificar especificamente sus `relatedResources`
  - Verificar si el Resource usa `$this->attribute` directamente — un rename rompe silenciosamente
  - Verificar si el Resource expone relaciones con `whenLoaded()` — un cambio en el nombre de la relacion causa mismatch
  - Verificar si hay `ResourceCollection` custom asociados
- **Pregunta clave**: El frontend o clientes de la API esperan una estructura especifica? Un campo renombrado o eliminado en el modelo que no se actualiza en el Resource genera un mismatch silencioso (retorna `null` en vez de error)

#### API Versioning
- Si la API tiene versionamiento, verificar que los Resources de versiones anteriores no se rompan

### 4. Request Validation y Form Requests

- Si se modifica la estructura de datos de un modelo:
  - Buscar `FormRequest` asociados en `config.directories.requests`
  - Verificar que las reglas de validacion coincidan con la nueva estructura
  - Revisar si hay `prepareForValidation()` o `passedValidation()` que manipulan datos
- Si se cambian reglas de validacion:
  - Considerar datos existentes que ya pasaron con reglas anteriores
  - Verificar si hay tests que validen contra las reglas viejas

### 5. Jobs, Queues y Listeners

- Buscar Jobs que procesan el modelo afectado en `config.directories.jobs`
- Buscar Listeners que reaccionan a eventos del modelo en `config.directories.listeners`
- Buscar Notifications que usan el modelo en `config.directories.notifications`
- **Pregunta clave**: Si un Job serializa el modelo (implements `SerializesModels`), un cambio en la estructura del modelo puede romper jobs que ya estan en cola
- Revisar si hay `ShouldQueue` listeners — el cambio puede afectar procesamiento asincrono con datos desactualizados

### 6. Relaciones y Eager Loading

- Si se modifica una relacion (`hasMany`, `belongsTo`, `morphTo`, etc.):
  - Buscar donde se usa esa relacion con eager loading (`with()`, `load()`)
  - Buscar donde se accede en blades o resources (`$model->relation`)
  - Verificar si hay `$with` (auto eager loading) en el modelo
- Si se renombra una relacion:
  - Buscar todos los usos de la relacion por nombre en el proyecto
  - Verificar Resources que usen `whenLoaded('nombreViejo')`
  - Si `config.notifications.warnOnRelationChange` es true: alertar explicitamente

### 7. Scopes, Accessors y Mutators

- Si se modifica/agrega un scope global (`addGlobalScope`):
  - Todos los queries del modelo se afectan — verificar que no rompa queries existentes
  - Buscar donde se usa `withoutGlobalScope()` para excluirlo
- Si se modifica un accessor (`get*Attribute` o nuevo estilo `Attribute::get()`):
  - Verificar Resources y Blades que leen ese atributo — el valor cambia silenciosamente
  - Verificar si se usa en condiciones logicas o calculos
- Si se modifica un mutator (`set*Attribute` o `Attribute::set()`):
  - Todos los `save()` y `update()` que setean ese campo se afectan
  - Verificar si hay seeders, factories o imports que dependen del comportamiento anterior

### 8. Middleware y Policies

- Si se modifica middleware (buscar en `config.directories.middleware`):
  - Verificar todas las rutas/grupos que usan ese middleware en `config.directories.routes`
  - Un cambio en middleware de autenticacion puede bloquear/abrir acceso inesperadamente
- Si se modifica una Policy (buscar en `config.directories.policies`):
  - Verificar que los `authorize()` en controladores y `can()` en blades sigan funcionando
  - Buscar `Gate::define()` y `Gate::before()` que puedan interferir

### 9. Casts, Enums y Value Objects

- Si se modifica un cast custom o se cambia el tipo de cast de un atributo:
  - Datos existentes en BD pueden no ser compatibles con el nuevo cast
  - Verificar serializacion/deserializacion
  - Si `config.notifications.warnOnCastChange` es true: alertar explicitamente
- Si se modifica un Enum usado en casts:
  - Datos existentes con valores del enum viejo pueden causar errores

### 10. Base de datos

- Migraciones: agregan, eliminan o modifican columnas/tablas?
- Verificar si las tablas afectadas estan en `config.criticalTables` — si es asi, elevar severidad
- Datos existentes: se pierden, corrompen o quedan inconsistentes?
- Indices: se afecta el rendimiento de queries existentes?
- Foreign keys: se rompen constraints?
- Seeders (en `config.directories.seeders`) y factories (en `config.directories.factories`): necesitan actualizacion?
- **Rollback**: la migracion tiene `down()` funcional?
- Si `config.notifications.warnOnMigrationWithoutDown` es true: alertar si falta `down()`

### 11. Config, Cache y Deployment

- Variables de entorno nuevas sin defaults?
- Se necesita `config:cache`, `route:cache`, `view:cache` clear?
- Cambios en `config.directories.config` que requieran re-deploy?
- Se afectan scheduled commands en `Console/Kernel.php` o `routes/console.php`?

### 12. Service Providers y Bindings

- Si se modifica una clase que esta bindeada en un ServiceProvider (buscar en `config.directories.providers`):
  - `$this->app->bind()`, `$this->app->singleton()`
  - Todos los que resuelven esa interfaz/clase del container se afectan
  - Verificar si hay fachadas (Facades) que apunten a ese binding

### 13. Mass Assignment y Fillable

- Si se agregan o eliminan campos del modelo:
  - Verificar que `$fillable` o `$guarded` esten actualizados
  - Si `config.notifications.warnOnFillableChange` es true: alertar explicitamente
  - Un campo nuevo sin `$fillable` causa MassAssignmentException
  - Un campo eliminado de `$guarded` puede abrir vectores de ataque

### 14. Seguridad

- Se expone informacion sensible en Resources o respuestas?
- Se debilitan gates, policies o middleware de autorizacion?
- Se afectan tokens, sesiones o mecanismos de autenticacion?

### 15. Rendimiento

- Se introducen queries N+1? (relaciones accedidas sin eager loading)
- Se invalida cache existente sin reconstruirlo?
- Se agregan operaciones costosas en request lifecycle?
- Se agregan queries dentro de loops?

### 16. Reglas custom del proyecto

Iterar sobre `config.customRules` y verificar cada una:

Para cada regla:
1. Evaluar si la condicion `when` aplica al cambio propuesto
2. Si aplica, ejecutar el `check` descrito
3. Clasificar el hallazgo segun la `severity` de la regla

## Clasificacion de hallazgos

### Critico (bloquean ejecucion)
Perdida de datos, downtime, vulnerabilidades de seguridad, mismatch silencioso en API responses que afecta clientes en produccion. **No proceder** sin resolver.

### Alto (requieren atencion)
Eventos/observers que se disparan inesperadamente, Resources desactualizados, relaciones rotas, jobs en cola que fallan. Resolver antes de continuar.

### Medio (considerar)
Efectos secundarios controlables, cache que necesita limpiarse, tests que necesitan actualizacion. Documentar y planificar.

### Bajo (informativo)
Cambios cosmeticos, mejoras opcionales, refactors sugeridos.

## Formato de salida

```
## Analisis de Impacto — [config.project.name]

**Cambio propuesto:** [descripcion breve]
**Modelos afectados:** [lista, marcar con ⚠ si estan en criticalModels]
**Tablas afectadas:** [lista, marcar con ⚠ si estan en criticalTables]
**Archivos afectados:** [lista]

### Cadena de efectos detectada

[Diagrama o lista de: cambio → evento/observer → listener → accion]

### Reglas custom evaluadas

- [regla.name]: [aplica/no aplica] — [resultado si aplica]

### Hallazgos

#### Critico
- [hallazgo]: [donde ocurre] → [que rompe] → [mitigacion]

#### Alto
- [hallazgo]: [donde ocurre] → [que rompe] → [mitigacion]

#### Medio
- [hallazgo]

#### Bajo
- [hallazgo]

### Conclusion
[Resumen: es seguro proceder? Con que precauciones? Que pasos adicionales se necesitan?]
```

## Reglas

- **No omitir pasos** - Completar el analisis completo aunque parezca obvio
- **Leer config primero** - Siempre leer `config.json` antes de analizar. Los modelos y tablas criticas definen prioridades
- **Ser especifico** - Nombrar modelos, observers, resources, middlewares y archivos concretos
- **Rastrear la cadena completa** - No parar en el primer nivel de efecto; seguir hasta el final (save → observer → event → listener → notification → ...)
- **Preferir falsos positivos** - Mejor alertar de un riesgo que no existe a ignorar uno real
- **Considerar produccion** - Siempre pensar en datos reales, jobs en cola, y clientes activos
- **Verificar mismatch silencioso** - Laravel no lanza error cuando un Resource accede a un atributo que ya no existe (retorna null). Buscar estos activamente
- **Evaluar reglas custom** - Siempre iterar sobre `config.customRules` y verificar cada una
- **Respetar notifications** - Los flags en `config.notifications` determinan que advertencias adicionales emitir
- **Documentar supuestos** - Si no puedes verificar algo, indicarlo como supuesto
- **Idioma** - Todo el analisis en espanol

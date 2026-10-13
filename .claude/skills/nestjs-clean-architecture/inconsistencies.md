# Inconsistencias del proyecto modelo: no son reglas

Referencia histórica: `wms-api`, commit `261d7ee`. El análisis fue estático; no se ejecutó la suite ni se comprobó el despliegue externo. Las rutas siguientes identifican evidencia del modelo, no archivos que deban existir en el proyecto destino.

No copiar estas excepciones para lograr una supuesta fidelidad arquitectónica. Tampoco corregirlas automáticamente en un proyecto ajeno al alcance solicitado.

## Diferencia respecto de la regla de organización del equipo

El modelo original contiene entidades y controllers relativamente planos y drivers agrupados por integración. Esa organización es evidencia histórica, pero no satisface la regla posterior del equipo: capa → dominio/recurso → funcionalidad/caso de uso. Los ejemplos se conservan como procedencia; nuevos archivos y flujos refactorizados deben seguir [architecture.md](architecture.md). La separación por dominio/caso de uso es una decisión explícita del equipo, no una conclusión atribuida al modelo.

## Dependencias y composición

| Evidencia de origen | Inconsistencia o límite | Cómo tratarla |
|---|---|---|
| `src/core/adapters/repositories/INotificationRepository.ts` | El puerto depende de `NotificationEntity`, decorada por TypeORM. | No usarlo como plantilla de contrato independiente; separar modelo propio de persistencia cuando esa corrección esté en alcance. |
| `src/core/drivers/repositories/notification/NotificationRepository.ts` | Contiene decoradores Nest e `update` no implementado. | No generalizar que todos los drivers de core son independientes del framework ni copiar el stub. |
| `src/app/module/WMS.module.ts` | Persistencia definida pero no importada por los módulos de arranque inspeccionados. | No presentarla como persistencia activa ni como base obligatoria de toda feature. |
| `src/app/drivers/NestERPWMSRepository.ts` | Entrega `strategies: []`; tests sí instancian `TransferOutDispatchedStrategy`. | Verificar wiring además de tests aislados. |
| `src/core/interactors/events/UpdateOrdersEvent.ts` y logger propio | Lectura directa de entorno; interactors importan logger concreto. | Son acoplamientos reales, no evidencia de puertos de configuración/logging ya existentes. |
| `src/core/interactors/types.ts` | Infraestructura de callbacks/listeners sin uso observado en los casos actuales. | Predomina async/await y Promise; no adoptar listeners como requisito. |

## Controllers y contratos HTTP

| Evidencia de origen | Inconsistencia o límite | Cómo tratarla |
|---|---|---|
| `src/app/controller/events.controller.ts` | Filtra eventos según estado/tipo y publica en cola desde el controller. | Excepción al controller puramente delegador; preservar comportamiento al extraer la política. |
| Controllers que extienden `HATEOASController` | Algunos métodos del objeto de operaciones son `undefined`, devuelven `undefined` o no tienen implementación. | No exponer rutas sin operación implementada; no copiar `loadService` que siempre lanza error. |
| `src/app/controller/hateoas/HATEOASController.ts` | Listados generan enlaces con `entity['id']`, aunque otros métodos identifican por `externalId`/`sku`; query params carecen de conversión explícita. | Verificar identidad y paginación en el controller destino. |
| `src/main.ts` | CORS permite GET/POST mientras existen PUT/DELETE; no se observan guards de autenticación de entrada. | No convertir esas omisiones en política general. Protección externa no verificada. |
| Upload controller y repositorio HTTP | Archivo tipado `Buffer`, pero se utiliza como objeto con `.buffer` procedente del upload. | Alinear tipos entre entrada, interactor y driver; el test HTTP muestra el shape real. |

## Validaciones y errores

| Evidencia de origen | Inconsistencia o límite | Cómo tratarla |
|---|---|---|
| Creación de ventas/compras/transferencias | Arrays ausentes o inválidos pueden alcanzar `map`/`forEach`; algunas iteraciones ocurren antes de validar. | Verificar forma/presencia primero; no copiar el orden defectuoso. |
| `DomainException.ts` | Status por defecto 403; otros flujos usan ValidationException 400 o Error ordinario. | No imponer una semántica única que no existe; elegir explícitamente según contrato del destino. |
| `APIAuroraWMSRepository.ts`, `APIERPWMSRepository.ts` | Condiciones `status < 200 && status >= 300` imposibles. | No usarlas como ejemplo de comprobación HTTP. |
| Lecturas de repositorios HTTP | Algunos catch sólo loguean y devuelven implícitamente undefined. | No equiparar caída externa con entidad inexistente; cambiar semántica de manera explícita. |
| `ShippingApiRepository.getZplLabelUrl` | En catch construye JSON sin invocar Logger. | No copiar ese bloque como logging funcional. |
| `IWMSRepository.ts` y su implementación | Método duplicado, retornos amplios/inconsistentes y firmas débilmente tipadas. | Extraer forma de contrato, no errores; revisar semántica real. |
| `InMemoryNotificationRepository.ts` | `save` lee índice length tras push y `update` no hace nada. | No presentarlo como doble completo/correcto. |

## Configuración, seguridad y herramientas

- `SpreadSheetReader.ts` contiene credenciales OAuth y una clave privada incrustadas; un script operativo también contiene una clave de API. No se incluyen sus valores en esta Skill. Si se trabaja sobre ese proyecto, corresponde revisar su vigencia y rotarlas/revocarlas según el acceso autorizado.
- Los helpers HTTP registran headers y body; no copiar ese comportamiento con información sensible. El logging estructurado es reutilizable, el contenido indiscriminado no.
- `.local.env`, archivos `deploy/env*` y SQLite están versionados. No usar esos archivos como configuración de otro proyecto ni asumir que sus nombres son cargados automáticamente por ConfigModule.
- La configuración mezcla imports de Nest 10 y microservices 11, y contiene dos lockfiles. No trasladar versiones ni reproducir conflictos de instalación.
- `form-data` se importa directamente sin declararse como dependencia directa en el manifiesto inspeccionado. No depender por accidente de su presencia transitiva.
- TypeScript desactiva varios controles estrictos y ESLint permite `any`. No bajar la exigencia de un destino para imitarlo.
- El build excluye `*.spec.ts` aunque los tests usan `.test.ts`; los scripts de producción y Docker señalan rutas de salida diferentes. Verificar salida efectiva; no copiar comandos sin comprobarlos.
- Los workflows solicitan omitir pruebas unitarias/integración. El script Jest e2e apunta a una configuración ausente; Cucumber tiene hooks pero no escenarios `.feature`. Ninguno es evidencia de validación completa.
- Los scripts operativos de la raíz hacen mutaciones remotas y contienen listados de datos; no forman parte de los flujos de desarrollo ni de las verificaciones de esta Skill.

## Qué no puede inferirse del modelo

No hay evidencia de un estándar implementado de Redis, CQRS, buses de comandos, símbolos propios de DI, mappers dedicados, agregados ricos, transacciones/Unit of Work, migraciones, reintentos con DLQ o idempotencia general. Tampoco de jobs cron concretos aunque exista un módulo de scheduling.

Introducir cualquiera de estas capacidades puede ser razonable según el destino. Debe describirse como una decisión nueva, con su necesidad, ubicación, configuración y pruebas, no como una regla deducida de este modelo.

# Convenciones de nombres e imports

## Elegir sin inventar uniformidad

Aplicá la regla del equipo de dominio/caso de uso a archivos nuevos y reorganizados. En código existente, conservá nombres públicos salvo que el refactor incluya cambiarlos. Para archivos nuevos sin precedente local, usá la forma predominante indicada aquí. Las variantes históricas no obligan a introducir nombres inconsistentes en módulos nuevos.

| Elemento | Forma reutilizable | Observación |
|---|---|---|
| Dominio/recurso | carpeta kebab-case, nombre estable | Usar el mismo nombre entre capas, por ejemplo `products`. |
| Funcionalidad/caso de uso | carpeta kebab-case que nombra la operación | Usar el mismo nombre entre capas, por ejemplo `list-products`. |
| Interactor | `VerbConcept.ts`, clase `VerbConcept` | Sin sufijo obligatorio `UseCase`, `Interactor` o `.service`. |
| Método de interactor | lowerCamelCase que nombra la operación | `create`, `findById`, `updateBySku`, `changeStatus`; no hay `execute()` general. |
| Dependencias de constructor | objeto tipado, parámetro `builder` | Hay interfaces `builderConcept`, `ConceptBuilder` y `BuilderConcept`; no existe un único nombre predominante. |
| Propiedad de dependencia | lowerCamelCase según capacidad | `repository`, `cacheManager`, o nombre funcional más específico. |
| Puerto | `IOperation.ts` / `IOperation`, o contrato compartido `IConceptRepository` | `I` predomina en contratos de integración; también hay `IQueueConnection` e `ICacheManager`. |
| Driver de caso de uso | `OperationRepository.ts` / `OperationRepository` | Por ejemplo `GetAllProductsRepository`; proveedor/tecnología se distingue sólo cuando aporta claridad. |
| Wrapper Nest | `NestIntegrationRepository.ts` / misma clase | En cache/queue hay drivers sin prefijo Nest; no renombrarlos automáticamente. |
| Implementación en memoria | `InMemoryConceptRepository.ts` | También existe `InMemoryCacheManager`. |
| Controller | `concept.controller.ts`, clase `ConceptController` | El basename es lowerCamelCase cuando tiene varias palabras. |
| Módulo funcional | `Concept.module.ts`, clase `ConceptModule` | Excepciones: `app.module.ts` y módulo de worker lowerCamelCase. |
| Servicio de ejecución | `Concept.service.ts`, clase `ConceptService` | Poca evidencia: un consumidor de mensajes, no un estándar de services CRUD. |
| Modelo | `Concept.ts`, clase `Concept` | Variantes relacionadas pueden compartir el archivo. |
| Entrada/salida | `ConceptRequest`, `ConceptResponse`, `ConceptListing`, `ConceptCreate`, `ConceptUpdate` | También se usa `UpdateConcept`; conservar consistencia local. |
| Excepción | `SpecificException.ts`, clase `SpecificException` | Herencia desde `DomainException` cuando corresponde. |
| Filtro | `domainException.filter.ts` / `DomainExceptionFilter` | Responsabilidad de transporte en `app/controller`. |
| Test | `Subject.test.ts` junto al código | Las integraciones HTTP también usan `subject.controller.test.ts` o `subjectController.integration.test.ts`. |
| Token de interactor | string con el nombre de clase | Mismo literal en `provide` y `@Inject`. |

No crear una convención global de `Symbol`, archivos `.port.ts`, `.adapter.ts`, `.use-case.ts` o carpetas `dto` por suponer que toda Clean Architecture los usa. Si ya existen en el destino, evaluar compatibilidad con el alcance antes de cambiarlos.

## Correspondencia entre capas

Ejemplo normativo del equipo: `products/list-products` contiene el puerto `IGetAllProducts`, driver `GetAllProductsRepository`, interactor `GetAllProducts`, controller `listProducts.controller.ts`, wrapper `NestGetAllProductsRepository` y módulo `ListProducts.module.ts`. `Product.ts` compartido vive en `core/entities/products/`.

No alternar `product`, `products`, `get-products` y `list-products` para la misma jerarquía. Mantener el token exactamente igual en la factory y el punto de inyección. Los nombres de proveedores del modelo son evidencia histórica, no el criterio primario para organizar drivers nuevos.

## Modelos y DTOs

La metodología de referencia utiliza clases de datos sin decoradores de transporte u ORM en `core/entities`. Los nombres Request/Response/Create/Update describen variantes; `Dto` aparece en variables, pero no es un sufijo obligatorio de las clases.

Puede haber más de una clase por archivo cuando pertenece al mismo concepto. No confundir este modelo de datos con un agregado de dominio que encapsula todas las reglas: las reglas predominan en interactors.

Si un nombre de input colisiona con un interactor, el modelo usa aliases al importar. En código nuevo sin contrato establecido, elegí nombres que distingan ambos papeles; esa elección es una normalización razonada, no un estándar único hallado en el repositorio.

## Imports

Los aliases disponibles en el modelo son `@core/*`, `@app/*`, `@utils/*`. Coexisten imports relativos y algunos desde `src/...`. Los aliases son una opción respaldada y frecuente; no afirmar que los relativos están prohibidos.

Para un destino nuevo:

1. Configurá los aliases que efectivamente se usarán en TypeScript y en el runner de tests.
2. Comprobá cómo los resuelve el build y el comando de producción. `paths` por sí solo no es garantía de resolución por Node del JavaScript emitido.
3. Usá rutas relativas para código vecino si es la convención local; evitá agregar una tercera forma innecesaria.

Para un destino existente, no copies automáticamente el `moduleNameMapper` ni cambies el bundler. Verificá su configuración real. La dirección de la dependencia importa más que la sintaxis de la ruta; ver [responsibilities.md](responsibilities.md).

## Formato y variantes que no deben copiarse

El modelo usa Prettier con comillas simples, trailing commas y punto y coma en el código mostrado. Su ESLint permite `any` y TypeScript tiene controles estrictos desactivados. Eso describe tolerancias del proyecto, no una regla para debilitar el tipado de otro repositorio.

No copiar typos, nombres de builders arrastrados de otra feature, diferencias accidentales entre nombre de archivo y clase, o imports sin uso. No renombrar todos los archivos existentes por estética: conservar contratos y limitar el diff al objetivo.

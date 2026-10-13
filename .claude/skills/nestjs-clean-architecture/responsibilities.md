# Responsabilidades y decisiones de ubicación

## Ubicación dentro de cada capa

Primero elegí la capa por responsabilidad; después el dominio y el caso de uso. Para componentes exclusivos, usá `<layer>/<domain>/<use-case>/`. Para componentes realmente compartidos, elegí el nivel común más cercano dentro de esa capa. La jerarquía conceptual debe coincidir entre `app` y `core`.

Una ubicación correcta por capa no compensa mezclar funcionalidades en una carpeta plana. Las entidades compartidas pertenecen a `core/entities/<domain>`; contratos/drivers comunes se elevan al dominio o área transversal explícita sin duplicarse. Consultá [architecture.md](architecture.md) para reglas y ejemplos de promoción.

## Matriz de decisión

| Pregunta | Ubicación | Límite |
|---|---|---|
| ¿Lee body, params, query, headers o archivos de HTTP? | `app/controller` | No consultar proveedor o base directamente. |
| ¿Construye enlaces o envoltorios de respuesta HTTP? | Controller/base de presentación | No hacerlo dentro del repositorio. |
| ¿Decide si una operación está permitida o qué pasos ejecutar? | `core/interactors/<domain>/<use-case>` | No depender del contenedor Nest ni del cliente externo concreto. |
| ¿Valida campos o invariantes requeridos por la operación? | Interactor + `interactors/validations` | No confiar en tipos TypeScript como validación runtime. |
| ¿Define estructuras de datos y enums compartidos? | `core/entities` | No introducir decoradores ORM o dependencias Nest en modelos ordinarios. |
| ¿Expresa una capacidad requerida de un sistema externo? | `core/adapters/<domain>/<use-case>` o nivel común | Interfaz orientada a capacidad, no reexportación de un SDK. |
| ¿Arma requests, serializa payloads o interpreta respuestas externas? | Driver de integración | No trasladar URLs, headers ni detalles del proveedor al interactor. |
| ¿Ejecuta consultas o gestiona entidades ORM? | Driver de persistencia | No propagar el EntityManager o entidad ORM mediante el puerto. |
| ¿Necesita `@Injectable`, `HttpService` o tokens Nest para conectar un cliente? | `app/drivers` y módulo | No convertir el interactor en injectable para resolver ese detalle. |
| ¿Decide qué implementación concreta recibe cada operación? | `app/module` | La composición no contiene lógica comercial. |
| ¿Recibe un mensaje o se dispara por scheduler? | `app/service`/adaptador de entrada registrado | Delegar la operación al interactor; configuración del transporte en infraestructura. |
| ¿Traduce DomainException a HTTP? | Filtro en `app/controller` | No usar Express dentro del dominio/interactor. |
| ¿Comparte utilidad puramente de integración? | Driver común o helper acotado | `utils` no es un lugar para esconder casos de uso. |

## Controllers

Reciben dependencias por constructor. En el patrón principal inyectan interactors mediante strings y usan un objeto `HateoasInteractor` para adaptar métodos a una base de presentación. Ese objeto es una interfaz de presentación, no un servicio obligatorio de negocio.

Los controllers pueden transformar una respuesta de listado a `{ data, total }`, manejar uploads y construir enlaces. No deben consultar ORM o APIs externas; no hay ejemplos de ese acceso directo en los controllers del modelo.

Excepción observada: el controller de eventos llama un puerto de cola e incluye decisiones por estado/tipo de evento. Conservar su comportamiento en un refactor compatible; no convertir esa mezcla en permiso general para poner reglas nuevas en controllers. Separar la política de aceptación de un evento en un interactor es una corrección explícita, no una descripción falsa del código original.

## Interactors

Son clases TypeScript ordinarias. El constructor recibe un objeto de dependencias tipadas por interfaces. Sus métodos ejecutan operaciones concretas; algunos agrupan lecturas o variantes de actualización relacionadas, de modo que no hay una regla de exactamente un método por clase.

Responsabilidades observadas: validación, verificación de existencia/conflictos, reglas sobre estados, coordinación de puertos, normalización de datos y retorno de modelos. Las reglas específicas del negocio original no son parte de esta Skill.

Los interactors no instancian repositorios concretos, no hacen HTTP ni consultas SQL y no importan `@app` o decoradores Nest. Sí hay uso de logger estático y `process.env` en casos concretos; registrar esos acoplamientos como excepciones, no declarar una pureza inexistente ni exigir puertos nuevos para todo cambio pequeño.

## Contratos, repositorios y drivers

Los contratos definen métodos asíncronos con entradas/salidas del modelo o primitivas. El término Repository también se usa para APIs remotas, no sólo para almacenamiento SQL.

Reutilizá un contrato si representa la capacidad requerida. El modelo tiene contratos amplios y otros pequeños; no exige ni un repositorio por entidad ni una única interfaz para todas las integraciones. Evitá expandir un contrato con operaciones ajenas por mera comodidad. Dividir uno existente requiere actualizar consumidores e implementaciones de manera compatible.

Los drivers encapsulan protocolo, autenticación saliente, query/path/body, serialización y errores de la dependencia. Las implementaciones Nest toman clientes/tokens del framework. El wrapper HTTP predominante extiende la implementación y entrega el cliente al constructor.

La nueva implementación debe satisfacer realmente el contrato: la presencia de `implements` con `any` o controles estrictos desactivados no garantiza retornos correctos. Verificar semántica de ausencia, errores y valores de retorno.

## Entidades, mapeos y validación

Los modelos ordinarios son estructuras sin ORM. Los modelos persistidos pertenecen al driver. No hay una capa DTO separada ni mappers obligatorios.

Los mapeos semánticos de estados y normalizaciones de la operación aparecen en el núcleo; el shape específico de una API externa se construye en su repositorio. Hay también una estrategia de transformación local. Elegí ubicación según qué cambia: regla de negocio o contrato del proveedor.

La validación predominante utiliza `Validate.validate`, `validateNonEmptyObject`, validaciones de arrays y acumulación de errores. Las opciones distinguen `null` de `undefined`. Los helpers del modelo tienen semántica particular: `string` exige texto no vacío, `number` exige valor no negativo y `array` exige arreglo no vacío. No copiar esos nombres suponiendo la semántica de los tipos nativos para todos los dominios.

Validá forma y presencia antes de iterar, desestructurar o modificar. Esto corrige una inconsistencia del modelo. No trasladar automáticamente validaciones al controller sólo porque hay `ValidationPipe`: en el patrón observado no hay DTOs con decoradores de class-validator.

## Errores

`DomainException` extiende `Error` e incluye status numérico. Subclases representan validación, ausencia y conflictos específicos. El filtro HTTP construye la respuesta con status, mensaje, timestamp y ruta.

La excepción de dominio ya conoce un status HTTP: es un acoplamiento del modelo. No presentar un sistema independiente de transporte como si ya estuviera implementado. Para otro proyecto, conservar su contrato de errores si existe; para adoptar el patrón, usar estas excepciones sin copiar su status por defecto a ciegas.

Distinguir ausencia de una entidad de fallo de red. No absorber todos los errores de una lectura y devolver `undefined`; no convertir cualquier error externo en un 404. En refactors, si esa semántica ya es pública, explicitá el cambio y probalo.

Un filtro que llama `switchToHttp()` es para HTTP. No asumir que maneja correctamente fallas de consumidores de mensajes. La política de reconocimiento/reintento requiere su propio análisis de transporte.

## Imports y dependencia inversa

| Consumidor | Dependencias esperables | Señal a revisar |
|---|---|---|
| Entidades ordinarias | Otras entidades, enums y tipos propios | Nest, ORM, HTTP, drivers concretos |
| Puertos | Entidades/tipos independientes y otros contratos pertinentes | Entidades ORM, wrappers Nest, tipos de SDK |
| Interactors | Puertos, entidades, validaciones, excepciones | `HttpService`, Axios directo, `EntityManager`, repositorio concreto, `@app` |
| Driver independiente | Puerto implementado, cliente/SDK, tipos y errores necesarios | Controller o módulo raíz |
| Wrapper Nest | Driver base o puerto, cliente y decoradores Nest | Reglas de negocio copiadas desde el interactor |
| Controller/consumidor | Framework de entrada, interactors, modelos | Consultas, HTTP saliente o lógica comercial extensa |
| Módulo | Controllers, interactors, implementaciones y módulos proveedores | Ejecución de la operación comercial dentro de factories |

El logger concreto importado por algunos interactors es una excepción conocida a una política de imports totalmente estricta. La interfaz de persistencia que importa un modelo ORM es una inconsistencia separada. No confundirlas con autorización para agregar nuevas dependencias arbitrarias hacia drivers.

## Tests y límites

Hay tres patrones funcionales:

1. Predominante: construir interactor + repositorio HTTP con Axios interceptado y caché en memoria. Es una prueba de varias unidades colaborando, aunque el script la llame unit test.
2. Variante aislada: mock de interfaz con Jest para comprobar reglas, llamadas y ausencia de efectos ante errores.
3. HTTP: TestingModule + Supertest, con sustitución de providers/módulos externos, filtro HTTP y cierre de la app.

Reutilizá el nivel de prueba pertinente. No arrancar servicios reales por copiar `AppModule` si la prueba necesita un módulo específico y dobles. Que un test pase con una estrategia inyectada manualmente no prueba que el módulo productivo la registre.

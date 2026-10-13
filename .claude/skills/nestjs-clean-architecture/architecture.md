# Arquitectura y composición

## Estructura reutilizable

Rutas relativas a la raíz del paquete NestJS destino. `<feature>` y `<integration>` representan nombres que se deben obtener de ese proyecto; no son carpetas literales.

```text
src/
├── main.ts
├── app/
│   ├── app.module.ts
│   ├── controller/
│   │   └── hateoas/              # sólo si se utiliza esa presentación
│   ├── module/
│   │   └── <group>/              # agrupación opcional de módulos relacionados
│   ├── drivers/
│   │   └── <integration>/        # cuando justifica agrupar varios archivos
│   └── service/                 # consumidores/adaptadores de ejecución Nest
├── core/
│   ├── entities/
│   ├── interactors/
│   │   ├── <feature>/
│   │   ├── exceptions/
│   │   └── validations/
│   ├── adapters/
│   │   ├── repositories/
│   │   ├── cache/               # opcional
│   │   └── queue/               # opcional
│   └── drivers/
│       ├── repositories/
│       │   ├── <integration>/
│       │   └── common/          # helpers de infraestructura compartidos
│       ├── cache/               # opcional
│       └── logger/
└── utils/                       # helpers compartidos con responsabilidad concreta
```

Los tests `.test.ts` se colocan junto al código que prueban. Si hay un worker independiente, su entrypoint está en `src`, con su módulo de composición en `app/module`; el nombre depende del trabajo del proceso.

No crear la estructura entera vacía. No imponer una carpeta `order`, `product`, `dto`, `ports`, `mappers`, `domain`, `application` o `infrastructure`: no son requisitos de esta metodología. En un proyecto ya organizado con nombres equivalentes, conservar esa organización puede ser la opción correcta para un cambio localizado.

## Dependencias lógicas y ejecución

```text
Entrada HTTP / consumidor Nest
              │ invoca
              ▼
           Interactor ── usa ──► contrato (interfaz)
              │                        ▲
              ▼                        │ implementa
      entidades / validaciones    driver concreto ──► sistema externo
                                       ▲
                                       │ adapta/compone
                                  wrapper Nest

Módulo Nest: conoce implementaciones y construye el interactor con ellas.
```

La llamada en ejecución atraviesa el contrato hasta su implementación. La dependencia de código del interactor apunta a la interfaz, no a esa implementación. No hace falta un token runtime para cada interfaz: una factory puede recibir un driver concreto y pasarlo al constructor tipado con la interfaz.

`core/adapters` es el nombre histórico de la ubicación de puertos. `core/drivers` contiene infraestructura; no se deben aplicar a toda la carpeta `core` las restricciones de las entidades y los interactors.

## Composición de un módulo

El patrón observado se puede expresar con nombres genéricos así; este fragmento es una **adaptación ilustrativa**, no una extracción literal:

```ts
{
  provide: 'RunOperation',
  useFactory: (repository: IRemoteRepository) =>
    new RunOperation({ repository }),
  inject: [NestRemoteRepository],
}
```

Las factories del modelo suelen declararse `async`, aunque sólo hagan `new`; eso no demuestra que la construcción necesite I/O. Preservá la convención local; usá async cuando exista una necesidad real de inicialización asíncrona.

Pasos de registro:

1. El módulo importa los módulos que proveen clientes externos utilizados por sus drivers.
2. Sus `providers` registran drivers y factories de interactors.
3. Sus `controllers` registran entradas HTTP. Los consumidores se registran como providers cuando así funciona la integración elegida.
4. El módulo raíz importa el módulo funcional.
5. Sólo exportá un provider cuando un módulo consumidor lo requiera; ese consumidor debe importar el módulo que lo exporta.

En el modelo, las áreas funcionales tienen módulos propios y algunas funcionalidades simples se componen directamente en el módulo raíz. No es obligatorio crear un módulo por clase. Tampoco es necesario declarar todo como global.

Las interfaces TypeScript no son tokens disponibles en runtime. El patrón utiliza strings con el nombre del interactor, clases concretas para drivers y tokens de bibliotecas como `APP_FILTER` o `CACHE_MANAGER`. No hay un estándar propio basado en `Symbol()`.

## HTTP y presentación

Los controllers delegan operaciones a interactors. La base HATEOAS agrupa respuestas en `{ data, links }`; para listados, `{ data: Resource[], total }`. Es un patrón de presentación existente, no un formato universal que deba imponerse a otra API.

Reutilizá esa base sólo si las operaciones expuestas tienen implementación. Un controller especializado puede delegar directamente en un interactor: la operación de obtención de etiqueta del ejemplo es evidencia de este estilo. No generar rutas CRUD que respondan mediante stubs sólo para satisfacer la herencia.

Las bibliotecas de transporte, uploads, headers y parámetros de ruta pertenecen a la entrada. Los payloads y URLs del proveedor saliente pertenecen al driver.

## Integraciones HTTP

Patrón observado:

- Clase de repositorio que implementa una interfaz de `core/adapters/repositories`.
- Cliente HTTP recibido por constructor, sin crear un contexto Nest dentro del repositorio.
- Helper común para acceso al cliente; métodos locales para construir URL, headers y body.
- Wrapper `Nest...Repository` injectable que entrega `HttpService.axiosRef` a la implementación.
- Traducción de respuesta/error del proveedor antes de devolver el resultado al interactor.

No se requiere una clase Mapper: hay transformaciones locales de payload y mapas de estados. Separá una transformación cuando su complejidad o reutilización lo justifique; identificá esa extracción como una mejora/adaptación, no como una carpeta obligatoria del patrón.

## Persistencia: patrón parcial

Existe integración TypeORM con modelos decorados en una subcarpeta `entities` del driver y un wrapper Nest que implementa el contrato usando `EntityManager`. También existe una implementación alternativa con `Repository<T>` y decoradores Nest dentro de `core/drivers`.

Ese subsistema no está conectado a los entrypoints inspeccionados del modelo y tiene fugas de tipos ORM al puerto. Por ello no hay evidencia para imponer una única estrategia de repositorio SQL, transacciones, migraciones o Unit of Work.

Para incorporar persistencia, **generalizá la separación predominante**: contrato con modelos independientes del ORM, implementación con tipos ORM, registro y conexión en el módulo. Si la implementación requiere directamente Nest, un driver en `app/drivers` es coherente con el wrapper de persistencia observado; no inventes una segunda clase vacía sólo para extenderla. No uses la interfaz defectuosa del modelo como plantilla.

## Workers, colas, caché y scheduler

Existe un puerto de cola y un driver RabbitMQ en `app/drivers/queue`; un consumidor Nest delega al interactor. El proceso worker tiene módulo y entrypoint propios. El arranque inspeccionado utiliza además transporte TCP: eso no es un requisito para todos los workers RabbitMQ.

El publisher del modelo usa el campo `queue` como routing key. No generalices que el nombre de cola siempre es la routing key: verificá el contrato de la integración destino.

No hay política propia completa de reintentos, DLQ ni idempotencia demostrada. Cuando una funcionalidad los necesite, diseñalos con los requisitos del destino y documentalos como nueva decisión. No atribuyas esas garantías al broker por la sola presencia de un consumidor.

La caché usa un contrato y wrappers Nest/in-memory. No hay Redis configurado. Se registra un módulo de scheduling, pero no hay jobs decorados; no crear cron jobs por copiar ese registro.

## Configuración y logging

El modelo registra `ConfigModule` y lee `process.env` directamente, incluso en un interactor y en un logger estático. No afirmar que ya existe una capa de configuración validada o un puerto de logging.

Reutilizá el mecanismo del destino. Si se necesita carga/validación asíncrona o un `ConfigService`, registralo en la composición y documentá la adaptación. Comprobá que las variables se cargan antes de construir clientes; no asumas que cualquier archivo de entorno se carga automáticamente.

Hay un logger propio con timestamp, nivel, servicio y mensaje, y helpers para HTTP. Conservar su utilidad no implica copiar headers completos, secretos o datos personales en logs.

## Tecnologías: evidencia, no lista de instalación

| Estado observado | Tecnologías |
|---|---|
| Aplicación e integraciones activas | NestJS, TypeScript, Express, Axios, `@nestjs/axios`, RabbitMQ con `@golevelup/nestjs-rabbitmq`, caché con `cache-manager`, configuración con `@nestjs/config` |
| Pruebas | Jest, ts-jest, Nest Testing, Supertest, axios-mock-adapter; herramientas in-memory propias |
| Subsistemas definidos con uso parcial o desconectado | TypeORM, mysql2, sqlite3; Google Sheets mediante google-spreadsheet/google-auth-library; scheduling registrado sin jobs |
| Dependencias declaradas sin uso directo observado en `src` | SDK S3 y sus utilidades de mocks, sharp, xml2js, middlewares helmet/response-time, class-transformer/class-validator sin DTOs decorados propios |
| Herramientas auxiliares | ESLint, Prettier, Docker, GitHub Actions; Cucumber/nyc configurados sin escenarios feature presentes |

No fijar versiones a partir de esta tabla. Elegir y verificar las compatibles en cada destino. La ausencia de uso directo no prueba que un paquete sea innecesario como dependencia indirecta de otra biblioteca.

## Incorporar una tecnología nueva

Esto es una extensión metodológica del patrón, no implementación existente de cada tecnología:

| Tecnología necesaria | Contrato consumido | Implementación y composición | Verificación |
|---|---|---|---|
| SDK de un proveedor | Capacidad de negocio, con modelos propios | Driver encapsula SDK; módulo construye cliente/wrapper | Errores, mapping, llamadas y configuración |
| ORM/base de datos | Puerto de persistencia sin tipos ORM | Driver contiene consultas/modelos ORM; módulo registra conexión | Semántica de lectura/escritura y aislamiento de tests |
| Broker | Operación de publicación; consumidor adapta mensajes | Driver gestiona cliente; consumidor llama interactor | Payload, routing y semántica de fallo requerida |
| Caché | Contrato de caché si la operación lo requiere | Driver adapta cliente; TTL/configuración explícitos | Hit/miss, expiración y fallos según requisitos |
| Job programado | Interactor reutilizable | Servicio Nest maneja el disparo y delega | Registro, ejecución y solapamiento si importa |

Los contratos nombran capacidades, no bibliotecas. No propagar tipos del SDK, conexiones ni objetos de respuesta del proveedor hasta el controller o interactor por comodidad.

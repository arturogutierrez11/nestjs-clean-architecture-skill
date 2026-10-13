# Ejemplos reales y autocontenidos

Estos fragmentos provienen del proyecto modelo `wms-api`, commit `261d7ee`. Sus rutas de origen son metadatos relativos de procedencia: **no se deben abrir para usar esta Skill en otro proyecto**. El código necesario para comprender cada patrón está incluido aquí.

Los nombres comerciales y proveedores se conservan sólo para demostrar la implementación real. Al aplicar el patrón, reemplazalos por los del destino. No copiar endpoints, reglas de estados, mensajes específicos ni identificadores como requisitos generales.

Los fragmentos son referencias de responsabilidades, no un scaffold ejecutable completo. Los imports `@core`/`@app` pertenecen al modelo y deben resolverse con la configuración del destino. Los recortes se identifican por archivo, símbolo y líneas de la revisión citada. Las limitaciones se explicitan para no reproducir defectos.

## Organización actual del equipo frente a rutas históricas

Las rutas de origen de estos fragmentos conservan la distribución histórica del modelo. **No son las ubicaciones a copiar al crear archivos nuevos.** Aplicá capa → dominio → caso de uso según [architecture.md](architecture.md). Conservá la forma de construcción, los contratos y la separación de responsabilidades, reubicándolos en la jerarquía correspondiente del destino.

## Índice

1. Interactor y contrato: construcción sin Nest.
2. Wrapper Nest y factory de DI.
3. Controller especializado y presentación.
4. Driver HTTP y transformación de payload.
5. Validación y errores.
6. Consumidor de mensajes.
7. Tests aislados y con HTTP interceptado.
8. Integración HTTP mediante Nest Testing.
9. Persistencia: variante parcial.


## 1. Interactor y contrato: construcción sin Nest

Un contrato pequeño define las operaciones externas. No tiene decoradores, ni conoce el cliente HTTP.


Origen: `src/core/adapters/repositories/IShippingApiRepository.ts` — IShippingApiRepository, archivo completo; líneas 1–4.

```ts
export interface IShippingApiRepository {
  getZplLabelUrl(orderId: string, jobName?: string): Promise<string>;
  getZplLabel(url: string, jobName?: string): Promise<string>;
}
```


El interactor recibe ese contrato en un objeto constructor, valida la entrada y coordina dos operaciones. No crea repositorios ni arranca Nest. El nombre `SendShippingLabelBuilder` no coincide con la clase consumidora: es una inconsistencia de naming del original, no una convención para copiar.


Origen: `src/core/interactors/sales-orders/GetShippingLabel.ts` — GetShippingLabel, archivo completo; líneas 1–31.

```ts
import { IShippingApiRepository } from '@core/adapters/repositories/IShippingApiRepository';
import { ValidationException } from '@core/interactors/exceptions/ValidationException';
import { isString } from '@core/interactors/validations/typeValidators';

interface SendShippingLabelBuilder {
  shippingApiRepository: IShippingApiRepository;
}

export class GetShippingLabel {
  private readonly shippingApiRepository: IShippingApiRepository;

  constructor(builder: SendShippingLabelBuilder) {
    this.shippingApiRepository = builder.shippingApiRepository;
  }

  async getZpl(orderId: string): Promise<{ zpl: string }> {
    if (!isString(orderId)) {
      throw new ValidationException('El order_id es requerido.');
    }

    const jobName = 'GetShippingLabel';
    const url: string = await this.shippingApiRepository.getZplLabelUrl(
      orderId,
      jobName,
    );
    const zpl = await this.shippingApiRepository.getZplLabel(url, jobName);
    return {
      zpl,
    };
  }
}
```


Otro interactor demuestra lectura y error de ausencia. Su método se llama `findById`, no `execute`. El status 404 y el vocabulario del recurso corresponden a este contrato concreto.


Origen: `src/core/interactors/product/FindProduct.ts` — FindProduct, archivo completo; líneas 1–24.

```ts
import { DomainException } from '@core/interactors/exceptions/DomainException';
import { Product } from '../../entities/Product';
import { IWMSRepository } from '../../adapters/repositories/IWMSRepository';

interface GetProductBuilder {
  wmsRepository: IWMSRepository;
}

export class FindProduct {
  private wmsRepository: IWMSRepository;

  constructor(builder: GetProductBuilder) {
    this.wmsRepository = builder.wmsRepository;
  }

  async findById(sku: string): Promise<Product> {
    const product: Product = await this.wmsRepository.findProductById(sku);
    if (!product) {
      throw new DomainException(`Article with sku: ${sku} not found`, 404);
    }

    return product;
  }
}
```


## 2. Wrapper Nest y factory de DI

El wrapper es injectable y entrega el cliente Nest a una implementación independiente del contenedor. La dependencia de caché aparece en el constructor original aunque estos métodos de etiquetas no la utilizan; no agregar caché a otra integración sólo por copiar este fragmento.


Origen: `src/app/drivers/NestShippingApiRepository.ts` — NestShippingApiRepository, archivo completo; líneas 1–17.

```ts
import { Injectable } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { CacheManager } from './cache/CacheManager';
import { ShippingApiRepository } from '@core/drivers/repositories/shipping/ShippingApiRepository';

@Injectable()
export class NestShippingApiRepository extends ShippingApiRepository {
  constructor(
    private readonly httpService: HttpService,
    private readonly _cacheManager: CacheManager,
  ) {
    super({
      axios: httpService.axiosRef,
      cacheManager: _cacheManager,
    });
  }
}
```


La factory conecta una clase concreta con un constructor tipado por interfaz. Este recorte pertenece a `providers` del módulo; el módulo también registra `NestAuroraRepository`, importa `HttpModule`/`CacheModule` y declara su controller.


Origen: `src/app/module/order/SaleOrder.module.ts` — SaleOrderModule.providers, primer interactor; líneas 22–28.

```ts
    {
      provide: 'GetSaleOrder',
      useFactory: async (auroraRepository: IWMSRepository) => {
        return new GetSaleOrder({ wmsRepository: auroraRepository });
      },
      inject: [NestAuroraRepository],
    },
```


El controller consume exactamente el mismo token. No intenta usar la interfaz TypeScript como un token runtime.


Origen: `src/app/controller/saleOrder.controller.ts` — SaleOrderController.constructor, parámetro; líneas 40–41.

```ts
    @Inject('GetSaleOrder')
    private readonly getOrder: GetSaleOrder,
```


## 3. Controller especializado y presentación

Este método adapta el nombre de campo HTTP y delega al interactor. Es un recorte de un controller que ya recibe `getShippingLabel` por inyección. No hace HTTP saliente ni consulta persistencia.


Origen: `src/app/controller/events.controller.ts` — EventsController.get y cierre de clase; líneas 128–133.

```ts
  @Post('aurora/packaging')
  @HttpCode(200)
  async get(@Body() entry: { order_id: string }) {
    return await this.getShippingLabel.getZpl(entry?.order_id);
  }
}
```


Para operaciones de recurso, la base muestra el envoltorio `{ data, links }`. Sus dependencias son `service: HateoasInteractor<T>` y `generateLinks`. Reutilizar el concepto sólo cuando el contrato HTTP del destino lo requiera; no copiar los stubs de otros métodos del modelo.


Origen: `src/app/controller/hateoas/HATEOASController.ts` — HATEOASController.findOne; líneas 60–65.

```ts
  @Get(':id')
  async findOne(@Param('id') id: string): Promise<Resource<T>> {
    const entity = await this.service.findOne(id);
    const links = this.generateLinks(id);
    return { data: entity, links };
  }
```


## 4. Driver HTTP y transformación de payload

El driver implementa el contrato. Recibe `AxiosInstance`, extiende el helper HTTP y no necesita el contenedor Nest. En este recorte se omite el método de consulta de URL, cuyo catch tiene una inconsistencia de logging.


Origen: `src/core/drivers/repositories/shipping/ShippingApiRepository.ts` — builder, declaración y constructor de ShippingApiRepository; líneas 8–22.

```ts
interface BuilderShippingApiRepository {
  axios: AxiosInstance;
  cacheManager: ICacheManager;
}

export class ShippingApiRepository
  extends AxiosRepository
  implements IShippingApiRepository
{
  private readonly cacheManager: ICacheManager;

  constructor(builder: BuilderShippingApiRepository) {
    super(builder.axios);
    this.cacheManager = builder.cacheManager;
  }
```


Origen: `src/core/drivers/repositories/shipping/ShippingApiRepository.ts` — ShippingApiRepository.getZplLabel; líneas 70–117.

```ts
  async getZplLabel(url: string, jobName?: string): Promise<string> {
    const config = {
      url,
      method: 'GET',
    };
    let response: AxiosResponse;
    const logContext = {
      ...config,
      job: jobName,
      services: 'shipping-api',
    };
    try {
      response = await this.client().request(config);
      const { status, statusText, data: responseData } = response;

      Logger.info(
        JSON.stringify({
          ...logContext,
          message: `Get ZPL Success`,
          response: {
            status,
            statusText,
            data: responseData,
          },
        }),
      );
      return responseData;
    } catch (error) {
      Logger.error(
        JSON.stringify({
          ...logContext,
          message: 'Error getting ZPL content',
          error: {
            status: error.status,
            data: error.data,
            statusText: error.message,
            stack: error.stack,
          },
          response: {
            status: response?.status,
            data: response?.data,
            statusText: response?.statusText,
          },
        }),
      );
      throw new DomainException('Error getting ZPL content', 500);
    }
  }
```


El uso de `Logger`/`DomainException` requiere sus imports; el helper expone `client()`. El fragmento usa el URL devuelto por el proveedor y conserva contexto de logging. En el destino, decidir qué datos pueden registrarse y cuáles deben omitirse; no copiar logging indiscriminado de configuración.

Ejemplo de mapping de un payload dentro de un repositorio. Es un recorte de `updatePurchaseOrder`: el formato `article`, la equivalencia entre sku/ean/name y la construcción del identificador son específicos de esa integración. Sólo es generalizable la ubicación del mapping, no su contenido.


Origen: `src/core/drivers/repositories/aurora/APIAuroraWMSRepository.ts` — APIAuroraWMSRepository.updatePurchaseOrder, payload; líneas 123–131.

```ts
    const body = {
      article: {
        sku: articleSku,
        ean: articleSku,
        name: articleSku,
      },
      quantity: quantity,
      lineOrder: lineOrder,
    };
```


## 5. Validación y errores

El helper devuelve una función que valida un campo, con tratamiento explícito de null/undefined. `Validate.validators` es un mapa de funciones propias de validación; no equivale automáticamente a class-validator.


Origen: `src/core/interactors/validations/Validate.ts` — Validate.validate; líneas 29–53.

```ts
  static validate(
    type:
      | 'string'
      | 'number'
      | 'boolean'
      | 'array'
      | 'nonNegativeInteger'
      | 'positiveInteger',
    { attribute, allowUndefined = false, allowNull = false }: ValidatorParams,
  ) {
    return (entity: Partial<Record<string, unknown>>): void => {
      const value = entity[attribute];

      if (allowUndefined && value === undefined) return;
      if (allowNull && value === null) return;

      const validator = Validate.validators[type];

      if (!validator(value)) {
        throw new ValidationException(
          `The '${attribute}' field is required and must be a ${type}.`,
        );
      }
    };
  }
```


La excepción base y la de validación son clases ordinarias. El 403 por defecto de la base es un dato del modelo, no una recomendación para errores nuevos.


Origen: `src/core/interactors/exceptions/DomainException.ts` — DomainException, archivo completo; líneas 1–8.

```ts
export class DomainException extends Error {
  status: number;

  constructor(message: string, status: number = 403) {
    super(message);
    this.status = status;
  }
}
```


Origen: `src/core/interactors/exceptions/ValidationException.ts` — ValidationException, archivo completo; líneas 1–11.

```ts
import { DomainException } from '@core/interactors/exceptions/DomainException';

export class ValidationException extends DomainException {
  public readonly errors: string[];

  constructor(errors: string[] | string) {
    const errorList = Array.isArray(errors) ? errors : [errors];
    super(`Validation error: ${errorList.join('; ')}`, 400);
    this.errors = errorList;
  }
}
```


El filtro adapta el error a HTTP y se registra con `{ provide: APP_FILTER, useClass: DomainExceptionFilter }`. No es un manejador genérico para colas.


Origen: `src/app/controller/domainException.filter.ts` — DomainExceptionFilter, archivo completo; líneas 1–20.

```ts
import { ExceptionFilter, Catch, ArgumentsHost } from '@nestjs/common';
import { Request, Response } from 'express';
import { DomainException } from '@core/interactors/exceptions/DomainException';

@Catch(DomainException)
export class DomainExceptionFilter implements ExceptionFilter {
  catch(exception: DomainException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.status;

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.message,
    });
  }
}
```


## 6. Consumidor de mensajes

El servicio registrado como provider consume un mensaje y delega. En el original, el interactor es inyectado por el token string `UpdateOrdersEvent`. Los nombres de exchange/routing/queue y el decorador Body pertenecen al original: verificar la API de la biblioteca elegida al incorporarlo a otro destino.


Origen: `src/app/service/UpdateStatus.service.ts` — UpdateStatusService.handleStatusEvent y cierre de clase; líneas 19–27.

```ts
  @RabbitSubscribe({
    exchange: 'wms-status',
    routingKey: 'wms-status',
    queue: 'wms-status',
  })
  async handleStatusEvent(@Body() attributes: Notification) {
    await this.updateOrdersEventInteractor.changeOrderStatus(attributes);
  }
}
```


## 7. Tests aislados y con HTTP interceptado

Variante aislada existente: un mock del puerto permite probar reglas y comprobar que un rechazo no genera efectos externos. El original usa un cast para un mock parcial de un contrato grande; no es obligatorio copiar esa debilidad de tipado.


Origen: `src/core/interactors/sales-orders/SendShippingLabel.test.ts` — SendShippingLabel: setup, cleanup y rechazo sin archivo; líneas 33–53.

```ts
  beforeEach(() => {
    wmsRepository = {
      findSaleOrderById: jest.fn(),
      postShippingLabel: jest.fn(),
    } as unknown as jest.Mocked<IWMSRepository>;

    interactor = new SendShippingLabel({ wmsRepository });
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  it('should throw a MissingFileException if no file is provided', async () => {
    await expect(
      interactor.sendShippingLabel(mockSaleOrder.externalId, undefined),
    ).rejects.toThrow(new MissingFileException('zplFile'));

    expect(wmsRepository.findSaleOrderById).not.toHaveBeenCalled();
    expect(wmsRepository.postShippingLabel).not.toHaveBeenCalled();
  });
```


Patrón predominante del modelo: interactor + driver real con Axios interceptado. Este setup utiliza imports de Axios, `ShippingApiRepository`, `InMemoryCacheManager` y `GetShippingLabel`. No invoca servicios reales una vez configurados los mocks requeridos.


Origen: `src/core/interactors/sales-orders/GetShippingLabel.test.ts` — GetShippingLabel: setup y limpieza de mocks; líneas 23–36.

```ts
  beforeAll(async () => {
    shippingApiRepository = new ShippingApiRepository({
      axios,
      cacheManager: new InMemoryCacheManager(),
    });

    interactor = new GetShippingLabel({
      shippingApiRepository,
    });
  });

  beforeEach(() => {
    shippingApiRepository.clearMocks();
  });
```


Este test verifica las dos llamadas salientes, además del resultado. `PATHS` construye las rutas de la integración, `orderId` es un fixture y `fakeZplContent` es texto de prueba; usar fixtures del dominio destino.


Origen: `src/core/interactors/sales-orders/GetShippingLabel.test.ts` — GetShippingLabel: caso de éxito con historial HTTP; líneas 45–72.

```ts
    it('should return zpl', async () => {
      const zplUrl = PATHS.SHIPPING.GET_ZPL(orderId);
      shippingApiRepository.addMock({
        status: 200,
        path: PATHS.SHIPPING.GET_LABEL_URL(orderId),
        method: 'GET',
        response: { zplLabelUrl: zplUrl },
      });

      shippingApiRepository.addMock({
        status: 200,
        path: zplUrl,
        method: 'GET',
        response: fakeZplContent,
      });

      const response = await interactor.getZpl(orderId);

      expect(response.zpl).toContain(fakeZplContent);

      const mocks = shippingApiRepository.getMock();
      expect(mocks.history.get).toHaveLength(2);
      expect(mocks.history.get[0].url).toBe(
        PATHS.SHIPPING.GET_LABEL_URL(orderId),
      );
      expect(mocks.history.get[1].url).toBe(PATHS.SHIPPING.GET_ZPL(orderId));
    });
  });
```


## 8. Integración HTTP mediante Nest Testing

Se importa el módulo de la feature, se registra el filtro HTTP y se sustituye un provider externo. Se inicializa/cierra la app y se obtiene el driver para configurar sus mocks. No hace falta importar todo el módulo raíz para probar un endpoint.


Origen: `src/app/controller/saleOrderController.integration.test.ts` — SaleOrderController: TestingModule y ciclo de vida; líneas 29–53.

```ts
  beforeAll(async () => {
    const moduleRef: TestingModule = await Test.createTestingModule({
      imports: [SaleOrderModule],
      providers: [
        {
          provide: APP_FILTER,
          useClass: DomainExceptionFilter,
        },
      ],
    })
      .overrideProvider(CacheManager)
      .useFactory({
        factory: () => new CacheManager(new InMemoryCacheManager() as any),
      })
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();

    auroraRepository = moduleRef.get(NestAuroraRepository);
  });

  afterAll(async () => {
    await app.close();
  });
```


El rechazo HTTP se verifica junto con la ausencia de requests salientes. Este recorte depende de `app`, `orderId`, `auroraRepository` y Supertest (`request`), definidos en el setup del test.


Origen: `src/app/controller/saleOrderController.integration.test.ts` — SaleOrderController: contrato de error HTTP; líneas 89–101.

```ts
    it('should throw a MissingFileException if no file is provided', async () => {
      const response = await request(app.getHttpServer()).post(
        `/api/sale-orders/${orderId}/labels`,
      );
      expect(response.status).toBe(400);
      expect(response.body.message).toBe(
        `No file was provided. The field 'zplFile' is required.`,
      );

      const mocks = auroraRepository.getMock();
      expect(mocks.history.get).toHaveLength(0);
      expect(mocks.history.post).toHaveLength(0);
    });
```


## 9. Persistencia: variante parcial

Este método es evidencia de persistencia ubicada en un driver Nest. La clase implementa `INotificationRepository` y recibe `EntityManager` con `@InjectEntityManager()`. **El puerto original depende de la entidad ORM y el módulo de persistencia no está conectado a los entrypoints actuales.** No copiar ese acoplamiento ni presentar este recorte como un flujo de persistencia completo.


Origen: `src/app/drivers/NestNotificationRepository.ts` — NestNotificationRepository.save; líneas 13–17.

```ts
  async save(
    notificationData: Partial<NotificationEntity>,
  ): Promise<NotificationEntity> {
    return await this.entityManager.save(NotificationEntity, notificationData);
  }
```


Para una integración de persistencia nueva, generalizar el límite: contrato con modelo propio, driver que convierte al modelo ORM y módulo que registra conexión/provider. La conversión explícita sería una adaptación correctiva, no un mapper ya implementado por el original.

## Cómo reutilizar estos ejemplos

Elegí el fragmento por responsabilidad, no por nombre comercial. Conservá la forma de construcción e inversión de dependencias; reemplazá tipos, métodos y configuración por los del destino. Revisá las advertencias de cada recorte y [inconsistencies.md](inconsistencies.md). No copiar un conjunto de fragmentos suponiendo que constituye un proyecto compilable sin imports, registro y contratos propios.


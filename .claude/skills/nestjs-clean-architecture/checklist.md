# Checklist de implementación y revisión

Aplicar a la parte afectada; marcar no aplicable con razón cuando corresponda. Las comprobaciones de compatibilidad e integración son generalizaciones operativas del patrón. No obligan a instalar todas las tecnologías del modelo.

## Antes de editar

- [ ] Identifiqué raíz/paquete, instrucciones locales, gestor, lockfile, scripts y versiones reales.
- [ ] Clasifiqué el pedido y respeté su alcance: revisión, feature, refactor, proyecto nuevo o integración.
- [ ] Leí un flujo vecino completo y sus tests; distinguí código importado de código desconectado.
- [ ] Enumeré entradas, salidas, reglas, errores, efectos externos y contratos a conservar.
- [ ] Asigné cada archivo a una responsabilidad; usé nombres del dominio destino.

## Estructura y naming

- [ ] Interactors en su feature, contratos en adapters, implementaciones en drivers y composición en módulos, o equivalentes locales justificados.
- [ ] Nombres consistentes con la feature; sin suffixes, capas o archivos vacíos impuestos por costumbre.
- [ ] No copié entidades, rutas, proveedores, identificadores ni reglas comerciales del ejemplo.
- [ ] Los imports resuelven en compilación, tests y ejecución, no sólo en el editor.

## Responsabilidades y contratos

- [ ] Controller/consumidor adapta transporte y delega; no consulta DB ni API externa directamente.
- [ ] Interactor se construye sin Nest y depende de interfaces para integración.
- [ ] Puerto no importa implementaciones, entidades ORM ni tipos de SDK.
- [ ] Driver encapsula detalles externos y cumple entradas, retornos y semántica del contrato.
- [ ] Modelos ordinarios no tienen decoradores de persistencia o dependencias de transporte añadidas.
- [ ] Mapeos están ubicados según sean semántica de la operación o formato del proveedor.
- [ ] Reconocí las excepciones documentadas; no las expandí inadvertidamente.

## DI y módulo

- [ ] Token de `provide` coincide exactamente con `@Inject`.
- [ ] Argumentos de la factory corresponden al orden de `inject` y al objeto constructor.
- [ ] Los drivers/clientes son visibles mediante providers locales o imports/exports correctos.
- [ ] El módulo funcional está importado desde un entrypoint efectivo.
- [ ] No añadí instancias duplicadas de clientes/proveedores compartidos sin necesidad.
- [ ] Estrategias y consumidores requeridos están registrados en runtime, no sólo en tests.
- [ ] Configuración está disponible cuando se construyen conexiones/clientes.

## Validación y errores

- [ ] Verifiqué `null`, `undefined`, vacío y tipos incorrectos donde afectan la operación.
- [ ] La forma del objeto/array se valida antes de iterarlo o modificarlo.
- [ ] Condiciones de existencia, conflicto y estado conservan las reglas acordadas.
- [ ] Los errores esperados se traducen al contrato establecido y no caen accidentalmente en un 500.
- [ ] Distinguí ausencia, validación y caída de dependencia; no absorbí fallas externas indiscriminadamente.
- [ ] No usé condiciones imposibles para rangos HTTP ni retornos ficticios para cumplir interfaces.
- [ ] Rutas heredadas tienen implementación real y los enlaces usan el identificador correcto.

## Dependencias y nuevas tecnologías

- [ ] Instalé sólo paquetes necesarios, con el gestor del proyecto y ubicación runtime/dev adecuada.
- [ ] Verifiqué compatibilidad/peer dependencies y mantuve el lockfile correspondiente.
- [ ] No incorporé versiones, credenciales o configuración del entorno original.
- [ ] El cliente quedó encapsulado detrás de un contrato y conectado a un módulo.
- [ ] Si hay conexiones, comprobé su inicialización/cierre y reemplazo en tests.
- [ ] Si hay cola/job, verifiqué routing, registro, fallos y sólo las garantías requeridas de reintento/idempotencia.
- [ ] Los logs aportan contexto sin copiar credenciales, headers sensibles o datos innecesarios.

## Pruebas y comandos

Leé los scripts antes de ejecutarlos: algunos comandos de lint/formato escriben archivos y los scripts de mantenimiento pueden operar sobre sistemas reales. No ejecutar esos scripts como mecanismo de validación.

1. Ejecutá las pruebas pertinentes al comportamiento cambiado: éxito, validación, ausencia/conflicto, error de dependencia y ausencia de efectos cuando la operación se rechaza.
2. Si cambiaste mappings o protocolo, verificá método, URL/path, payload y respuesta/error del driver mediante un cliente interceptado o doble apropiado.
3. Si cambiaste controllers/DI, verificá resolución del módulo y comportamiento HTTP con dependencias externas reemplazadas. Si cambiaste worker, verificá el registro efectivo de su consumidor.
4. Ejecutá build/typecheck y lint según los scripts disponibles. Identificá fallas preexistentes y no ocultes errores bajando controles de TypeScript.
5. Inspeccioná el diff para detectar cambios ajenos, archivos generados o lockfiles nuevos accidentales.

Ejemplos de comandos, **condicionados a que esos scripts existan en el destino**:

```sh
npm run test -- --runInBand
npm run build
```

```sh
yarn test --runInBand
yarn build
```

Elegí un gestor, no ejecutes ambas variantes. Ajustá filtros de tests a la configuración real. No asumir que `test:e2e` funciona porque aparece en package.json; verificá el archivo de configuración y los escenarios existentes. Una prueba HTTP puede formar parte del mismo runner Jest que las de interactors.

## Criterio de entrega

- [ ] El flujo y wiring están completos, sin métodos vacíos o imports pendientes.
- [ ] Los contratos conservados y los cambios de comportamiento intencionales están explícitos.
- [ ] Informé comandos ejecutados y resultados reales, sin confundir lectura estática con ejecución.
- [ ] Si una verificación no pudo ejecutarse, declaré qué falta y qué alcance quedó sin comprobar.
- [ ] En una revisión, cada hallazgo tiene ubicación, impacto y corrección mínima; recomendaciones opcionales están separadas de violaciones.

## Comprobar portabilidad de esta Skill

Al copiar o actualizar la carpeta, verificá que los enlaces Markdown sólo referencien documentos incluidos, que los ejemplos contengan el contexto necesario y que las instrucciones no requieran abrir el repositorio modelo. Las rutas de procedencia dentro de ejemplos son metadatos, no dependencias ejecutables.

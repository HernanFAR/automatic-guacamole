# Serviu — exploración de Ticket Service v2 y Flow Orchestration Service

**Estado:** pseudocódigo de diseño, no código ejecutable ni contrato implementado. Primera iteración: 2026-11-12.

Este repositorio es el **cuaderno de modelado y composición** de los servicios y el Product de Ticket Support, tomando la plantilla Serviu anterior como procedencia histórica de la primera iteración. La siguiente realización debe actualizar los cuatro consumidores desde **[template-service v0.1.1-preview.1](https://github.com/atom-dev-serviu/template-service/tree/v0.1.1-preview.1)** hasta **[v0.2.0-preview.1](https://github.com/atom-dev-serviu/template-service/tree/v0.2.0-preview.1)**, según la [guía de actualización de ese tag](https://github.com/atom-dev-serviu/template-service/blob/v0.2.0-preview.1/UPGRADING.md) y el [plan de adopción de Authentication](implementation-readiness.md). No sustituye la documentación propietaria del caso Serviu, no constituye una especificación normativa de VSlices y no presupone un motor de procesos configurable.

## Recorrido

1. [Ticket Service v2 — Spaces y Features](ticket-service-v2.md).
2. [Flow Orchestration Service — Spaces y Features](flow-orchestration-service.md).
3. [Contrato transversal del Ticket Support Product — conexiones, coordinación y decisiones](integration-contract.md).
4. [Folder Service — estado actual y preparación de integración](folder-service.md).
5. [Ticket Support Product / BFF — composición y Features](ticket-support-bff.md).
6. [Decisión 001 — asignación anticipada de número de Ticket y FolderPath](decisions/001-early-ticket-number.md).
7. [Preparación de implementación — migración incremental de plantilla y Authentication privado](implementation-readiness.md).

## Decisiones establecidas

- `ticket-service-v2` posee el Ticket **inalterable tras el registro**, los informes únicos por Ticket y sus reglas de validez/formalización, referencias de evidencias y catálogos de clasificación.
- `flow-orchestration-service` reconoce **FlowSpecification**, versiones identificables, transiciones constantes e instancias de ejecución con estado e historia. No escribe informes ni Tickets.
- El primer flujo de Tickets se define **solo mediante constantes**, identificado por `TicketSupport / V1`: `Evaluación | Aprobación | Supervisión | Cerrado`. Cada versión tiene ID y unicidad por `(specification, version)`; cada transición condicional tiene ID y unicidad por `(specificationVersion, currentState, nextState)`. Sus **condiciones se implementan en código**, no mediante características configurables del Space por ahora.
- El orquestador **es propietario de la definición y consulta** de una **proyección de lectura con cinco características**, obtenida sobre tablas de Ticket Service v2 sin escribirlas ni replicarlas: evaluación completada, alcance, aprobación completada, supervisión completada y estado de cierre. El alcance y cierre solo están presentes tras completar sus respectivos informes.
- Los Points son inmutables. Evolución de un informe borrador e iteración de una instancia de flujo producen nuevos Points, no mutaciones del Point anterior.
- Guardar un borrador **no avanza el flujo**. Formalizar datos en Ticket Service y ejecutar una transición de Flow Orchestration son responsabilidades distintas.
- No recuperar las Features descartadas de **edición de Ticket** ni **asignación de responsable**; tampoco introducir edición posterior de adjuntos. Conservar asociación de evidencias durante registro.
- **FlowSpecification** es el Space principal del orquestador; **FlowInstance** es el de mayor uso y ejecuta una versión compatible con el tipo de elemento. La política de admisión de instancias se define en `FlowSpecificationVersion`; en `TicketSupport/V1`, cada Ticket admite **una única FlowInstance**, incluso tras finalizar y sin duplicados por cambio de versión. No añadir editor de estados, gestión dinámica de definiciones ni un motor genérico sin necesidad comprobada.

## Siguiente realización: plantilla y autenticación

Los repositorios `ticket-service-v2`, `folder-service-v2`, `flow-orquestration-service` y `ticket-support-product-v2` deben adoptar incrementalmente **`atom-dev-serviu/template-service v0.2.0-preview.1`** (origen `v0.1.1-preview.1`), preservando sus implementaciones. El salto de VSlices a `0.6.0-preview.2` es una dependencia de la nueva plantilla, no la versión del template. No usar el script destructivo de reemplazo para migrar código personalizado.

**Los cuatro servicios**, no solo Product/BFF, deben instalar las referencias y configuración de Authentication de la plantilla y **reconocer cookie SSO e identidad del usuario autenticado**. Comprobar aparte la propagación de la sesión en llamadas entre servicios y la autorización funcional.

Asimismo, Ticket Service debe **asignar durablemente el número antes del registro** y `RegisterTicket` debe exigir ese número existente, válido y libre; Product solicita la asignación antes de subir adjuntos. La [Decisión 001](decisions/001-early-ticket-number.md) y la [preparación de implementación](implementation-readiness.md) establecen condiciones y pruebas. **No se afirma que estos cambios ya se hayan ejecutado en los cuatro repositorios.**

## Alcance de la prueba de concepto

**Fecha límite: 13 de noviembre de 2026.** `FlowSubject` es una unión discriminada cuya única variante inicial es `TicketReference`; el futuro `EndorsmentRequestReference` se incorporará cuando exista ese consumidor, sin almacenar además un string de tipo. La atomicidad entre el nuevo Point de `FlowInstance` y `ExecutedTransition`, la concurrencia optimista y las garantías fuertes de consistencia son mejoras **post-PoC**, no condiciones para esta entrega. Las fallas parciales deben seguir siendo visibles, sin atribuir al prototipo garantías que no ofrece.

## Convención de referencias

- **Space interno del servicio:** conservar únicamente su identificador, por ejemplo `SystemId`, `ModuleId`, `IncidenceTypeId` o `FlowInstanceId`.
- **Space de otro servicio:** referencia explícita denominada `<Concepto>Reference`, por ejemplo `TicketReference` para Flow Orchestration, o `AccountReference` para Ticket Service v2. No introducir `Reference<A>`.
- Los nombres `ManualReference`, `ResolvedReference` y `RecognizableReference` describen variantes de la referencia a un **elemento externo de negocio**, no sustituyen esta convención de propiedad.

## Convención del pseudocódigo

Los bloques `pseudo` describen **significado, admisibilidad, Feature, requisito y resultado**. No son sintaxis C# ni una API VSlices válida. `Choice<...>`, `SpaceId` (interno), `SpaceReference` (externo), `Optional<...>`, `Fin<...>` y `Point<...>` son notación de diseño. Los nombres de Features son candidatos; pueden agruparse o dividirse tras estudiar los contratos reales.

## Fuentes y procedencia

- [Spaces del sistema Tickets](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/spaces.md)
- [Features del sistema Tickets y distribución de responsabilidades](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/system-features.md)
- [Spaces y transiciones de Flow Orchestration V1](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/flow-orchestration-spaces.md)
- [Work Process organizacional](https://github.com/atom-dev-serviu/documentation/blob/main/organization/tickets/management-process.md)
- [Service Template v0.2.0-preview.1 — arquitectura](https://github.com/atom-dev-serviu/template-service/blob/v0.2.0-preview.1/docs/architecture/README.md)

**Límite:** crear estos documentos no crea los servicios, tablas, endpoints ni validaciones runtime.

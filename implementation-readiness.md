# Preparación de implementación — actualización de plantilla y Authentication privado

**Estado:** coordinación de realización pendiente; no acredita ejecución en los cuatro servicios. **Fecha de revisión:** 2026-11-14. **Referencia inmutable:** [`atom-dev-serviu/template-service@v0.2.0-preview.1`](https://github.com/atom-dev-serviu/template-service/tree/v0.2.0-preview.1), commit `6f5b258b58323ee8d5bbe88032cf0503e8398df2` (coincidía con `main` al verificarlo; la rama predeterminada es `main`, no `master`).

Este plan complementa los modelos conceptuales y la [Decisión 001](decisions/001-early-ticket-number.md). No reemplaza las responsabilidades de Ticket, Folder, Flow ni del Product, ni afirma que las rutas de la PoC ya estén implementadas.

## Alcance y repositorios

| Consumidor | Repositorio | Intervención prevista |
| --- | --- | --- |
| Ticket Service v2 | [ticket-service-v2](https://github.com/atom-dev-serviu/ticket-service-v2) | Adoptar `template-service v0.2.0-preview.1`; reconocer cookie SSO y usuario autenticado en Hosting; autorizar operaciones sobre Tickets |
| Folder Service v2 | [folder-service-v2](https://github.com/atom-dev-serviu/folder-service-v2) | Adoptar la misma versión; reconocer cookie SSO y usuario autenticado; verificar autoridad y pertenencia antes de operar sobre archivos |
| Flow Orchestration | [flow-orquestration-service](https://github.com/atom-dev-serviu/flow-orquestration-service) | Adoptar la misma versión; reconocer cookie SSO y usuario autenticado; autorizar inicio y avance de flujos |
| Ticket Support Product/BFF | [ticket-support-product-v2](https://github.com/atom-dev-serviu/ticket-support-product-v2) | Adoptar la misma versión; reconocer cookie SSO y usuario autenticado; coordinar los servicios respetando la autoridad por recurso |

Estos cuatro repositorios se originaron desde la plantilla, pero evolucionan por separado. **No se propone reinstalar/regenerar la plantilla ni copiar su árbol:** se aplicarán las diferencias relevantes sobre ramas de cada consumidor, preservando cambios de dominio, scripts, workflows y configuraciones.

## 1. Actualizar los cuatro consumidores de `template-service v0.1.1-preview.1` a `v0.2.0-preview.1`

**Objetivo primario:** migrar la **plantilla Serviu** `atom-dev-serviu/template-service` desde `v0.1.1-preview.1` hasta el tag **`v0.2.0-preview.1`** en los cuatro repositorios. No confundir esa versión de plantilla con la versión de los paquetes **VSlices `0.6.0-preview.2`**, que forman parte de sus dependencias.

**Fuente fija:** [UPGRADING.md del tag destino](https://github.com/atom-dev-serviu/template-service/blob/v0.2.0-preview.1/UPGRADING.md), [onboarding privado](https://github.com/atom-dev-serviu/template-service/blob/v0.2.0-preview.1/docs/development/private-authentication-onboarding.md) e [informe de migración VSlices](https://github.com/atom-dev-serviu/template-service/blob/v0.2.0-preview.1/docs/development/vslices-0.6.0-preview.2-migration.md). Resolver las diferencias contra los **archivos y contratos del tag**, no contra el estado mutable de una rama.

**Disponibilidad local declarada:** Hernán ya dejó una copia actualizada de la plantilla y preparó una caché NuGet local con los cuatro paquetes Authentication y tres paquetes VSlices.Authentication para el sandbox de GPT Work sin NuGet online. Esa preparación **no acredita** que los cuatro repositorios consumidores estén migrados ni que la caché contenga todas las dependencias transitivas; comprobar restore y build offline por consumidor.

1. En cada repositorio: registrar SHA de partida, estado real de personalizaciones, referencias NuGet, versión de origen de la plantilla y pruebas iniciales. Abrir una rama de migración por consumidor; usar el árbol local actualizado únicamente como referencia de diferencias.
2. **No usar** `scripts/switch-template-version.py --apply` en un servicio con personalizaciones: reemplaza archivos versionados; no es un merge ni una migración incremental.
3. En los proyectos reales `*.Spaces` y `*.Work`, alinear los paquetes `VSlices.Spaces`, `VSlices.Work` y `VSlices.Work.DataAccess` requeridos a `0.6.0-preview.2`. Los átomos `PointReader`, `PointWriter`, `PointRemover` y `HasPoint*` están ahora en el paquete `VSlices.Work.DataAccess`, manteniendo `VSlices.Work.Algebra`.
4. Revisar cada Feature frente a **FM011**: declarar `Feature<F, F.Algebra, Request, Response>` y una interfaz `Algebra` (o `Alg`) **anidada y pública**, con sus capacidades mínimas. El runtime concreto compone las interfaces de sus Features.
5. Revisar cada Space concreto frente a **SM008**: `DiscreteSpace<SELF>` e igualdad correcta para su semántica real. No convertir indiscriminadamente todas las clases en records ni modificar identidad/invariantes para conformar una prueba.
6. Traer además los cambios pertinentes de `template-service v0.2.0-preview.1` que **no** se reducen a actualizar VSlices: las dependencias privadas `Authentication.Work` y `Authentication.Hosting`, la Feature de lectura del contexto de autenticación, la configuración HTTP/cookie, las pruebas y los ajustes documentales/CI relevantes. Contrastar cada diferencia con los contratos del tag; no sobrescribir desarrollos propios.
7. Verificar restore/build, Laws y regresiones del consumidor. La plantilla usa xUnit v3/Microsoft Testing Platform: comprobar el runner real antes de elegir `dotnet run` o `dotnet test`. Ejecutar SQL Server/Testcontainers cuando corresponda; documentar cualquier prueba omitida.
8. Registrar lo realizado, resultados y límites antes de abrir PR; no presentar la migración de Framework como prueba de seguridad HTTP o despliegue.

Esta migración **no cambia por sí misma** la Decisión 001 sobre la numeración anticipada de Ticket: el nuevo comportamiento debe implementarse y probarse aparte en Ticket Service v2 y Product/BFF.

## 2. Instalar e integrar Authentication **en los cuatro servicios**

**Decisión explícita:** `ticket-service-v2`, `folder-service-v2`, `flow-orquestration-service` y `ticket-support-product-v2` **deben reconocer la cookie SSO de Serviu y la identidad del usuario autenticado**. No es una capacidad opcional o exclusiva del Product/BFF. El modelo de integración toma como referencia la configuración existente en [`template-service v0.2.0-preview.1`](https://github.com/atom-dev-serviu/template-service/tree/v0.2.0-preview.1).

**Distribución:** [`auth-dependency v0.1.0-preview.1`](https://github.com/atom-dev-serviu/auth-dependency/releases/tag/v0.1.0-preview.1), privado en GitHub Packages. La plantilla destino ya declara **`Authentication.Work 0.1.0-preview.1` en Work** y **`Authentication.Hosting 0.1.0-preview.1` en Hosting**; `Authentication.Spaces` y `Authentication.Grounding` llegan transitivamente. Los paquetes VSlices `0.6.0-preview.2` se distribuyen por NuGet.org.

```text
Feed privado: https://nuget.pkg.github.com/atom-dev-serviu/index.json
```

Cada uno de los cuatro consumidores debe:

1. Adoptar las referencias de `Authentication.Work` y `Authentication.Hosting` de la plantilla destino en sus proyectos correspondientes; ajustar a nombres reales si fueron renombrados. Evitar copias de los proyectos de Authentication o VSlices.
2. Incorporar la Feature de lectura de autenticación y la composición del Hosting del tag, adaptándolas a sus módulos y sus rutas reales, en lugar de limitarse a añadir un `PackageReference`.
3. Configurar explícitamente el esquema cookie de Serviu, su nombre, validación y Data Protection/keyring compatibles con la sesión emitida por Accounts. Account Service sigue siendo el **emisor** de la cookie; Authentication no emite credenciales.
4. Instalar `UseAuthentication()` y `UseAuthorization()` en el orden correcto y exigir identidad **reconocible y autenticada** en las rutas protegidas. Una cookie presente pero inválida, expirada o con claims ambiguos no acredita identidad.
5. Distinguir esa **autenticación obligatoria en cada servicio** de la **autorización funcional y por recurso**. Ticket, Folder y Flow deben verificar permisos pertinentes antes de realizar operaciones; el BFF no sustituye esos controles.
6. Comprobar si las llamadas del Product a Ticket/Folder/Flow transportan realmente la cookie esperada o requieren un mecanismo adicional de invocación. **No dar por resuelta la propagación interservicio** al instalar el paquete, ni reenviar cookies indiscriminadamente: definir el contrato y probarlo con el mecanismo acordado. No confundir al usuario de la sesión con el servicio invocador.
7. Asegurar acceso de lectura a GitHub Packages tanto para desarrollo como para los cuatro workflows consumidores. Personas: PAT classic personal `read:packages` en configuración local, nunca versionado. GitHub Actions: conceder a cada repositorio acceso en **Manage Actions access** de los paquetes y usar su `GITHUB_TOKEN` según permisos de organización. Evitar almacenar secretos en `NuGet.Config` o logs.
8. En GPT Work, verificar que la caché local transferida contiene los paquetes y **todas** sus dependencias transitivas y ejecutar restore/build offline, sin asumir que la presencia de siete carpetas demuestra restauración completa.

**Evidencia mínima por servicio:** ruta protegida sin cookie → rechazo; cookie inválida/expirada → rechazo; cookie Serviu válida → usuario correcto reconocido; identidad ambigua → rechazo; permiso de negocio insuficiente → rechazo. Verificar adicionalmente autorización por Ticket, archivo o instancia, según su propietario.

**Límite explícito:** que los cuatro servicios consuman y reconozcan cookies **no demuestra** que el BFF ya pueda hacer llamadas autenticadas a todos ellos, ni que el transporte esté protegido, ni que una sesión válida conceda permisos de negocio.

## 3. Asignación anticipada obligatoria del número de Ticket

Se preserva íntegramente la [Decisión 001](decisions/001-early-ticket-number.md), pero su cumplimiento debe incluirse **expresamente** en el alcance de implementación.

- **Ticket Service v2** es propietario de la numeración. Una Feature/operación **POST** asigna y persiste de manera durable un número único **antes** de que exista el Ticket, con exclusividad concurrente y sin reutilizar asignaciones.
- **`RegisterTicket` exige un número suministrado**. Nunca lo genera implícitamente: debe comprobar que la asignación existe, que es válida y que el número aún **no está asociado a ningún Ticket**. La asociación debe quedar protegida por una restricción efectiva de persistencia frente a carreras.
- **Product/BFF** solicita primero el número, deriva `FolderPath(code, "TicketSupport", "Attachments")`, sube adjuntos, registra el Ticket incluyendo el número previamente asignado y las referencias iniciales, y luego intenta crear la FlowInstance.
- **Folder** utiliza esa ubicación sin inventar números ni conocer el dominio de Tickets; **Flow** recibe la referencia del Ticket ya registrado.
- La pérdida de un número asignado sin Ticket se admite; no se presume idempotencia de asignación o registro ni atomicidad entre servicios.

**Pruebas indispensables:** registrar sin número asignado → rechazo; número desconocido → rechazo; número ya asociado → rechazo; dos asignaciones concurrentes → diferentes; asignación válida libre → Ticket con ese mismo número; simultaneidad de dos registros con el mismo número → a lo sumo uno; ruta de carpeta igual antes/después; los adjuntos nacen vinculados y el Ticket no admite edición posterior. Registrar efectos parciales reales si falla Folder, Ticket o Flow.

## 4. Orden de integración y verificación

1. Migrar **los cuatro repositorios** de `template-service v0.1.1-preview.1` al tag fijo `v0.2.0-preview.1` mediante las guías, conservando cambios propios y validando las diferencias.
2. Instalar/configurar Authentication **en los cuatro**, incluida lectura de identidad de cookie SSO; ejecutar pruebas positivas y negativas de cada Hosting. Conservar antiforgery en formularios humanos.
3. Determinar y comprobar la propagación/autenticación efectiva Product → Ticket/Folder/Flow, manteniendo separadas la identidad del usuario, la identidad del llamante y la autorización funcional.
4. Implementar/validar `AssignNextTicketNumber` en Ticket Service v2 y hacer obligatorio el número preasignado y no consumido en `RegisterTicket`.
5. Actualizar la composición del Product: `AssignNextTicketNumber → Folder.AddFile → Ticket.RegisterTicket(code, ...) → Flow.StartFlow`, con fallos parciales observables y sin inventar transacciones ni idempotencia.
6. Probar restore/build/Laws/regresiones, cookies, seguridad por recurso, concurrencia de numeración, flujo de adjuntos y recuperación de fallos; documentar límites de sandbox/CI.
7. En cada PR consignar SHA inicial y final, versión exacta del tag de template, NuGet utilizados, evidencia real de CI, accesos al feed y pendientes.

**Criterio de salida:** los cuatro consumidores están migrados al tag de plantilla indicado, restauran sus dependencias, reconocen usuario autenticado con cookie Serviu, compilan y superan sus pruebas; Ticket registra únicamente números asignados y libres con integridad concurrente. La coordinación de llamadas interservicio requiere evidencia propia, no una inferencia por instalación de paquetes.

## Relación con el diseño anterior

La [Decisión 001](decisions/001-early-ticket-number.md) sigue vigente. El [contrato de integración](integration-contract.md) y los diseños de [Ticket](ticket-service-v2.md), [Folder](folder-service.md), [Flow](flow-orchestration-service.md) y [BFF](ticket-support-bff.md) conservan su autoridad conceptual sobre los procesos y modelos. Este documento guía una **siguiente realización**, no modifica retrospectivamente esos acuerdos.

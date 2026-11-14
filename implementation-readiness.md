# Preparación de implementación — actualización de plantilla y Authentication privado

**Estado:** plan de realización pendiente, no ejecución de los cuatro servicios. **Fecha de revisión:** 2026-11-14.

Este plan complementa los modelos conceptuales y la [Decisión 001](decisions/001-early-ticket-number.md). No reemplaza las responsabilidades de Ticket, Folder, Flow ni del Product, ni afirma que las rutas de la PoC ya estén implementadas.

## Alcance y repositorios

| Consumidor | Repositorio | Intervención prevista |
| --- | --- | --- |
| Ticket Service v2 | [ticket-service-v2](https://github.com/atom-dev-serviu/ticket-service-v2) | Actualización incremental del template; proteger endpoints y atribuir invocaciones |
| Folder Service v2 | [folder-service-v2](https://github.com/atom-dev-serviu/folder-service-v2) | Actualización incremental; autenticación del invocador antes de autorizar operaciones de archivo |
| Flow Orchestration | [flow-orquestration-service](https://github.com/atom-dev-serviu/flow-orquestration-service) | Actualización incremental; verificar invocador y autoridad para iniciar/avanzar instancias |
| Ticket Support Product/BFF | [ticket-support-product-v2](https://github.com/atom-dev-serviu/ticket-support-product-v2) | Actualización incremental; integrar autenticación por cookie desde Authentication; coordinar servicios sin propagar cookies arbitrariamente |

Estos cuatro repositorios se originaron desde la plantilla, pero evolucionan por separado. **No se propone reinstalar/regenerar la plantilla ni copiar su árbol:** se aplicarán las diferencias relevantes sobre ramas de cada consumidor, preservando cambios de dominio, scripts, workflows y configuraciones.

## 1. Migrar cada consumidor a la plantilla vigente, de manera incremental

**Autoridad:** [UPGRADING.md en template-service](https://github.com/atom-dev-serviu/template-service/blob/main/UPGRADING.md), y su [informe de migración](https://github.com/atom-dev-serviu/template-service/blob/main/docs/development/vslices-0.6.0-preview.2-migration.md). La versión de template-service y la de VSlices **no** son la misma numeración.

1. En cada repositorio: registrar SHA de partida, NuGet instalados, archivos modificados respecto del template y ejecución inicial de pruebas. Abrir una rama de migración por consumidor.
2. **No usar** `scripts/switch-template-version.py --apply` en un servicio con personalizaciones: reemplaza archivos versionados; no es un merge ni una migración incremental.
3. En los proyectos reales `*.Spaces` y `*.Work`, alinear los paquetes `VSlices.Spaces`, `VSlices.Work` y `VSlices.Work.DataAccess` requeridos a `0.6.0-preview.2`. Los átomos `PointReader`, `PointWriter`, `PointRemover` y `HasPoint*` están ahora en el paquete `VSlices.Work.DataAccess`, manteniendo `VSlices.Work.Algebra`.
4. Revisar cada Feature frente a **FM011**: declarar `Feature<F, F.Algebra, Request, Response>` y una interfaz `Algebra` (o `Alg`) **anidada y pública**, con sus capacidades mínimas. El runtime concreto compone las interfaces de sus Features.
5. Revisar cada Space concreto frente a **SM008**: `DiscreteSpace<SELF>` e igualdad correcta para su semántica real. No convertir indiscriminadamente todas las clases en records ni modificar identidad/invariantes para conformar una prueba.
6. Mantener las referencias a Skills y Laws del tag realmente instalado; contrastar código fuente y contratos de `v0.6.0-preview.2` cuando la documentación histórica discrepe.
7. Verificar restore/build, Laws y regresiones del consumidor. La plantilla usa xUnit v3/Microsoft Testing Platform: comprobar el runner real antes de elegir `dotnet run` o `dotnet test`. Ejecutar SQL Server/Testcontainers cuando corresponda; documentar cualquier prueba omitida.
8. Registrar lo realizado, resultados y límites antes de abrir PR; no presentar la migración de Framework como prueba de seguridad HTTP o despliegue.

Esta migración **no cambia por sí misma** la Decisión 001 sobre la numeración anticipada de Ticket ni los contratos de dominio del resto de los documentos.

## 2. Incorporar los paquetes privados Authentication

**Fuente:** [auth-dependency](https://github.com/atom-dev-serviu/auth-dependency), release `v0.1.0-preview.1`, publicado en el feed privado de GitHub Packages de `atom-dev-serviu`. Estos paquetes consumen las bibliotecas públicas de autenticación de VSlices en `0.6.0-preview.2`. No clonar ni insertar los antiguos proyectos `VSlices.*` o código `Serviu.Authentication.*` en cada consumidor.

**Feed privado:**

```text
https://nuget.pkg.github.com/atom-dev-serviu/index.json
```

**Paquetes disponibles:**

| Paquete | Responsabilidad | Consumo indicado |
| --- | --- | --- |
| `Authentication.Spaces` | `AccountReference`, `ServiuIdentity`, reglas semánticas | Proyectos que efectivamente modelen o examinen estas identidades |
| `Authentication.Work` | Frontera de dependencia del lector `AuthenticationContextReader` | Proyectos que requieran esa capacidad, sin duplicarla |
| `Authentication.Grounding` | Interpretación de principal de cookie ASP.NET Core a identidad Serviu | Hosting que utilice esa realización de cookie |
| `Authentication.Hosting` | Composición de cookie y políticas de autorización ASP.NET Core | **Ticket Support Product/BFF**, en el Hosting HTTP adecuado |

`Authentication.Hosting` arrastra las dependencias anteriores vía NuGet. En el Product, instalar inicialmente:

```bash
dotnet add src/Templates.Hosting/Templates.Hosting.csproj package Authentication.Hosting --version 0.1.0-preview.1
```

Sustituir `Templates.Hosting` por el nombre real si el repositorio ya se renombró. Configurar feed privado (con lectura del paquete) **más** nuget.org; usar un PAT classic personal con `read:packages` localmente, guardado fuera de Git. Para GitHub Actions, conceder al repositorio consumidor acceso en **Manage Actions access** de cada paquete y utilizar su `GITHUB_TOKEN` cuando las políticas de la organización lo permitan. No incluir tokens en el código, workflow, archivo `NuGet.Config` versionado, release ni pseudocódigo.

### Importante: cookie de usuario no equivale a credencial entre servicios

El paquete publicado `Authentication.Hosting` configura **cookie ASP.NET Core** y lectores de identidad Serviu; no constituye por sí solo una implementación de JWT Bearer, un emisor/renovador de tokens de Accounts, ni una política completa de autorización de todas las APIs. **No instalar `Authentication.Hosting` indiscriminadamente en Ticket, Folder y Flow como si eso los protegiera mediante Bearer.**

Para los **tres servicios HTTP internos**, queda una tarea de realización por servicio:

- Establecer el mecanismo real para autenticar una llamada del Product/BFF y cómo se verifica el token/credencial emitido por Accounts u otra autoridad autorizada. La implementación de emisión, propagación y renovación no está suministrada por estos paquetes.
- Incorporar la capa `Authentication.Spaces`/`Authentication.Work` solo donde la semántica o las capacidades se requieran; el adaptador de autenticación del transporte interservicio debe existir y estar probado antes de afirmar que las rutas están protegidas.
- Aplicar autorización funcional/por recurso en la frontera adecuada **después de** reconocer al invocador. Una cookie válida o identidad reconocida no demuestra acceso a un Ticket, archivo o transición. El secreto técnico `X-Folder-Invocation` preexistente tampoco es autorización del usuario.
- Conservar frontera navegador → Product/BFF → Services. No reenviar la cookie del navegador entre servicios por defecto ni incluir tokens en logs.

No confundir `AccountReference` del solicitante (atribución) con la identidad del servicio llamante (autenticación de invocación) ni con la autorización para cada Feature. El Account Service continúa siendo emisor de la sesión; el paquete Authentication no emite credenciales.

## 3. Orden de integración y verificación

1. Actualizar incrementalmente los cuatro repositorios contra las guías vigentes y ejecutar sus suites actuales, sin introducir nuevos contratos HTTP como efecto lateral.
2. Incorporar `Authentication.Hosting 0.1.0-preview.1` al Product/BFF; configurar nombre/esquema de cookie y Data Protection del host, el `AuthenticationContextReader` y las políticas de las operaciones; conservar antiforgery en formularios.
3. Definir y comprobar la autenticación **real** Product → Ticket/Folder/Flow y sus políticas de invocación/autoridad, sin suponer que el paquete de cookie implementa Bearer. Agregar adaptadores cuando los contratos estén comprobados.
4. Ejecutar casos de consumidor: sesión válida, ausente e identidad Serviu inválida; falta de autorización; llamada interna sin credencial y con credencial inválida; acceso a recurso ajeno; rechazo de transiciones no autorizadas; antiforgery; y correlación sin filtrar secretos.
5. Revalidar la coordinación de registro `AssignNextTicketNumber → Folder.AddFile → Ticket.RegisterTicket → Flow.StartFlow`, junto con fallos parciales, sin afirmar transacción distribuida ni idempotencia automática.
6. En cada PR: reportar la versión de template/VSlices, paquetes Authentication instalados, evidencia real de CI, configuración de acceso al feed y pendientes operativos.

**Criterio de salida:** los paquetes restauran desde sus feeds auténticos, los cuatro servicios compilan y pasan sus pruebas correspondientes, y existe evidencia de autenticación y autorización real en los límites que estén expuestos. Hasta entonces, autenticación del Product y autenticación interservicio siguen siendo **trabajos pendientes**, no propiedades acreditadas de la PoC.

## Relación con el diseño anterior

La [Decisión 001](decisions/001-early-ticket-number.md) sigue vigente. El [contrato de integración](integration-contract.md) y los diseños de [Ticket](ticket-service-v2.md), [Folder](folder-service.md), [Flow](flow-orchestration-service.md) y [BFF](ticket-support-bff.md) conservan su autoridad conceptual sobre los procesos y modelos. Este documento guía una **siguiente realización**, no modifica retrospectivamente esos acuerdos.

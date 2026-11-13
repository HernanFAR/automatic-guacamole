# Ticket Support Product / BFF — modelo de composición para la PoC

**Decisión vigente:** [Decisión 001 — asignación anticipada de TicketCode y derivación de FolderPath](decisions/001-early-ticket-number.md) sustituye la preparación independiente de carpeta mediante GUID. El Product obtiene el código de Ticket Service **antes de subir archivos**.

**Estado:** pseudocódigo de responsabilidades y Features candidato a realización, no reemplazo ya implementado. **Fecha de revisión:** 2026-11-12. **Plazo PoC:** 2026-11-13. **Fuente de código examinada:** [`atom-dev-serviu/ticket-support-product@4c4ee69d8e7d776e0136c2c55cfe792c91f3f6f4`](https://github.com/atom-dev-serviu/ticket-support-product/tree/4c4ee69d8e7d776e0136c2c55cfe792c91f3f6f4).

El análisis aplica el mismo método de **Ticket Service v2**, **Flow Orchestration** y **Folder Service**: identificar significado, propiedad, contratos reales, Features, admisibilidad, resultados y preguntas abiertas. El BFF **no es propietario de los Spaces de negocio** de esos servicios; coordina actuaciones y construye representaciones para las Views.

## 1. Fuentes revisadas y evidencia

- [`TicketWorkflowOperations.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Work/Tickets/TicketWorkflowOperations.cs): registro, borradores, completados, monitoreo y referencias; solicitante obtenido de cuenta confiable, no del formulario.
- [`TicketAttachmentOperations.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Work/Tickets/TicketAttachmentOperations.cs): subida a Folder, asociación al Ticket, compensación de subida fallida, descarga y retiro con verificación de pertenencia.
- [`HttpTicketFolderFiles.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Grounding/Tickets/HttpTicketFolderFiles.cs): POST, GET metadata, GET contenido, DELETE privados de Folder.
- [`HttpTicketCommandPort.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Grounding/Tickets/HttpTicketCommandPort.cs): comandos privados al Ticket Service **actual**; no son aún los contratos de Ticket Service v2.
- [`TicketSupportFormEndpoints.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Hosting/TicketSupportFormEndpoints.cs): formularios, antiforgery, policies, registro, informes, mantenedores, descarga y exportación.
- [`TicketSupportHostingExtensions.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Hosting/TicketSupportHostingExtensions.cs): SSO de Accounts, DI, consultas SQL de lectura, cliente privado Folder, rutas; no establece una integración con Flow Orchestration.
- [`TicketActionCoordinator.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Work/Tickets/TicketActionCoordinator.cs) y [`AuditedTicketWorkflowOperations.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Work/Tickets/AuditedTicketWorkflowOperations.cs): proceso de autorización/auditoría modelado y probado con dobles; **las rutas actuales invocan `TicketWorkflowOperations` directamente**, no este coordinador auditado.
- [`ListTicketAttachmentDetails.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Work/Tickets/ListTicketAttachmentDetails.cs): enriquece referencias con metadatos reales, conserva referencias aunque Folder no responda.
- [`MaintainClassifications.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Work/Classifications/MaintainClassifications.cs) y lectores de Work: catálogo, filtros, bandejas, workspace, historia y exportación.
- [`TicketEvidenceUploadPolicy.cs`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Hosting/TicketEvidenceUploadPolicy.cs): 0..5 adjuntos, no vacíos, hasta 10 MiB cada uno, PDF/PNG/JPG/JPEG, rechazo de duplicados por nombre y tamaño.
- [Inventario de responsabilidades](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/docs/migration-responsibility-map.md) y [delimitación de auditoría](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/docs/audit-coordination-boundary-2026-10-31.md).

**Verificación de repositorio:** CI de regresión [run 43152727339](https://github.com/atom-dev-serviu/ticket-support-product/actions/runs/43152727339), `success` para HEAD `4c4ee69`. No se ejecutó aquí un E2E nuevo ni se infiere que Ticket v2, Flow Orchestration o sus nuevos contratos estén integrados.

## 2. Propiedad y composición

| Responsable | Posee | El BFF puede |
| --- | --- | --- |
| Ticket Service v2 | Ticket inalterable, informes Draft/Completed, catálogos, referencias de adjuntos | Solicitar registro, guardar/completar informes, mantener catálogos y leer datos autorizados |
| Folder Service | Carpetas, archivos, bytes, nombre, MIME, metadatos | Subir archivos, leer metadatos/contenido y retirar contenido para compensación si corresponde |
| Flow Orchestration | FlowSpecification/Version, instancias, transiciones y `TicketFlowFacts` | Solicitar StartFlow y AdvanceFlow; leer instancia, historial y elegibilidad |
| Accounts / Access | Identidad, autenticación y decisiones de acceso por definir | Resolver identidad de sesión y utilizar permisos demostrados, no inventarlos |
| Audit Service (futuro) | Registro de observaciones de actuaciones | Solicitar observaciones cuando exista contrato real; separar resultado funcional/auditado |
| Ticket Support Product/BFF | Coordinación de llamadas, validación de formularios, filtros, respuestas para Views | **No** escribir tablas de los servicios, definir estados del flujo ni ser dueño de archivos |

**Regla:** navegador → Product → capacidades de servicios; no navegador → Folder/Ticket/Flow. Las lecturas SQL de la implementación actual acceden a tablas compartidas propietarias de otros servicios: se preservan como evidencia, **no se legitiman como permisos generales para escritura**.

## 3. Spaces y representaciones del Product (no entidades persistidas nuevas)

El BFF necesita **datos de entrada, referencias externas y resultados de composición**, no un `TicketSupportProcess` persistido creado solo por simetría con los servicios.

```pseudo
// Tipos externos; en Product no son identificadores de Spaces propios.
Value TicketReference
Value AccountReference
Value FolderFileReference
Value FlowInstanceReference           // instancia propietaria de Flow Orchestration

Value TicketRegistrationInput {
    system: SystemReference
    module: ModuleReference
    incidenceType: IncidenceTypeReference
    incidentDate: Date
    profile: AssociatedProfileInput
    risk: Risk
    description: Text(min=1, max=500)
    // otros datos de registro siguen el contrato de Ticket Service v2
    attachments: Sequence<PendingEvidence>  // 0..5
}

Value PendingEvidence {
    name: Text(min=1, max=50)         // regla de Ticket v2, extensión incluida
    contentType: FileContentType
    bytes: Bytes
}

Value TicketFolderLocation {
    correlative: Text
    concept: Text
    pathTo: Text
    // Actual: (workspace.Code, "Ticket", workspace.Reference).
    // Para Ticket v2 (Decisión 001): (assignedTicketCode, "TicketSupport", "Attachments").
}

Value StoredEvidence {
    file: FolderFileReference
    name: Text
    contentType: FileContentType
    size: Int64
}

Projection TicketAttachmentDisplay {
    file: FolderFileReference
    ordinal: Int
    name: Optional<Text>
    contentType: Optional<FileContentType>
    size: Optional<Int64>
    // Missing Folder metadata MUST NOT erase existing Ticket reference.
}

Choice ProductCommandOutcome =
    Succeeded | Rejected | Failed | PartiallyCompleted

Derived TicketRegistrationReceipt {
    ticket: Optional<TicketReference>
    flowInstance: Optional<FlowInstanceReference>
    attachmentReferences: Sequence<FolderFileReference>
    outcome: ProductCommandOutcome
    // No afirmar atomicidad entre servicios.
}
```

**Corrección de nomenclatura:** `SystemReference`, `ModuleReference` e `IncidenceTypeReference` son referencias externas *desde el Product*, aunque en Ticket Service se denominen `SystemId`, `ModuleId`, `IncidenceTypeId`. Igual ocurre con `TicketReference` y `FlowInstanceReference`.

**Nota:** `TicketRegistrationReceipt` es una propuesta de representación de coordinación; no corresponde a una respuesta HTTP del Product actual. `TicketFolderLocation` contiene las coordenadas observadas del código, no una definición aprobada para v2. No introducir un almacenamiento adicional para estas representaciones.

## 4. Feature principal: registrar Ticket con evidencias e iniciar flujo

### Situación actual observada

El endpoint `/soporte/acciones/tickets/crear` valida archivos, **registra el Ticket primero** y después sube cada archivo a Folder, añadiendo su referencia al Ticket Service existente. Si falla una subida posterior, redirige indicando archivos pendientes. Ese comportamiento es incompatible con nuestra regla **Ticket v2 inalterable con adjuntos registrados desde el inicio**.

### Composición candidata para v2

```pseudo
Feature RegisterTicketSupport {
    input: TicketRegistrationInput + TrustedSession
    requires:
        caller authorized for registration
        && requester resolved from trusted Accounts session
        && registration input admissible by Ticket Service v2
        && evidenceCount <= 5
        && each evidence has admissible name and nonempty bytes
        && configured intake MIME/extension/size policy passes

    sequence:
        // Preparar referencias ANTES de registrar Ticket.
        code = TicketServiceV2.AssignNextTicketNumber() // POST, asignación durable y única
        location = TicketFolderLocation(code, "TicketSupport", "Attachments")
        stored = for each evidence:
            FolderService.AddFile(location, evidence, requester)

        registered = TicketServiceV2.RegisterTicket(
            input.withRequester(requester).withAssignedCode(code)
                 .withAttachments(stored.references)
        )

        started = FlowOrchestration.StartFlow(
            TicketSupportVersionV1Id,
            FlowSubject.TicketReference(registered.ticket)
        )

    returns: TicketRegistrationReceipt
    doesNot:
        modify registered Ticket after creation
        create second FlowInstance for same Ticket
        claim distributed transaction or guaranteed rollback
}
```

**Tensión resuelta conceptualmente:** el Product actual obtenía la carpeta de un Ticket ya registrado. [Decisión 001](decisions/001-early-ticket-number.md) introduce la asignación anticipada de `TicketCode` por Ticket Service, que permite derivar la carpeta tanto antes como después del registro. Quedan por definir la persistencia y exclusividad efectiva del número y la semántica de reintentos; no confundir código único con idempotencia.

**Fallo parcial:** si Folder recibe algunos archivos y falla el registro de Ticket, quedan potencialmente huérfanos. Si el Ticket se registra y `StartFlow` falla, queda registrado sin instancia; la regla de una instancia por Ticket no inventa reintentos ni orquestación transaccional. El Product debe distinguir estados observables, evitar duplicar registros por redirección y no presentar éxito integral ficticio. Para la PoC no exigimos transacciones distribuidas, outbox ni mecanismos de recuperación fuertes.

## 5. Features de informes y avance de flujo

Los informes son Spaces propietarios de Ticket Service v2; `TicketFlowFacts` y las transiciones se calculan en Flow Orchestration. El Product **solicita** ambas actuaciones y reporta sus resultados, sin asumir que completar un informe realiza automáticamente una transición.

```pseudo
Feature SaveEvaluationDraft {
    input: TicketReference + DraftEvaluationInput
    calls: TicketServiceV2.SaveEvaluationDraft
    guarantees: saving a draft does not advance the FlowInstance
}

Feature CompleteEvaluationAndAdvance {
    input: TicketReference + DraftEvaluationInput
    sequence:
        TicketServiceV2.CompleteEvaluation
        FlowOrchestration.InspectEligibleTransitions
        FlowOrchestration.AdvanceFlow // Solo si corresponde y está autorizada
    doesNot: select destination independent of the FlowSpecificationVersion
}

Feature SaveApprovalDraft {
    input: TicketReference + DraftApprovalInput
    calls: TicketServiceV2.SaveApprovalDraft
}

Feature CompleteApprovalAndAdvance {
    input: TicketReference + DraftApprovalInput
    sequence:
        TicketServiceV2.CompleteApproval
        FlowOrchestration.InspectEligibleTransitions
        FlowOrchestration.AdvanceFlow
}

Feature SaveSupervisionDraft {
    input: TicketReference + DraftSupervisionInput
    calls: TicketServiceV2.SaveSupervisionDraft
    guarantees: closure != "Cerrado" does not close the flow
}

Feature CompleteSupervisionAndMaybeClose {
    input: TicketReference + DraftSupervisionInput
    requires: closure == "Cerrado" and other required fields admissible
    sequence:
        TicketServiceV2.CompleteSupervision
        FlowOrchestration.InspectEligibleTransitions
        FlowOrchestration.AdvanceFlow // Supervisión -> Cerrado cuando sea elegible
}
```

Una `FlowInstance` interpreta siempre su `FlowSpecificationVersion`; el Product no persiste una segunda etapa `TicketStage` ni decide la condición por inspeccionar formularios. **No** recuperar edición del Ticket, responsable asignado ni edición posterior de adjuntos: decisiones descartadas del modelo v2.

## 6. Features de lectura y presentación

El código actual contiene lectores para catálogo, bandeja, conteos, recientes, workspace, historial, referencias y metadatos, más exportación Excel. Son operaciones **de Product**, que combinan información de servicios sin apropiarse de sus Spaces.

```pseudo
Feature GetTicketSupportWorkspace {
    input: TicketReference
    reads:
        TicketServiceV2.Ticket + EvaluationReport + ApprovalReport + SupervisionReport
        FlowOrchestration.FlowInstance + ExecutedTransition history
        TicketServiceV2.AttachmentReferences
        FolderService.FileMetadata for linked references
    returns: TicketWorkspacePresentation
    doesNot: fabricate a history from a fallback/demo
}

Feature ListTicketSupportTray {
    input: TicketTrayFilter
    reads:
        TicketServiceV2 tickets
        FlowOrchestration current FlowInstance states
    returns: TicketTrayPresentation
    // no persistir TicketStage en Ticket Service v2
}

Feature GetTicketSupportStageCounts {
    reads: FlowOrchestration states joined with Ticket references
    returns: counts per state including zero for absent states
}

Feature ListTicketSupportClassifications {
    reads: TicketServiceV2.Systems + Modules + IncidenceTypes
    returns: catalogue choices with activationStatus
}

Feature ListTicketAttachmentDetails {
    input: TicketReference
    reads: TicketServiceV2.AttachmentReferences then FolderService.FileMetadata
    returns: ordered TicketAttachmentDisplay
    invariant: unavailable metadata does not erase reference or fabricate filename
}

Feature DownloadTicketAttachment {
    input: TicketReference + FolderFileReference
    requires:
        caller authorized for Ticket
        && FolderFileReference belongs to that Ticket
    calls: FolderService.DownloadFile with owner FolderPath
    returns: bytes + MIME + filename
}

Feature ExportTicketSupport {
    input: TicketExportFilter
    reads: authorized Ticket and Flow data
    materializes: XLSX in Grounding using ClosedXML
    doesNot: grant export rights merely because data is readable
}
```

**Alcance:** `ListTicketTray`, `GetTicketWorkspace`, `GetTicketStageCounts`, `GetRecentTickets`, `GetTicketStageHistory`, `ListTicketAttachmentReferences`, `ListTicketAttachmentDetails` y `ExportTickets` existen en Work. Sus adaptadores actuales leen tablas SQL de Tickets y el campo legacy `Stage`; **reutilizar su presentación es razonable, pero migrar sus lecturas al ownership de Flow Orchestration requiere una iteración verificable**. La exportación y las consultas de catálogos pueden posponerse si la demostración del recorrido principal no las necesita.

## 7. Mantenimiento de catálogos desde Product

```pseudo
Feature MaintainSystem {
    operations: CreateSystem | RenameSystem | ActivateSystem | DeactivateSystem
    delegates: TicketServiceV2
}

Feature MaintainModule {
    operations: CreateModule | RenameModule | ActivateModule | DeactivateModule
    delegates: TicketServiceV2
}

Feature MaintainIncidenceType {
    operations: CreateIncidenceType | RenameIncidenceType
              | ActivateIncidenceType | DeactivateIncidenceType
    delegates: TicketServiceV2
}
```

El Product interpreta intención, formulario y permisos; **Ticket Service v2 mantiene unicidad normalizada y estados de activación**. Crear un elemento produce `Active` por defecto; no exponer una opción inicial `Inactive` heredada del endpoint actual. No incorporar `DeleteModule`.

Las acciones del Product pueden compartir infraestructura para invocación, pero no deben redefinir la admisibilidad de los catálogos.

## 8. Autorización, atribución y auditoría

- **Identidad:** `ICurrentRequesterAccount` obtiene el solicitante desde contexto confiable; el navegador no elige `AccountReference`.
- **Autorización:** Hosting exige sesión y políticas por ruta/formulario. El código contiene una superficie `TicketActionCoordinator` que falla cerrado ante acceso desconocido o actor ausente; su conexión con Access Service y las reglas por recurso **no están demostradas** para la PoC. No confundir cookie SSO con permiso sobre un Ticket.
- **CSRF:** endpoints de formularios validan antiforgery; conservarlo.
- **Folder:** secreto temporal `X-Folder-Invocation` solo entre Product y servicio; no conceder permiso por conocer un GUID.
- **Auditoría:** `AuditedTicketWorkflowOperations` distingue resultado funcional y resultado de Audit. Falta adaptador real para `ITicketAuditObservationPort`; no fingir registro persistido ni invocarlo como si estuviera conectado.
- **Resultados:** cuando una secuencia interservicios se interrumpe, representar éxito parcial real, rechazo o fallo conocido; no inventar cambios de estado, permisos o registros de auditoría.

## 9. Reglas y casos de verificación

| Caso | Resultado semántico esperado |
| --- | --- |
| Registro con 0 adjuntos | Ticket creado sin archivos; se solicita iniciar una única instancia V1 |
| Registro con 1..5 adjuntos válidos | Referencias de Folder conocidas al registrar el Ticket; Ticket no cambia después |
| Adjunto vacío, >10 MiB o extensión no admitida | Rechazo previo a invocaciones de los servicios |
| Fallo en una carga de Folder | No afirmar Ticket creado; registrar qué archivos alcanzaron a persistir |
| Ticket registrado, falla StartFlow | Mostrar éxito parcial sin duplicar Ticket ni comenzar segunda instancia |
| Guardar borrador de evaluación/aprobación/supervisión | No cambia FlowInstance |
| Completar evaluación Soporte | Flow Orchestration habilita Evaluación → Supervisión |
| Completar evaluación Mejora | Habilita Evaluación → Aprobación |
| Completar aprobación | Habilita Aprobación → Supervisión |
| Supervisión distinta de Cerrado | Permanece sin transición a Cerrado |
| Supervisión Cerrado | Habilita Supervisión → Cerrado |
| Descarga con referencia que no pertenece al Ticket | Rechazar sin consultar bytes en Folder |
| Metadatos ausentes | Mantener referencia y orden; no inventar nombre |
| Catálogo nuevo | Elemento `Active` por defecto; unicidad pertenece al Service |
| Autorización desconocida | No ejecutar comando; reportar decisión como desconocida |

La regresión actual cubre varias piezas con dobles (acciones, clasificación, adjuntos, HTTP y lecturas). **No demuestra la secuencia de Ticket v2 + Flow Orchestration**, ni una implementación de `RegisterTicketSupport` que resuelva sus dependencias.

## 10. Decisiones abiertas y propuesta de prioridades

**Bloqueos semánticos directos:**
1. **Realización de asignación anticipada:** derivación de `FolderPath` decidida en [Decisión 001](decisions/001-early-ticket-number.md); falta adaptar `PrepareAttachmentLocation` y formalizar POST, persistencia, unicidad y contratos de reintento.
2. **Contrato de Ticket v2:** forma real del registro que recibe `FolderFileReference`, y de lecturas/proyecciones para el Product.
3. **Inicio del flujo:** cómo encontrar la instancia de un Ticket y evitar el doble inicio durante reintentos.
4. **Resultados parciales:** qué respuesta visible debe producir el Product si Folder, Ticket o Flow fallan en distintos pasos.

**Fuera del alcance inicial, salvo bloqueo observado:** transacciones distribuidas, outbox, reintentos garantizados, concurrencia fuerte, autorización definitiva de Access Service, integración Audit Service, nuevos formatos de exportación y CRUD dinámico de FlowSpecification.

**Orden de realización recomendado:** primero `RegisterTicketSupport` con y sin adjuntos y una `FlowInstance` inicial; luego operaciones de informes y avance; después workspace/adjuntos y vistas. Mantener catálogos y exportación según necesidad observable, sin trasladar a Product persistencia de Tickets, archivos ni Flow.

**No realizado en este documento:** modificaciones a `atom-dev-serviu/ticket-support-product`, ejecución de E2E ni integración real de nuevos servicios. Se trata de la **primera propuesta revisable de pseudocódigo**, no de una funcionalidad entregada.

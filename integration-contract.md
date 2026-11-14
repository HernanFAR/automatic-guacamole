# Ticket Support Product — contrato transversal de integración

**Decisión vigente sobre numeración y carpeta:** [Decisión 001](decisions/001-early-ticket-number.md), que sustituye la preparación provisional de FolderPath mediante GUID independiente.

**Estado:** contrato **conceptual y revisable** para la PoC del 13 de noviembre de 2026. Define propiedad, conexiones, secuencias y fallos observables; **no** afirma que los nuevos servicios estén implementados o integrados E2E, ni fija endpoints, transporte o transacciones distribuidas. Complementa [Ticket Service v2](ticket-service-v2.md), [Folder Service](folder-service.md), [Flow Orchestration](flow-orchestration-service.md) y [el modelo del BFF](ticket-support-bff.md).

**Procedencia de la composición existente:** [`atom-dev-serviu/ticket-support-product@4c4ee69`](https://github.com/atom-dev-serviu/ticket-support-product/tree/4c4ee69d8e7d776e0136c2c55cfe792c91f3f6f4). Los comportamientos de ese código pertenecen al Product actual y **no** implican contratos vigentes del Ticket Service v2 ni del orquestador futuro.

## 1. Topología y fronteras

```pseudo
Browser / TicketSupport.Views
    |
    | sesión, formularios, consulta y descarga (sin acceso directo a Services)
    v
Ticket Support Product / BFF
    |-- Accounts / Access : identidad confiable y decisiones de acceso
    |-- Ticket Service v2 : Ticket, informes, catálogos, referencias de archivos
    |-- Folder Service    : contenido, ubicación, metadatos y descarga
    |-- Flow Orchestration: instancias, estados, elegibilidad e historial
    `-- Audit Service     : observaciones de actuaciones, integración futura

Flow Orchestration
    `-- read-only SQL JOIN/view -> tablas de informes de Ticket Service v2
        // TicketFlowFacts pertenece al orquestador, no a Ticket Service.
```

El **Product/BFF coordina** las invocaciones y compone respuestas para las Views. **No es propietario** de los Spaces de negocio de esos servicios y no debe replicar `TicketStage` como fuente autoritativa de progreso.

| Pregunta | Propietario / responsabilidad |
| --- | --- |
| ¿Qué constituye un Ticket válido y sus adjuntos iniciales? | Ticket Service v2; Ticket registrado inalterable |
| ¿Quién posee evaluación, aprobación y supervisión Draft/Completed? | Ticket Service v2 |
| ¿Quién posee el nombre, tipo MIME, bytes y tamaño de archivos? | Folder Service; Ticket conserva únicamente referencias |
| ¿Quién determina ubicación `FolderPath` durante la composición? | Ticket Service v2 asigna previamente el código; Product deriva `FolderPath(code, "TicketSupport", "Attachments")` por convención estable; Folder valida la identidad estructurada |
| ¿Quién posee la instancia y las transiciones ejecutadas? | Flow Orchestration |
| ¿Quién define y calcula `TicketFlowFacts`? | Flow Orchestration, mediante lectura SQL de solo lectura sobre datos propietarios de Ticket Service v2 |
| ¿Quién evalúa si una transición es elegible? | Flow Orchestration; elegibilidad **no** sustituye autorización |
| ¿Quién coordina el orden de operaciones y resultados parciales? | Ticket Support Product/BFF |
| ¿De dónde proviene la identidad del solicitante? | Sesión confiable de Accounts; nunca un parámetro libre del navegador |
| ¿Quién concede autorización funcional? | Autoridad de Access por concretar; el Product debe verificarla antes de actuar |
| ¿Quién registra auditoría? | Audit Service futuro; el Product solicita observaciones cuando exista conexión real |

## 2. Conexiones actuales versus integración propuesta

| Dependencia | Código actual de Ticket Support Product | Contrato conceptual de PoC |
| --- | --- | --- |
| Browser → Product | Razor/Hosting, formularios con antiforgery, políticas de ruta y SSO compartido | Mantener al Product como única superficie de usuario |
| Product → Tickets | `HttpTicketCommandPort` para registrar, guardar/completar evaluación y aprobación, monitoreo y editar referencias; consultas SQL para bandejas, workspace, catálogos | Adaptar a Ticket Service **v2**: Ticket inalterable, informes Draft/Completed, catálogos; sin mutar adjuntos tras registro |
| Product → Folder | `HttpTicketFolderFiles`: `Upload`, `ReadMetadata`, `Download`, `Remove` con `X-Folder-Invocation`; carpeta actual derivada del Ticket registrado | Usar `AddFile`, `GetFileMetadata`, `DownloadFile` y `RemoveFile` cuando proceda para compensación/compatibilidad; no `CopyFile` |
| Product → Flow | No hay integración real de los Spaces y Features nuevos | `StartFlow`, `InspectFlow`, `InspectEligibleTransitions`, `AdvanceFlow` |
| Product → Accounts / Access | Lector de cookie SSO y políticas actuales; no suponer que equivalen a facultades definitivas por recurso | Resolver origen confiable y exigir permisos pertinentes sin inventar concesiones |
| Product → Audit | `TicketActionCoordinator` y `AuditedTicketWorkflowOperations` existen como procesos probados con dobles; no están conectados a las rutas actuales mediante Audit real | Preservar distinción entre resultado funcional e intento de auditoría; integración posterior |

**Regla de privacidad y autoridad:** navegador → Product → Services. No exponer credenciales de invocación Folder ni permitir descargar un archivo por conocer solamente `AttachedFileId`.

## 3. Contratos entre fronteras

### 3.1. Referencias y datos

```pseudo
Ticket Service v2 exports:
    TicketReference                // identificador externo desde Product o Flow
    Ticket, EvaluationReport, ApprovalReport, SupervisionReport
    AttachmentReferences          // pertenencia y orden establecidos por Tickets

Folder Service exports:
    FolderPath(Correlative, Concept, PathTo)
    AttachedFileId
    FileMetadata(id, name, contentType, size)
    FileContent                   // bytes obtenidos solo para descarga

Flow Orchestration exports:
    FlowSpecificationVersionId
    FlowInstanceId
    FlowSubject = TicketReference // unión discriminada; futuros tipos si hay consumidores
    ConditionalTransitionId
    ExecutedTransition history
```

`TicketSupport/V1` admite **una sola FlowInstance por Ticket**, incluso si se cerró; la política se expresa en `FlowSpecificationVersion` y no es universal para todos los flujos.

La referencia a Folder debe permitir al Product ubicar el archivo de forma verificable **sin** convertir `FolderPath` en una ruta física arbitraria ni atribuir a Folder el conocimiento de Tickets. La forma exacta del `FolderFileReference` de Ticket v2 está pendiente; no asumir que un GUID aislado basta para autorizar o localizar contenido.

### 3.2. Proyección para condiciones de transición

```pseudo
FlowOrchestration.ReadTicketFlowFacts(TicketReference)
    reads: Ticket Service v2 report tables (read-only JOIN/view)
    derives:
        evaluationCompleted
        evaluationScope
        approvalCompleted
        supervisionCompleted
        closureState
```

Las tablas e informes siguen perteneciendo a Ticket Service; la proyección pertenece al orquestador. No se añade una copia persistida en el Product ni una API obligatoria de `TicketFlowFacts` a Ticket Service.

La supervisión solo está **completada** cuando el informe registra `closure = "Cerrado"`. En la proyección esto equivale a `supervisionCompleted = true` y `closureState = Some("Cerrada")`; cualquier otro valor permanece sin cierre.

## 4. Secuencias funcionales coordinadas por Product/BFF

### 4.1. Registrar Ticket con archivos e iniciar su flujo

```pseudo
case RegisterTicketSupport {
    Product.ValidateFormAndAuthorization()
    requester = Accounts.ResolveTrustedAccountReference()

    // Regla v2: Ticket nace con sus referencias; no agregar después.
    code = TicketServiceV2.AssignNextTicketNumber() // POST, asignación única y durable
    location = FolderPath(code, "TicketSupport", "Attachments")
    stored = each(valid evidence, FolderService.AddFile(location, evidence, requester))

    ticket = TicketServiceV2.RegisterTicket(
        registration.withRequester(requester)
                    .withAssignedCode(code)
                    .withAttachmentReferences(stored.references)
    )

    flow = FlowOrchestration.StartFlow(
        TicketSupportVersionV1Id,
        FlowSubject.TicketReference(ticket.id)
    )

    return Product.ComposeRegistrationResult(ticket, stored, flow)
}
```

**No confundir con el Product actual:** su Hosting ejecuta `RegisterTicket → Upload → AddAttachmentReference`, porque obtiene la carpeta desde `TicketWorkspace(Code, Reference)` después de registrar. Ese orden **no puede trasladarse directamente** al Ticket v2 inalterable.

**Decisión material resuelta (diseño):** [Decisión 001](decisions/001-early-ticket-number.md) establece que Ticket Service asigna previamente un código único y durable. Product deriva de él la carpeta y Ticket Service recibe ese mismo código al registrar. Siguen pendientes los contratos físicos, la forma de las referencias y el tratamiento de reintentos; la asignación anticipada no implica idempotencia automática.

**Fallos parciales a representar, sin fingir transacción:** archivos subidos pero registro fallido; Ticket registrado pero `StartFlow` fallido; no duplicar Ticket o FlowInstance como respuesta automática a un reintento. El tratamiento exacto de limpieza, reintentos y recuperación sigue abierto.

### 4.2. Evaluación y aprobación

```pseudo
case SaveEvaluationDraft {
    Product.AuthorizeAction()
    TicketServiceV2.SaveEvaluationDraft(ticket, draft)
    // No avanzar FlowInstance.
}

case CompleteEvaluationAndAdvance {
    Product.AuthorizeAction()
    TicketServiceV2.CompleteEvaluation(ticket, draft)
    instance = FlowOrchestration.InspectFlow(instanceIdFor(ticket))
    eligible = FlowOrchestration.InspectEligibleTransitions(instance.id)
    requested = Product.SelectMatchingEligibleTransition(eligible)
    FlowOrchestration.AdvanceFlow(instance.id, instance.currentState, requested)
}

case SaveApprovalDraft {
    Product.AuthorizeAction()
    TicketServiceV2.SaveApprovalDraft(ticket, draft)
    // No avanzar FlowInstance.
}

case CompleteApprovalAndAdvance {
    Product.AuthorizeAction()
    TicketServiceV2.CompleteApproval(ticket, draft)
    instance = FlowOrchestration.InspectFlow(instanceIdFor(ticket))
    eligible = FlowOrchestration.InspectEligibleTransitions(instance.id)
    requested = Product.SelectMatchingEligibleTransition(eligible)
    FlowOrchestration.AdvanceFlow(instance.id, instance.currentState, requested)
}
```

El orquestador **decide elegibilidad**, usando `ConditionalTransitionId` y hechos actuales; el Product no reconstruye predicados ni escribe un segundo estado de flujo. `instanceIdFor(ticket)` indica una necesidad de lookup todavía sin contrato físico. La selección de transición elegible es una responsabilidad de coordinación y no autoriza elegir destinos que el orquestador no devolvió.

### 4.3. Supervisión y eventual cierre

```pseudo
case SaveSupervisionDraft {
    Product.AuthorizeAction()
    TicketServiceV2.SaveSupervisionDraft(ticket, draft)
    // closure != "Cerrado"; no cierra el flujo.
}

case CompleteSupervisionAndClose {
    Product.AuthorizeAction()
    require draft.closure == "Cerrado"
    TicketServiceV2.CompleteSupervision(ticket, draft)
    instance = FlowOrchestration.InspectFlow(instanceIdFor(ticket))
    eligible = FlowOrchestration.InspectEligibleTransitions(instance.id)
    require TicketSupervisionToClosedId in eligible
    FlowOrchestration.AdvanceFlow(
        instance.id, instance.currentState, TicketSupervisionToClosedId
    )
}
```

No presentar una supervisión `"Con observaciones"` o `"Sin Cierre"` como `CompletedSupervisionReport`; no pasar a `Cerrado` por mera presencia de un informe guardado.

### 4.4. Leer bandeja, workspace y adjuntos

```pseudo
case BuildTicketWorkspace {
    Product.AuthorizeTicketRead(ticket)
    ticketData = TicketServiceV2.ReadTicketAndReports(ticket)
    instanceAndHistory = FlowOrchestration.InspectFlow(instanceIdFor(ticket))
    references = TicketServiceV2.ReadAttachmentReferences(ticket)
    metadata = for each references:
        FolderService.GetFileMetadata(ResolveFolderPath(ticket), reference)
    return Product.ComposeWorkspace(ticketData, instanceAndHistory, references, metadata)
}

case DownloadTicketAttachment {
    Product.AuthorizeTicketRead(ticket)
    references = TicketServiceV2.ReadAttachmentReferences(ticket)
    require requestedReference belongs to references
    return FolderService.DownloadFile(ResolveFolderPath(ticket), requestedReference)
}

case BuildTicketTrayAndCounts {
    // La etapa autoritativa ya no es Ticket.Stage.
    tickets = TicketServiceV2.ReadTicketsForTray(...)
    states = FlowOrchestration.ReadInstancesForTickets(tickets) // contrato batch pendiente
    return Product.ComposeTrayAndStageCounts(tickets, states)
}
```

**No inventar metadatos:** una referencia existe aunque la consulta a Folder falle. El Product actual conserva asociación y orden, mostrando metadatos no disponibles. No reconstruir un historial desde fixtures ni desde `TicketStageTransitions` legacy si los hechos reales deben pertenecer a Flow.

`ReadTicketAndReports`, `ReadAttachmentReferences`, `ReadTicketsForTray` y `ReadInstancesForTickets` expresan **capacidades requeridas**, no endpoints o queries ya confirmadas en los servicios v2. La estrategia para evitar N+1 lecturas y para preservar filtros/exportación se decidirá durante la realización.

### 4.5. Catálogos y exportación

El Product conserva lectura/mantenimiento de `System`, `Module` e `IncidenceType`, delegando sus reglas y unicidad a Ticket Service v2. Una creación inicia `Active`; no introducir `DeleteModule`. La exportación XLSX es una Feature del Product con materialización ClosedXML en Grounding y permisos específicos; no es una Feature del Flow Service.

## 5. Seguridad, resultados y consistencia

- **SSO:** aceptar identidad únicamente desde sesión confiable; no tomar solicitante, autorización o secretos entre servicios desde campos arbitrarios del browser.
- **Autorización:** el Product verifica acceso de ruta, actuación y pertenencia de cada Ticket según autoridad efectiva; cookie válida no equivale por sí sola a permiso de negocio.
- **Antiforgery:** conservar validación en formularios humanos.
- **Folder:** `X-Folder-Invocation` es secreto técnico privado, **no** autorización del usuario. La descarga solo ocurre después de verificar referencia vinculada y acceso al Ticket.
- **Audit:** el código actual contiene coordinación abstracta, pero no hay adaptador Audit integrado ni garantía de recibo. Separar resultado funcional, intento de auditoría y fallos.
- **Fallos:** no etiquetar como fracaso total una operación de Ticket que ya quedó persistida cuando falle Flow o Audit; devolver estado parcial reconocible.
- **PoC:** atomicidad de Points y `ExecutedTransition`, concurrencia optimista, consistencia fuerte entre servicios, colas/outbox y compensaciones garantizadas son **deseables posteriores**, no bloqueos para el 13 de noviembre. Los errores observables no deben ocultarse.

### Distribución e identidad para la siguiente implementación

La [preparación de implementación](implementation-readiness.md) exige migrar incrementalmente los cuatro consumidores de template-service siguiendo `UPGRADING.md` (VSlices `0.6.0-preview.2`), sin sobrescribir árboles personalizados.

El Product/BFF deberá consumir el paquete privado `Authentication.Hosting 0.1.0-preview.1` para interpretar la sesión cookie e identidad Serviu. Este paquete **no** autentica automáticamente las invocaciones a Ticket, Folder o Flow: las credenciales del invocador, su verificación y la autorización por recurso son contratos y realizaciones separados. No hacer pasar una cookie de navegador ni `X-Folder-Invocation` por prueba suficiente de autoridad; preservar la procedencia de solicitante y servicio llamante.

La publicación del NuGet en GitHub Packages y la restauración de sus dependencias VSlices desde nuget.org son hechos de distribución, no pruebas de que los cuatro repositorios hayan instalado el paquete ni de que sus rutas estén protegidas.

## 6. Pruebas mínimas de contrato transversal

| Escenario | Qué se verifica |
| --- | --- |
| Registro sin adjuntos | Ticket registrado e intento de crear una sola instancia `TicketSupport/V1` |
| Registro con adjuntos | Folder devuelve referencias antes del registro inalterable de Ticket |
| Fallo de Folder antes del registro | No se afirma Ticket creado; se informa posibilidad de archivos ya subidos |
| Fallo de Flow después del registro | Ticket sigue existiendo; estado parcial visible, sin duplicación automática |
| Completar evaluación Soporte/Mejora | Orquestador ofrece la transición según `TicketFlowFacts` |
| Guardar borradores | No avanzar FlowInstance |
| Completar aprobación | Transición por ID elegible hacia Supervisión |
| Guardar supervisión distinta de Cerrado | Flujo no cierra |
| Completar supervisión Cerrado | Orquestador permite Supervisión → Cerrado |
| Descargar archivo no asociado | Rechazo previo a consultar bytes |
| Metadatos de Folder ausentes | Se conserva referencia, sin inventar nombre |
| Sesión o autorización insuficiente | No invocar comandos privados por apariencia de sesión |
| Fallo de Audit | No reemplazar resultado funcional ya conocido por uno inventado |

Estas pruebas son **criterios para implementar**, no declaración de que ya fueron ejecutadas con todos los servicios.

## 7. Decisiones pendientes, priorizadas

1. **Realizar la asignación anticipada de TicketCode:** decisión de derivación de FolderPath cerrada en [Decisión 001](decisions/001-early-ticket-number.md); faltan persistencia concurrente, contrato POST, referencias externas y semántica de reintento.
2. **Contratos reales de Ticket v2:** registro con referencias y lecturas requeridas por las Views.
3. **Lookup de FlowInstance por Ticket y lecturas de bandeja:** no existen contratos concretos confirmados para `instanceIdFor(ticket)` o las consultas batch.
4. **Respuestas parciales del Product:** forma observable para el usuario ante fallo de Folder, registro o Flow; recuperación proporcional a la PoC.
5. **Autoridad y auditoría:** reglas efectivas de Access y conexión de Audit; no adjudicar permisos basados únicamente en SSO.
6. **Historia y sucesión:** conservar `ExecutedTransition` como historia de Flow y planificar reemplazo de `TicketStageTransitions` legacy.
7. **Versiones:** selección de versiones para nuevos Tickets, mantenimiento de ejecuciones anteriores y migraciones explícitas.
8. **Seguimientos:** forma de representar múltiples seguimientos de supervisión sin confundirlos con cierre.
9. **Post-PoC:** atomicidad reforzada, control de concurrencia, idempotencia y recuperación garantizada.

**Prioridad inmediata:** resolver el punto 1 para poder realizar `RegisterTicketSupport`; después verificar el recorrido funcional de informes y transición. Este documento no reemplaza los modelos propietarios ni impone abstracciones de infraestructura que los casos todavía no exigen.

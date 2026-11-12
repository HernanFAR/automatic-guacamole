# Ticket Service v2 — modelo de Spaces y Features

**Estado:** pseudocódigo candidato; la semántica de Ticket e informes procede del estudio de [Serviu Tickets](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/spaces.md). Los mecanismos concretos y las firmas todavía no están aprobados.

## 1. Propiedad semántica

Ticket Service v2 reconoce la **solicitud formalizada** y los informes que la describen. **No posee** el estado de la máquina ni decide si corresponde transitar a otra etapa. El Ticket es inalterable después de su registro; sus informes son conceptos independientes referenciados por el identificador del Ticket y tienen **cardinalidad máxima de uno por Ticket**.

```pseudo
Space Ticket {
    id: TicketId                  // único
    requester: Reference<Account>
    incidentDate: PastOrPresentDate
    description: Text(max=500)
    system: Reference<System>
    module: Reference<Module>
    incidentType: Reference<IncidentType>
    risk: Choice<"Bajo", "Medio", "Alto">
    associatedProfile: AssociatedProfile | Reference<HR.Profile>
    associatedElement: RecognizableReference | Text(max=50)
    attachments: Sequence<Attachment>(min=0, max=5)

    invariant: After(RegisterTicket), no characteristic evolves.
    NOT INCLUDED: FlowState, FlowHistory, EvaluationReport, ApprovalReport, SupervisionReport.
}

Space Attachment {
    resource: Reference<FolderFile>
    name: Text(max=50)
    sizeBytes: Integer(>0)
    invariant: no change after Ticket is registered.
}

Space EvaluationReport {
    id: TicketId                     // identity = owning Ticket reference
    formalization: Choice<"Borrador", "Formalizado">
    scope: Optional<Choice<"Soporte", "Mejora">>
    // Remaining report characteristics: from authoritative Spaces spec
    fields: EvaluationData

    invariant: one report per Ticket
    evolution: draft -> new immutable Point with revised data
    invariant: formalized report cannot evolve
}

Space ApprovalReport {
    id: TicketId
    formalization: Choice<"Borrador", "Formalizado">
    fields: ApprovalData
    invariant: one report per Ticket
    evolution: draft -> revised immutable Point
    invariant: formalized report cannot evolve
}

Space SupervisionReport {
    id: TicketId
    closure: Choice<"Sin cierre", "Cerrada", "Con observaciones"> // recognized choices
    fields: SupervisionData
    // The completion criterion and permitted evolution require refinement.
}
```

`EvaluationData`, `ApprovalData` y `SupervisionData` son referencias a la composición estudiada, **no** propuestas de un `Dictionary<string, object>` ni licencias para inventar campos. La fuente propietaria detalla características, opciones y restricciones: [spaces.md](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/spaces.md). Los catálogos de Sistema/Módulo/Tipo de incidencia y Perfil asociado permanecen allí.

### Distinción crítica

```pseudo
Ticket.Immutable != Report.ImmutableDuringDraft
EvaluationReport.Formalized != Flow.CurrentState == "Aprobación"
SupervisionReport.Closure == "Cerrada" != Flow.CurrentState == "Cerrado"
```

El estado del informe es dato propio; el estado del flujo es hecho de otra responsabilidad.

## 2. Features candidatas

```pseudo
Feature RegisterTicket {
    input: RegistrationData + Optional<InitialEvidenceReferences>
    requires: mandatory registration fields are admissible
    writes: Ticket
    returns: TicketId
    guarantees: Ticket exists and its constitutive characteristics cannot be edited
    doesNot: create the FlowInstance by implicit cross-service transaction
}

Feature SaveEvaluationDraft {
    input: TicketId + EvaluationData
    requires: report may still evolve as draft
    writes: EvaluationReport(new Point)
    guarantees: preserves draft; does not formalize; does not advance the flow
}

Feature FormalizeEvaluation {
    input: TicketId + EvaluationData
    requires: report complete and admissible; not already formalized
    writes: EvaluationReport(formalized Point)
    guarantees: report immutable after formalization
    doesNot: select next flow state
}

Feature SaveApprovalDraft {
    input: TicketId + ApprovalData
    requires: report may still evolve as draft
    writes: ApprovalReport(new Point)
    guarantees: no flow transition
}

Feature FormalizeApproval {
    input: TicketId + ApprovalData
    requires: report complete and admissible
    writes: ApprovalReport(formalized Point)
    guarantees: no flow transition
}

Feature RecordSupervision {
    input: TicketId + SupervisionData
    requires: supervision report admissibility rules
    writes: SupervisionReport(new Point when permitted)
    guarantees: records supervision information
    doesNot: close the flow itself
}

Feature MaintainClassificationCatalogue {
    // Candidate grouping only: System / Module / IncidentType
    ownership: TicketService
    note: existing creation, editing and activation operations remain in scope
}
```

La división entre `RecordSupervision` y una eventual Feature de **completar supervisión** no está resuelta: la proyección acordada distingue `supervisionCompleted` del valor `closure`, pero aún debemos definir exactamente qué acto formaliza esa completitud.

## 3. Contrato de lectura requerido por el orquestador

El consumidor necesita **solo cinco características**, proyectadas por lectura SQL. No se convierte en una Feature de escritura ni en datos duplicados.

```pseudo
ReadProjection TicketFlowFacts(ticketId) -> TicketFlowFacts {
    evaluationCompleted: Bool
    evaluationScope: Optional<Choice<"Soporte", "Mejora">>
    approvalCompleted: Bool
    supervisionCompleted: Bool
    closureState: Optional<Choice<"Sin cierre", "Cerrada", "Con observaciones">>

    invariant: !evaluationCompleted => evaluationScope == None
    invariant:  evaluationCompleted => evaluationScope is Some
    invariant: !supervisionCompleted => closureState == None
    invariant:  supervisionCompleted => closureState is Some
}
```

**Sin datos adicionales** para decidir el avance: no se proyectan solicitante, responsable, descripción, fechas ni contenido de informes. La elegibilidad de negocio no implica autorización de quien ejecuta el comando.

## 4. Exclusiones y validación

- No recuperar `EditTicketInformation`, `AssignResponsible` ni edición de adjuntos posterior al registro.
- Conservar la capacidad técnica de asociar referencias de Folder en el registro; las compensaciones de fallos deben analizarse aparte.
- No copiar la propiedad legacy de `Stage` ni `TicketStageTransitions` a v2: pertenecen al orquestador.
- Las pruebas de sucesión deberán confrontar el comportamiento vigente con las reglas acordadas, no medir solo build verde.
- Es necesario contrastar la admisibilidad detallada de cada informe con la fuente antes de codificar tipos reales.

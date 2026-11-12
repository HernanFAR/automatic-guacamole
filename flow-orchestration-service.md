# Flow Orchestration Service — máquina TicketSupport V1

**Estado:** pseudocódigo de la versión fija inicial, con FlowSpecification como Space principal; basado en [la definición propietaria](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/flow-orchestration-spaces.md). No se implementa un diseñador de flujos ni un grafo persistido.

## Convención de referencias

Los identificadores de Spaces **internos** son `<Space>Id`. Un elemento propietario de otro servicio se expresa con una referencia externa explícita, por ejemplo `TicketReference`. La instancia no se limita conceptualmente a Tickets: conserva **tipo de elemento y referencia**, y exige compatibilidad con su versión de especificación.

```pseudo
Value FlowSpecificationId
Value FlowSpecificationVersionId
Value ConditionalTransitionId
Value FlowInstanceId
Value ExecutedTransitionId

Value TicketReference                  // Space de Ticket Service v2, externo
Choice ElementType = "Ticket"          // otros tipos se incorporarán cuando existan
Choice FlowVersion = "V1"              // versión constante por ahora
Choice TicketSupportV1State =
    "Evaluación" | "Aprobación" | "Supervisión" | "Cerrado"

Derived FlowSubject {
    elementType: ElementType
    reference: ExternalElementReference
    // V1: elementType="Ticket" y reference: TicketReference
    // La forma general del tipo de referencia aún se examinará con otro consumidor.
}
```

## 1. Especificación y versiones

**FlowSpecification es el Space conceptual principal.** Identifica una máquina de estados reconocible, con **nombre obligatorio de máximo 50 caracteres y único globalmente entre todas las especificaciones de flujo**; una especificación puede tener varias versiones. **FlowInstance es el Space de ejecución más utilizado**, no la definición de la máquina.

```pseudo
Space FlowSpecification {
    id: FlowSpecificationId
    name: Text(min=1, max=50)

    unique: name globally across all FlowSpecifications
    // No two FlowSpecifications may share the same name.
    // Case/whitespace/accent normalization has not yet been agreed.
}

Space FlowSpecificationVersion {
    id: FlowSpecificationVersionId
    specification: FlowSpecificationId
    version: FlowVersion
    compatibleElementTypes: Set<ElementType>
    initialState: FlowState
    states: Set<FlowState>

    unique: (specification, version)
    invariant: specification is not changed
    invariant: a published version retains its original semantics
    transitions: Derived<Set<ConditionalTransitionId>>
}

Space ConditionalTransition {
    id: ConditionalTransitionId
    specificationVersion: FlowSpecificationVersionId
    currentState: FlowState
    nextState: FlowState

    unique: (specificationVersion, currentState, nextState)
    // La misma pareja de estados puede existir en versiones distintas.
    // La condición de habilitación NO se modela como característica todavía.
}
```

**Reglas de transición en código, por ahora.** `ConditionalTransition` identifica una relación admisible entre dos estados. El cálculo que determina si está habilitada vive en **funciones de código específicas de la versión**, no en un `Expression`, `Predicate`, JSON ni catálogo de reglas dentro del Space. Revisaremos si conviene modelar las condiciones como características cuando existan varios casos reales que justifiquen el mecanismo.

**Estado de la realización:** la especificación `TicketSupport`, su versión V1 y las transiciones son **definidas por constantes**. Estos Spaces reconocen identidad y unicidad semánticas, pero **no obligan a crear tablas o Features CRUD** para especificaciones, versiones o transiciones. La gestión dinámica sigue fuera de alcance.

### TicketSupport V1 — valores constantes

```pseudo
const FlowSpecification TicketSupport {
    id: TicketSupportSpecificationId
    name: "TicketSupport"
}

const FlowSpecificationVersion TicketSupportV1 {
    id: TicketSupportVersionV1Id
    specification: TicketSupportSpecificationId
    version: "V1"
    compatibleElementTypes: { "Ticket" }
    initialState: "Evaluación"
    states: { "Evaluación", "Aprobación", "Supervisión", "Cerrado" }
}

const ConditionalTransition TicketEvaluationToApproval {
    id: TicketEvaluationToApprovalId
    specificationVersion: TicketSupportVersionV1Id
    currentState: "Evaluación"
    nextState: "Aprobación"
}

const ConditionalTransition TicketEvaluationToSupervision {
    id: TicketEvaluationToSupervisionId
    specificationVersion: TicketSupportVersionV1Id
    currentState: "Evaluación"
    nextState: "Supervisión"
}

const ConditionalTransition TicketApprovalToSupervision {
    id: TicketApprovalToSupervisionId
    specificationVersion: TicketSupportVersionV1Id
    currentState: "Aprobación"
    nextState: "Supervisión"
}

const ConditionalTransition TicketSupervisionToClosed {
    id: TicketSupervisionToClosedId
    specificationVersion: TicketSupportVersionV1Id
    currentState: "Supervisión"
    nextState: "Cerrado"
}

constant TicketSupportV1.TerminalStates = { "Cerrado" }
```

Los nombres de los identificadores de constantes sirven de notación, no representan valores concretos aprobados.

## 2. Instancia e historia de ejecución

```pseudo
Space FlowInstance {
    id: FlowInstanceId
    specificationVersion: FlowSpecificationVersionId
    subject: FlowSubject
    currentState: FlowState

    requires: subject.elementType in specificationVersion.compatibleElementTypes
    invariant: specificationVersion and subject do not change across Points
    invariant: currentState belongs to specificationVersion.states
    invariant: currentState evolves only through an eligible ConditionalTransition
    invariant: V1 instances continue under V1 rules after new versions appear
}

Space ExecutedTransition {
    id: ExecutedTransitionId
    instance: FlowInstanceId
    transition: ConditionalTransitionId
    from: FlowState
    to: FlowState
    occurredAt: Instant

    invariant: immutable historical fact, not a potential transition
    invariant: transition belongs to the version of the instance
}
```

Los estados representan **situaciones de la máquina**, no necesariamente actividades de trabajo. `"Cerrado"` es el estado terminal de TicketSupport V1. Iniciar una instancia en `"Evaluación"` es distinto de ejecutar una transición entre dos estados existentes.

## 3. Proyección de Ticket propietaria de Flow Orchestration

**Responsabilidad:** Flow Orchestration **define, construye y consume** la proyección `TicketFlowFacts` para evaluar las condiciones de transición. Su propietario de definición y consulta es el orquestador; **los datos fuente siguen perteneciendo a Ticket Service v2**.

```pseudo
Choice EvaluationScope = "Soporte" | "Mejora"
Choice ProjectedClosureState = "Sin cierre" | "Cerrada" | "Con observaciones"

Projection TicketFlowFacts {
    evaluationCompleted: Bool
    evaluationScope: Optional<EvaluationScope>
    approvalCompleted: Bool
    supervisionCompleted: Bool
    closureState: Optional<ProjectedClosureState>

    valid: !evaluationCompleted => evaluationScope == None
    valid:  evaluationCompleted => evaluationScope is Some
    valid: !supervisionCompleted => closureState == None
    valid:  supervisionCompleted => closureState is Some
}

// Propiedad de lectura: Flow Orchestration.
// Fuentes: informes de evaluación, aprobación y supervisión de Ticket Service.
Query ReadTicketFlowFacts(ticket: TicketReference) -> Fin<TicketFlowFacts> {
    source: read-only JOIN/view of Ticket Service-owned tables
    doesNot: write Ticket data or duplicate report persistence
}

derive evaluationCompleted =
    current EvaluationReport is CompletedEvaluationReport

derive approvalCompleted =
    current ApprovalReport is CompletedApprovalReport

derive supervisionCompleted =
    current SupervisionReport is CompletedSupervisionReport

derive evaluationScope = match current EvaluationReport {
    CompletedEvaluationReport completed => Some(completed.scope)
    otherwise => None
}

derive closureState = match current SupervisionReport {
    CompletedSupervisionReport completed
        => Some(ExplicitClosureMapping(completed.closure))
    otherwise => None
}
```

`EvaluationReport`, `ApprovalReport` y `SupervisionReport` son **Spaces externos** para el orquestador; el pseudocódigo de la consulta describe cómo interpretar sus datos persistidos, no incorpora sus Spaces al modelo propietario del flujo. [Las variantes y características están definidas en Ticket Service v2](ticket-service-v2.md#5-informe-de-evaluación--unión-discriminada).

**Límites explícitos:** la derivación de `supervisionCompleted` depende de aprobar el criterio para producir `CompletedSupervisionReport`. El formulario de Ticket Service documenta `"Sin Cierre" | "Cerrado" | "Con observaciones"`, mientras que el contrato proyectado requiere `"Sin cierre" | "Cerrada" | "Con observaciones"`: el mapeo exacto sigue abierto y no debe implementarse silenciosamente. Las garantías de consistencia/concurrencia de lecturas y transiciones también están por definir.

`TicketFlowFacts` es una **proyección calculada de solo lectura**, no un agregado persistido, ni un nuevo Space con identidad independiente. Se mantiene exactamente en cinco características; no se agregan antecedentes del Ticket que no participen en condiciones.

## 4. Condiciones de transición definidas en código

Una `ConditionalTransition` existe como relación constante `(currentState, nextState)`, pero **su existencia no autoriza el avance**. La condición se evalúa como función de código para `TicketSupport/V1`, sobre `TicketFlowFacts` (proyección de lectura definida en este servicio).

```pseudo
function EligibleTransitions(
    version: FlowSpecificationVersionId,
    currentState: FlowState,
    facts: TicketFlowFacts
) -> Set<ConditionalTransitionId> =
    match (version, currentState) {
        (TicketSupportVersionV1Id, "Evaluación") when
            facts.evaluationCompleted
            && facts.evaluationScope == Some("Mejora")
            => { TicketEvaluationToApprovalId }

        (TicketSupportVersionV1Id, "Evaluación") when
            facts.evaluationCompleted
            && facts.evaluationScope == Some("Soporte")
            => { TicketEvaluationToSupervisionId }

        (TicketSupportVersionV1Id, "Aprobación") when facts.approvalCompleted
            => { TicketApprovalToSupervisionId }

        (TicketSupportVersionV1Id, "Supervisión") when
            facts.supervisionCompleted
            && facts.closureState == Some("Cerrada")
            => { TicketSupervisionToClosedId }

        otherwise => {}
    }

// El destino de cada transición se obtiene de su definición constante.
// "Sin cierre" y "Con observaciones" no habilitan rutas nuevas.
// "Cerrado" no tiene transiciones salientes en V1.
```

No se crea un DSL de condiciones, almacenamiento de predicados o un `RuleSpace`. La posibilidad de representar condiciones como características del Space es una **investigación posterior**, basada en varios casos implementados.

## 5. Features candidatas

```pseudo
Feature StartFlow {
    input: FlowSpecificationVersionId + FlowSubject
    requires:
        specificationVersion exists among recognized constant versions
        && subject.elementType in specificationVersion.compatibleElementTypes
        && referenced element exists (verified by appropriate owner)
        && duplicate/uniqueness policy for instances is resolved
    writes: FlowInstance(
        specificationVersion,
        subject,
        currentState=specificationVersion.initialState
    )
    guarantees: instance is bound to the selected version
    note: starting is not a transition between two existing states
}

Feature InspectFlow {
    input: FlowInstanceId
    reads: FlowInstance + ExecutedTransitions
    returns: currentState + history
}

Feature InspectEligibleTransitions {
    input: FlowInstanceId
    reads: FlowInstance + TicketFlowFacts
    returns: EligibleTransitions(specificationVersion, currentState, facts)
    doesNot: grant authority to act
}

Feature AdvanceFlow {
    input: FlowInstanceId + expectedCurrentState + requestedTransitionId
    reads: current FlowInstance + current TicketFlowFacts
    requires:
        requestedTransitionId in EligibleTransitions(specificationVersion, currentState, facts)
        && concurrency checks succeed
        && actor authorization has been verified by its owner
    writes: next immutable FlowInstance Point + ExecutedTransition with requestedTransitionId
    guarantees: either a coherent state change and its fact, or failure
    doesNot: write Ticket, EvaluationReport, ApprovalReport, SupervisionReport
}
```

`InspectEligibleTransitions` puede terminar como consulta de `InspectFlow`; no se fija una división de Features hasta tener necesidad observable.

## 6. Casos mínimos para validar el modelo

| Estado | Proyección relevante | Destinos de las transiciones habilitadas |
| --- | --- | --- |
| Evaluación | Evaluación incompleta | Ninguna |
| Evaluación | Completa, Alcance Soporte | Supervisión |
| Evaluación | Completa, Alcance Mejora | Aprobación |
| Aprobación | Aprobación incompleta | Ninguna |
| Aprobación | Aprobación completa | Supervisión |
| Supervisión | Supervisión incompleta | Ninguna |
| Supervisión | Completa, Sin cierre | Ninguna |
| Supervisión | Completa, Con observaciones | Ninguna |
| Supervisión | Completa, Cerrada | Cerrado |
| Cerrado | Cualquier proyección | Ninguna |

Estos casos prueban la **semántica de elegibilidad**; las pruebas de autorización, coherencia de escritura y concurrencia son adicionales.

# Flow Orchestration Service — máquina TicketSupport V1

**Estado:** pseudocódigo de la versión fija inicial; basado en [la definición propietaria](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/flow-orchestration-spaces.md). No se implementa un diseñador de flujos ni un grafo persistido.

## 1. Selecciones constantes

```pseudo
Choice FlowKind = "TicketSupport"
Choice FlowVersion = "V1"

Choice TicketSupportV1State =
    "Evaluación" | "Aprobación" | "Supervisión" | "Cerrado"

constant TicketSupportV1.InitialState = "Evaluación"
constant TicketSupportV1.TerminalStates = ["Cerrado"]
```

El estado representa la **situación de la máquina**, no un Work Flow organizacional necesariamente activo. `Cerrado` es válido como estado terminal.

## 2. Spaces candidatos

```pseudo
Space FlowInstance {
    id: FlowInstanceId
    subject: Reference<TicketId>       // subject belongs to Ticket Service
    kind: FlowKind                     // "TicketSupport" in V1
    version: FlowVersion               // "V1" in V1
    currentState: TicketSupportV1State

    invariant: subject, kind and version do not change across Points
    invariant: currentState evolves only by an admissible transition
    invariant: all Points are immutable
    invariant: a V1 instance is interpreted by V1 rules, also after V2 exists
}

Space ExecutedTransition {
    id: TransitionId
    instance: Reference<FlowInstanceId>
    from: TicketSupportV1State
    to: TicketSupportV1State
    occurredAt: Instant

    invariant: immutable past fact, not a proposed transition or failed attempt
}
```

La definición `(kind, version) -> transition rules` existe como **conjunto de constantes**. No hay `FlowDefinition` persistido inicialmente; si alguna vez necesitamos V2, debe conservarse la interpretación de las instancias V1.

## 3. Proyección externa para las condiciones

```pseudo
Projection TicketFlowFacts {
    evaluationCompleted: Bool
    evaluationScope: Optional<Choice<"Soporte", "Mejora">>
    approvalCompleted: Bool
    supervisionCompleted: Bool
    closureState: Optional<Choice<"Sin cierre", "Cerrada", "Con observaciones">>

    valid: !evaluationCompleted => evaluationScope == None
    valid:  evaluationCompleted => evaluationScope is Some
    valid: !supervisionCompleted => closureState == None
    valid:  supervisionCompleted => closureState is Some
}
```

No es agregado persistido ni Space propietario de Flow Orchestration. La lectura se obtiene mediante JOIN/vista SQL sobre Ticket Service. La implementación de esa consulta y garantías de concurrencia quedan pendientes.

## 4. Función pura de transiciones admisibles

```pseudo
function EligibleTransitions(
    state: TicketSupportV1State,
    facts: TicketFlowFacts
) -> Set<TicketSupportV1State> =
    match state {
        "Evaluación" when
            facts.evaluationCompleted
            && facts.evaluationScope == Some("Mejora")
            => { "Aprobación" }

        "Evaluación" when
            facts.evaluationCompleted
            && facts.evaluationScope == Some("Soporte")
            => { "Supervisión" }

        "Aprobación" when facts.approvalCompleted
            => { "Supervisión" }

        "Supervisión" when
            facts.supervisionCompleted
            && facts.closureState == Some("Cerrada")
            => { "Cerrado" }

        otherwise => {}
    }

// No implicit routes for "Sin cierre" or "Con observaciones".
// No outgoing transitions from "Cerrado" in V1.
```

La relación de origen y destino **no basta**: cada transición requiere satisfacer su condición sobre la proyección. Un nuevo borrador no cambia el estado del flujo.

## 5. Features candidatas

```pseudo
Feature StartFlow {
    input: TicketId + FlowKind("TicketSupport") + FlowVersion("V1")
    requires: Ticket exists; duplicate/uniqueness policy to define
    writes: FlowInstance(initialState="Evaluación")
    guarantees: instance is bound to V1 rules
    note: start is not a transition between two existing states
}

Feature InspectFlow {
    input: FlowInstanceId
    reads: FlowInstance + ExecutedTransitions
    returns: currentState + history
}

Feature InspectEligibleTransitions {
    input: FlowInstanceId
    reads: FlowInstance + TicketFlowFacts
    returns: EligibleTransitions(currentState, facts)
    doesNot: grant authority to act
}

Feature AdvanceFlow {
    input: FlowInstanceId + expectedCurrentState + requestedDestination
    reads: current FlowInstance + current TicketFlowFacts
    requires:
        requestedDestination in EligibleTransitions(currentState, facts)
        && concurrency checks succeed
        && actor authorization has been verified by its owner
    writes: next immutable FlowInstance Point + ExecutedTransition
    guarantees: either a coherent state change and its fact, or failure
    doesNot: write Ticket, EvaluationReport, ApprovalReport, SupervisionReport
}
```

`InspectEligibleTransitions` puede terminar como consulta de `InspectFlow`; no se fija una división de Features hasta tener necesidad observable.

## 6. Casos mínimos para validar el modelo

| Estado | Proyección relevante | Transiciones habilitadas |
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

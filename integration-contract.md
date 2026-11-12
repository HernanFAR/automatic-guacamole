# Ticket Service v2 ↔ Flow Orchestration — contrato conceptual

**Estado:** fronteras de responsabilidad acordadas; protocolo interservicio **no decidido**. Este documento no define endpoints ni obliga a dos servicios a compartir transacción.

## 1. Frontera de autoridades

| Pregunta | Responsable |
| --- | --- |
| ¿Existe una solicitud válida y qué antecedentes constituyen su identidad? | Ticket Service v2 |
| ¿Puede editarse el Ticket ya registrado? | No; Ticket Service v2 garantiza su inalterabilidad |
| ¿Es válida / está completada la variante de un informe? | Ticket Service v2 |
| ¿En qué estado está una instancia de la máquina? | Flow Orchestration |
| ¿Quién define y construye la proyección de condiciones? | Flow Orchestration: consulta SQL de solo lectura sobre tablas propietarias de Ticket Service v2 |
| ¿Se cumplen las condiciones del próximo estado? | Flow Orchestration, evaluando su propia proyección `TicketFlowFacts` |
| ¿Qué archivos se almacenan? | Folder Service; Ticket Service conserva referencias asociadas al registro |
| ¿Quién puede actuar? | Autoridad de acceso por resolver; la elegibilidad de una transición no concede permiso |
| ¿Quién audita? | Audit Service a futuro; no incluir dependencia falsa hoy |

## 2. Secuencias de negocio (no protocolos implementados)

```pseudo
case RegisterAndStart {
    TicketService.RegisterTicket(registration)
    // After successful registration, process coordination must ensure:
    FlowOrchestration.StartFlow(ticketId, "TicketSupport", "V1")
    // Open: partial failure, retries, uniqueness, orchestration of the two writes.
}

case EvaluateAndAdvance {
    TicketService.CompleteEvaluation(ticketId, draftEvaluation)
    facts = FlowOrchestration.ReadTicketFlowFacts(TicketReference(ticketId))
    // Flow service evaluates the projection and performs an allowed transition.
    FlowOrchestration.AdvanceFlow(instanceId, expectedState, target)
    // Open: ordering, transactional consistency, retries, partial failure.
}

case ApproveAndAdvance {
    TicketService.CompleteApproval(ticketId, draftApproval)
    FlowOrchestration.AdvanceFlow(instanceId, "Aprobación", "Supervisión")
}

case SuperviseAndMaybeClose {
    TicketService.SaveSupervisionDraft(ticketId, draftSupervision)
    // CompleteSupervision only succeeds when closure == "Cerrado"
    // and the required supervision fields are valid.
    TicketService.CompleteSupervision(ticketId, draftSupervision)
    // Only the verified fact of CompletedSupervisionReport whose projected
    // closure is "Cerrada" enables FlowOrchestration.AdvanceFlow(..., "Cerrado").
}
```

**Propiedad de la proyección:** `TicketFlowFacts` se define únicamente en [Flow Orchestration Service](flow-orchestration-service.md#3-proyección-de-ticket-propietaria-de-flow-orchestration); Ticket Service v2 sigue siendo propietario de las tablas y los informes de origen. Esta distribución no crea una nueva API obligatoria entre los servicios.

**Importante:** el pseudocódigo anterior describe orden lógico, **no** una decisión sobre si el Product coordina llamadas, si el orquestador ofrece una actuación compuesta o si existe un mecanismo de reintento. No sugiere escribir datos de Ticket desde Flow Orchestration.

## 3. Decisiones todavía abiertas

1. **Inicio:** quién solicita iniciar la instancia y cómo se resuelve un Ticket creado sin instancia. **Ya decidido:** `TicketSupport/V1` admite una sola `FlowInstance` por Ticket; su política pertenece a `FlowSpecificationVersion`, con comprobación atómica pendiente de realización.
2. **Formalización y avance:** interfaz real de coordinación y tratamiento de fallos parciales.
3. **Concurrencia:** cómo impedir dos avances incompatibles y garantizar que la proyección siga siendo vigente cuando se confirma la transición.
4. **Datos históricos:** migración de `Stage` y `TicketStageTransitions` del Ticket Service actual hacia propiedad del orquestador, con reconstrucción de historia.
5. **Historial de supervisión:** cierre decidido: solo `"Cerrado"` permite `CompletedSupervisionReport`; `"Sin Cierre"` y `"Con observaciones"` permanecen incompletos. Falta decidir cómo conservar los seguimientos sucesivos.
6. **Autoridad:** reglas efectivas de acceso y eventual incorporación de Audit Service.
7. **Contrato de versión:** cómo mantener versiones antiguas, seleccionar la versión inicial para nuevas instancias y definir migraciones explícitas.
8. **Sucesión:** qué pruebas de equivalencia y comportamiento permitirán retirar `ticket-service` en favor de `ticket-service-v2`.

## 4. Prioridad de examen

Primero validar la **admisibilidad de los Spaces** y la función pura de elegibilidad, después concretar **Features y sus resultados**, y finalmente elegir la coordinación interservicio usando casos reales de fallo y concurrencia. La plantilla ofrece estructura de proyectos, algebra y pruebas: **no decide por nosotros la semántica del dominio**.

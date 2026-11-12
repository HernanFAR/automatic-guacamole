# Ticket Service v2 — Spaces, características y Features

**Estado:** pseudocódigo conceptual para desarrollar `ticket-service-v2`, primera ampliación 2026-11-12. Es **autosuficiente como inventario de los conceptos aquí nombrados**, pero **no equivale a C# compilable ni a contratos definitivos de Framework**.

**Fuente de los significados y admisibilidades:** [Spaces — Serviu Tickets](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/spaces.md), [Features de sistema](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/system-features.md) y [Flow Orchestration V1](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/flow-orchestration-spaces.md). Se distingue en cada sección lo **acordado**, lo **observado** y lo **por definir**. No transformar las dudas en restricciones silenciosas.

## 0. Convenciones del pseudocódigo

```pseudo
Choice<"A", "B">         // selección explícita entre valores constantes
Optional<T>             // Some<T> | None: no presume valor por defecto
Reference<T>            // referencia conceptual, no FK ni comprobación automática
Sequence<T>             // colección con restricciones indicadas
Text(max=N)             // longitud máxima acordada si se indica N
Date                   // fecha sin semántica de timezone todavía acordada
Point<T>               // valor inmutable dentro de un Space
PendingRule("...")      // falta una decisión o evidencia
External<T>             // propiedad de otro sistema, no definición de este Service
Derived<T>              // construcción derivada, no necesariamente Space independiente

// Un símbolo nombrado debajo se define aquí, o se declara External con frontera explícita.
// Las firmas de Features indican intención; no son el API de VSlices Framework.
```

**Los Points son inmutables.** Una característica que puede variar solo varía **entre nuevos Points de la misma identidad**.

## 1. Conceptos externos y de apoyo

```pseudo
External Account {
    owner: ServiuAccounts
    note: el solicitante es una referencia a una Cuenta existente;
          la definición y autorización pertenecen a Serviu Cuentas.
}

External HRProfile {
    owner: HumanResources
    note: no equivale a los perfiles organizacionales ni a AssociatedProfileOption.
}

External FolderFile {
    owner: FolderService
    note: bytes, almacenamiento y metadatos propietarios viven allí.
}

External ReferencedBusinessElement {
    owner: sistema al que pertenece el elemento, cuando existe identificación real
    note: Proyecto, Solicitud u otros; NO es necesariamente Ticket ni un Space de Tickets.
}

// Identificadores reconocibles sin imponer UUID, string, ni reglas de generación
Value TicketId
Value SystemId
Value ModuleId
Value IncidenceTypeId
Value FolderFileId
```

## 2. Selecciones de constantes y referencias del Ticket

### 2.1 Perfil asociado

No usamos el nombre `AssociatedProfile` como un tipo indefinido. Distinguimos **la selección de catálogo** de **la característica del Ticket**, que también admite una referencia a RR. HH.

```pseudo
MaintainedSpace AssociatedProfileOption {
    value: Choice<
        "Receptor", "Revisor", "Validador", "Aprobador",
        "Todos los perfiles", "Gerente de Proyecto", "Gerente Operativo",
        "Propietario de Proyecto", "Encargado de Soporte",
        "Administrador", "Otro"
    >
    immutable: true
}

Derived AssociatedProfile {
    variant: AssociatedProfileOption | Reference<External<HRProfile>>
    note: unión discriminada; no confundir con permisos de sesión.
}
```

El catálogo separado se conserva por legibilidad, según el estudio del caso. El campo `associatedProfile` es **propio del Ticket**, mientras que el valor `HRProfile` es una referencia externa. La presencia de un selector no crea permisos.

### 2.2 Elemento referenciado y referencia reconocible

La documentación propietaria **todavía no decidió** si «Elemento referenciado» y «Referencia» son dos Spaces o una sola estructura. Por eso `RecognizableReference` se define aquí como **unión conceptual provisional**, no como Space confirmado.

```pseudo
Derived RecognizableReference {
    variant:
        ManualReference {
            value: Text(max=50)
            verifiedExistence: false  // el texto no acredita la existencia
        }
      | ResolvedReference {
            system: Reference<System>
            element: Reference<External<ReferencedBusinessElement>>
            // Identidad, clase de elemento y permisos dependen del sistema externo
        }

    admission:
        NonIntegratedSystem => ManualReference
        IntegratedSystem    => ManualReference | ResolvedReference
    note: nunca confundir referencia a elemento externo con TicketId.
}
```

La referencia manual debe seguir disponible incluso en sistemas integrados. La capacidad de búsqueda requiere autorización externa. **Las clases concretas de elementos integrados y el mecanismo de resolución no están acordados**. El máximo de 50 caracteres proviene de la característica del Ticket; su aplicabilidad a cada tipo de identificador externo debe contrastarse.

## 3. Catálogos propios de Tickets

### 3.1 Sistema

```pseudo
Choice IntegrationCondition = "Integrated" | "NonIntegrated"

Space System {
    id: SystemId
    integration: IntegrationCondition
    // Además: nombre/identificación visible y actividad se observan en UI,
    // pero las reglas de nombre, identidad y evolución NO están acordadas.
    displayName: PendingRule("denominación, obligatoriedad y longitud")
    activity: PendingRule("vigencia y reglas para consulta/mantenimiento")

    modules: Derived<Sequence<Reference<Module>>>
    incidenceTypes: Derived<Sequence<Reference<IncidenceType>>>
    // No decidir aún si relaciones anteriores se almacenan o se calculan.

    rule: createdThroughMaintainer => integration == "NonIntegrated"
    rule: "Integrated" => incorporatedByServiuTeamByDefault
    note: integración no significa pertenencia del software a Serviu.
}
```

Ejemplos reconocidos como integrados previstos: ServiuPRO IdP, Pagos, PAVES y Refrendación. No asumir que sus integraciones o consultas ya existan.

### 3.2 Módulo de Sistema

```pseudo
Space Module {
    id: ModuleId
    name: Text                           // máximo y unicidad por definir
    system: Reference<System>
    active: PendingRule("actividad observada en selector, no aprobada como invariante")
    evolution: PendingRule("identidad, edición y vigencia")

    invariant: module belongs to its associated System
}
```

### 3.3 Tipo de incidencia

```pseudo
Space IncidenceType {
    id: IncidenceTypeId
    name: Text                           // máximo y unicidad por definir
    system: Reference<System>
    active: PendingRule("actividad observada, reglas por contrastar")
    evolution: PendingRule("identidad, edición y vigencia")
    note: no está acreditada dependencia adicional respecto de Module.
}
```

El formulario muestra módulos y tipos de incidencia según el Sistema seleccionado y ofrece activos. Eso **no prueba por sí solo** qué política de mantenimiento o invariancia debe adoptar el dominio. La referencia del Ticket a Módulo y Tipo de incidencia debe corresponder a su Sistema; las reglas precisas de validación histórica quedan por contrastar.

## 4. Ticket y adjuntos

```pseudo
Choice Risk = "Bajo" | "Medio" | "Alto"

Space Attachment {
    file: Reference<External<FolderFile>>
    name: Text(max=50)                     // incluye extensión
    sizeBytes: Integer(>0)
    immutable: true
}

Space Ticket {
    id: TicketId                            // identidad única
    requester: Reference<External<Account>>
    incidentDate: Date(<= today)
    description: Text(max=500)
    system: Reference<System>
    module: Reference<Module>
    incidenceType: Reference<IncidenceType>
    risk: Risk
    associatedProfile: AssociatedProfile
    associatedElement: RecognizableReference
    attachments: Sequence<Attachment>(min=0, max=5)

    invariant: after RegisterTicket, all characteristics are inalterable.
    invariant: selected Module and IncidenceType belong to selected System.
    DOES_NOT_CONTAIN: FlowState, FlowHistory, EvaluationReport,
                      ApprovalReport, SupervisionReport
}
```

Los archivos se asocian durante el registro. **La modificación posterior de adjuntos, la edición del Ticket y la asignación de responsable no forman parte de las Features del sistema objetivo.** `FolderService` posee bytes y metadatos; Ticket Service posee el significado de la asociación.

El formulario registra también **código de proyecto** y **nombre de proyecto** como contextuales opcionales; su ubicación exacta respecto de `Ticket` y `RecognizableReference` necesita revisión semántica antes de tratarlos como características aprobadas. **No se omiten como desconocidos** ni se declara un nuevo Space `Project` por su sola presencia.

## 5. Informe de evaluación

Una evaluación es un Space propio, con **máximo uno por Ticket** y el `TicketId` como identidad/referencia. `EvaluationData` ahora nombra **una composición de características definida completamente**, no un placeholder ni necesariamente un Space adicional.

```pseudo
Choice EvaluationFormalization = "Borrador" | "Formalizado"
Choice SupportType = "Interno" | "Externo" | "Interno/externo"
Choice SupportLevel =
    "Nivel 1 (SLA 3 hrs)" | "Nivel 2 (SLA 8 hrs)" | "Nivel 3 (SLA +1 día)"
Choice EvaluationScope = "Soporte" | "Mejora"
Choice IncidentSolution =
    "Sí" | "No" | "En Proceso" | "Ambiente Testing" | "Derivar a Gerente Operativo"

Derived EvaluationData {
    supportType: Optional<SupportType>
    supportLevel: Optional<SupportLevel>
    supportEntryDate: Optional<Date(<= today)>
    scope: Optional<EvaluationScope>
    actionDescription: Optional<Text(max=500)>
    proposedSolution: Optional<Text(max=1000)>
    expectedSolutionTime: Optional<Text(max=100)>
    incidentSolution: Optional<IncidentSolution>
    activityStartDate: Optional<Date>
    activityEndDate: Optional<Date>

    admission:
        if both activity dates present => activityStartDate <= activityEndDate
        // Borrador admite información parcial.
        // Formalizar requiere las diez características, con admisibilidad válida.
}

Space EvaluationReport {
    id: TicketId
    data: EvaluationData
    formalization: EvaluationFormalization
    cardinality: 0..1 per Ticket

    evolution:
        Borrador -> Borrador   // conserva borradores como nuevos Points
        Borrador -> Formalizado
        Formalizado -> no further evolution
}
```

**Dudas preservadas:** finalidad de otra fecha de ingreso a soporte; autoridad y procedencia de la descripción de acción y propuesta de solución; pertinencia de «¿Se solucionó el incidente?» dentro de evaluación en vez de supervisión. Los límites 500/1000/100 están **aprobados para el modelo**, aunque no se ha verificado su implementación anterior. El nombre del SLA no crea por sí solo garantías de tiempo.

## 6. Informe de aprobación operacional

`ApprovalData` conserva las **cinco características** del formulario; la formalización requiere tres.

```pseudo
Choice ApprovalSatisfaction = "Sí" | "No" | "En Proceso" | "Otro"
Choice ApprovalIndication =
    "Iniciar mejora" | "Iniciar soporte" |
    "Mejora en evaluación" | "Soporte en evaluación" | "Otro"
Choice ApprovalFormalization = "Borrador" | "Formalizado"

Derived ApprovalData {
    satisfaction: Optional<ApprovalSatisfaction>          // requerido al formalizar
    observationDate: Optional<Date>                       // opcional
    observation: Optional<Text>                           // max por definir
    indication: Optional<ApprovalIndication>              // requerido al formalizar
    indicationDetail: Optional<Text(nonEmpty=true)>       // requerido al formalizar; max por definir
}

Space ApprovalReport {
    id: TicketId
    data: ApprovalData
    formalization: ApprovalFormalization
    cardinality: 0..1 per Ticket
    context: route "Mejora"

    evolution:
        Borrador -> Borrador
        Borrador -> Formalizado
        Formalizado -> no further evolution

    formalizationRequires:
        satisfaction is Some
        && indication is Some
        && indicationDetail is Some(nonEmpty)
}
```

Las opciones `"No"` o `"En Proceso"` **no demuestran** rutas de rechazo, devolución o reapertura. Falta acordar qué se aprueba y qué significa aprobación organizacional válida. La regla de avance del flujo V1 actualmente observa solo si la aprobación está formalizada.

## 7. Informe de supervisión

`SupervisionData` define las **tres características** observadas, con evolución mientras el informe no haya alcanzado cierre. La noción `supervisionCompleted` requerida por el orquestador **no está definida todavía como característica del informe**; debe precisarse cómo se deriva.

```pseudo
// Valores observados literalmente en el formulario documentado:
Choice ReportClosureState =
    "Sin Cierre" | "Cerrado" | "Con observaciones"

Choice OutOfSystemPaymentRegularization =
    "No procede" |
    "Actualizado en Plataforma" |
    "Pendiente por actualizar en Plataforma"

Derived SupervisionData {
    closure: ReportClosureState
    paymentRegularization: OutOfSystemPaymentRegularization
    monitoringObservations: Optional<Text(nonEmpty=true)>  // max por definir
}

Space SupervisionReport {
    id: TicketId
    data: SupervisionData
    cardinality: 0..1 per Ticket

    evolution: may produce new Points while closure != "Cerrado"
    invariant: after closure == "Cerrado", no further evolution
    unanswered: repeated monitoring within one report; valid closure criteria
}

Derived SupervisionCompletion {
    value: Bool
    rule: PendingRule("qué hecho significa Supervisión completada")
}
```

**Diferencia de vocabulario real:** el formulario / modelo de informe documenta `"Sin Cierre" | "Cerrado" | "Con observaciones"`, mientras que la proyección de Flow Orchestration acordada utiliza `"Sin cierre" | "Cerrada" | "Con observaciones"`. **No se igualan silenciosamente**: el contrato de lectura deberá definir un mapeo explícito y verificable, en particular `"Cerrado" (informe) → "Cerrada" (proyección)`.

## 8. Proyección de solo lectura para Flow Orchestration

No es un agregado persistente de Ticket Service ni un Space que Flow Orchestration pueda escribir.

```pseudo
Choice ProjectedClosureState = "Sin cierre" | "Cerrada" | "Con observaciones"

Projection TicketFlowFacts {
    evaluationCompleted: Bool
    evaluationScope: Optional<EvaluationScope>
    approvalCompleted: Bool
    supervisionCompleted: Bool
    closureState: Optional<ProjectedClosureState>

    invariant: !evaluationCompleted => evaluationScope == None
    invariant:  evaluationCompleted => evaluationScope is Some
    invariant: !supervisionCompleted => closureState == None
    invariant:  supervisionCompleted => closureState is Some
}

derive evaluationCompleted = EvaluationReport.formalization == "Formalizado"
derive approvalCompleted   = ApprovalReport.formalization == "Formalizado"
derive supervisionCompleted = PendingRule("completitud de supervisión")
derive evaluationScope = evaluationCompleted
    ? Some(EvaluationReport.data.scope) : None
derive closureState = supervisionCompleted
    ? Some(ExplicitClosureMapping(SupervisionReport.data.closure)) : None
```

Los cinco campos y las reglas de presencia **sí están acordados**. La derivación exacta de supervisión y el mapeo de constantes del cierre **no**. No inventar una bandera persistida nueva para satisfacer el pseudocódigo.

## 9. Features candidatas — Ticket Service v2

```pseudo
Feature RegisterTicket {
    input: RegistrationData<Ticket> + Optional<InitialAttachments>
    requires: admissible Ticket and mandatory registration data
    writes: Ticket
    returns: TicketId
    guarantees: Ticket registered and inalterable
    doesNot: start FlowInstance by an assumed distributed transaction
}

Feature SaveEvaluationDraft {
    input: TicketId + EvaluationData
    requires: report is new or is still Borrador
    writes: EvaluationReport(new Point)
    guarantees: no flow transition
}

Feature FormalizeEvaluation {
    input: TicketId + EvaluationData
    requires: ten required characteristics valid; report not yet Formalizado
    writes: EvaluationReport(formalized Point)
    guarantees: subsequent edits rejected
    doesNot: select the flow destination
}

Feature SaveApprovalDraft {
    input: TicketId + ApprovalData
    requires: report is new or Borrador
    writes: ApprovalReport(new Point)
    guarantees: no flow transition
}

Feature FormalizeApproval {
    input: TicketId + ApprovalData
    requires: satisfaction, indication, indicationDetail present and valid
    writes: ApprovalReport(formalized Point)
    doesNot: transition the machine
}

Feature RecordSupervision {
    input: TicketId + SupervisionData
    requires: valid report; evolution still allowed
    writes: SupervisionReport(new Point)
    doesNot: close the machine
}

Feature MaintainSystem {
    operations: create, edit, activate/deactivate // actual intent to contrast
    owner: TicketService
}
Feature MaintainModule {
    operations: create, edit, activate/deactivate // actual intent to contrast
    owner: TicketService
}
Feature MaintainIncidenceType {
    operations: create, edit, activate/deactivate // actual intent to contrast
    owner: TicketService
}
```

Los mantenedores se describen como candidatos funcionales; **no se interpretan las operaciones agrupadas como firmas aprobadas de una Feature**. Las reglas organizacionales de creación, modificación y vigencia todavía deben profundizarse.

## 10. Exclusiones y asuntos para siguiente revisión

1. No trasladar `Stage` ni `TicketStageTransitions` del legacy a Ticket Service v2: propiedad de Flow Orchestration.
2. No recuperar edición del Ticket, asignación responsable ni gestión posterior de adjuntos como Feature.
3. **Completitud de supervisión:** determinar cuándo `SupervisionCompletion` es verdadero; no confundir guardado con completitud.
4. **Normalización de cierre:** reconciliar etiquetas de formulario e interfaz de proyección mediante decisión expresa.
5. **Referencias a elementos externos:** precisar identidades, tipos y permisos sin asumir que los sistemas externos están integrados.
6. **Catálogos:** verificar identidad, vigencia y cambios admisibles contra los mantenedores reales; no inventar reglas de unicidad.
7. **Datos contextuales de proyecto:** precisar ubicación de código y nombre de proyecto en el modelo.
8. **Compatibilidad:** definir pruebas de equivalencia semántica para reemplazar `ticket-service` por `ticket-service-v2`.
9. **Coordinación:** formalización de informes y avance de Flow Orchestration siguen siendo escrituras separadas cuyo protocolo no está decidido.

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

**Acuerdo 2026-11-12:** los tres nombres tienen máximo 50 caracteres y unicidad contextual. Los cambios de nombre y actividad admiten nuevos Points de la misma identidad. La unidad de unicidad de `System` es todo su catálogo; la de `Module` y `IncidenceType` es el `System` asociado.

### 3.1 Características compartidas

```pseudo
Choice Activity = "Activo" | "Desactivado"
Choice IntegrationCondition = "Integrated" | "NonIntegrated"

// No crea otro Space. Es una característica de cada catálogo.
// Todo Point es inmutable: editar / cambiar actividad produce otro Point.
Value NormalizedName {
    presented: Text(nonEmpty=true, max=50)
    semanticKey: NormalizeName(presented)

    // Regla de equivalencia acordada:
    // "Pagos" == "pagos " == "PAGOs"
    // NormalizeName ignora mayúsculas/minúsculas y espacios exteriores.
    // Una comparación para unicidad o búsqueda utiliza semanticKey.
    invariant: NormalizeName(presented) is not empty
    invariant: length(TrimOuterWhitespace(presented)) <= 50
    note: preserve a readable presentation; do not require uppercase display.
}
Alias CatalogueName = NormalizedName

// La unicidad se comprueba por clave semántica y alcance, sobre TODOS
// los registros, incluidos los Desactivados:
Unique(System.name.semanticKey)
Unique(Module.system, Module.name.semanticKey)
Unique(IncidenceType.system, IncidenceType.name.semanticKey)

// El nombre de un elemento Desactivado continúa reservado.
// La comparación exacta de acentos, Unicode y espacios interiores sigue abierta.
// NormalizedName aplica también a otras características que se usen para
// identidad o búsqueda por nombre: no es una restricción de cada Text descriptivo.
```

**Nombre normalizado:** concepto de valor reutilizable para toda característica que funcione como nombre identificativo, clave de unicidad o criterio de búsqueda por nombre. Las Features no comparan texto bruto; comparan su equivalencia semántica normalizada. La normalización especificada hasta ahora elimina diferencias por capitalización y espacios exteriores. No presupone eliminar acentos ni modificar espacios interiores. Los nombres de elementos desactivados **siguen reservados** en su ámbito. El nombre visible y la clave semántica son dos representaciones del mismo valor, no dos Spaces de negocio.

**Actividad:** característica evolucionable que identifica si un elemento está activo o desactivado, sin cambiar su identidad. Se expresa mediante constantes, sin un nuevo Space. Queda por definir solo la actividad inicial de creación.

### 3.2 System

```pseudo
Space System {
    id: SystemId
    name: CatalogueName                // clave normalizada única, incluso desactivados
    integration: IntegrationCondition
    activity: Activity                 // nuevos Points para cambios admitidos

    modules: Derived<Sequence<Reference<Module>>>
    incidenceTypes: Derived<Sequence<Reference<IncidenceType>>>
    // Por decidir si ambas relaciones se almacenan o se consultan.

    invariant: id does not vary within the same System
    invariant: integration does not change through rename/activate/deactivate
    invariant: name.semanticKey is unique in the full System catalogue, including inactive

    rule: createdThroughMaintainer => integration == "NonIntegrated"
    rule: integration == "Integrated" => provisionedByServiuTeam
    initiallyIntegrated: ["ServiuPro - Soporte"]
}
```

**Único System integrado inicial:** `"ServiuPro - Soporte"`. Los anteriores ejemplos de IdP, Pagos, PAVES y Refrendación **no** se incorporan como integrados en esta versión.

### 3.3 Module

```pseudo
Space Module {
    id: ModuleId
    name: CatalogueName
    system: Reference<System>
    activity: Activity

    invariant: id and system do not change across Points
    invariant: name.semanticKey unique within referenced System, including inactive
    // Módulos de Systems distintos sí pueden compartir nombre.
}
```

**Crear Module** requiere nombre y referencia a un System relacionado. Queda pendiente confirmar si la creación exige que ese System esté activo; no introducir una regla implícita.

### 3.4 IncidenceType

```pseudo
Space IncidenceType {
    id: IncidenceTypeId
    name: CatalogueName
    system: Reference<System>
    activity: Activity

    invariant: id and system do not change across Points
    invariant: name unique within referenced System
    // IncidenceTypes de Systems distintos pueden compartir nombre.
    // No se declara relación obligatoria con Module.
}
```

### 3.5 Features de mantenimiento: reglas por catálogo

```pseudo
Feature CreateSystem {
    input: name: CatalogueName
    requires: no System with same normalized name, including inactive
    writes: System(name, integration="NonIntegrated", activity=PendingInitialActivity)
    // Los Systems Integrated no se crean mediante este mantenedor.
}

Feature EditSystem {
    input: SystemId + newName: CatalogueName
    requires: System exists by ID
              && System.activity == "Activo"
              && newName normalized unique in System catalogue, including inactive
    writes: new Point with same ID, IntegrationCondition and Activity; new Name only
}

Feature CreateModule {
    input: system: Reference<System> + name: CatalogueName
    requires: related System exists
              && normalized name unique among all Module of that System, including inactive
    writes: Module(system, name, activity=PendingInitialActivity)
}
Feature EditModule {
    input: ModuleId + newName: CatalogueName
    requires: Module exists by ID
              && Module.activity == "Activo"
              && newName normalized unique among all Modules of same System, including inactive
    writes: new Point with same ID, same System, same Activity; new Name only
}

Feature CreateIncidenceType {
    input: system: Reference<System> + name: CatalogueName
    requires: related System exists
              && normalized name unique among all IncidenceType of that System, including inactive
    writes: IncidenceType(system, name, activity=PendingInitialActivity)
}
Feature EditIncidenceType {
    input: IncidenceTypeId + newName: CatalogueName
    requires: IncidenceType exists by ID
              && IncidenceType.activity == "Activo"
              && newName normalized unique among all IncidenceType of same System, including inactive
    writes: new Point with same ID, same System, same Activity; new Name only
}
```

**Para cada catálogo (`System`, `Module`, `IncidenceType`)** existen dos Features de cambio de actividad. Se expresan paramétricamente para no repetir reglas idénticas sin confundir su propiedad:

```pseudo
Feature Activate<Catalogue> {
    input: Id<Catalogue>
    requires: exists by ID && current.activity == "Desactivado"
    rejects: missing ID; current.activity == "Activo" // ya está activo
    writes: new Point(activity="Activo"), same identity and other characteristics
}

Feature Deactivate<Catalogue> {
    input: Id<Catalogue>
    requires: exists by ID && current.activity == "Activo"
    rejects: missing ID; current.activity == "Desactivado"
    writes: new Point(activity="Desactivado"), same identity and other characteristics
}

ActivityTransitionRules {
    "Desactivado" -> "Activo"       // Activate: permitido
    "Activo" -> "Activo"           // Activate: error, ya activo
    "Activo" -> "Desactivado"       // Deactivate: permitido
    "Desactivado" -> "Desactivado" // Deactivate: error, ya desactivado
}
```

**Reglas de error acordadas:** editar y desactivar requieren identidad existente y actividad `"Activo"`; activar requiere identidad existente y actividad `"Desactivado"`. ID inexistente siempre produce error. Activar algo ya activo y desactivar algo ya desactivado producen error. La errata anterior sobre Activar quedó expresamente corregida por Hernán.

**Pendientes que sí importan:** estado inicial de actividad al crear; equivalencia de acentos, normalización Unicode y espacios interiores; posibilidad de activar/desactivar módulos o incidencias cuando su System esté desactivado; precondición de actividad del System al crear Module/IncidenceType. Ninguno se deduce de la UI.

### 3.6 Ejemplos de equivalencia para pruebas

```pseudo
NormalizeName("Pagos") == NormalizeName("pagos ")
NormalizeName("Pagos") == NormalizeName("PAGOs")
CreateSystem("Pagos"); CreateSystem("pagos ") => Error(DuplicateName)
CreateSystem("Pagos"); DeactivateSystem(id); CreateSystem("PAGOs") => Error(DuplicateName)

CreateModule(systemA, "Pagos"); CreateModule(systemB, "pagos ") => allowed
CreateModule(systemA, "Pagos"); CreateModule(systemA, "PAGOs") => Error(DuplicateName)
CreateIncidenceType(systemA, "Pagos"); CreateIncidenceType(systemA, "pagos ") => Error(DuplicateName)

ActivateSystem(activeId) => Error(AlreadyActive)
DeactivateSystem(inactiveId) => Error(AlreadyInactive)
ActivateSystem(inactiveId) => Point(activity="Activo")
DeactivateSystem(activeId) => Point(activity="Desactivado")
```

Estos ejemplos documentan reglas semánticas, no firmas ni códigos de error implementados.

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

// Las Features de catálogos se definen con contratos y errores en §3.5.
// Create/Edit son específicas de System, Module e IncidenceType.
// Activate/Deactivate comparten el esquema, pero no su identidad funcional.
```

Los mantenedores se describen como candidatos funcionales; **no se interpretan las operaciones agrupadas como firmas aprobadas de una Feature**. Las reglas organizacionales de creación, modificación y vigencia todavía deben profundizarse.

## 10. Exclusiones y asuntos para siguiente revisión

1. No trasladar `Stage` ni `TicketStageTransitions` del legacy a Ticket Service v2: propiedad de Flow Orchestration.
2. No recuperar edición del Ticket, asignación responsable ni gestión posterior de adjuntos como Feature.
3. **Completitud de supervisión:** determinar cuándo `SupervisionCompletion` es verdadero; no confundir guardado con completitud.
4. **Normalización de cierre:** reconciliar etiquetas de formulario e interfaz de proyección mediante decisión expresa.
5. **Referencias a elementos externos:** precisar identidades, tipos y permisos sin asumir que los sistemas externos están integrados.
6. **Catálogos:** nombre normalizado, reserva de nombres desactivados y reglas de Activar/Desactivar acordadas en §3; confirmar actividad inicial y detalles de normalización aún abiertos.
7. **Datos contextuales de proyecto:** precisar ubicación de código y nombre de proyecto en el modelo.
8. **Compatibilidad:** definir pruebas de equivalencia semántica para reemplazar `ticket-service` por `ticket-service-v2`.
9. **Coordinación:** formalización de informes y avance de Flow Orchestration siguen siendo escrituras separadas cuyo protocolo no está decidido.

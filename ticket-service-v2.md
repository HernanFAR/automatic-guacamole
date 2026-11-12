# Ticket Service v2 — Spaces, características y Features

**Estado:** pseudocódigo conceptual para desarrollar `ticket-service-v2`, primera ampliación 2026-11-12. Es **autosuficiente como inventario de los conceptos aquí nombrados**, pero **no equivale a C# compilable ni a contratos definitivos de Framework**.

**Fuente de los significados y admisibilidades:** [Spaces — Serviu Tickets](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/spaces.md), [Features de sistema](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/system-features.md) y [Flow Orchestration V1](https://github.com/atom-dev-serviu/documentation/blob/main/systems/serviu-tickets/flow-orchestration-spaces.md). Se distingue en cada sección lo **acordado**, lo **observado** y lo **por definir**. No transformar las dudas en restricciones silenciosas.

## 0. Convenciones del pseudocódigo

```pseudo
Choice<"A", "B">         // selección explícita entre valores constantes
Optional<T>             // Some<T> | None: no presume valor por defecto
Internal Space => <Space>Id      // solo identificador: SystemId, ModuleId, ...
External Space => <Space>Reference  // referencia nombrada: AccountReference, ...
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
Value FolderFileReference       // referencia a Space externo (Folder Service)
Value AccountReference          // referencia a Space externo (Serviu Cuentas)
Value HRProfileReference        // referencia a Space externo (Recursos Humanos)
Value ReferencedBusinessElementReference // referencia a concepto de otro sistema
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
    variant: AssociatedProfileOption | HRProfileReference
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
            system: SystemId
            element: ReferencedBusinessElementReference
            // Identidad, clase de elemento y permisos dependen del sistema externo
        }

    admission:
        NonIntegratedSystem => ManualReference
        IntegratedSystem    => ManualReference | ResolvedReference
    note: nunca confundir referencia a elemento externo con TicketId.
}
```

Los campos que apuntan a Spaces propios de Ticket Service usan sus identificadores (`SystemId`, `ModuleId`, `IncidenceTypeId`). Las referencias a Spaces o elementos de otros servicios se nombran `<Concepto>Reference`; no se declara `Reference<A>` genérico. La referencia manual debe seguir disponible incluso en sistemas integrados. La capacidad de búsqueda requiere autorización externa. **Las clases concretas de elementos integrados y el mecanismo de resolución no están acordados**. El máximo de 50 caracteres proviene de la característica del Ticket; su aplicabilidad a cada tipo de identificador externo debe contrastarse.

## 3. Catálogos propios de Tickets

**Acuerdo 2026-11-12:** los tres nombres tienen máximo 50 caracteres y unicidad contextual. Los cambios de nombre y actividad admiten nuevos Points de la misma identidad. La unidad de unicidad de `System` es todo su catálogo; la de `Module` y `IncidenceType` es el `System` asociado.

### 3.1 Características compartidas

```pseudo
Choice ActivationStatus = "Active" | "Inactive"
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

**ActivationStatus:** característica evolucionable que identifica si un elemento está activo o desactivado, sin cambiar su identidad. Se expresa mediante constantes, sin un nuevo Space. Toda creación mediante las Features de mantenimiento comienza con `activationStatus = "Active"`.

### 3.2 System

```pseudo
Space System {
    id: SystemId
    name: CatalogueName                // clave normalizada única, incluso desactivados
    integration: IntegrationCondition
    activationStatus: ActivationStatus                 // nuevos Points para cambios admitidos

    modules: Derived<Sequence<ModuleId>>
    incidenceTypes: Derived<Sequence<IncidenceTypeId>>
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
    system: SystemId
    activationStatus: ActivationStatus

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
    system: SystemId
    activationStatus: ActivationStatus

    invariant: id and system do not change across Points
    invariant: name unique within referenced System
    // IncidenceTypes de Systems distintos pueden compartir nombre.
    // No se declara relación obligatoria con Module.
}
```

### 3.5 Features de mantenimiento: reglas por catálogo

**Regla acordada:** `CreateSystem`, `CreateModule` y `CreateIncidenceType` siempre generan inicialmente un Point con `ActivationStatus = "Active"`.

```pseudo
Feature CreateSystem {
    input: name: CatalogueName
    requires: no System with same normalized name, including inactive
    writes: System(name, integration="NonIntegrated", activationStatus="Active")
    // Los Systems Integrated no se crean mediante este mantenedor.
}

Feature EditSystem {
    input: SystemId + newName: CatalogueName
    requires: System exists by ID
              && System.activationStatus == "Active"
              && newName normalized unique in System catalogue, including inactive
    writes: new Point with same ID, IntegrationCondition and ActivationStatus; new Name only
}

Feature CreateModule {
    input: system: SystemId + name: CatalogueName
    requires: related System exists
              && normalized name unique among all Module of that System, including inactive
    writes: Module(system, name, activationStatus="Active")
}
Feature EditModule {
    input: ModuleId + newName: CatalogueName
    requires: Module exists by ID
              && Module.activationStatus == "Active"
              && newName normalized unique among all Modules of same System, including inactive
    writes: new Point with same ID, same System, same ActivationStatus; new Name only
}

Feature CreateIncidenceType {
    input: system: SystemId + name: CatalogueName
    requires: related System exists
              && normalized name unique among all IncidenceType of that System, including inactive
    writes: IncidenceType(system, name, activationStatus="Active")
}
Feature EditIncidenceType {
    input: IncidenceTypeId + newName: CatalogueName
    requires: IncidenceType exists by ID
              && IncidenceType.activationStatus == "Active"
              && newName normalized unique among all IncidenceType of same System, including inactive
    writes: new Point with same ID, same System, same ActivationStatus; new Name only
}
```

**Para cada catálogo (`System`, `Module`, `IncidenceType`)** existen dos Features de cambio de actividad. Se expresan paramétricamente para no repetir reglas idénticas sin confundir su propiedad:

```pseudo
Feature Activate<Catalogue> {
    input: Id<Catalogue>
    requires: exists by ID && current.activationStatus == "Inactive"
    rejects: missing ID; current.activationStatus == "Active" // ya está activo
    writes: new Point(activationStatus="Active"), same identity and other characteristics
}

Feature Deactivate<Catalogue> {
    input: Id<Catalogue>
    requires: exists by ID && current.activationStatus == "Active"
    rejects: missing ID; current.activationStatus == "Inactive"
    writes: new Point(activationStatus="Inactive"), same identity and other characteristics
}

ActivationStatusTransitionRules {
    "Inactive" -> "Active"       // Activate: permitido
    "Active" -> "Active"           // Activate: error, ya activo
    "Active" -> "Inactive"       // Deactivate: permitido
    "Inactive" -> "Inactive" // Deactivate: error, ya desactivado
}
```

**Reglas de error acordadas:** editar y desactivar requieren identidad existente y actividad `"Activo"`; activar requiere identidad existente y actividad `"Desactivado"`. ID inexistente siempre produce error. Activar algo ya activo y desactivar algo ya desactivado producen error. La errata anterior sobre Activar quedó expresamente corregida por Hernán.

**Pendientes que sí importan:** equivalencia de acentos, normalización Unicode y espacios interiores; posibilidad de activar/desactivar módulos o incidencias cuando su System esté desactivado; precondición de estado de activación del System al crear Module/IncidenceType. Ninguno se deduce de la UI.

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
ActivateSystem(inactiveId) => Point(activationStatus="Active")
DeactivateSystem(activeId) => Point(activationStatus="Inactive")
```

Estos ejemplos documentan reglas semánticas, no firmas ni códigos de error implementados.

## 4. Ticket y adjuntos

```pseudo
Choice Risk = "Bajo" | "Medio" | "Alto"

Space Attachment {
    file: FolderFileReference
    name: Text(max=50)                     // incluye extensión
    sizeBytes: Integer(>0)
    immutable: true
}

Space Ticket {
    id: TicketId                            // identidad única
    requester: AccountReference
    incidentDate: Date(<= today)
    description: Text(max=500)
    system: SystemId
    module: ModuleId
    incidenceType: IncidenceTypeId
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

## 5. Informe de evaluación — unión discriminada

**Acuerdo de modelado:** `EvaluationReport` es el Space base con identidad `TicketId`. Sus únicas variantes son `DraftEvaluationReport` y `CompletedEvaluationReport` (herencia de Spaces como notación conceptual). La variante determina la completitud: no se conserva `formalization` como campo redundante.

```pseudo
Choice SupportType = "Interno" | "Externo" | "Interno/externo"
Choice SupportLevel =
    "Nivel 1 (SLA 3 hrs)" | "Nivel 2 (SLA 8 hrs)" | "Nivel 3 (SLA +1 día)"
Choice EvaluationScope = "Soporte" | "Mejora"
Choice IncidentSolution =
    "Sí" | "No" | "En Proceso" | "Ambiente Testing" | "Derivar a Gerente Operativo"

Space EvaluationReport {
    id: TicketId                          // identidad y referencia interna compartida
    cardinality: 0..1 per Ticket          // Draft y Completed NO coexisten
}

Space DraftEvaluationReport extends EvaluationReport {
    supportType: Optional<SupportType>
    supportLevel: Optional<SupportLevel>
    supportEntryDate: Optional<Date(<= today)>
    scope: Optional<EvaluationScope>
    actionDescription: Text(min=0, max=500)
    proposedSolution: Text(min=0, max=1000)
    expectedSolutionTime: Text(min=0, max=100)
    incidentSolution: Optional<IncidentSolution>
    activityStartDate: Optional<Date>
    activityEndDate: Optional<Date>

    admission: if both dates exist => activityStartDate <= activityEndDate
    evolution: Draft -> Draft (new immutable Point)
}

Space CompletedEvaluationReport extends EvaluationReport {
    supportType: SupportType
    supportLevel: SupportLevel
    supportEntryDate: Date(<= today)
    scope: EvaluationScope
    actionDescription: Text(min=1, max=500)
    proposedSolution: Text(min=1, max=1000)
    expectedSolutionTime: Text(min=1, max=100)
    incidentSolution: IncidentSolution
    activityStartDate: Date
    activityEndDate: Date

    admission: activityStartDate <= activityEndDate
    evolution: none // Point definitivo: no edición posterior
}

Transform CompleteEvaluation(draft: DraftEvaluationReport)
    -> Fin<CompletedEvaluationReport> {
    requires: all ten fields present, semantically valid, and dates ordered
    guarantees: preserves same TicketId; consumes the logical Draft variant
}
```

Las selecciones y fechas opcionales del borrador, junto con los textos `min=0`, permiten capturas parciales; la variante completada exige las diez características. Los **Points de ambas variantes son siempre inmutables**: que el borrador sea evolucionable no significa que sus propiedades se muten.

**Preguntas preservadas:** justificación de fecha de ingreso, procedencia de acción/propuesta y pertinencia de «¿Se solucionó el incidente?» en evaluación. Los máximos 500/1.000/100 están acordados para el modelo, no verificados en el servicio legado.

## 6. Informe de aprobación — unión discriminada

Un único informe lógico por Ticket, ahora expresado como **`ApprovalReport = DraftApprovalReport | CompletedApprovalReport`**. Tres características son obligatorias al completar; fecha y observación **siguen siendo legítimamente opcionales**, incluso en la variante completada. Para textos sin contenido, se utiliza `Text(min=0)` en vez de `Optional<Text>`.

```pseudo
Choice ApprovalSatisfaction = "Sí" | "No" | "En Proceso" | "Otro"
Choice ApprovalIndication =
    "Iniciar mejora" | "Iniciar soporte" |
    "Mejora en evaluación" | "Soporte en evaluación" | "Otro"

Space ApprovalReport {
    id: TicketId
    cardinality: 0..1 per Ticket
    context: route "Mejora"
}

Space DraftApprovalReport extends ApprovalReport {
    satisfaction: Optional<ApprovalSatisfaction>
    observationDate: Optional<Date>
    observation: Text(min=0, max=1000)
    indication: Optional<ApprovalIndication>
    indicationDetail: Text(min=0, max=1000)

    evolution: Draft -> Draft (new immutable Point)
}

Space CompletedApprovalReport extends ApprovalReport {
    satisfaction: ApprovalSatisfaction
    observationDate: Optional<Date>               // sigue siendo opcional
    observation: Text(min=0, max=1000)          // sigue siendo opcional
    indication: ApprovalIndication
    indicationDetail: Text(min=1, max=1000)

    evolution: none
}

Transform CompleteApproval(draft: DraftApprovalReport)
    -> Fin<CompletedApprovalReport> {
    requires: satisfaction, indication and indicationDetail present and valid
    guarantees: preserves same TicketId; completed report immutable
}
```

**Límites acordados (2026-11-12):** `observation` tiene máximo 1.000 caracteres; `indicationDetail` tiene máximo 1.000. La variante completada exige contenido significativo en `indicationDetail` (los espacios en blanco por sí solos no cuentan).

**No inferir** rutas de rechazo o devolución de las opciones `"No"` o `"En Proceso"`. Falta contrastar el significado organizacional de una aprobación válida; la condición de Flow Orchestration V1 observa únicamente que exista la variante Completed.

## 7. Informe de supervisión — unión discriminada candidata

Aplicamos también el patrón **`SupervisionReport = DraftSupervisionReport | CompletedSupervisionReport`**. Conservamos los tres campos del modelo observado; las observaciones son **opcionales en ambas variantes** y se representan mediante `Text(min=0, max=2000)`.

**Límite importante:** que un informe posea estado de cierre `"Cerrado"` no es lo mismo que declarar la variante `Completed`. La nueva distinción de completitud está propuesta para nuestro diseño; **aún debe contrastarse con los seguimientos sucesivos del legacy** y con el acto organizacional de completar supervisión. No convertir todo guardado en una formalización automática.

```pseudo
// Selecciones del informe según formulario documentado
Choice ReportClosureState = "Sin Cierre" | "Cerrado" | "Con observaciones"
Choice OutOfSystemPaymentRegularization =
    "No procede" |
    "Actualizado en Plataforma" |
    "Pendiente por actualizar en Plataforma"

Space SupervisionReport {
    id: TicketId
    cardinality: 0..1 per Ticket
}

Space DraftSupervisionReport extends SupervisionReport {
    closure: Optional<ReportClosureState>
    paymentRegularization: Optional<OutOfSystemPaymentRegularization>
    monitoringObservations: Text(min=0, max=2000)

    evolution: Draft -> Draft (new immutable Point; repeated monitoring to examine)
}

Space CompletedSupervisionReport extends SupervisionReport {
    closure: ReportClosureState
    paymentRegularization: OutOfSystemPaymentRegularization
    monitoringObservations: Text(min=0, max=2000) // legítimamente opcional

    evolution: none // mismo patrón de variante definitiva; verificar contra negocio
}

Transform CompleteSupervision(draft: DraftSupervisionReport)
    -> Fin<CompletedSupervisionReport> {
    requires: closure and paymentRegularization present and valid
    requires: PendingRule("cuándo se declara completada la supervisión")
    guarantees: same TicketId; Completed immutability
}
```

Esta transformación **no presupone** que solo `"Cerrado"` puede formar un Completed; la proyección acordada contempla `"Sin cierre"`, `"Cerrada"` y `"Con observaciones"` cuando `supervisionCompleted=true`. El criterio de completar, su relación con seguimientos sucesivos y la restricción de evolución posterior requieren validación; no presentamos este modelo candidato como comportamiento legacy comprobado.

**Límite acordado (2026-11-12):** `monitoringObservations` admite hasta 2.000 caracteres y puede estar vacío. Esta longitud corresponde al campo del informe actual; todavía falta decidir cómo representar múltiples seguimientos históricos, sin concatenarlos automáticamente.

**Discrepancia de vocabulario:** formulario `"Sin Cierre" | "Cerrado" | "Con observaciones"` frente a proyección `"Sin cierre" | "Cerrada" | "Con observaciones"`. Requiere mapeo explícito antes de implementar.

## 8. Contrato de lectura consumido por Flow Orchestration

**La definición y realización de `TicketFlowFacts` pertenecen a [Flow Orchestration Service](flow-orchestration-service.md#3-proyección-de-ticket-propietaria-de-flow-orchestration).** Este Service solo posee Ticket y sus informes (incluidas las variantes Draft/Completed). No crea, conserva ni expone como responsabilidad propia una proyección específica del orquestador.

Flow Orchestration consulta **directamente, en modo de solo lectura**, las tablas propietarias de Ticket Service mediante JOIN/vista SQL y construye sus cinco hechos a partir de los informes. No necesita escribir ni duplicar los datos de Ticket.

La formalización de cada informe sigue siendo una responsabilidad de Ticket Service v2. El criterio organizacional de completar supervisión y el mapeo de valores de cierre siguen pendientes de acuerdo.

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
    input: TicketId + DraftEvaluationReport fields
    requires: report absent or current variant is DraftEvaluationReport
    writes: DraftEvaluationReport(new Point)
    guarantees: no flow transition
}

Feature CompleteEvaluation {
    input: TicketId + DraftEvaluationReport
    requires: CompleteEvaluation(draft) succeeds (all ten fields)
    writes: CompletedEvaluationReport replacing same logical report
    guarantees: no further report evolution
    doesNot: choose next FlowState
}

Feature SaveApprovalDraft {
    input: TicketId + DraftApprovalReport fields
    requires: report absent or current variant is DraftApprovalReport
    writes: DraftApprovalReport(new Point)
    guarantees: no flow transition
}

Feature CompleteApproval {
    input: TicketId + DraftApprovalReport
    requires: CompleteApproval(draft) succeeds (three required fields)
    writes: CompletedApprovalReport replacing same logical report
    guarantees: optional observation fields remain optional
    doesNot: advance the machine
}

Feature SaveSupervisionDraft {
    input: TicketId + DraftSupervisionReport fields
    requires: report absent or current variant is DraftSupervisionReport
    writes: DraftSupervisionReport(new Point)
    doesNot: advance the machine
}

Feature CompleteSupervision {
    input: TicketId + DraftSupervisionReport
    requires: required fields present AND pending organizational completion criterion resolved
    writes: CompletedSupervisionReport replacing same logical report
    guarantees: no further evolution under the candidate model
    doesNot: close the machine (Flow Orchestration decides that)
}

// Las Features de catálogos se definen con contratos y errores en §3.5.
// Create/Edit son específicas de System, Module e IncidenceType.
// Activate/Deactivate comparten el esquema, pero no su identidad funcional.
```

Los mantenedores se describen como candidatos funcionales; **no se interpretan las operaciones agrupadas como firmas aprobadas de una Feature**. Las reglas organizacionales de creación, modificación y vigencia todavía deben profundizarse.

## 10. Exclusiones y asuntos para siguiente revisión

1. No trasladar `Stage` ni `TicketStageTransitions` del legacy a Ticket Service v2: propiedad de Flow Orchestration.
2. No recuperar edición del Ticket, asignación responsable ni gestión posterior de adjuntos como Feature.
3. **Completitud de supervisión:** aprobar el criterio para crear `CompletedSupervisionReport` y contrastar su irreversibilidad con seguimientos repetidos. No confundir guardar un borrador con completar.
4. **Normalización de cierre:** reconciliar etiquetas de formulario e interfaz de proyección mediante decisión expresa.
5. **Referencias a elementos externos:** precisar identidades, tipos y permisos sin asumir que los sistemas externos están integrados.
6. **Catálogos:** nombre normalizado, reserva de nombres desactivados y reglas de Activar/Desactivar acordadas en §3; confirmar actividad inicial y detalles de normalización aún abiertos.
7. **Datos contextuales de proyecto:** precisar ubicación de código y nombre de proyecto en el modelo.
8. **Compatibilidad:** definir pruebas de equivalencia semántica para reemplazar `ticket-service` por `ticket-service-v2`.
9. **Coordinación:** completar informes y avanzar Flow Orchestration siguen siendo escrituras separadas cuyo protocolo no está decidido.

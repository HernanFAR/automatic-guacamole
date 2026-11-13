# Decisión 001 — Asignación anticipada del número de Ticket y derivación de FolderPath

**Estado:** decisión de diseño aceptada para la siguiente iteración, **pendiente de realización**. **Fecha:** 2026-11-13. **Ámbito:** Ticket Service v2, Ticket Support Product/BFF, Folder Service v2. **Origen:** revisión del contrato transversal tras comprobar que el registro inalterable necesita referencias de archivos anteriores al Ticket.

## Problema

Ticket Service v2 registra Tickets inalterables y debe recibir las referencias a sus adjuntos al crearlos. Folder Service necesita una identidad de carpeta estable antes de subir archivos. En el Product histórico, FolderPath se deducía de un Ticket ya registrado; en el PR de Product v2 actual (`ef7311c`) se genera una carpeta independiente `T:{Guid:N}`. Esta última preparación resuelve el alta previa, pero no ofrece por sí sola una derivación posterior desde el Ticket.

Se decide **obtener el número/código de Ticket antes de subir archivos y antes de registrar el Ticket**, de modo que BFF y Folder compartan una identidad recuperable sin conocer un TicketId todavía inexistente.

## Decisión

1. **Ticket Service v2 es autoridad de numeración** y expone una Feature invocable mediante **POST** para asignar anticipadamente el siguiente número/código de Ticket. El POST queda **expuesto al Product/BFF** mediante una superficie HTTP apropiadamente autenticada/autorizada; `POST` no significa endpoint anónimo.
2. La asignación proporciona un **código único que no se volverá a asignar a otra solicitud**, aun cuando el Ticket todavía no se registre. Para cumplirlo, Ticket Service debe conservar duraderamente el avance/asignación de numeración y garantizar exclusividad concurrente. No basta con calcular `MAX(número)+1` ni con comprobar unicidad solamente al registrar.
3. **`RegisterTicket` recibe ese código** y no genera otro. Debe verificar que el código fue asignado válidamente y que **no está consumido por otro Ticket**, y confirmar su uso con unicidad efectiva al persistir. Un código asignado y aún no registrado puede quedar sin uso: **no exigimos ausencia de saltos** entre los Tickets finalmente registrados.
4. Ticket Support Product deriva `FolderPath` de manera determinista **a partir de ese código asignado**. Para el modelo candidato se conserva la estructura `FolderPath(Correlative, Concept, PathTo)`:
   - `Correlative = assignedTicketCode` (ejemplo: `SP-0042`);
   - `Concept = "TicketSupport"`;
   - `PathTo = "Attachments"`.
   
   Estos dos literales son **convención propuesta para el Product v2**, compatible con la forma estructural del Folder Service, no una afirmación de que sean la convención histórica del sistema. Sus valores deberán permanecer estables para recuperar los archivos.
5. El BFF prepara/sube archivos a esa carpeta, obtiene sus `AttachedFileId`, y **registra el Ticket con el código asignado y todas las referencias iniciales**. Después solicita iniciar una única `FlowInstance` `TicketSupport/V1`. No se agrega ni elimina una referencia del Ticket registrado.
6. El mismo código del Ticket registrado permite al Product reconstruir el mismo `FolderPath` para consultar metadatos o descargar archivos; **la pertenencia y autorización se comprueban por la lista de referencias del Ticket antes de recuperar bytes**. El código conocido no concede acceso.

## Secuencia de referencia

```pseudo
case RegisterTicketWithEvidence {
    Product.ValidateAndAuthorizeRegistration()
    requester = Accounts.ResolveTrustedRequester()

    code = TicketServiceV2.AssignNextTicketNumber()  // POST; asignación durable y exclusiva
    folder = FolderPath(code, "TicketSupport", "Attachments")
    files = for each admissible evidence:
        FolderService.AddFile(folder, evidence, requester)

    ticket = TicketServiceV2.RegisterTicket(
        code = code,
        registration = validatedRegistration,
        attachments = files.ids
    )

    flow = FlowOrchestration.StartFlow(
        TicketSupportVersionV1Id,
        FlowSubject.TicketReference(ticket.id)
    )
    return Product.ComposeObservedOutcome(ticket, files, flow)
}

case ReadTicketEvidence {
    Product.AuthorizeTicketRead(ticket)
    code = TicketServiceV2.ReadTicketCode(ticket)
    folder = FolderPath(code, "TicketSupport", "Attachments")
    require fileReference in TicketServiceV2.ReadAttachmentReferences(ticket)
    return FolderService.DownloadFile(folder, fileReference)
}
```

Los nombres de Features y contratos anteriores son **intencionales**, no firmas C# ni rutas definitivas: la ruta HTTP concreta, DTO de respuesta y forma del tipo `TicketNumber/TicketCode` se fijarán con el código real.

## Garantías distinguidas

| Propiedad | Estado tras esta decisión |
| --- | --- |
| Código anticipado, único y durable | **Exigido** al mecanismo de asignación de Ticket Service |
| Conocimiento de carpeta antes de crear el Ticket | **Resuelto conceptualmente** por derivación desde código |
| Reconstrucción de carpeta después del registro | **Resuelta conceptualmente** mientras código y convención sean estables |
| Unicidad de código del Ticket registrado | **Exigida** con validación y restricción efectiva de persistencia |
| Reutilizar archivos ya subidos en reintentos | **Posibilitado**, si el Product conserva código y referencias o puede recuperarlas |
| Idempotencia de la petición POST de numeración | **Abierta**: repetir una petición no debe suponerse equivalente a recuperar la asignación anterior |
| Idempotencia de `RegisterTicket` | **Abierta**: mismo código no demuestra misma intención; debe decidirse cómo reconocer resultado previo |
| Recuperar archivos tras una interrupción sin referencias conservadas | **Abierta**: definir lectura/listado o comprobante durable si el caso lo necesita |
| Atomicidad de Folder + Ticket + Flow | **No garantizada**; reportar efectos parciales reales |
| Autorización por conocer código o ruta | **No**; siempre requiere autoridad de Product y pertenencia a Ticket |

**Diferencia conceptual:** se adopta **asignación anticipada no reutilizable**, no se introduce por ahora una entidad `RegistrationAttempt`, un timeout/lease, ni una política de caducidad. Llamarla «reserva» en conversaciones no debe hacer asumir capacidades de reintento o liberación no acordadas.

## Consecuencias y sustitución de propuesta anterior

- **Sustituye** la decisión provisional `T:{Guid.NewGuid():N}` del [PR de Product v2 #1](https://github.com/atom-dev-serviu/ticket-support-product-v2/pull/1). La Feature `PrepareAttachmentLocation` deberá **consumir el código emitido por Ticket Service**, o ser reemplazada por composición equivalente; no volver a inventar otro GUID de carpeta.
- No exige un `FolderId` nuevo ni que Folder Service conozca el dominio de Tickets. La identidad del archivo sigue siendo su `AttachedFileId` dentro de una carpeta determinada.
- El código del Ticket es una referencia de negocio, **no necesariamente su `TicketId`**. Flow Orchestration sigue usando `TicketReference(ticket.id)`.
- Un POST de numeración puede dejar huecos si el registro posterior falla. Se acepta como consecuencia explícita de asignar identidad antes del registro.
- Quedan pendientes el esquema de asignación y consumo de códigos, la verificación concurrente, el tratamiento de múltiples llamadas a POST, los contratos de referencias de archivos y la recuperación tras fallos parciales.
- Esta decisión **no modifica** automáticamente las ramas de implementación ni `COORDINATION.md`; esos documentos constituyen evidencia histórica de la iteración anterior y deberán enlazar esta decisión en el siguiente handoff.

## Pruebas requeridas al realizar

1. Dos asignaciones concurrentes devuelven códigos distintos; no se reutilizan números asignados sin registrar.
2. Registrar un Ticket con código ya consumido se rechaza; registrar con código válido asignado conserva ese código.
3. La carpeta derivada antes y después del registro es idéntica.
4. Un Ticket nace con referencias de archivos previamente subidos y no permite edición posterior.
5. Un archivo no asociado al Ticket no puede descargarse aunque su código y GUID sean conocidos.
6. Fallos entre subida, registro y comienzo de flujo dejan resultados parciales reconocibles.
7. Los reintentos no se anuncian como idempotentes hasta existir contrato y prueba específica.

**Fuentes de diseño:** [contrato transversal](integration-contract.md), [Ticket Service v2](ticket-service-v2.md), [BFF](ticket-support-bff.md), [Folder Service](folder-service.md). Esta decisión tiene precedencia sobre las notas provisionales de carpeta con GUID, sin reescribirlas retrospectivamente.

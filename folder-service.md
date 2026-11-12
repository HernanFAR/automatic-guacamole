# Folder Service — modelo semántico y Features para la PoC

**Estado:** pseudocódigo examinable derivado de código real; aún no es implementación VSlices nueva. **Fecha:** 2026-11-12. **Objetivo:** preparar el consumo del BFF de Ticket Support para la PoC del 13 de noviembre. **Repositorio propietario revisado:** [`atom-dev-serviu/folder-service@15e2bd058c06e8f4954cb85f7cc19192f8536653`](https://github.com/atom-dev-serviu/folder-service/tree/15e2bd058c06e8f4954cb85f7cc19192f8536653).

Este documento aplica el recorrido usado con Ticket Service v2 y Flow Orchestration: **evidencia → significado → Spaces y valores → invariantes → Features → pruebas → límites**. No supone que la forma actual de las clases de .NET sea la única realización posible.

## 1. Responsabilidad y límites

**Folder Service** posee la identidad estructurada de las carpetas y el contenido de los archivos, su nombre, MIME, tamaño derivado y operaciones genéricas. **No** posee Tickets, la asociación funcional entre Ticket y archivo, las decisiones de autorización de usuarios finales ni la coordinación del proceso.

**Ticket Service v2** conserva referencias opacas a los archivos asociados durante el registro de un Ticket inalterable. **Ticket Support Product/BFF** coordina el registro con esas referencias y controla qué recursos puede consultar o descargar un usuario. La API privada de Folder Service no debe exponerse directamente al navegador.

[Contrato propietario](https://github.com/atom-dev-serviu/documentation/blob/main/es/proyectos/ticket-support-reconstruction/contrato-folder-service.md) · [Responsabilidades en el repositorio](https://github.com/atom-dev-serviu/folder-service/blob/main/RESPONSIBILITIES.md).

## 2. Evidencia observada en código

| Aspecto | Contrato o comportamiento observado |
| --- | --- |
| Identidad de carpeta | `FolderPath(Correlative, Concept, PathTo)`; tres cadenas 1..92 caracteres, sin `FolderId` independiente |
| Identidad de archivo | `AttachedFileId` como GUID válido; SQL almacena clave compuesta `(FolderPath, Id)` |
| Nombre de archivo | Base y extensión obligatorias de 1..128 caracteres **cada una**; el largo total puede superar 128 |
| MIME | `FileContentType`: 1..128 caracteres e incluye `/`; no se valida una gramática MIME completa |
| Contenido | `FileContent`: copia defensiva de bytes; `Size` derivado de la longitud; **la construcción admite arreglo vacío** |
| Auditoría | `AuditStamp(ById, At)`; `ById` no vacío y `At` no futuro; es atribución opaca, no autenticación |
| Carpeta | Colección ordenada de archivos; agregar ID repetido se rechaza, retirar inexistente falla; revisiones inmutables |
| Copia | Genera nuevo `AttachedFileId`, conserva nombre, contenido y **AuditStamp del origen** |
| Metadatos | Consulta `Id, Name, ContentType, Size` sin cargar bytes del almacenamiento SQL |
| Límite de carga HTTP | `POST /files` rechaza `Bytes == null` y tamaños **superiores a 10 MiB** con 413; no es invariante de `FileContent` |
| Almacenamiento | SQL Server en esquema `Folders`: `FolderStates` y `FolderFiles`; EF Core solo para migraciones, SQL directo para operaciones |

Fuentes: [`FolderPath.cs`](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Spaces/FolderPath.cs), [`Folder.cs`](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Spaces/Folder.cs), [`FileValues.cs`](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Spaces/FileValues.cs), [`FolderSchemaDbContext.cs`](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Grounding/FolderSchemaDbContext.cs), [`FoldersHostingExtensions.cs`](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Hosting/FoldersHostingExtensions.cs).

## 3. Modelo semántico candidato

### 3.1. Carpeta: identidad estructurada, sin ID artificial

```pseudo
Value FolderPath {
    correlative: Text(min=1, max=92)
    concept: Text(min=1, max=92)
    pathTo: Text(min=1, max=92)

    identity: (correlative, concept, pathTo)
    invariant: not an arbitrary filesystem path
}

Space Folder {
    id: FolderPath                        // identidad interna compuesta
    files: Sequence<AttachedFileId>       // archivos pertenecientes a esta carpeta

    evolution:
        Empty -> WithAddedFile
        WithFiles -> WithAddedFile
        WithFiles -> WithRemovedFile

    invariant: unique AttachedFileId within folder
    invariant: Points are immutable
}
```

**Nota de fidelidad:** `Folder` actualmente usa `Path`, `Files` y una `Revision` de infraestructura; `id: FolderPath` y la secuencia de IDs son una **interpretación semántica propuesta**, no una firma actual del Framework. No creamos `FolderId` sin presión de negocio. `Revision` pertenece a la estrategia de concurrencia de la realización actual, no se introduce como característica indispensable del modelo.

### 3.2. Archivo y contenido

```pseudo
Value AttachedFileId {
    value: Guid
}

Value FileName {
    base: Text(min=1, max=128)
    extension: Text(min=1, max=128)
    represented: Derived<base + extension>
}

Value FileContentType {
    value: Text(min=1, max=128)
    admission: contains("/")
}

Value FileContent {
    bytes: Bytes
    type: FileContentType
    size: Derived<Length(bytes)>
}

Value AuditStamp {
    byId: Text(min=1)                     // identidad atribuida, sin autenticación propia
    at: Instant(<= now)
}

Space AttachedFile {
    id: AttachedFileId
    folder: FolderPath                     // asociación interna por identidad estructurada
    name: FileName
    content: FileContent
    updated: AuditStamp

    invariant: not editable by current Features
    invariant: changing folder by Copy creates new AttachedFileId
}
```

**Distinción importante:** se propone `AttachedFile` como Space semántico con identidad propia para examinarlo como hicimos en los otros dos servicios. El código vigente lo realiza como elemento de `Folder`; **no hay decisión de separar su persistencia o establecer ciclo de vida independiente**. `folder: FolderPath` representa pertenencia, no una referencia a un servicio externo.

El límite de 10 MiB pertenece **hoy al Hosting**. No trasladarlo silenciosamente a `FileContent` como si fuese regla universal. Tampoco asumir por el nombre `FileContent` que un arreglo vacío está prohibido: el código lo admite.

### 3.3. Proyección de metadatos

```pseudo
Projection FileMetadata {
    id: AttachedFileId
    name: Text
    contentType: FileContentType
    size: Int64

    source: AttachedFile within FolderPath
    excludes: bytes, AuditStamp
}
```

La proyección **pertenece a Folder Service**, porque él es propietario de los archivos y de esta consulta; no debe moverse al orquestador. No es un nuevo Space persistido. La realización SQL usa `DATALENGTH(Content)`.

## 4. Features seleccionadas por consumo real de Ticket Support Product

**Alcance acordado:** modelar únicamente las capacidades utilizadas por [`ITicketFolderFiles` y `TicketAttachmentOperations`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Work/Tickets/TicketAttachmentOperations.cs): `AddFile` (llamada `Upload` en el adaptador), `GetFileMetadata` (`ReadMetadata`), `DownloadFile` (`Download`) y `RemoveFile` (`Remove`). [`HttpTicketFolderFiles`](https://github.com/atom-dev-serviu/ticket-support-product/blob/main/src/TicketSupport.Grounding/Tickets/HttpTicketFolderFiles.cs) confirma las cuatro llamadas HTTP. **Se omite `CopyFile` de este modelo** porque el Product no la consume. Esto no implica retirar código ni endpoints existentes del repositorio propietario de Folder Service.

`RemoveFile` está presente en el **Product actual** como retiro y como limpieza compensatoria tras falla al conservar una referencia. En la PoC de Ticket v2, que no permite editar adjuntos luego del registro, **no habilitaremos esa operación como Feature de usuario**. Su existencia genérica es útil para limpieza/compatibilidad, sin redefinir la inalterabilidad del Ticket.

```pseudo
Feature AddFile {
    input: FolderPath + FileName + FileContent + AuditStamp
    requires: admissible values; generated AttachedFileId not duplicated in Folder
    writes: new Folder Point including new AttachedFile
    returns: AttachedFileId + metadata
    guarantees: newly generated identity and retained bytes, MIME, name, audit
}

Feature DownloadFile {
    input: FolderPath + AttachedFileId
    requires: file exists in indicated folder
    reads: AttachedFile
    returns: name + MIME + bytes + identity
    doesNot: write Folder
}

Feature RemoveFile {
    input: FolderPath + AttachedFileId
    requires: file exists in indicated folder
    writes: new Folder Point without that file
    doesNot: decide whether a Ticket may remove an attachment
}

Feature GetFileMetadata {
    input: FolderPath + AttachedFileId
    requires: file exists in indicated folder
    reads: selected file metadata only
    returns: FileMetadata
    doesNot: load or expose file bytes
}
```

**Nota:** el `AddFileResponse` actual devuelve también el Folder completo y `CopyFileResponse` devuelve Folder, del que Hosting extrae el último archivo. La salida anterior prioriza **significado para el consumidor**, no pretende ser una firma literal. La implementación del BFF deberá ajustarse al contrato HTTP real.

## 5. Errores, contratos HTTP y autoridad

La API privada de [Hosting](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Hosting/FoldersHostingExtensions.cs) expone:

| Operación | Ruta | Respuesta funcional |
| --- | --- | --- |
| Añadir | `POST /files` | 201 con ID, nombre, MIME y tamaño |
| Metadatos | `GET /files/{fileId}/metadata` + `correlative,concept,pathTo` | 200 sin bytes |
| Descargar | `GET /files/{fileId}` + coordenadas | 200 con bytes en respuesta JSON |
| Copiar | `POST /files/{fileId}/copy` | 200 con nuevo ID |
| Retirar | `DELETE /files/{fileId}` + coordenadas | 204 |

Todas las rutas `/files` requieren el secreto temporal `X-Folder-Invocation`; ausencia o diferencia devuelve 403. No es autorización funcional por usuario final. `/healthz` y `/readyz` son anónimos.

La función `Safe` del Hosting traduce `FileNotFoundException` a 404, `ArgumentException` a 400, `InvalidOperationException` a 409 y `SqlException` a 503. El cuerpo de alta limita bytes a 10 MiB con 413; los otros posibles fallos no quedan completamente clasificados aquí.

**Observación de implementación:** `SqlFolderRepository.UpdateAsync` utiliza transacción SQL Server, revisión optimista y reemplaza la colección de filas de archivos. Esa implementación evita pérdidas silenciosas frente a revisiones obsoletas, pero puede reescribir muchos bytes al añadir/quitar un archivo. No constituye una razón para cambiar el modelo semántico.

## 6. Pruebas de comportamiento relevantes para el Product

| Caso | Comportamiento esperado | Evidencia actual |
| --- | --- | --- |
| Añadir a carpeta vacía | Se crea un archivo y queda visible en lectura | `AddFile_persists_file_content_and_audit` |
| Descargar | Devuelve bytes del archivo seleccionado sin escribir | `DownloadFile_reads_selected_file_without_writing` |
| Consultar metadatos | No incluye bytes y exige carpeta correcta | `Read_metadata_retains_original_file_name_type_size_without_exposing_bytes` |
| Retirar | El archivo deja de existir en la carpeta | `RemoveFile_persists_removal` |
| Operar sobre inexistente | Download/Remove fallan; la prueba existente también cubre Copy, fuera del alcance seleccionado | `Missing_file_fails_copy_download_and_remove_without_changing_folders` |
| Actualizar Point obsoleto | No se pierden archivos existentes | `Stale_aggregate_update_is_rejected_instead_of_losing_files` |
| Archivo inmutable | Los bytes se copian defensivamente; tamaño derivado | `File_content_is_defensively_copied_and_size_is_derived` |
| Persistencia SQL real | Archivo sobrevive recreación de repositorio | `Uploaded_file_survives_repository_recreation_with_bytes_and_audit` (Testcontainers, opt-in) |

Pruebas en [`FolderFeatureTests.cs`](https://github.com/atom-dev-serviu/folder-service/blob/main/tests/Folders.Tests.Regression/FolderFeatureTests.cs) y [`FolderSqlServerIntegrationTests.cs`](https://github.com/atom-dev-serviu/folder-service/blob/main/tests/Folders.Tests.Regression/FolderSqlServerIntegrationTests.cs).

**Evidencia CI revisada:** [Verify Folders, run 42695456673](https://github.com/atom-dev-serviu/folder-service/actions/runs/42695456673) sobre HEAD `15e2bd0` terminó en `success` (2026-11-09). El workflow de `main` ejecuta regresión rápida y build; las pruebas Testcontainers SQL Server **requieren `workflow_dispatch`**. No atribuir al CI estándar haber ejecutado esa integración ni afirmar E2E transversal.

## 7. Recorrido necesario para Ticket Support Product/BFF

```pseudo
case RegisterTicketWithAttachments {
    // El Product conoce las reglas de registro del Ticket.
    validate registration and caller authority
    derive FolderPath for registration
    upload each selected file with FolderService.AddFile
    collect AttachedFileIds / FolderFileReferences
    invoke TicketServiceV2.RegisterTicket with opaque attachment references

    // La coordinación, tratamiento de subida parcial, reintentos
    // y limpieza de archivos huérfanos aún requiere decisión del Product.
}

case ReadRegisteredAttachment {
    verify authorized caller
    verify the referenced file belongs to the Ticket
    derive owner FolderPath
    request FolderService.GetFileMetadata or DownloadFile
}
```

**Restricciones acordadas en Ticket v2:** el Ticket registrado no se edita; no hay Feature para agregar ni retirar adjuntos posteriormente. Los límites de Ticket v2 (`0..5` adjuntos y `Attachment.name` máximo 50 caracteres) se validan en la frontera que conoce el Ticket, sin convertirlos automáticamente en restricciones genéricas de Folder Service. La política de tamaño del formulario y su compatibilidad con el máximo HTTP de 10 MiB debe verificarse al integrar.

**Tensión conocida:** el contrato histórico BFF sugería `RegisterTicket → AddFile → conservar referencias`. Como el nuevo Ticket debe nacer ya con sus referencias y permanece inalterable, esa secuencia no puede copiarse literalmente. Se propone subir primero y registrar después; **deja posible archivos huérfanos si falla el registro**. No prometer atomicidad distribuida para la PoC.

## 8. Decisiones y límites para el siguiente incremento

**Acordado:**
- Folder Service sigue siendo genérico y privado.
- `FolderPath` identifica estructuralmente una carpeta.
- `AttachedFileId` identifica un archivo. No modelamos `CopyFile` porque el Product actual no la consume.
- Los metadatos se proyectan en Folder Service, sin cargar bytes.
- Ticket Support Product/BFF verifica pertenencia y autorización antes de leer/descargar.
- Solo asociar archivos durante el registro de Ticket v2; `RemoveFile` se conserva para compatibilidad/limpieza, pero no se publica como edición del Ticket. `CopyFile` queda fuera del modelo de la PoC.

**Por examinar (sin bloquear la definición semántica):**
1. Forma exacta de `FolderFileReference` para el Ticket: referencia opaca y coordenadas necesarias para localizar el archivo sin adivinar la carpeta.
2. Derivación real de `FolderPath` desde el escenario de registro.
3. Secuencia y tratamiento mínimo de subida parcialmente exitosa y Ticket no registrado.
4. Contrato de errores y restricciones de tamaño en el BFF.
5. Evidencia E2E de Hosting + SQL Server + BFF; pruebas SQL opt-in y autenticación definitiva no se presuponen.
6. Posibilidad de optimizar la persistencia sin reescribir todos los archivos; **no forma parte de la PoC del 13**.

**No implementado en este documento:** migración al nuevo VSlices Framework, cambios al repositorio propietario, despliegue, nuevas Features y pruebas ejecutadas localmente. Este modelo está limitado deliberadamente a **cuatro Features consumidas por el Product actual**, sin trasladar automáticamente sus operaciones de edición de adjuntos al nuevo Ticket v2. Sirve como base para el siguiente incremento concreto con el código real, preservando su procedencia.

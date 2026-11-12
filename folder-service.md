# Folder Service — preparación para la PoC de Ticket Support

**Estado:** levantamiento de contratos y delimitación de trabajo; **no es una implementación nueva**. Fecha de revisión: 2026-11-12. Fecha límite de la PoC: 2026-11-13.

## 1. Fuentes y realidad verificada

- [Repositorio propietario `atom-dev-serviu/folder-service`](https://github.com/atom-dev-serviu/folder-service), rama `main`: `Folders.Spaces`, `Folders.Work`, `Folders.Grounding`, `Folders.Hosting` y `Folders.Tests.Regression`.
- [Contrato de responsabilidad](https://github.com/atom-dev-serviu/documentation/blob/main/es/proyectos/ticket-support-reconstruction/contrato-folder-service.md): servicio privado genérico; no determina pertenencia a Tickets ni autorización de usuarios.
- [README del servicio](https://github.com/atom-dev-serviu/folder-service/blob/main/README.md): SQL Server, migraciones EF Core, operaciones privadas y consulta de metadatos sin bytes. Esta documentación informa sobre trabajos previos; **no sustituye pruebas E2E actuales**.
- [Work actual](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Work/FolderFeatures.cs): `AddFile`, `CopyFile`, `DownloadFile`, `RemoveFile`.
- [Metadatos](https://github.com/atom-dev-serviu/folder-service/blob/main/src/Folders.Work/FileMetadata.cs): `GetFileMetadata` con `FolderPath` y `AttachedFileId`.

## 2. Significado y frontera

Folder Service posee carpetas y archivos: estructura de `FolderPath`, identidad del archivo, nombre, MIME, contenido y operaciones genéricas. No posee Tickets ni la autorización funcional del usuario final.

Ticket Service v2 posee la **referencia** al archivo asociado durante el registro del Ticket, no sus bytes. Ticket Support Product/BFF coordina solicitudes y verifica las reglas de acceso al contenido antes de utilizar Folder Service.

El contrato histórico menciona eliminación y modificación de adjuntos desde el BFF. **La decisión vigente para Ticket Service v2 prevalece para esta PoC:** no editar el Ticket ni añadir o retirar sus adjuntos después del registro. Esto **no exige eliminar** las Features genéricas de Folder Service.

## 3. Modelo existente (hechos de código, no una propuesta de reemplazo)

```pseudo
Value FolderPath {
    correlative: Text(min=1, max=92)
    concept: Text(min=1, max=92)
    pathTo: Text(min=1, max=92)
    // Identidad estructurada: NO ruta arbitraria del sistema de archivos.
}

Space Folder {
    path: FolderPath
    files: Sequence<AttachedFile>
    revision: Number
}

Space AttachedFile {
    id: AttachedFileId
    name: FileName
    content: FileContent
    updated: AuditStamp
}

Value FileContent {
    data: Bytes
    type: FileContentType
    size: Derived<Length(data)>
}
```

**Precisión:** son representaciones conceptuales del código .NET actual, no una afirmación de que estas clases ya realizan los nuevos contratos VSlices de Spaces. El código actual comprueba longitud 1..92 de cada componente de `FolderPath`, partes de `FileName` de 1..128 y un MIME de 1..128 que incluye `/`. No se han aprobado cambios de límites por el solo hecho de documentarlos.

## 4. Capacidades que ya existen

| Capacidad | Requisito | Resultado |
| --- | --- | --- |
| `AddFile` | Carpeta estructurada, nombre, contenido y atribución | Nuevo archivo con `AttachedFileId` |
| `CopyFile` | Origen, archivo y carpeta destino | Copia con identidad nueva |
| `DownloadFile` | Carpeta y archivo existente | Contenido y metadatos |
| `RemoveFile` | Carpeta y archivo existente | Archivo retirado de la carpeta |
| `GetFileMetadata` | Carpeta y archivo existente | ID, nombre, MIME y tamaño **sin descargar bytes** |

## 5. Recorrido prioritario para la PoC

```pseudo
case RegisterTicketWithAttachments {
    // Responsabilidad de Ticket Support Product/BFF:
    validate registration and authorized context
    derive structured FolderPath
    for each attachment: call FolderService.AddFile
    register Ticket with opaque references in Ticket Service v2

    // El orden real, reversión ante fallos y compensación se precisan al
    // realizar el BFF; no prometer transacción distribuida.
}

case ReadRegisteredAttachment {
    BFF verifies association of reference with Ticket and authorization
    BFF calls FolderService.GetFileMetadata or DownloadFile as needed
}
```

El contrato histórico muestra una secuencia de registro cuyo orden exacto **debe revisarse** frente al Ticket inalterable y las referencias producidas por `AddFile`; no reproducir mecánicamente el legado.

## 6. Verificación necesaria antes de modificar

1. Confirmar HEAD y CI actuales de `folder-service/main`, más diferencia con `migration/initial-capabilities` y PR activo si corresponde.
2. Verificar host privado, protección `X-Folder-Invocation`, SQL Server, migraciones y contratos de errores con evidencia del código y pruebas; no tomar el README como prueba suficiente de integración.
3. Ejecutar/consultar evidencia de pruebas existentes para alta, consulta de metadatos y descarga sobre almacenamiento durable.
4. Revisar cómo el BFF deriva `FolderPath` y garantiza asociación de referencias con el Ticket, sin asumir acceso directo al navegador.
5. Para la PoC, priorizar registro con adjuntos y lectura. Copia, eliminación y edición posterior de adjuntos **no** forman parte del recorrido necesario de Ticket v2.

## 7. Alcance consciente de la PoC

No convertir Folder Service en propietario de relaciones de negocio, no introducir un motor de permisos genérico ni prometer atomicidad entre servicios. Las garantías fuertes de concurrencia, recuperación y auditoría futura son asuntos posteriores, salvo que un fallo observado impida el recorrido mínimo.

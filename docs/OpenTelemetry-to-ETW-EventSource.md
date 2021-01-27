# OpenTelemetry ETW exporter to EventSource mapping

EventSource defines certain common standard fields used in ETW transport [specification](https://msdnshared.blob.core.windows.net/media/MSDNBlogsFS/prod.evol.blogs.msdn.com/CommunityServer.Components.PostAttachments/00/10/44/08/22/_EventSourceUsersGuide.docx).

## Standard ETW / EventSource fields

Standard ETW fields:

| Field Name | Description | Example |
|------------|---------|---------|
| `Timestamp` | Binary-packed. Represented as ISO-8601 string in local time | `2021-01-27T12:00:41.9080032-08:00` |
| `ProviderName` or `Provider GUID` | `MyCompany-MyApplication-ComponentName` |
| `Id` | Event Id or Sequence Number (0 - for automatic sequence numbers) | 9 |
| `Opcode` | Definitions:<br>0 - WINEVENT_OPCODE_INFO<br>1 - WINEVENT_OPCODE_START<br>2 - WINEVENT_OPCODE_STOP<br>[Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-opcodetype-complextype#remarks). | 0 |
| `Task` | [Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-tasktype-complextype) | - |
| `Keywords` | [Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-keywordtype-complextype) | - |
| `Level` | Definitions:<br>1 - WINEVENT_LEVEL_CRITICAL<br>2 - WINEVENT_LEVEL_ERROR<br>3 - WINEVENT_LEVEL_WARNING<br>4 - WINEVENT_LEVEL_INFO<br>5 - WINEVENT_LEVEL_VERBOSE<br>[Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-leveltype-complextype) | |




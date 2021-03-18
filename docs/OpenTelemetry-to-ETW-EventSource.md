# OpenTelemetry ETW exporter to EventSource mapping

EventSource defines certain common standard fields used in ETW transport [specification](https://msdnshared.blob.core.windows.net/media/MSDNBlogsFS/prod.evol.blogs.msdn.com/CommunityServer.Components.PostAttachments/00/10/44/08/22/_EventSourceUsersGuide.docx).

## ETW / EventSource Fields

ETW fields:

| Field Name | Description | Example |
|------------|-------------|---------|
| `Timestamp` | Binary-packed. Represented as ISO-8601 string in local time | `2021-01-27T12:00:41.9080032-08:00` |
| `ProviderName` or `Provider GUID` | `MyCompany-MyApplication-ComponentName` |
| `Id` | Event Id or Sequence Number (0 - for automatic sequence numbers) | 9 |
| `Opcode` | Definitions:<br>0 - WINEVENT_OPCODE_INFO<br>1 - WINEVENT_OPCODE_START<br>2 - WINEVENT_OPCODE_STOP<br>[Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-opcodetype-complextype#remarks). | 0 |
| `Task` | [Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-tasktype-complextype) | - |
| `Keywords` | [Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-keywordtype-complextype) | - |
| `Level` | Definitions:<br>1 - WINEVENT_LEVEL_CRITICAL<br>2 - WINEVENT_LEVEL_ERROR<br>3 - WINEVENT_LEVEL_WARNING<br>4 - WINEVENT_LEVEL_INFO<br>5 - WINEVENT_LEVEL_VERBOSE<br>[Defined here](https://docs.microsoft.com/en-us/windows/win32/wes/eventmanifestschema-leveltype-complextype) | |
| `ActivityId` | A value that uniquely identifies the current activity. | |
| `RelatedActivityId` | A value that uniquely identifies the parent activity to which this activity is related. | |

ETW custom properties can logically be placed in strongly-typed `Payload` property bag, or string key-value map or dictionary.

ETW event represented in JSON notation:

```json
{
  "Timestamp": "2021-01-27T12:00:36.9024479-08:00",
  "ProviderName": "OpenTelemetry",
  "Id": 1,
  "Message": null,
  "ProcessId": 8604,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MySpanTop/Start",
  "ActivityID": "12f64b71-192a-4691-0000-cccccccccccc",
  "RelatedActivityID": null,
  "Payload": {
    "SpanLinks": "",
    "TraceId": "e2a9f03263d86e408dabf4f12b06c1a2"
  }
}
```

## OpenTelemetry Concepts, Attributes, Fields

| Field Name       | Description |
|------------------|-------------|
| `trace_id`       |             |
| `span_id`        |             |
| `parent_span_id` |             |
| (Span)`name`     | Span name   |
| (Span)`kind`     | Span kind   |
| (Span)`status`   | Span status |
| (Event)`name`    | Event name  |
| *attributes*     | Strongly typed key-value dictionary |

## OpenTelemetry to ETW Mapping

| OT Field Name    | OT Type  | ETW / EventSource Field Name | ETW Type       | Location    |
|------------------|----------|------------------------------|----------------|-------------|
| trace_id         | 16 bytes | TraceId                      | UTF-8 string   | Payload     |
| span_id          | 8 bytes  | ActivityId                   | GUID           | Envelope    |
| parent_span_id   | 8 bytes  | RelatedActivityId            | GUID           | Envelope    |
| name             |          | EventName                    | UTF-8 string   | Envelope    |
| kind             |          | SpanKind                     | number         | Envelope    |
| status           |          | Status                       | UTF-8 string   | Envelope    |
| *attributes*     |          | Payload                      |                | Payload     |
| span (start)     |          | Opcode=1                     | number         | Envelope    |
| span (end)       |          | Opcode=2                     | number         | Envelope    |

## OpenTelemetry Correlation to ETW Correlation concepts

A `SpanContext` represents the portion of a `Span` which must be serialized and
propagated along side of a distributed context. `SpanContext`s are immutable.

The OpenTelemetry `SpanContext` representation conforms to the [W3C TraceContext
specification](https://www.w3.org/TR/trace-context/). It contains two
identifiers - a `TraceId` and a `SpanId` - along with a set of common
`TraceFlags` and system-specific `TraceState` values.

`TraceId` A valid trace identifier is a 16-byte array with at least one
non-zero byte.

`SpanId` A valid span identifier is an 8-byte array with at least one non-zero
byte.

`TraceFlags` contain details about the trace. Unlike TraceState values,
TraceFlags are present in all traces. The current version of the specification
only supports a single flag called [sampled](https://www.w3.org/TR/trace-context/#sampled-flag).

`TraceState` carries vendor-specific trace identification data, represented as a list
of key-value pairs. TraceState allows multiple tracing
systems to participate in the same trace. It is fully described in the [W3C Trace Context
specification](https://www.w3.org/TR/trace-context/#tracestate-header)

One `Tracer` is associated with exactly one `TraceId`.

`Tracer` objects create `Span` objects using `StartSpan` API. Each `Span` associated
with the `Tracer.TraceId` and has its own unique `SpanId`.

When the first top-level `Span` object is created:
- `ETW.RelatedActivityId` (GUID, 16 bytes) = parent tracer `Tracer.TraceId` (16 bytes)
- `ETW.ActivityId` (GUID, 16 bytes) = `Span.SpanContext.SpanId`

When a child `Span` object is created:
- `ETW.RelatedActivityId` (GUID, 16 bytes) = parent `Span.SpanContext.SpanId` (8 bytes)
- `ETW.ActivityId` (GUID, 16 bytes) = current `Span.SpanContext.SpanId` (8 bytes)

Custom decoration could be optionally retained on each individual ETW event, to enable
quick search of all `Span` objects associated with the current `Trace` by populating
`ETW.Payload.TraceId` - custom attribute inside the `Payload` property bag.

### Attributes

Attributes could be expressed at two levels:
- Span-level attributes
- Event-level attributes

OpenTelemetry Span with Events represented in OTLP-JSON notation:

```json
{
    "resource_spans": [
        {
            "instrumentation_library_spans": [
                {
                    "spans": [
                        {
                            "trace_id": "kDMI7LTxLxTj220awNARJw==",
                            "span_id": "9ir6veJ4Hdw=",
                            "parent_span_id": "bgnsqqPvjYQ=",
                            "name": "Sample-8",
                            "kind": 1,
                            "start_time_unix_nano": 1588334156464409000,
                            "end_time_unix_nano": 1588334156470454639,
                            "attributes": [
                                {
                                    "key": "attr",
                                    "string_value": "value3"
                                },
                                {
                                    "key": "attr3",
                                    "string_value": "value4"
                                }
                            ],
                            "events": [
                                {
                                    "time_unix_nano": 464430000,
                                    "name": "event1",
                                    "attributes": [
                                        {
                                            "key": "key1",
                                            "type": 3,
                                            "bool_value": true
                                        }
                                    ]
                                },
                                {
                                    "time_unix_nano": 464438000,
                                    "name": "event2",
                                    "attributes": [
                                        {
                                            "key": "key2",
                                            "string_value": "value2"
                                        }
                                    ]
                                }
                            ],
                            "status": {
                                "message": "ok"
                            }
                        }
                    ]
                }
            ]
        }
    ]
}
```

## Example OpenTelemetry Spans and Events exported via ETW exporter

C++ code that emits two nested Spans and Event on inner span:

```cpp
  std::string providerName = "OpenTelemetry"; // supply unique instrumentation name here
  exporter::ETW::TracerProvider tp;
  auto tracer = tp.GetTracer(providerName, "TLD");
  // Span attributes
  Properties attribs =
  {
    {"attrib1", 1},
    {"attrib2", 2}
  };
  // Top-level outder span
  auto topSpan = tracer->StartSpan("MySpanTop");
  std::this_thread::sleep_for (std::chrono::seconds(1));
  // Inner nested span with parent
  auto outerSpan = tracer->StartSpan("MySpanL2", attribs);
  // Add event
  std::string eventName1 = "MyEvent1";
  Properties event1 =
  {
    {"uint32Key", (uint32_t)1234},
    {"uint64Key", (uint64_t)1234567890},
    {"strKey", "someValue"}
  };
```

Generates the following payload over ETW channel:

```json
[
{
  "Timestamp": "2021-01-27T12:00:36.9024479-08:00",
  "ProviderName": "OpenTelemetry",
  "Id": 1,
  "Message": null,
  "ProcessId": 8604,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MySpanTop/Start",
  "ActivityID": "12f64b71-192a-4691-0000-cccccccccccc",
  "RelatedActivityID": null,
  "Payload": {
    "SpanLinks": "",
    "TraceId": "e2a9f03263d86e408dabf4f12b06c1a2"
  }
},
{
  "Timestamp": "2021-01-27T12:00:37.9035262-08:00",
  "ProviderName": "OpenTelemetry",
  "Id": 2,
  "Message": null,
  "ProcessId": 8604,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MySpanL2/Start",
  "ActivityID": "776d932b-5cdc-4d11-0000-cccccccccccc",
  "RelatedActivityID": "12f64b71-192a-4691-0000-cccccccccccc",
  "Payload": {
    "SpanLinks": "",
    "TraceId": "e2a9f03263d86e408dabf4f12b06c1a2",
    "attrib1": 1,
    "attrib2": 2
  }
},
{
  "Timestamp": "2021-01-27T12:00:37.9039311-08:00",
  "ProviderName": "OpenTelemetry",
  "Id": 3,
  "Message": null,
  "ProcessId": 8604,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MyEvent1",
  "ActivityID": null,
  "RelatedActivityID": "776d932b-5cdc-4d11-0000-cccccccccccc",
  "Payload": {
    "SpanId": "2b936d77dc5c114d",
    "strKey": "someValue",
    "uint32Key": 1234,
    "uint64Key": 1234567890
  }
}
]
```

# Implementation Notes

Starting a Span emits `Opcode=WINEVENT_OPCODE_START` event and populate `ActivityId` / `RelatedActivityId` in accordance to [EventSource spec](https://msdnshared.blob.core.windows.net/media/MSDNBlogsFS/prod.evol.blogs.msdn.com/CommunityServer.Components.PostAttachments/00/10/44/08/22/_EventSourceUsersGuide.docx) ,
whereas stopping a Span emits `Opcode=WINEVENT_OPCODE_STOP`. This feature operation is functionally equivalent to [EventSource guidance for .NET](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.tracing.eventsource?view=net-5.0#examples).

Auto-instrumentation macros may be provided for C++ code to replicate the automated injection of `Start` and `Stop` operations for scoped spans similar to .NET. OpenTelemetry `span_id` is mapped to ETW `ActivityId`, `parent_span_id` is mapped to ETW `RelatedActivityId`. Event-level Payload also carries `SpanId` field to simplify matching back to parent span. `TraceId` field may be populated on  Span/Start and Span/Events only. Since casual relationship from individual event back to parent span is already defined via `ActivityId`, `RelatedActivityId` and `SpanId` attributes. Events on Span belong only to one parent span, and the span belongs to only one unique trace identified via `TraceId`.

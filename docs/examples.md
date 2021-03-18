# OpenTelemetry C++ SDK to ETW export

```cpp
  std::string providerName = "OpenTelemetry-ETW-Provider"; // supply unique instrumentation name here
  exporter::ETW::TracerProvider tp;

  // TODO: this code should fallback to MsgPack if TLD is not available
  auto tracer = tp.GetTracer(providerName, "TLD");
  auto topSpan = tracer->StartSpan("MySpanTop");
  ```
  
  Generates the following ETW event:
  
  ```json
  {
  "Timestamp": "2021-03-18T12:32:17.8948138-07:00",
  "ProviderName": "OpenTelemetry-ETW-Provider",
  "Id": 1,
  "Message": null,
  "ProcessId": 15036,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MySpanTop/Start",
  "ActivityID": "390cd47b-46d0-418c-0000-000000000000",
  "RelatedActivityID": null,
  "Payload": {
    "SpanLinks": "",
    "TraceId": "ddd5d76c4a74cf45a6791acf7c9aa71b"
  }
}
```

Note the `Start` suffix after the `Span` name - **MySpanTop/Start**.

Note that there are two options how `TraceId` may be expressed:
- `TraceId` added to `Payload` property bag and `RelatedActivityId` assigned to `null`; OR
- `TraceId` value assigned to `RelatedActivityId` GUID type, to associate the span with its parent `Tracer.TraceId`

Child spans can be associated with their parents as follows:

```cpp
  // Span attributes
  Properties attribs =
  {
    {"attrib1", 1},
    {"attrib2", 2}
  };

  auto outerSpan = tracer->StartSpan("MySpanL2", attribs);
  auto innerSpan = tracer->StartSpan("MySpanL3", attribs);
```

Note that the `Tracer` object remembers the `current` span that is
obtained using `Tracer::GetCurrentSpan()` API. ETW exporter performs
the following assignment:
- set `ETW.RelatedActivityId` to `current` (becoming parent) `SpanContext.Id`
- set `ETW.ActivityId` to new child `SpanContext.Id`
- assign new child to `current` Span

Sequence of ETW events:

Child of top-level span:

```json
{
  "Timestamp": "2021-03-18T12:32:18.8952602-07:00",
  "ProviderName": "OpenTelemetry-ETW-Provider",
  "Id": 2,
  "Message": null,
  "ProcessId": 15036,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MySpanL2/Start",
  "ActivityID": "6c13b971-6930-4f9d-0000-000000000000",
  "RelatedActivityID": "390cd47b-46d0-418c-0078-aa9bac010000",
  "Payload": {
    "SpanLinks": "",
    "TraceId": "ddd5d76c4a74cf45a6791acf7c9aa71b",
    "attrib1": 1,
    "attrib2": 2
  }
}
```

Grandchild of top-level span:

```json
{
  "Timestamp": "2021-03-18T12:32:18.8952709-07:00",
  "ProviderName": "OpenTelemetry-ETW-Provider",
  "Id": 3,
  "Message": null,
  "ProcessId": 15036,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MySpanL3/Start",
  "ActivityID": "c1ffb283-0432-46aa-0000-000000000000",
  "RelatedActivityID": "6c13b971-6930-4f9d-007e-aa9bac010000",
  "Payload": {
    "SpanLinks": "",
    "TraceId": "ddd5d76c4a74cf45a6791acf7c9aa71b",
    "attrib1": 1,
    "attrib2": 2
  }
}
```

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

Note that in this example all events carry the same `TraceId=ddd5d76c4a74cf45a6791acf7c9aa71b` string (_may be represented as 16-byte GUID_) value inside the `Payload` property bag. This enables joining all spans together into one trace, if necessary. Populating `TraceId` on every event could be wasteful from the application instrumentation perspective, this function of assigning `TraceId` to individual events may be implemented in the ETW listener (side-car).

Adding events on `Span` can be implemented as follows. Note we are adding:
- `MyEvent1` and `MyEvent2` on outer (2nd-level from top) span
- `MyEvent3` on inner (3rd-level from top) span

```cpp
  // Add first event
  std::string eventName1 = "MyEvent1";
  Properties event1 =
  {
    {"uint32Key", (uint32_t)1234},
    {"uint64Key", (uint64_t)1234567890},
    {"strKey", "someValue"}
  };
  outerSpan->AddEvent(eventName1, event1);
  
  // Add second event
  std::string eventName2 = "MyEvent2";
  Properties event2 =
  {
    {"uint32Key", (uint32_t)9876},
    {"uint64Key", (uint64_t)987654321},
    {"strKey", "anotherValue"}
  };
  outerSpan->AddEvent(eventName2, event2);

  std::string eventName3= "MyEvent3";
    Properties event3 =
  {
    /* Extra metadata that allows event to flow to A.I. pipeline */
    {"metadata", "ai_event"},
    {"uint32Key", (uint32_t)9876},
    {"uint64Key", (uint64_t)987654321},
    // {"int32array", {{-1,0,1,2,3}} },
    {"tempString", getTemporaryValue() }
  };
  innerSpan->AddEvent(eventName3, event3);
```

Corresponding ETW event payload:

**MyEvent1** : 

```json
{
  "Timestamp": "2021-03-18T12:32:18.8952787-07:00",
  "ProviderName": "OpenTelemetry-ETW-Provider",
  "Id": 4,
  "Message": null,
  "ProcessId": 15036,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MyEvent1",
  "ActivityID": null,
  "RelatedActivityID": "6c13b971-6930-4f9d-0000-000000000000",
  "Payload": {
    "SpanId": "71b9136c30699d4f",
    "strKey": "someValue",
    "uint32Key": 1234,
    "uint64Key": 1234567890
  }
}
```

**MyEvent2** :

```json
{
  "Timestamp": "2021-03-18T12:32:19.8963662-07:00",
  "ProviderName": "OpenTelemetry-ETW-Provider",
  "Id": 5,
  "Message": null,
  "ProcessId": 15036,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MyEvent2",
  "ActivityID": null,
  "RelatedActivityID": "6c13b971-6930-4f9d-0000-000000000000",
  "Payload": {
    "SpanId": "71b9136c30699d4f",
    "strKey": "anotherValue",
    "uint32Key": 9876,
    "uint64Key": 987654321
  }
}
```

**MyEvent3** :

```json
{
  "Timestamp": "2021-03-18T12:32:21.8975498-07:00",
  "ProviderName": "OpenTelemetry-ETW-Provider",
  "Id": 6,
  "Message": null,
  "ProcessId": 15036,
  "Level": "Always",
  "Keywords": "0x0000000000000000",
  "EventName": "MyEvent3",
  "ActivityID": null,
  "RelatedActivityID": "c1ffb283-0432-46aa-0000-000000000000",
  "Payload": {
    "SpanId": "83b2ffc13204aa46",
    "metadata": "ai_event",
    "tempString": "Value from Temporary std::string",
    "uint32Key": 9876,
    "uint64Key": 987654321
  }
}
```

# Comparison with .NET Microsoft-Diagnostics-DiagnosticSource ETW provider - Event API

ETW Provider `Microsoft-Diagnostics-DiagnosticSource` can be used to listen
to events emitted using `DiagnosticSource`.

C# Instrumentation:

```csharp
   DiagnosticSource ds = new DiagnosticListener("Diag1");
   ds.Write("ev1", new { X = "str" }); 
```

ETW event contents:

```json
{
  "Timestamp": "2021-03-18T12:57:28.0311936-07:00",
  "ProviderName": "Microsoft-Diagnostics-DiagnosticSource",
  "Id": 3,
  "Message": null,
  "ProcessId": 1544,
  "Level": "Informational",
  "Keywords": "0x0000F00000000002",
  "EventName": "Event",
  "ActivityID": null,
  "RelatedActivityID": null,
  "Payload": {
    "SourceName": "Diag1",
    "EventName": "ev1",
    "Arguments": "[[{\"Key\":\"Key\",\"Value\":\"X\"},{\"Key\":\"Value\",\"Value\":\"str\"}]]"
  }
}
```

# Comparison with .NET Microsoft-Diagnostics-DiagnosticSource ETW Provider - Activity API

`Activity` is conceptually similar to the concept of `Span`.

C# Instrumentation using `DiagnosticSource.StartActivity` :

```csharp
   Activity activity = new Activity("MyOperation");
   ds.StartActivity(activity, null);
```

**Activity Start** ETW event contents:

```json
{
  "Timestamp": "2021-03-18T12:57:28.0386165-07:00",
  "ProviderName": "Microsoft-Diagnostics-DiagnosticSource",
  "Id": 3,
  "Message": null,
  "ProcessId": 1544,
  "Level": "Informational",
  "Keywords": "0x0000F00000000002",
  "EventName": "Event",
  "ActivityID": null,
  "RelatedActivityID": null,
  "Payload": {
    "SourceName": "Diag1",
    "EventName": "MyOperation.Start",
    "Arguments": "[]"
  }
}
```

```csharp
   ds.StopActivity(activity, null);
```

**Activity Stop**  ETW event contents:

```json
{
  "Timestamp": "2021-03-18T12:57:28.0408691-07:00",
  "ProviderName": "Microsoft-Diagnostics-DiagnosticSource",
  "Id": 3,
  "Message": null,
  "ProcessId": 1544,
  "Level": "Informational",
  "Keywords": "0x0000F00000000002",
  "EventName": "Event",
  "ActivityID": null,
  "RelatedActivityID": null,
  "Payload": {
    "SourceName": "Diag1",
    "EventName": "MyOperation.Stop",
    "Arguments": "[]"
  }
}

However, C# API does not automatically track parent-child relationship. `ActivityId` and `RelatedActivityId` fields remain empty. Even if a child activity is explicitly linked to parent activity using `Activity.SetParentId` API, relationship remains unreflected in ETW events.

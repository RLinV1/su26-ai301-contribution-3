# Contribution [3]: [OpenTelemetry]

**Contribution Number:** [3]  
**Student:** [Raymond Lin]  
**Issue:** [#5924 — [feature request] Being able to filter logs like activities with a processor](https://github.com/open-telemetry/opentelemetry-dotnet/issues/5924)  
**Repository:** [open-telemetry/opentelemetry-dotnet](https://github.com/open-telemetry/opentelemetry-dotnet)  
**Pull Request:** [#___ — PR title](pr-link)  
**Status:** Issue selected — working on reproduction.

---

## Candidate Issues

I was deciding between these two:

- [pygmt #1393](https://github.com/GenericMappingTools/pygmt/issues/1393)
- [opentelemetry-dotnet #5924](https://github.com/open-telemetry/opentelemetry-dotnet/issues/5924) ← **chosen**

---

## Why I Chose This Issue

I picked opentelemetry-dotnet #5924 mainly because I've used OpenTelemetry for logs before and
I genuinely like working with it. Having already wired up `builder.Logging.AddOpenTelemetry(...)`
in a real project means I'm not starting from zero — I already understand the logging pipeline
(logger provider → processors → exporter), so I can spend my time on the actual contribution
instead of learning the library from scratch. It's also a tool I expect to keep using, so
anything I learn here pays off outside the class.

A few other things made it a good fit:

- **It's labeled `good first issue`, `documentation`, and `contribfest`** — the maintainers have
  explicitly flagged it as newcomer-friendly and small in scope.
- **It's still open and unassigned** after being filed in October 2024, and it carries the
  `keep-open` label, so it won't be closed as stale out from under me while I work on it.
- **The scope is understandable.** The maintainer already pointed at a working reference
  implementation, and said they intend to document the pattern in this repo when they have
  the cycles — so the contribution is a well-defined gap, not an open-ended design problem.
- **It's a real pain point.** The reporter is trying to stop health-check/liveness logs from
  flooding Application Insights, which is a problem I can relate to from my own logging setup.
- **C#/.NET is my stack**, whereas the pygmt issue would have meant learning both Python
  plotting internals and the GMT C library at the same time.

---

## Understanding the Issue

### Problem Description

In OpenTelemetry .NET, traces have a built-in way to drop telemetry before export: you install a
`Sampler`, or clear the `Recorded` flag on an `Activity`, and the SDK won't export it. Logs have
no equivalent. There is no log sampler, and `LogRecord` exposes no "don't export me" switch, so a
custom `BaseProcessor<LogRecord>` that's registered with `AddProcessor(...)` can inspect and
mutate a log record but cannot suppress it.

The reporter (tovich37) hit this while sending logs to Azure Monitor / Application Insights. They
wrote a `FilterLogsProcessor` intending to drop records whose `Path` attribute contains
`liveness`, `health`, or `metrics`, and their code literally has a
`// TODO: Configure the log record here, so that it is not exported` where the drop should
happen — because the SDK gives them no way to express it. The result is that health-check noise
pollutes their Application Insights data with no available workaround.

Worth noting: the built-in `ILogger` filtering (`builder.Logging.AddFilter(...)`) only filters by
category and log level. It can't make a decision based on the *attributes* of an individual log
record, which is exactly what this scenario needs, so it isn't a substitute.

### Expected Behavior

A developer should be able to write a processor that decides, per log record, whether that record
continues down the pipeline to the exporter — the same mental model that already exists for
activities. Something like:

```csharp
public class FilterLogsProcessor : BaseProcessor<LogRecord>
{
    public override void OnEnd(LogRecord data)
    {
        var path = data.Attributes?.FirstOrDefault(a => a.Key == "Path").Value?.ToString();
        if (!string.IsNullOrEmpty(path) && path.Contains("health"))
        {
            // this record should not be exported
            return;
        }

        base.OnEnd(data);
    }
}
```

And, at minimum, the supported way to achieve this should be discoverable from the repo's own
documentation.

### Current Behavior

`AddProcessor` registers processors into a chain that the SDK owns. Returning early from `OnEnd`
in your own processor doesn't stop the record — the SDK still hands it to the next processor and
ultimately to the exporter. Filtering only works if a processor *owns* the downstream processor
and chooses whether to call it, and that pattern isn't documented anywhere in
`opentelemetry-dotnet`, so users don't discover it.

The maintainer response confirms the shape of the fix. rajkumar-rangaraj
([comment](https://github.com/open-telemetry/opentelemetry-dotnet/issues/5924#issuecomment-2438357263))
pointed to [Azure/azure-sdk-for-net#41493](https://github.com/Azure/azure-sdk-for-net/pull/41493),
whose `LogFilteringProcessor` does exactly this — it wraps an inner processor and only forwards
records that pass the check:

```csharp
internal class LogFilteringProcessor : BaseProcessor<LogRecord>
{
    private readonly BaseProcessor<LogRecord> _processor;

    public override void OnEnd(LogRecord logRecord)
    {
        if (/* record passes the filter */)
        {
            _processor.OnEnd(logRecord);   // forward
        }
        // otherwise: dropped
    }

    // OnForceFlush / OnShutdown / Dispose delegate to _processor
}
```

He added that they "plan to document this in the repo … when we have the cycles to update the
documentation." That, plus the `documentation` label, tells me the deliverable here is most
likely a documented sample in `opentelemetry-dotnet` rather than a new SDK API.

There's already a strong precedent to follow: `docs/logs/routing/RoutingProcessor.cs` is a
wrapping processor that sends log records to one of two inner processors based on category name,
and correctly forwards `OnForceFlush`, `OnShutdown`, and `Dispose`. A filtering sample would be
the same structure with one inner processor and a boolean decision.

### Affected Components

- **Package:** `OpenTelemetry` (the SDK), logs pipeline
- **Types involved:** `BaseProcessor<LogRecord>`, `LogRecord`,
  `OpenTelemetryLoggerOptions.AddProcessor` / `LoggerProviderBuilder.AddProcessor`
- **Likely place for the fix:** the `docs/logs/` samples — either a new `filtering/` sample
  directory alongside `routing/`, or an addition to `docs/logs/extending-the-sdk/`
- **Reference implementation:** `LogFilteringProcessor` in
  [Azure/azure-sdk-for-net#41493](https://github.com/Azure/azure-sdk-for-net/pull/41493)
- **Existing precedent in-repo:** `docs/logs/routing/RoutingProcessor.cs`

---

## Reproduction Process

### Environment Setup

**Prerequisites**
- [...]

**Steps**

1. [...]

### Steps to Reproduce

1. [...]

### Reproduction Evidence

- [...]

---

## Solution Approach

### Analysis

[...]

### Proposed Solution

[...]

### Implementation Plan

1. [...]

---

## Testing Strategy

### Unit Tests

[...]

### Integration Tests

[...]

### Manual Testing

[...]

---

## Implementation Notes

### Week [N] Progress

[...]

### Code Changes

- **Files modified:**
  - [...]
- **Key commits:**
  - [...]
- **Approach decisions:** [...]

---

## Pull Request

**PR Link:** [...]

**PR Description:** [...]

**Maintainer Feedback:** [...]

**Status:** [...]

---

## Learnings & Reflections

### Technical Skills Gained

[...]

### Challenges Overcome

[...]

### What I'd Do Differently Next Time

[...]

---

## Resources Used

- [...]

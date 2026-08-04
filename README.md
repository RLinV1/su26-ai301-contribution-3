# Contribution [3]: [OpenTelemetry]

**Contribution Number:** [3]  
**Student:** [Raymond Lin]  
**Issue:** [#5924 — [feature request] Being able to filter logs like activities with a processor](https://github.com/open-telemetry/opentelemetry-dotnet/issues/5924)  
**Repository:** [open-telemetry/opentelemetry-dotnet](https://github.com/open-telemetry/opentelemetry-dotnet)  
**Pull Request:** [#___ — PR title](pr-link)  
**Status:** Reproduction process and solution approach drafted — repro captures pending .NET SDK install.

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

Because this is a documentation-gap issue, there are two things to reproduce: the *behavioral*
symptom (a processor that returns early still can't stop a record from being exported) and the
*documentation* symptom (nothing in the repo tells you the supported way to do it).

### Environment Setup

**Prerequisites**

- .NET SDK 8.0 or later. The repo pins an exact SDK in `global.json`, so if the build complains
  about a version mismatch, install the version it names rather than editing the file.
- Git, and a fork of `open-telemetry/opentelemetry-dotnet`.
- No collector or Azure resource is needed. `ConsoleExporter` is enough to see whether a record
  reached the exporter, and it keeps the repro self-contained.

**Steps**

1. Fork and clone the repository, then confirm the toolchain works before changing anything:

   ```bash
   git clone https://github.com/<my-username>/opentelemetry-dotnet.git
   cd opentelemetry-dotnet
   dotnet build docs/logs/routing/routing.csproj
   ```

   I'm building the `routing` sample specifically because it's the closest existing analogue to
   what I'll be adding, so if it builds, my new sample will build the same way.

2. Create a scratch console project outside the repo for the behavioral repro, referencing the
   released packages rather than the local source. This keeps the repro honest — it shows the
   behavior a user actually hits, not something specific to a local build:

   ```bash
   dotnet new console -o LogFilterRepro
   cd LogFilterRepro
   dotnet add package OpenTelemetry
   dotnet add package OpenTelemetry.Exporter.Console
   dotnet add package Microsoft.Extensions.Logging
   ```

### Steps to Reproduce

**Part 1 — the naive processor doesn't drop anything.**

1. Write the processor the way the issue reporter did: inspect the record's attributes and return
   early when it looks like health-check noise.

   ```csharp
   using System.Linq;
   using Microsoft.Extensions.Logging;
   using OpenTelemetry;
   using OpenTelemetry.Logs;

   internal sealed class FilterLogsProcessor : BaseProcessor<LogRecord>
   {
       public override void OnEnd(LogRecord data)
       {
           var path = data.Attributes?
               .FirstOrDefault(a => a.Key == "Path").Value?.ToString();

           if (!string.IsNullOrEmpty(path) &&
               (path.Contains("health") || path.Contains("liveness") || path.Contains("metrics")))
           {
               // TODO: Configure the log record here, so that it is not exported.
               return;
           }

           base.OnEnd(data);
       }
   }
   ```

2. Register it ahead of the exporter and emit one record that should be dropped and one that
   should survive:

   ```csharp
   using var loggerFactory = LoggerFactory.Create(builder =>
   {
       builder.AddOpenTelemetry(logging =>
       {
           logging.AddProcessor(new FilterLogsProcessor());
           logging.AddConsoleExporter();
       });
   });

   var logger = loggerFactory.CreateLogger("Repro");

   using (logger.BeginScope(new[] { new KeyValuePair<string, object?>("Path", "/health") }))
   {
       logger.LogInformation("Health check probe {Path}", "/health");
   }

   logger.LogInformation("Real request {Path}", "/api/orders");
   ```

3. Run it:

   ```bash
   dotnet run
   ```

4. Observe that **both** records are printed by the console exporter. The `return` in `OnEnd` ends
   *that processor's* work, not the record's journey.

**Part 2 — confirm there's no supported alternative and no documentation for one.**

5. Search the SDK surface for a drop switch on the log record:

   ```bash
   grep -rn "class LogRecord" src/OpenTelemetry/Logs/
   grep -rn "Sampler" src/OpenTelemetry/Logs/
   ```

   There is no `Sampler` type in the logs pipeline and no `Recorded` / `IsRecorded` / `Dropped`
   member on `LogRecord` — the escape hatches that exist for `Activity` simply have no counterpart
   here.

6. Search the docs for any filtering guidance:

   ```bash
   grep -rni "filter" docs/logs/
   ```

   The hits are about `ILogger` category/level filtering (`AddFilter`), which can't see an
   individual record's attributes. Nothing describes the wrapping-processor pattern. `docs/logs/`
   currently contains `complex-objects`, `correlation`, `customizing-the-sdk`, `dedicated-pipeline`,
   `extending-the-sdk`, `getting-started-aspnetcore`, `getting-started-console`, `redaction`, and
   `routing` — no `filtering`.

7. Confirm the mechanism in the SDK source, which explains *why* step 4 behaves that way:

   ```bash
   grep -rn "CompositeProcessor" src/OpenTelemetry/
   ```

   Each `AddProcessor(...)` call appends to a `CompositeProcessor<LogRecord>`, and
   `AddConsoleExporter()` appends its own `SimpleLogRecordExportProcessor`. The composite walks its
   chain and calls `OnEnd` on every entry unconditionally. A processor has no way to tell the
   composite to stop, because the composite — not the user's processor — owns the iteration.

### Reproduction Evidence

Expected versus actual for the run in step 3:

| Log record | Expected (what the reporter wanted) | Actual |
| --- | --- | --- |
| `Health check probe /health` | dropped, never reaches the exporter | printed by `ConsoleExporter` |
| `Real request /api/orders` | exported | printed by `ConsoleExporter` |

The filtering decision is computed correctly — a breakpoint or `Console.Error.WriteLine` inside the
`if` block shows it being taken for the `/health` record — but it has no effect on export. That gap
between "the processor decided to drop it" and "it got exported anyway" is the whole bug.

Artifacts to attach once I've run this end to end:

- `repro/Program.cs` and `repro/FilterLogsProcessor.cs` (the exact files above).
- Terminal capture of `dotnet run` showing both records in the console exporter output.
- Terminal capture of the `grep -rni "filter" docs/logs/` result showing no filtering sample.

> Note: I have the .NET runtime but not the SDK on my current machine (`dotnet --list-sdks` returns
> nothing), so these captures are still pending. The expected/actual table above is derived from the
> SDK source path traced in step 7, not from a run I've already done. I'll replace this note with
> the real output once the SDK is installed.

---

## Solution Approach

### Analysis

The root cause is an ownership problem, not a missing feature. In the traces pipeline the SDK asks
*before* the fact — a `Sampler` runs at `Activity` creation, and an unsampled activity is either
never created or never gets `ActivityTraceFlags.Recorded`. In the logs pipeline everything happens
*after* the fact: the record already exists, and every registered processor is called by a
`CompositeProcessor<LogRecord>` that iterates its whole chain. A user processor sits inside that
chain as a peer, so it can read and mutate the record but can't remove itself from the loop.

The only place the iteration can be interrupted is a processor that *owns* its downstream processor
and decides whether to call it. That inverts the relationship: instead of registering the filter and
the exporter as two siblings in the composite, you register one filter that holds the exporter's
processor. This is exactly what `LogFilteringProcessor` in
[Azure/azure-sdk-for-net#41493](https://github.com/Azure/azure-sdk-for-net/pull/41493) does, and it's
structurally identical to the in-repo `docs/logs/routing/RoutingProcessor.cs`, which already wraps
two inner processors and picks one per record.

So the capability exists today; it's just undiscoverable. Combined with the `documentation` label and
rajkumar-rangaraj's comment that they "plan to document this in the repo," the right deliverable is a
documented sample, not a new SDK API. Proposing an API change here would be scope creep against a
`good first issue` and would need a spec discussion upstream in OpenTelemetry, which isn't what the
maintainers asked for.

### Proposed Solution

Add a new `docs/logs/filtering/` sample, modelled directly on `docs/logs/routing/`:

```
docs/logs/filtering/
├── FilteringProcessor.cs
├── Program.cs
├── README.md
└── filtering.csproj
```

`FilteringProcessor.cs` — one inner processor plus a predicate, with the full lifecycle forwarded:

```csharp
// Copyright The OpenTelemetry Authors
// SPDX-License-Identifier: Apache-2.0

using OpenTelemetry;
using OpenTelemetry.Logs;

/// <summary>
/// A processor which only forwards log records that satisfy a predicate to the
/// inner processor. Records that fail the predicate are dropped and never exported.
/// </summary>
internal sealed class FilteringProcessor : BaseProcessor<LogRecord>
{
    private readonly BaseProcessor<LogRecord> innerProcessor;
    private readonly Func<LogRecord, bool> shouldExport;

    public FilteringProcessor(
        BaseProcessor<LogRecord> innerProcessor,
        Func<LogRecord, bool> shouldExport)
    {
        this.innerProcessor = innerProcessor ?? throw new ArgumentNullException(nameof(innerProcessor));
        this.shouldExport = shouldExport ?? throw new ArgumentNullException(nameof(shouldExport));
    }

    public override void OnEnd(LogRecord data)
    {
        if (this.shouldExport(data))
        {
            this.innerProcessor.OnEnd(data);
        }

        // Otherwise the record is dropped: the inner processor never sees it,
        // so it is never handed to the exporter.
    }

    protected override bool OnForceFlush(int timeoutMilliseconds)
        => this.innerProcessor.ForceFlush(timeoutMilliseconds);

    protected override bool OnShutdown(int timeoutMilliseconds)
        => this.innerProcessor.Shutdown(timeoutMilliseconds);

    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            this.innerProcessor.Dispose();
        }

        base.Dispose(disposing);
    }
}
```

`Program.cs` — the health-check scenario from the issue, so the sample answers the reporter's actual
question:

```csharp
var filteringProcessor = new FilteringProcessor(
    innerProcessor: new BatchLogRecordExportProcessor(new ConsoleExporter<LogRecord>(new())),
    shouldExport: static logRecord =>
    {
        var path = logRecord.Attributes?
            .FirstOrDefault(a => a.Key == "Path").Value as string;

        return path is null
            || (!path.Contains("health", StringComparison.OrdinalIgnoreCase)
                && !path.Contains("liveness", StringComparison.OrdinalIgnoreCase)
                && !path.Contains("metrics", StringComparison.OrdinalIgnoreCase));
    });

using var loggerFactory = LoggerFactory.Create(builder =>
    builder.AddOpenTelemetry(logging => logging.AddProcessor(filteringProcessor)));
```

The README for the sample is where the real value is, and it needs to call out the three things that
make this pattern easy to get wrong:

1. **Don't also register the exporter separately.** Calling `AddConsoleExporter()` /
   `AddOtlpExporter()` alongside the filtering processor puts a second export processor in the
   composite that the filter doesn't own, and every record reaches it — the filter appears to do
   nothing. The exporter must be reachable *only* through the wrapping processor. This is the
   failure mode I expect most people hit, and it's precisely why the earlier repro shows both
   records: `AddProcessor` and `AddConsoleExporter` were siblings.
2. **Forward `OnForceFlush`, `OnShutdown`, and `Dispose`.** A wrapper that only overrides `OnEnd`
   silently swallows shutdown, so buffered records are lost on exit.
3. **Filtering by attribute vs. by category.** If the decision can be made from category name and
   level, `builder.Logging.AddFilter(...)` is cheaper and stops the record before it's ever
   allocated. The wrapping processor is for decisions that need the record's attributes — which is
   the case in this issue, and the reason `AddFilter` isn't a workaround.

I'll also link the new sample from `docs/logs/README.md` and cross-reference it from
`docs/logs/extending-the-sdk/README.md`, since that's where someone writing a custom processor will
look first.

### Implementation Plan

1. Comment on [#5924](https://github.com/open-telemetry/opentelemetry-dotnet/issues/5924) describing
   the plan (a `docs/logs/filtering/` sample mirroring `routing/`) and ask
   rajkumar-rangaraj to confirm that's the deliverable he had in mind before I write it. The issue
   is unassigned and `contribfest`-labeled, so claiming it explicitly avoids duplicate work.
2. Finish the reproduction captures above so the PR can state the problem concretely.
3. Branch from `main` as `docs/logs-filtering-sample`.
4. Scaffold `docs/logs/filtering/` by copying the structure of `docs/logs/routing/` — including the
   Apache-2.0 SPDX headers and the `$(RepoRoot)\src\...` `ProjectReference` style used by
   `routing.csproj`, rather than package references.
5. Implement `FilteringProcessor.cs` and `Program.cs` as above.
6. Write `docs/logs/filtering/README.md`: the problem, the wrapping pattern, the annotated code, the
   three pitfalls, and a "when to use `AddFilter` instead" section.
7. Add links from `docs/logs/README.md` and `docs/logs/extending-the-sdk/README.md`.
8. Verify locally: `dotnet build docs/logs/filtering/filtering.csproj`, then `dotnet run` and confirm
   the health-check records are absent from the output while the others are present. Run the repo's
   analyzers/format check and markdownlint over the new docs, since the CI gate for a docs PR is
   mostly lint.
9. Open the PR referencing "Fixes #5924", with before/after output showing the health-check record
   dropped. No `CHANGELOG.md` entry — this ships no shipping-package change.
10. Respond to review. The likeliest maintainer asks are naming (`FilteringProcessor` vs.
    `LogFilteringProcessor`), whether the predicate should be a `Func<>` or a subclass hook, and
    whether the sample belongs under `extending-the-sdk` instead of its own directory — I'll follow
    whatever they prefer rather than defending the layout.

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

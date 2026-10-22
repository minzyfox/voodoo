# Vx flight recorder and placement simulation

Historical proposal from October 6, 2026. See the [Voodoo roadmap](ROADMAP.md) for the current milestones and updated assessment of existing tools.

Assessment date: October 6, 2026. Vx checkout HEAD: `c3039ed9`; implementation observations describe the files inspected in that working tree.

A compiler explanation and memory-flow viewer is a strong fit for Vx. A runtime flight recorder could add useful evidence about copies, execution, and failures. Accurate timing simulation is a plausible later step for bounded workloads, but the current cost model does not justify a general prediction tool.

| Tool | Assessment | Main requirement |
| --- | --- | --- |
| Compiler placement and memory-flow viewer | Useful starting point | Connect analysis results to source operations, dependencies, and lowerings |
| Runtime flight recorder | Viable | Record actual operations and correlate them with the compiler plan |
| Placement timing simulator | Conditional | Calibrated costs, execution resources, and a complete dependency graph |

## Compiler decisions and memory flow

Vx already reports transfer routes, byte counts, cost provenance, resident memory totals, capacity margins, and traffic per kernel region and placed buffer through [`--diagnostics-json`](../../Vx/src/diagnostics_json.rs). Counts can be exact, upper bounds, or unavailable. These facts support questions such as which tensor requires a transfer, which buffer causes repeated reads, and why a placement exceeds capacity.

Vx validates declared placement and selects transfer routes. Its [memory-algebra design](../../Vx/docs/memory_algebra.md) separates those responsibilities from a future placement optimizer. The viewer should identify placement requested by the programmer and explain the decisions the compiler actually makes.

A static view could show allocation, transfer, consumer execution, readback, and release, with source locations and dependencies. Selecting an operation could expose its destination, size, selected route, lowering, and validation obligations. Independent operations should remain independent; a display must not invent their execution order.

The existing reports are ingredients rather than a complete execution graph. They publish traffic weights but do not expose all producer–consumer edges. The [report types](../../Vx/src/report.rs) also need operation identities and source mappings for reliable navigation and correlation. Aggregate resident totals alone cannot establish buffer lifetimes or a precise peak-memory curve.

Program order and dependencies can answer **where a copy occurs in execution** without predicting its timestamp. This is already useful: XProf's [Memory Viewer](https://openxla.org/xprof/memory_viewer) displays compiler-derived memory usage and buffer lifetimes against program order, explicitly distinguishing that axis from elapsed time.

## Runtime recording

Fixed placement does not imply fixed execution. Sizes, invocation counts, backend selection, allocation failures, and delays can vary. The [Vx runtime ABI](../../Vx/include/vx_hardware_runtime.h) provides natural recording points for allocation and transfer, dispatch, await, readback, peer transfer, and release.

Two current implementation details demonstrate why observations and lowerings matter:

- In the [CUDA backend](../../Vx/runtime/cuda_dispatch.cpp), device-image execution synchronizes before dispatch returns, and `vx_plugin_await_future` does nothing.
- In [raw operation lowering](../../Vx/src/codegen/lower/mod.rs), `raw::async_copy` currently emits synchronous loads and stores.

A timeline based on API names would suggest overlap that these paths do not provide. Compiler explanations should expose the selected mechanism; runtime traces can establish what happened in a particular execution.

[dial9](https://github.com/dial9-rs/dial9) provides the relevant design precedent: collect individual events, retain them, and inspect their causes after a problem. It supports custom application events alongside runtime and system information. Its Tokio instrumentation would not automatically reveal Vx transfers or GPU execution; Vx needs its own semantic events.

A proposed recorder would retain a bounded history of allocations, copies, launches, completion, waits, and failures, linked to compiler operation IDs and buffer identities. Host-call timestamps must be distinguished from device execution timestamps. NVIDIA's [CUPTI Activity API](https://docs.nvidia.com/cupti/main/main.html) provides GPU activity records and correlation IDs linking CUDA calls to kernels and copies. [Perfetto](https://perfetto.dev/docs/instrumentation/tracing-sdk) offers an existing event format and viewer; its [buffer documentation](https://perfetto.dev/docs/concepts/buffers) describes bounded recording and detection of lost events.

Ordinary runtime hooks do not record every memory operation inside a kernel. Vx's logical read counts are also not physical HBM traffic measurements: caches and backend transformations affect what reaches memory.

## Limits of timing simulation

The [memory-algebra assessment](../../Vx/docs/memory_algebra.md) documents missing transfer startup latency, unsettled multi-hop composition, and contention detection without a calibrated sharing law. It also records instrument errors that confused memory levels. These are reasons to withhold precise timing claims.

The diagnostics schema defines `total_cost` as a unitless route-selection weight. It cannot be used as an event duration. `derived_cost` is a bandwidth estimate with units, not an observed duration. Region traffic is reported per launch; summing static sites does not establish total traffic for a run containing loops or conditional execution.

A credible simulator needs dependencies, actual execution mechanisms, resource and queue constraints, compute costs, calibrated transfer costs, and assumptions about dynamic inputs. Changing placement must also account for any changed lowering or compute performance. Facts, conservative bounds, estimates, and measurements should remain visibly distinguishable. Unsupported cases should remain unknown.

## Research precedents

- **[FlexFlow — Beyond Data and Model Parallelism for Deep Neural Networks](https://arxiv.org/html/1807.05358v1), Jia, Zaharia, and Aiken, 2018.** Constructs compute and communication tasks and simulates their dependencies and resource queues. Compute durations come from measurements; communication uses a bandwidth model under explicit assumptions. It supports placement simulation within a defined workload model, without establishing accuracy for arbitrary Vx programs.
- **[Daydream — Accurately Estimating the Efficacy of Optimizations for DNN Training](https://www.usenix.org/conference/atc20/presentation/zhu-hongyu), Zhu, Phanishayee, and Pekhimenko, USENIX ATC 2020.** Builds a dependency graph from CUPTI traces, transforms the graph to represent optimizations, and simulates execution. This is a direct precedent for using recorded behavior to support later what-if analysis.
- **[XProf Memory Viewer](https://openxla.org/xprof/memory_viewer).** Demonstrates the value of static memory planning, buffer lifetimes, and source attribution before timing simulation is available.

## Suggested scope

Start with source-linked compiler decisions and a dependency view. Add runtime observations using the same operation identities when concrete debugging questions justify recording. Use measured behavior to calibrate narrowly scoped placement comparisons later.

The value is making Vx's decisions inspectable and checking their relationship to execution. A precise-looking animation built from unverified durations would undermine that value. The explanation viewer is worth pursuing now; a general timing simulator should wait for trustworthy graph structure and calibration.

# Voodoo roadmap

Voodoo is the codename for Vx tooling that explains where data lives, why it moves, what actually executed, and which alternative placements are supported by evidence. Start with a useful compiler explanation joined to a small execution trace. Extend that foundation toward a bounded flight recorder and, when measurements justify it, placement simulation.

Status: proposed milestones, October 7, 2026. The implementation baseline is Vx `33e727d18696560e51aa089ef4dfe764498839eb`. Milestone completion requires the evidence below; this roadmap does not claim that the proposed tooling exists.

## Direction and scope

The immediate value is answering concrete questions: did a region run as generated GPU code, through a vendor library, or on the host? Which copies did that choice introduce? Why did operations serialize? Which storage remains allocated?

Vx already exposes placement, route and logical traffic facts. It also carries source locations into MLIR and performs ownership and transfer-lifetime analysis. These foundations make a first tool practical. They do not yet form a complete, exported execution plan correlated with runtime behavior.

The tool should distinguish programmer intent, compiler decisions and observed execution. Each claim should retain its units, assumptions and provenance. Unknown information stays unknown. A static program-order view must not look like a measured timeline, and logical access counts must not be presented as physical HBM traffic.

No new user-facing language syntax is required for the first milestone. The initial work belongs mainly in compiler metadata, runtime instrumentation and their shared contracts. A complete graph for arbitrary programs, real asynchronous overlap, multi-backend support and a custom timeline UI are later work.

## Milestones

| Milestone | User outcome | Main work | Completion evidence |
| --- | --- | --- | --- |
| 1. Explain one computation from source to execution | Identify backend choices, staging copies and serialization for a small CUDA workload | Operation and invocation identities, source attribution, compiler decision records, internal runtime events, existing trace viewer | A reproducible residency comparison with correct results and attributable events; an independent timing check; explicit coverage and overhead |
| 2. Explain dependencies and memory lifetime | Follow why an operation waits and why storage remains allocated | Control flow, aliases, read/write dependencies, allocation and pool lifetime, partial graph export | Branch, loop and alias cases agree with execution; unsupported dependencies and storage are visibly incomplete |
| 3. Retain useful history when problems occur | Diagnose an intermittent slowdown or failure after the event | Bounded retention, triggers, loss detection, durable metadata, failure handling | Retention and failure drills preserve interpretable evidence; recording stays within a declared resource budget |
| 4. Compare measured placement alternatives | Choose between a small set of legal placements with measured evidence | Candidate compilation, correctness checks, repeated runs, structural comparison and experiment metadata | Comparable alternatives reproduce a useful result across held-out inputs; changed lowerings are accounted for |
| 5. Simulate a bounded placement space | Rank more alternatives than are practical to execute exhaustively | Calibrated task costs, resource scheduling, latency and contention models, uncertainty | Held-out placement ranking and selection loss meet predeclared targets; unsupported cases are refused |

Milestone 1 has a separate [implementation TODO](MILESTONE_1_TODO.md). Milestone 2 and retention work in milestone 3 can proceed independently after the milestone 1 identity and event contracts settle. Small measured comparisons can begin during milestone 1; milestone 4 makes them repeatable. Milestone 5 depends on validated dependencies and measurement, not on completing every production recorder feature.

## Milestone 1 scope

Use one process, one CUDA GPU and static-shape tensor workloads. The primary example is repeated matrix multiplication, comparing operands kept resident with operands staged on each iteration. Add a small generated kernel to cover a dispatch path distinct from cuBLAS. Start from existing Vx workloads and tests where possible.

Produce a versioned compiler manifest, an execution trace, and a compact report that connects the two. Reuse a timeline viewer. A source operation may expand into several runtime operations, and a source site may execute many times; both relationships must be represented.

Include allocations, staging, readback and pool activity inside the backend. Recording only calls to the public plugin ABI would hide work performed inside a dispatch. Record the actual backend choice and its reason when available; runtime choices cannot always be settled by compilation.

The initial report should establish structural agreement before discussing timing: which invocations occurred, which buffers moved, and which mechanism executed. No duration prediction is required. Device timing must be checked independently, and missing device timing must not be replaced with a host-call duration under the same label.

## Milestone 2 scope

Export dependencies from appropriate compiler stages while preserving source origins. Include data dependencies, mutation and alias ordering, barriers, calls and runtime synchronization. Keep loops and guards in the static representation instead of claiming every program has one fixed directed acyclic graph. An observed run supplies particular loop iterations and branch outcomes.

Reuse existing ownership, drop, alias and liveness analyses. Distinguish a logical value's lifetime, an allocation's lifetime and memory retained by an allocator pool. Backend temporaries and views must be represented without double-counting storage. Report supported regions rather than blocking the entire tool on complete whole-program analysis.

Static hints should expose evidence such as repeated transfers or repeated logical reads. A hint is not a speedup claim. Capacity totals remain different from measured peak device memory unless their coverage and lifetime model justify the comparison.

## Milestone 3 scope

Add a bounded recorder with explicit event-loss and truncation records. Retained events must remain decodable when their original creation events or metadata precede the retained window. Define dump behavior for a manual request, a timeout and a handled failure. State the limits for abrupt process termination and device failures.

Measure disabled and enabled overhead, memory use and output volume. Keep cheap semantic recording separate from expensive device profiling. Extend to another backend only with an explicit capability description for timing, correlation and memory accounting; absent capabilities remain visible.

## Milestone 4 scope

Compile and execute a small set of legal placement variants. Check numerical results and record shapes, dtypes, layouts, compiler and kernel identities, backend and library versions, device configuration, warmup and repetition policy. Separate initialization and JIT work from steady-state execution where the question requires it.

Compare operation structure before attributing a duration difference to placement. A placement change can change generated code, vendor-library routing, occupancy, synchronization or temporary storage. Measurements from one mechanism cannot silently stand in for another.

## Milestone 5 scope

Start with static-shape task graphs and a named set of transformations. Model compute and communication tasks, their dependencies and the resources they occupy. Add transfer startup latency before relying on bandwidth alone; account for the resulting size-dependent route choice. Validate multi-hop composition and resource contention against the mechanisms actually used.

Assess useful decisions on held-out cases. In addition to duration error, measure placement ranking and selection loss: how much slower the chosen placement is than the best measured candidate. Define numerical acceptance targets before evaluating the held-out set. Report interval coverage and width together so an arbitrarily broad interval cannot pass as useful prediction.

A statistical prediction interval is not a proven bound. Some observations will fall outside it at its stated coverage. A changed workload, code path or hardware configuration must invalidate or qualify calibration rather than silently reusing it.

## Existing Vx foundations and gaps

| Area | Baseline and implication |
| --- | --- |
| [Diagnostics](../../Vx/src/diagnostics_json.rs) and [report types](../../Vx/src/report.rs) | Routes, cost provenance, capacity and logical traffic exist. Correlated operation identities, source attribution and dependency export need work. `total_cost` is a route-selection weight, not a duration. |
| [Source locations](../../Vx/src/codegen/generator.rs) and [Vx operations](../../Vx/include/VxDialect.td) | Reuse existing MLIR locations, explicit operations and memory effects. Preserve origins through lowering, expansion and fusion. |
| [Ownership and transfer lowering](../../Vx/src/dialect/VxLowering.cpp) | Drops and transfer frees already use lifetime information. Export and correlate it instead of creating a competing lifetime analysis. |
| [Region traffic](../../Vx/src/hir/check/region_traffic.rs) | Counts logical accesses per launch, with bounds or absence reasons. It does not measure device memory traffic. |
| [Runtime ABI](../../Vx/include/vx_hardware_runtime.h) and [CUDA backend](../../Vx/runtime/cuda_dispatch.cpp) | Natural tracing points exist, but dispatch also performs internal staging and allocation. The current device-image path synchronizes before returning. |
| [Raw lowering](../../Vx/src/codegen/lower/mod.rs) | `raw::async_copy` currently uses synchronous loads and stores. Record the selected mechanism instead of deriving overlap from its name. |
| [Memory algebra](../../Vx/docs/memory_algebra.md) | Transfer startup latency, composition and contention remain modeling work. They do not block structural explanation. |

The existing `VX_TIME_KERNEL` device-image timing path records its stop event after synchronization. Audit this placement before using those durations for calibration; source inspection identifies a possible timing gap, not a measured error magnitude.

## References and design lessons

- [FlexFlow, Beyond Data and Model Parallelism for Deep Neural Networks](https://arxiv.org/html/1807.05358v1): measured compute costs and explicit task/resource simulation make bounded placement search feasible. Its structured operator graph and runtime assumptions do not establish accuracy for arbitrary Vx programs.
- [Daydream](https://www.usenix.org/conference/atc20/presentation/zhu-hongyu): trace-derived dependencies and graph transformations support measured what-if analysis. A trace still describes a particular execution.
- [Dial9](https://github.com/dial9-rs/dial9): retaining individual runtime and application events supports investigation after a problem. Vx requires its own semantic instrumentation.
- [XProf Graph Viewer](https://openxla.org/xprof/graph_viewer) and [Memory Viewer](https://openxla.org/xprof/memory_viewer): compiler graphs can be joined with execution measurements, while static lifetimes can be shown against program order. This integration is established work, not a novelty claim for Voodoo.
- [IREE device profiling](https://iree.dev/developers/performance/device-profiling/) and [Tracy integration](https://iree.dev/developers/performance/profiling-with-tracy/): reuse the lessons from structured dispatch, queue, memory and executable records rather than treating existing compiler tooling as only text dumps.
- [SC22 interactive data movement visualization](https://arxiv.org/abs/2207.07433): static dataflow and parameterized simulation can guide optimization before full timing prediction.
- [TileLens, formerly Triton-Viz](https://github.com/Deep-Learning-Profiling-Tools/tilelens): visualization, profiling and sanitization provide useful kernel tooling; the current project includes optional NKI support.
- [Perfetto SDK](https://perfetto.dev/docs/instrumentation/tracing-sdk) and [buffer behavior](https://perfetto.dev/docs/concepts/buffers): existing trace infrastructure can host Vx events and bounded recording.
- [CUPTI](https://docs.nvidia.com/cupti/main/main.html): CUDA API and device-activity correlation provides a basis for checking attribution and timing.

## Earlier proposals

The [placement simulation assessment](vx-flight-recorder-and-placement-simulation.md) and [explanation and recorder proposal](vx-explanation-and-flight-recorder-tooling.md) retain the original motivation and their October 6 baseline. This roadmap governs the proposed milestones. It replaces the requirement to finish a complete graph before shipping anything, does not require a prediction for every operation, and updates the comparison with existing tools.

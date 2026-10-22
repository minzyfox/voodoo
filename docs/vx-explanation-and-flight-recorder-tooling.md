# Vx explanation and flight-recorder tooling

Historical proposal from October 6, 2026. See the [Voodoo roadmap](ROADMAP.md) for the current milestones and updated assessment of existing tools.

Oct 6, 2026 · @desia

## Summary

Build two tools around one shared prerequisite: a compile-time explainer and a runtime flight recorder, joined by stable operation IDs so every compiler prediction can be checked against what actually ran. A general timing simulator comes later, gated on accuracy measured by that join.

| Tool | Verdict | First deliverable | Main dependency |
| --- | --- | --- | --- |
| Compiler explainer (placement, dataflow, static hints) | Start now | Source-linked dataflow graph with static hints | Operation IDs, source maps, dependency edges |
| Runtime flight recorder | Viable once IDs exist | Bounded event history keyed by operation ID, with a predicted-versus-observed diff | Same IDs; a device-timestamp source per backend |
| Timing prediction and what-if simulator | Conditional | Interval predictions with provenance on narrow workloads | Calibration data from the recorder; accuracy gates |

Observations about Vx describe the working tree at `c3039ed9` and are carried over from the earlier assessment dated October 6, 2026 without re-checking. Findings about other tools come from a limited web search and are listed under Existing tools and gaps.

## Proposal: two tools joined by operation IDs

The compile-time tool explains what the compiler decided and what it expects. The runtime tool records what happened. A diff step compares the two and feeds the residuals back into the cost model.

1. **Compiler explainer.** Emits a dataflow graph with stable operation IDs, source locations, destinations, sizes, selected routes and lowerings. Optional load metrics (entries per tensor, sizes, types) turn symbolic sizes into concrete ones. Each operation carries a prediction record.
2. **Runtime flight recorder.** Keeps a bounded history of allocations, copies, launches, waits, completions and failures under the same IDs, with host and device timestamps kept separate.
3. **Diff and calibration.** Joins both by operation ID, reports where observed behavior fell outside the predicted interval, and stores the residuals as calibration data.

&#91;embedded content: feedback loop · 4 steps\]

The loop is what makes prediction trustworthy over time. Each prediction becomes a checkable claim, and each miss either improves the model or exposes a compiler or system problem. Measuring first and simulating second is also the order used by FlexFlow and Daydream (see Research precedents).

## Compiler decisions and memory flow

Vx already reports the raw material for an explainer but not yet an execution graph. The `--diagnostics-json` output (`Vx/src/diagnostics_json.rs`) publishes transfer routes, byte counts, cost provenance, resident memory totals, capacity margins, and traffic per kernel region and placed buffer. Counts can be exact, upper bounds, or unavailable.

Three gaps must close before a viewer is reliable (phase 0):

- **Operation identities and source mappings.** The report types (`Vx/src/report.rs`) need both for navigation and for correlation with runtime events.
- **Producer–consumer edges.** Reports publish traffic weights but not all dependencies.
- **Buffer lifetimes.** Aggregate resident totals cannot establish lifetimes or a precise peak-memory curve.

Program order plus dependencies can answer where a copy occurs in execution without predicting its timestamp. XProf's Memory Viewer takes this approach: it shows compiler-derived buffer lifetimes against program order and says explicitly that the axis is not elapsed time. Independent operations must stay independent in the display, which must not invent an execution order.

Vx validates declared placement and selects transfer routes. A future placement optimizer is a separate responsibility (`Vx/docs/memory_algebra.md`), so the viewer should show the placement the programmer requested and explain the decisions the compiler actually makes.

## Compile-time tool: dataflow graph and prediction records

The tool turns Vx code into a dataflow graph and attaches a prediction to every operation. Nodes are allocation, transfer, consumer execution, readback and release. Edges are dependencies. Selecting a node shows its source location, destination, size, selected route, lowering and validation obligations.

Load metrics are optional. With them, symbolic sizes become concrete. Without them, the tool reports symbolic or bounded values and does not guess.

The tool also shows the mechanism the compiler selected, because API names mislead. In the CUDA backend (`Vx/runtime/cuda_dispatch.cpp`), device-image execution synchronizes before dispatch returns and `vx_plugin_await_future` does nothing. In raw operation lowering (`Vx/src/codegen/lower/mod.rs`), `raw::async_copy` currently emits synchronous loads and stores. A timeline drawn from API names would show overlap that these paths do not provide. These two observations describe the inspected working tree and are the claims most likely to go stale.

Each operation gets a prediction record:

| Field | Meaning |
| --- | --- |
| Operation ID | Stable key shared with the recorder |
| Predicted range | A low–high interval, not a single number |
| Cost source | Fact, conservative bound, estimate, or calibrated measurement |
| Assumptions | Shapes, backend, lowering and dynamic-input assumptions the range depends on |
| Status | Exact, upper bound, or unknown; unsupported cases stay unknown |

Why intervals and not cycle counts is covered under Limits of timing prediction.

## Hints

Hints split into two classes with different readiness: rules over static facts can ship with the first viewer, while what-if hints must wait for an accurate cost model.

- **Static rule hints (early).** These need no timing model. Examples: a buffer re-transferred across regions; a tensor read repeatedly from a slower memory level; a placement that exceeds capacity or its margin; declared layouts or strides that are misaligned; chains of dependent indirect loads (pointer chasing), flagged but not quantified because their cost depends on data.
- **What-if hints (gated).** A claim such as "moving X to another memory level saves N cycles" needs calibrated costs. It must also account for any changed lowering or compute performance. Withhold these until the accuracy gates under Phasing are met.

Alignment is mostly checkable at compile time from declared layouts and strides. The recorder can log actual pointer alignment at allocation and transfer events. Whether misalignment cost anything needs measured throughput, which the diff step supplies.

## Runtime recorder: promised versus actual

The recorder logs what each operation actually did and compares it with the compiler's prediction for the same operation ID. Fixed placement does not imply fixed execution: sizes, invocation counts, backend selection, allocation failures and delays can vary.

The Vx runtime ABI (`Vx/include/vx_hardware_runtime.h`) provides natural recording points for allocation and transfer, dispatch, await, readback, peer transfer and release. [dial9](https://github.com/dial9-rs/dial9) is the design precedent: collect individual events, retain them, and inspect causes after a problem. Its Tokio instrumentation would not reveal Vx transfers or GPU execution, so Vx needs its own semantic events.

Proposed event fields:

- Operation ID and buffer identity, linking the event to the compiler plan.
- Event kind, bytes, and source and destination.
- Pointer address, alignment and stride at allocation and transfer.
- Host-call timestamps and device-execution timestamps, kept separate.
- Outcome, including failures.

Retention is a bounded history. Lost events must be detectable; the Perfetto [buffer documentation](https://perfetto.dev/docs/concepts/buffers) describes bounded recording with loss detection. What triggers a dump (failure, timeout, manual request) is an open decision.

Device timing is backend-specific. NVIDIA's [CUPTI Activity API](https://docs.nvidia.com/cupti/main/main.html) provides GPU activity records and correlation IDs linking CUDA calls to kernels and copies, but it is NVIDIA-only while the runtime ABI is backend-neutral. Each backend needs its own device-timestamp source. [Perfetto](https://perfetto.dev/docs/instrumentation/tracing-sdk) offers an existing event format and viewer. Before building a custom recorder, compare it with a reuse baseline: NVTX ranges carrying Vx operation IDs, consumed by Nsight Systems and CUPTI tooling.

Three limits apply. Ordinary runtime hooks do not record memory operations inside a kernel. Logical read counts are not physical HBM traffic, because caches and backend transformations affect what reaches memory. Recording overhead can change the overlap being observed, so it must be measured and reported with the results.

## Limits of timing prediction

Knowing the backend at compile time fixes the instruction set and the lowering, but not what the hardware does at run time, so most predictions should be intervals with provenance and not cycle counts. Warp scheduling, cache state, clock throttling, contention from other processes and driver behavior all vary. Cycle-exact prediction is realistic only for deterministic parts: statically scheduled accelerators, fixed-latency DMA engines, and pure compute on a quiet device. When a measurement lands outside its interval, the model or the system is wrong, and that is useful to know.

The current Vx cost model adds its own limits:

- The memory-algebra assessment documents missing transfer startup latency, unsettled multi-hop composition, and contention detection without a calibrated sharing law. It also records instrument errors that confused memory levels.
- `total_cost` in the diagnostics schema is a unitless route-selection weight and cannot be used as an event duration.
- `derived_cost` is a bandwidth estimate with units, not an observed duration.
- Region traffic is reported per launch. Summing static sites does not give total traffic for a run with loops or conditional execution.

Published work points the same way. An analytical model for pipelining in a deep learning compiler, [ALCOP](https://arxiv.org/pdf/2210.16691), notes that such models cannot be very accurate because memory-system factors are hard to capture, while learned models need large amounts of measured data. A study of [auto-vectorizer cost models](https://www.iris.unisa.it/handle/11386/4753087) found speedup is generally over-estimated, with weak to medium correlation between predicted and actual gains.

A credible simulator needs dependencies, actual execution mechanisms, resource and queue constraints, compute costs, calibrated transfer costs, and assumptions about dynamic inputs. Facts, conservative bounds, estimates and measurements must stay visibly distinct, and unsupported cases must stay unknown.

## Existing tools and gaps

The closest precedent is the SC22 data-movement visualization by Schaad, Ben-Nun and Hoefler. No tool found combines source-linked compiler decisions, predictions with provenance, and a comparison against a runtime recording.

| Tool | What it covers | Gap relative to this proposal |
| --- | --- | --- |
| [Schaad, Ben-Nun and Hoefler (SC22)](https://arxiv.org/pdf/2207.07433) | Data movement and reuse analysis without running the program, using static dataflow analysis plus parameterized simulations, shown on the program representation | Published 2022; no runtime comparison found; fit to heterogeneous AI frameworks not established |
| [Triton-Viz](https://github.com/Deep-Learning-Profiling-Tools/triton-viz/blob/main/README.md) | Visualizes tensor operations and memory usage for Triton; examples need no GPU | Triton only; no predictions |
| [Triton-Sanitizer](https://hgpu.org/?p=30696) | Memory access errors found from symbolic address and mask expressions checked by an SMT solver, with eager simulation for indirect accesses | Correctness, not performance |
| [GPA](https://arxiv.org/pdf/2009.04061) and [GPUscout](https://ce.cit.tum.de/fileadmin/w00cgn/caps/vanecek/gpuscout_protools.pdf) | Static analysis of GPU binaries combined with profiles or metrics; GPA estimates the speedup of each suggestion | Binary level; needs a run |
| [Modular MAX](https://docs.modular.com/develop/debugging-errors/) | An IR dump per lowering stage (fusion, memory planning, device placement) through `ir-output-dir`; the older [max visualize](https://docs.modular.com/max/visualize) showed a pre-optimization graph in Netron | Files to diff; no memory-flow viewer found |
| [IREE](https://opensecura.googlesource.com/3p/openxla/iree/+/8d5cf9689fdbbfa646e452842ecc11ee15997b92%5E) | Stream dialect handles device placement and asynchronous scheduling; an option dumps scheduling statistics such as dispatch counts | Dumps and statistics; no viewer found |
| [XProf Memory Viewer](https://openxla.org/xprof/memory_viewer) | Compiler-derived buffer lifetimes against program order | Memory only; no runtime comparison |
| LLVM optimization remarks and GCC `-fopt-info`; `llvm-mca`, uiCA, OSACA | Missed-optimization reasons; static CPU throughput prediction | Hard to read; not tied to data movement; assume warm caches. Listed from general knowledge, not re-checked here |

The search behind this table was limited to five queries read as snippets, and the pages were not opened in full. Not finding a Mojo or Vx-style compile-time memory viewer is a lead to confirm, not proof that none exists.

## Research precedents

Vx targets AI frameworks (inference, a PyTorch equivalent, and similar), so deep-learning placement simulators are its closest precedents. Four gaps remain between those systems and Vx.

- **[FlexFlow](https://arxiv.org/html/1807.05358v1)** (Jia, Zaharia and Aiken, 2018). Constructs compute and communication tasks and simulates their dependencies and resource queues. Compute durations come from measurements; communication uses a bandwidth model under explicit assumptions. It supports placement simulation within a defined workload model.
- **[Daydream](https://www.usenix.org/conference/atc20/presentation/zhu-hongyu)** (Zhu, Phanishayee and Pekhimenko, USENIX ATC 2020). Builds a dependency graph from CUPTI traces, transforms it to represent optimizations, and simulates execution. It is a direct precedent for using recorded behavior to support later what-if analysis.
- **[XProf Memory Viewer](https://openxla.org/xprof/memory_viewer).** Shows static memory planning, buffer lifetimes and source attribution before any timing simulation.
- **[Schaad, Ben-Nun and Hoefler, SC22](https://arxiv.org/pdf/2207.07433).** Static dataflow analysis plus parameterized simulations for data-movement views without execution.

The gaps to close for Vx:

1. **Graph derivation.** In these systems the framework supplies the dependency graph. Vx must derive it, which is phase 0.
2. **Compiler-generated kernels.** Profiled operations are fixed units in FlexFlow. In Vx a placement change can change the lowering and the compute time, so a duration measured under one placement may not carry over to another.
3. **Execution model.** These systems assume asynchronous overlap with resource queues. Vx's current CUDA path synchronizes at dispatch, so the model must describe what Vx does today.
4. **Inference dynamics.** Variable sequence lengths, KV-cache growth and batching likely make inference less regular than the training workloads in these papers. Calibration matters more, and unknown must stay unknown.

None of this establishes accuracy for arbitrary Vx programs. It supports starting with static-shape tensor graphs and widening only as measurements allow.

## Phasing and acceptance gates

Phase 0 is the critical path: stable operation IDs, source maps, dependency edges and buffer lifetimes. Two gates separate it from the explainer and the recorder from calibrated what-if hints.

&#91;embedded content: roadmap · 4 phases, 2 gates\]

Phases 1 and 2 both depend only on phase 0, so the recorder can start alongside the explainer; the predicted-versus-observed diff needs both. The gate criteria name no numbers, and the team must set the thresholds. Gate 2 is the answer to when the simulator should be revisited: after the recorded diffs show predictions holding on named workloads, not before.

## Open questions and risks

- **Missed-optimization hints.** Detecting a missed optimization requires a model of what should have happened. Can Vx state that expectation for its own passes, or only report what it did?
- **Retention triggers.** Should the recorder dump on failure, on timeout, on request, or all three?
- **Build or reuse.** Is NVTX plus Nsight Systems, or Perfetto's event format, enough, or does Vx need its own recorder and viewer?
- **Backend-neutral device timestamps.** What is the device-time source on each non-NVIDIA backend?
- **Perturbation.** How much does recording change overlap and timing, and how is that reported?
- **First calibration workloads.** Static-shape inference graphs first, then training steps, is the suggested order; the team should name the actual set.
- **Gate thresholds.** The accuracy gates below name the criteria but not the numbers; the team must set them.
- **Gap confirmation.** The newer-tools search was limited, so the claim that no integrated tool exists should be re-checked before it is stated externally.

## Sources

Sources from the original assessment and from the newer search. Pages from the newer search were read as search snippets, not opened in full.

- [FlexFlow: Beyond Data and Model Parallelism for Deep Neural Networks](https://arxiv.org/html/1807.05358v1)
- [Daydream: Accurately Estimating the Efficacy of Optimizations for DNN Training](https://www.usenix.org/conference/atc20/presentation/zhu-hongyu)
- [XProf Memory Viewer](https://openxla.org/xprof/memory_viewer)
- [dial9](https://github.com/dial9-rs/dial9)
- [NVIDIA CUPTI](https://docs.nvidia.com/cupti/main/main.html)
- [Perfetto tracing SDK](https://perfetto.dev/docs/instrumentation/tracing-sdk) and [buffer documentation](https://perfetto.dev/docs/concepts/buffers)
- [Boosting Performance Optimization with Interactive Data Movement Visualization (SC22)](https://arxiv.org/pdf/2207.07433)
- [Triton-Viz](https://github.com/Deep-Learning-Profiling-Tools/triton-viz/blob/main/README.md)
- [Triton-Sanitizer](https://hgpu.org/?p=30696)
- [GPA: A GPU Performance Advisor](https://arxiv.org/pdf/2009.04061)
- [GPUscout slides](https://ce.cit.tum.de/fileadmin/w00cgn/caps/vanecek/gpuscout_protools.pdf)
- [MAX: debugging errors](https://docs.modular.com/develop/debugging-errors/) and [MAX: visualize a model](https://docs.modular.com/max/visualize)
- [IREE scheduling statistics commit](https://opensecura.googlesource.com/3p/openxla/iree/+/8d5cf9689fdbbfa646e452842ecc11ee15997b92%5E)
- [ALCOP](https://arxiv.org/pdf/2210.16691)
- [Portable cost modeling for auto-vectorizers](https://www.iris.unisa.it/handle/11386/4753087)

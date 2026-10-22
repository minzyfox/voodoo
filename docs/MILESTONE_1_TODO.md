# Voodoo milestone 1 TODO

Milestone 1 connects a small Vx computation from source to observed execution on one CUDA GPU. A programmer should be able to identify the chosen backend, copies introduced inside dispatch, buffer allocation and reuse, and synchronization that prevents overlap.

Status: not started. All implementation and validation tasks below remain open. Scope and later milestones are defined in the [roadmap](ROADMAP.md). The implementation baseline is Vx `33e727d18696560e51aa089ef4dfe764498839eb`, assessed October 7, 2026.

## Deliverables

- [ ] A versioned compiler manifest containing source origins, operation relationships, placements and selected mechanisms for the supported workloads.
- [ ] An opt-in execution trace joined to that manifest, including internal CUDA staging, allocation, copy, launch and synchronization events.
- [ ] A compact explanation report and a trace readable in an existing viewer.
- [ ] Reproducible workload variants, expected structural behavior and a validation report covering attribution, timing, correctness and recording overhead.

The milestone is deliberately limited to one process, one CUDA GPU, static tensor shapes and a controlled number of host-loop iterations. Include a generated GPU kernel and a cuBLAS-routed computation. A host fallback, if exercised, must be labeled as such. Complete program graphs, automatic placement, duration predictions, production crash retention and real asynchronous overlap are deferred.

## Workloads and expected behavior

- [ ] Build the main example from existing [GPU matmul tests](../../Vx/tests/backend/pass/gpu_matmul_roles.vx), [dtype coverage](../../Vx/tests/backend/pass/gpu_matmul_dtypes.vx) and [matmul workloads](../../Vx/benchmarks/matmul_execution.vx). Select a CUDA-compatible fixture; do not assume an NPU example runs unchanged on CUDA.
- [ ] Create two numerically equivalent variants: inputs transferred once and kept resident across repeated matmuls, and host inputs staged on each invocation. Define the expected input-copy counts and byte totals for each variant before collecting traces.
- [ ] Add a small generated-kernel workload that does not use cuBLAS. Record the expected dispatch path explicitly.
- [ ] Include repeated execution of the same source site so invocation identity is exercised. Include allocation reuse so an address reused later cannot be mistaken for the original allocation.
- [ ] Specify shapes, dtypes, layout, iteration counts, warmup, output checking and whether reported times include initialization or JIT compilation.
- [ ] Record the required GPU and tool versions and choose the independent CUDA tracing baseline. GPU validation is required to complete the milestone even if compiler and host-side checks run elsewhere.

## Language and execution contracts

These tasks clarify existing behavior and metadata. New profiling syntax is not a prerequisite.

- [ ] Write down the meaning of requested topology, compiler-selected lowering and actual runtime backend. Report them separately when a runtime choice or fallback is possible.
- [ ] Define submission, start, completion, await and release events. Describe current synchronous dispatch honestly; the word `async` must not create an inferred overlap or an invented queue.
- [ ] Document the storage ownership and completion conditions used by the supported workloads. Ensure a reported release corresponds to the actual owner or temporary storage action, not merely the last appearance of a variable name.
- [ ] Preserve existing numerical and placement behavior with recording enabled. Any correction to execution or timing instrumentation should be a separately reviewable change with focused validation.
- [ ] Define the coverage boundary for arbitrary calls, aliases, dynamic shapes and raw kernel memory operations. Unsupported detail gets an absence reason; it is not silently counted as zero.

Start from the [runtime ABI](../../Vx/include/vx_hardware_runtime.h), [Vx operation definitions](../../Vx/include/VxDialect.td) and [raw primitive lowering](../../Vx/src/codegen/lower/mod.rs).

## Compiler identities and source attribution

- [ ] Define a build identity that binds the manifest to its executable and compilation configuration. Reject joins with a different build instead of matching similar-looking names.
- [ ] Define source-operation IDs and generated-operation IDs. Require deterministic IDs for the same inputs and configuration, including parallel compilation. Do not promise unchanged IDs across source edits or compiler versions.
- [ ] Carry origin relationships through outlining, expansion and supported fusion paths. One source operation may produce several generated operations; a generated operation may have several source origins.
- [ ] Reuse existing file, line and column locations from [the generator](../../Vx/src/codegen/generator.rs). Preserve source paths and a source-content identity so navigation can detect a changed source file.
- [ ] Introduce buffer/value identities independent of display names. Keep aliases or views distinct from their underlying storage where the supported workload exposes them.
- [ ] Carry the required operation identity into runtime calls or dispatch metadata. Choose an ABI-compatible extension or an explicitly versioned contract; test behavior when metadata is absent.

## Compiler explanation manifest

- [ ] Choose the export points: capture source intent before lowering and capture selected mechanisms after relevant lowering decisions. Avoid presenting an early plan as a complete description of the final executable.
- [ ] Define and document the manifest schema and compatibility rules. Reuse [existing reports](../../Vx/src/report.rs) and [diagnostics facts](../../Vx/src/diagnostics_json.rs) where appropriate without silently changing existing consumers.
- [ ] Export operation kind, origins, placement, route, shape, dtype, byte count, selected mechanism and the available decision reason. Distinguish a compiler decision from a backend choice only known at runtime.
- [ ] Preserve facts, conservative bounds, estimates and unknown values as different states. Keep units attached to estimates. Do not turn `total_cost` into elapsed time.
- [ ] Record whether logical traffic is per launch, exact or bounded, and why it is absent when uncountable. Preserve the distinction from measured physical memory traffic.
- [ ] Export only dependencies needed to explain the selected workloads, including direct data use and relevant synchronization. Label graph coverage explicitly; do not infer independence from missing edges.
- [ ] Reuse existing transfer-free and drop information from [lowering](../../Vx/src/dialect/VxLowering.cpp). Export supported allocation/release relationships without claiming a complete peak-memory model.

## Runtime events and correlation

- [ ] Define run, invocation and runtime-event identities. Repeated executions of one static operation need distinct invocation IDs, and internal work needs its parent invocation.
- [ ] Record buffer allocation IDs with reuse generations or equivalent unique identities. Addresses may be useful attributes, but are not identities.
- [ ] Record device, thread and stream or execution context when known. Include bytes, direction, shape or stride where relevant, outcome, and the clock domain of each timestamp.
- [ ] Record the actual dispatch path: cuBLAS, generated device image, host execution or failure. Capture the reason when available.
- [ ] Instrument supported public [plugin entry points](../../Vx/include/vx_hardware_runtime.h) for allocation/transfer, dispatch, await, readback and release.
- [ ] Instrument internal work in [the CUDA backend](../../Vx/runtime/cuda_dispatch.cpp): `stage`, `run_gemm`, device-image launch, output copies, temporary allocations and pool acquire/release. Link each to the containing invocation.
- [ ] Distinguish logical buffer release, return to a pool and physical device deallocation. Report tracked live bytes and pool retention separately, with their coverage; neither is total device memory by default.
- [ ] Keep compile-time facts and observed runtime attributes separately inspectable. Report unexpected copies or mechanisms without overwriting the original plan.

A full recorder for every backend is outside this milestone. Every selected path must either produce its required events or report that coverage is incomplete.

## Timing and trace output

- [ ] Compare a Perfetto-based semantic trace with a small NVTX/Nsight or CUPTI baseline, then select and document one implementation path. Do not build a new timeline renderer for this milestone.
- [ ] Use a monotonic host clock. Distinguish host API duration, synchronization duration and device execution duration. Declare clock mapping uncertainty when combining domains.
- [ ] Audit `VX_TIME_KERNEL`. In the baseline device-image path, the stop event is submitted after `cudaDeviceSynchronize`; check whether this includes an idle gap and correct the instrument before reusing its measurements.
- [ ] Obtain independent device timing and copy counts using CUDA profiling tooling. Avoid adding synchronizations merely to make event reporting convenient.
- [ ] Make recording opt-in and impose a capture budget. Emit explicit loss, truncation or incomplete-capture status when the budget is exhausted; long-running rolling retention belongs to milestone 3.
- [ ] Export metadata sufficient to interpret the saved trace without access to the live process. Support a clean end-of-run flush and a manual capture stop for the demonstration.
- [ ] Record errors on the supported paths where feasible. Do not claim survival of abrupt termination before durable recording is implemented and tested.

References: [Perfetto SDK](https://perfetto.dev/docs/instrumentation/tracing-sdk), [Perfetto buffers](https://perfetto.dev/docs/concepts/buffers) and [CUPTI correlation](https://docs.nvidia.com/cupti/main/main.html).

## Explanation report

- [ ] Given a source operation, show its requested placement, known compiler decisions, actual backend and all recorded invocations with their child work.
- [ ] Show copy counts and bytes, including staging and readbacks hidden inside dispatch. Explain the resident-versus-restaged workload difference using this evidence.
- [ ] Show synchronization and distinguish observed serialization from a claim that language semantics require serialization in all implementations.
- [ ] Show supported buffer allocation, release and pool events. Leave a full lifetime graph and peak-memory explanation to milestone 2.
- [ ] Mark unsupported facts, incomplete traces, lost events and absent device timings visibly. Refuse a manifest/trace build mismatch.
- [ ] Provide machine-readable output as well as the concise human report, so validation can inspect attribution and counts without scraping a visualization.

## Validation and completion gate

- [ ] Numerical results agree between workload variants and between recording enabled and disabled.
- [ ] Every required event in the selected workload maps to the correct invocation and source origin, or is explicitly labeled runtime setup. There are no unexplained in-scope operations and no event loss in acceptance captures.
- [ ] Expected repeated-invocation counts, input-copy counts and byte totals match the trace. A deliberately added restaging operation changes both the report and independent CUDA evidence.
- [ ] Source mapping survives supported outlining and expansion. Identity checks distinguish repeated calls, source-name collisions, allocation reuse and mismatched builds.
- [ ] Lifetime-related records agree with actual allocation and pool actions for the selected examples. Extend existing transfer-free coverage rather than claiming complete lifetime analysis from this fixture.
- [ ] Device timing is cross-checked against the independent baseline under a declared repetition and warmup policy. Define the acceptable discrepancy before the comparison; record both variability and any remaining bias.
- [ ] Measure disabled and enabled overhead, capture size and memory use on small and larger workloads. Set a development-use budget before acceptance and report the results; production always-on overhead is a later target.
- [ ] Exercise an undersized capture budget and missing metadata to prove the tool reports incomplete evidence instead of presenting a complete-looking explanation.
- [ ] Demonstrate that a developer can identify repeated staging or unnecessary serialization from the saved artifacts without reading runtime source. Document the explanation and the behavior change it supports.
- [ ] Save the reproduction commands, exact build/device metadata, artifact paths, coverage limits and results. Mark the milestone complete only when all required acceptance evidence is available.

Compiler contract checks, runtime event checks and the GPU experiment should verify observable behavior. Do not add tests that only repeat the implementation's internal structure. Apply the Vx repository's test and change guidance when implementation begins.

## Suggested implementation order

1. Freeze workload expectations and the build/source/invocation identity contract.
2. Export a minimal compiler manifest with source origins and mechanisms.
3. Trace one complete dispatch, including its internal copies and allocations, and join it to the manifest.
4. Add repeated invocations, residency variants, generated-kernel coverage and the explanation report.
5. Validate timing, event coverage, loss behavior and overhead; produce the acceptance artifacts.

Defer full alias/dependency recovery, new stream scheduling, multi-GPU and remote execution, dynamic inference workloads, automated hints about saved cycles, and placement optimization to the roadmap milestones that need them.

# Microsoft Agent Framework XPU Enablement Report

## Summary

Microsoft Agent Framework orchestrates models but does not run inference itself;
hosted or local services perform the computation. XPU enablement focuses on
validating existing local-service integrations on Intel GPU hardware.

- **Ollama chat:** Complete. All four chat integration tests passed with full
  Intel GPU offload and no framework changes.
- **Ollama embeddings:** Complete. The embedding integration test passed with
  model and compute buffers allocated on the Intel GPU.
- **Foundry Local:** Complete. Foundry Local CLI 0.10.3 detected four Intel Arc
  Pro B60 GPUs on Linux and initialized successfully, but exposed only CPU
  model variants. There is no supported Linux Intel GPU path today.

## Repos Analyzed

| Repository | Role | Recommendation |
|---|---|---|
| [`microsoft/agent-framework`](https://github.com/microsoft/agent-framework) | Core framework: Python + .NET packages, provider clients, samples, tests | Ollama integration validated on Intel GPU; no framework changes needed |
| [`microsoft/Foundry-Local`](https://github.com/microsoft/Foundry-Local) | External local-inference runtime consumed by `agent-framework-foundry-local` | No Linux Intel GPU path exists today; out of scope |

## Testing

### Unit Test Analysis (Summary)

As of WW29, the Agent Framework Python unit test status:

| Metric | Count | Notes |
|---|---:|---|
| Total unit tests across run scope | 8,529 | Excludes `devui` and `lab` packages, integration tests, and all .NET tests |
| Pure software unit tests | 8,529 | No GPU/XPU/CUDA-specific test paths, markers, or hardware dependencies |
| Passing pure software unit tests | 8,241 | Plus 2 xfail (not included in passing count) |
| Blocked pure software unit tests | 286 | 286 skipped |
| Failed pure software unit tests | 0 | |
| XPU-runnable unit tests | 0 | |
| Passing XPU-runnable unit tests | 0 | |
| Blocked XPU-runnable unit tests | 0 | |
| Failed XPU-runnable unit tests | 0 | |
| Newly added XPU-specific unit tests | 0 | |

**Key takeaway:** unit-test coverage is software-only. Tests provide regression
signal for configuration plumbing but do not prove XPU execution.

### Existing Ollama Test Coverage

| Tests | Coverage | Execution | XPU Relevance |
|---|---|---|---|
| [`test_cmc_integration_with_chat_completion`](python/packages/ollama/tests/test_ollama_chat_client.py) | Non-streaming chat completion against live Ollama | Opt-in; skipped unless `OLLAMA_MODEL` is set. **Passed on XPU (WW33)** | Full chat path; container logs confirm 25/25 layers on Intel GPU |
| [`test_cmc_streaming_integration_with_chat_completion`](python/packages/ollama/tests/test_ollama_chat_client.py) | Streaming chat completion against live Ollama | Same condition. **Passed on XPU (WW33)** | Streaming variant of the above |
| [`test_cmc_integration_with_tool_call`](python/packages/ollama/tests/test_ollama_chat_client.py) | Non-streaming chat with function tool invocation | Same condition. **Passed on XPU (WW33)** | Tool-calling on Intel GPU |
| [`test_cmc_streaming_integration_with_tool_call`](python/packages/ollama/tests/test_ollama_chat_client.py) | Streaming chat with function tool invocation | Same condition. **Passed on XPU (WW33)** | Streaming + tool-calling on Intel GPU |
| [`test_ollama_embedding_integration`](python/packages/ollama/tests/ollama/test_ollama_embedding_client.py) | Generates embeddings for two inputs and validates non-empty float vectors | Opt-in; skipped unless `OLLAMA_EMBEDDING_MODEL` is set. **Passed on XPU (WW33)** | Embedding path; container logs confirm model and compute buffers on Intel GPU |

### XPU Coverage Status

| Test | Status |
|---|---|
| Ollama chat completion on Intel GPU | **4/4 passing, GPU confirmed** (WW33) — zero framework changes |
| Ollama embedding generation on Intel GPU | **1/1 passing, GPU confirmed** (WW33) — zero framework changes |
| Foundry Local on Intel GPU | **Smoke test complete; unsupported on Linux** — hardware detected, but no Intel-compatible GPU variants available |

An XPU test must confirm that model computation uses the Intel GPU. A successful
response alone is not sufficient; GPU offload must be independently observed.

## Contributions

| # | Contribution | Status | Acceptance criteria |
|---|---|---|---|
| E2E-1 | Validate Ollama chat and embedding integration tests against XPU-backed Ollama | **COMPLETE (WW33)** | 5/5 tests pass, GPU offload confirmed via container logs |
| Smoke Test | Validate Foundry Local on Intel GPU hardware | **COMPLETE — failed as expected** | Runtime initialized, GPUs detected, and GPU catalog queried; no compatible variants available |
| PR-1 | Document Intel GPU configuration and prerequisites for Foundry Local | Not viable | No supported configuration exists on Linux |
| PR-2 | Add configuration regression test for Intel GPU device selection in Foundry Local | Not viable | Device forwarding cannot provide missing runtime support |
| PR-3 | Add opt-in Foundry Local integration tests and companion sample | Not viable | No Intel GPU variant is available to test |
| Issue-1 | Request Linux Intel GPU support from Foundry Local | Optional upstream follow-up | Request accepted and tracked upstream |

## XPU Test Plan

### E2E-1: Ollama chat and embeddings on Intel XPU

**Goal:** Prove the framework's Ollama client paths work on Intel XPU with no
framework code changes.

- Run native Ollama with Vulkan backend on Intel XPU host
- Load chat and embedding models, verify GPU offload
- Run the 5 existing opt-in integration tests
- Confirm GPU utilization via container logs

**Status: COMPLETE.**

On an Intel Arc Pro B60 using Ollama's Vulkan backend, all five tests passed (four chat and one embedding). Logs confirmed full GPU offload and allocation.

### Smoke Test: Foundry Local on Intel GPU

**Goal:** Confirm whether Foundry Local can run inference on Intel GPU on Linux.

- Installed Foundry Local CLI 0.10.3 on Ubuntu 25.10 with four Intel Arc Pro B60 GPUs
- Started the runtime and confirmed that it detected the Intel GPUs
- Queried all GPU model variants with `foundry model list --device gpu --variants --verbose`

**Result:** Failed as expected. The GPU catalog was empty, while the same runtime
returned CPU variants using `CPUExecutionProvider`.

**Rationale:** Hardware detection does not enable inference by itself. Foundry
Local also needs a compatible ONNX Runtime execution provider. Its OpenVINO path
is provided through Windows-only WinML, WebGPU is not supported on Linux, and
its Linux GPU path is CUDA for NVIDIA hardware. With no Intel-capable provider,
there is no GPU model to run; an alias would select a CPU variant instead.

**Status: COMPLETE.**

### Upstreaming

No Agent Framework PR is warranted. Ollama already works without framework
changes, while Foundry Local lacks Linux Intel GPU runtime support. The proposed
documentation, configuration test, integration tests, and sample therefore do
not justify OSPDT approval. An upstream Foundry Local enhancement request is a possible
follow-up.

## Next Steps

- [x] Run the Agent Framework Python unit suite: **8,241 passed** (WW29 baseline)
- [x] Analyze the repo for local-inference provider paths and integration-test patterns
- [x] Confirm no in-process ML frameworks in the Python packages
- [x] Run Ollama chat integration tests on XPU: **4/4 passed** (WW33)
- [x] Run Ollama embedding integration test on XPU: **1/1 passed** (WW33)
- [x] Confirm Intel GPU offload for both chat and embedding models
- [x] Inspect `foundry-local-sdk` device enum: CPU, GPU, NPU present; XPU is not
- [x] Review `microsoft/Foundry-Local` project for XPU support
- [x] Run the Foundry Local smoke test on Intel GPU hardware: **no GPU variants available**
- [x] Conclude PR-1, PR-2, and PR-3 are not viable for Linux Intel GPU enablement
- [ ] Optional: Request Linux Intel GPU support from Foundry Local upstream

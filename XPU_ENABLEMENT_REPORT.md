# Microsoft Agent Framework XPU Enablement Report

## Summary

Microsoft Agent Framework orchestrates models but does not run inference itself;
hosted or local services perform the computation. XPU enablement focuses on
validating existing local-service integrations on Intel GPU hardware.

- **Ollama chat:** Complete. All four chat integration tests passed with full
  Intel GPU offload and no framework changes.
- **Ollama embeddings:** Complete. The embedding integration test passed with
  model and compute buffers allocated on the Intel GPU.
- **Foundry Local:** Initial investigation of `microsoft/Foundry-Local` reveals
  that its Intel GPU execution providers (OpenVINO via WinML, WebGPU via DX12) are
  Windows-only. There is no Linux Intel GPU path today. Probably out of scope.

## Repos Analyzed

| Repository | Role | Recommendation |
|---|---|---|
| [`microsoft/agent-framework`](https://github.com/microsoft/agent-framework) | Core framework: Python + .NET packages, provider clients, samples, tests | Ollama integration validated on Intel GPU; no framework changes needed |
| [`microsoft/Foundry-Local`](https://github.com/microsoft/Foundry-Local) | External local-inference runtime consumed by `agent-framework-foundry-local` | No Linux Intel GPU path exists today; probably out of scope |

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
| Foundry Local on Intel GPU | Not possible on Linux; Intel GPU execution providers are Windows-only |

An XPU test must confirm that model computation uses the Intel GPU. A successful
response alone is not sufficient; GPU offload must be independently observed.

## Contributions

| # | Contribution | Status | Acceptance criteria |
|---|---|---|---|
| E2E-1 | Validate Ollama chat and embedding integration tests against XPU-backed Ollama | **COMPLETE (WW33)** | 5/5 tests pass, GPU offload confirmed via container logs |
| Smoke Test | Validate Foundry Local chat completion on Intel GPU hardware | Not started; now expected to fail based on code review | Correct response, confirmed Intel GPU use, no CPU fallback |
| PR-1 | Document Intel GPU configuration and prerequisites for Foundry Local | Unlikely; depends on Smoke Test | Documentation reflects the proven device value, model variant, and execution provider |
| PR-2 | Add configuration regression test for Intel GPU device selection in Foundry Local | Unlikely; depends on Smoke Test | Unit test proves the device value is forwarded correctly; configuration coverage, not hardware support |
| PR-3 | Add opt-in Foundry Local integration tests and companion sample | Unlikely; depends on Smoke Test | Tests cover non-streaming, streaming, and tool-calling against a real `FoundryLocalClient` on Intel GPU |
| Issue-1 | Open a tracking issue describing the Intel XPU enablement plan | Proposed | Issue triaged and linked from PRs |

## XPU Test Plan

### E2E-1: Ollama chat and embeddings on Intel XPU

**Goal:** Prove the framework's Ollama client paths work on Intel XPU with no
framework code changes.

- Run native Ollama with Vulkan backend on Intel XPU host
- Load chat and embedding models, verify GPU offload
- Run the 5 existing opt-in integration tests
- Confirm GPU utilization via container logs

**Status: COMPLETE.**

| Component | Value |
|---|---|
| GPU | Intel Arc Pro B60 (BMG G21), 23.9 GiB discrete |
| CPU | Intel Xeon 6980P, 237 GiB system RAM |
| Compute backend | Vulkan (`OLLAMA_VULKAN=true`, `GGML_VK_VISIBLE_DEVICES=0`) |
| Chat model | `qwen2.5:0.5b` — 25/25 layers on GPU |
| Embedding model | `nomic-embed-text` — 216 MiB model + 92 MiB compute on GPU |
| Chat result | 4 passed in 5.03s |
| Embedding result | 1 passed |

### Smoke Test: Foundry Local on Intel GPU

**Goal:** Confirm whether Foundry Local can run inference on Intel GPU on Linux.

- Install Foundry Local runtime on the Intel GPU test host
- Attempt to load a model with `DeviceType.GPU`
- Observe whether the runtime selects the Intel discrete GPU or fails

**Expected outcome:** Failure. A code review of `microsoft/Foundry-Local` found
that the runtime's Intel-capable GPU paths (OpenVINO EP via WinML, WebGPU EP via
DX12) are both Windows-only. On Linux, only CPU and CUDA (NVIDIA) execution
providers are available. The smoke test will confirm this empirically.

**Status:** Not started.

### Upstreaming

All proposed PRs are tests, documentation, and sample code — expected to be a
small amount of code changes. Whether to submit through OSPDT depends on whether
the Foundry Local smoke test succeeds. E2E-1 (Ollama) required zero framework
changes and zero upstream PRs. If Foundry Local does require changes, batching
into one OSPDT submission keeps process cost low.

Update: a code review of `microsoft/Foundry-Local` found that its Intel GPU
execution providers are Windows-only, making the smoke test unlikely to succeed
on Linux. The Foundry Local PRs may not be viable.

- Submit documentation and configuration test justified by the smoke test (PR-1, PR-2)
- Submit opt-in integration tests and companion sample (PR-3)
- File an upstream request if Foundry Local cannot select Intel GPU

## Next Steps

- [x] Run the Agent Framework Python unit suite: **8,241 passed** (WW29 baseline)
- [x] Analyze the repo for local-inference provider paths and integration-test patterns
- [x] Confirm no in-process ML frameworks in the Python packages
- [x] Run Ollama chat integration tests on XPU: **4/4 passed** (WW33)
- [x] Run Ollama embedding integration test on XPU: **1/1 passed** (WW33)
- [x] Confirm Intel GPU offload for both chat and embedding models
- [x] Inspect `foundry-local-sdk` device enum: CPU, GPU, NPU present; XPU is not
- [x] Review `microsoft/Foundry-Local` project for XPU support
- [ ] Run the Foundry Local smoke test on Intel GPU hardware (now expected to fail)
- [ ] Prepare PR-1 (documentation) if smoke test succeeds
- [ ] Prepare PR-2 (configuration regression test) if smoke test succeeds
- [ ] Prepare PR-3 (opt-in integration tests and companion sample)
- [ ] Open Issue-1 describing the Intel XPU enablement plan
- [ ] Follow up on PRs through review and merge

# Microsoft Agent Framework XPU Enablement Report

## Summary

Microsoft Agent Framework is primarily an orchestration framework, not a model
runtime. It contains no in-process ML code paths; inference is always delegated to
an external service — hosted (Azure OpenAI, OpenAI, Anthropic, Foundry) or local
(Foundry Local, Ollama). XPU enablement focuses on two practical steps:

1. Run the existing Ollama integration tests against a local LLM server on Intel XPU.
2. Add configuration tests confirming that `FoundryLocalClient(device=DeviceType.XPU)`
   is accepted and threaded through to the underlying `foundry-local-sdk`, plus
   documentation and a CI job on XPU hardware.

The Ollama path (E2E-1) is now complete: all 4 integration tests pass with 100% GPU
offload on Intel Arc Pro B60 via Vulkan, with zero framework code changes. The
Foundry Local path (PR-1, PR-2) is proposed but blocked on confirming upstream SDK
support for `DeviceType.XPU`.

## Repos Analyzed

| Repository | Role | Recommendation |
|---|---|---|
| [`microsoft/agent-framework`](https://github.com/microsoft/agent-framework) | Core framework: Python + .NET packages, provider clients, samples, tests | Add configuration regression tests for `DeviceType.XPU`; add a Foundry Local integration CI job on Intel XPU hardware |
| [`microsoft/Foundry-Local`](https://github.com/microsoft/Foundry-Local) | External local-inference runtime consumed by `agent-framework-foundry-local` | Confirm `DeviceType.XPU` is exposed by the shipping `foundry-local-sdk`; file upstream request if not |
| [`ollama/ollama`](https://github.com/ollama/ollama) | External local-inference runtime consumed by `agent-framework-ollama` | Ollama integration tests are backend-agnostic and pass against XPU Ollama with no framework code changes |

## Testing

### Unit Test Analysis (Summary)

As of WW29, the Agent Framework Python unit test status:

| Metric | Count | Notes |
|---|---:|---|
| Total unit tests across run scope | 8,529 | Excludes `devui` and `lab` packages, integration tests, and all .NET tests |
| Pure software unit tests | 8,529 | No GPU/XPU/CUDA-specific test paths, markers, or hardware dependencies |
| Passing pure software unit tests | 8,241 | Plus 2 xfail (not included in passing count) |
| Blocked pure software unit tests | 286 | 286 skipped (missing optional deps, provider-specific env vars, etc.) |
| Failed pure software unit tests | 0 | |
| XPU-runnable unit tests | 0 | No unit tests in this scope directly execute XPU-specific paths |
| Passing XPU-runnable unit tests | 0 | N/A |
| Blocked XPU-runnable unit tests | 0 | N/A |
| Failed XPU-runnable unit tests | 0 | N/A |
| Newly added XPU-specific unit tests | 0 | N/A |

**Key takeaway:** current Agent Framework unit-test coverage is software-only.
Unit tests provide useful regression signal for factory and configuration plumbing,
but do not prove XPU execution.

### Two Test Command Scopes

Two commands are used across sessions:

| Command | Intended scope | Observed role in this report |
|---|---|---|
| `poe test -A` (from `python/`) | Aggregate across workspace, excluding `devui` and `lab`, passes `-m "not integration"` and `pytest-xdist` flags | Primary command for baseline validation and PR checks |
| `pytest packages/foundry_local/tests -m "not integration"` | Targeted per-package unit run | Used for iterating on `FoundryLocalClient` device-parameter tests |

This report treats `poe test -A` as the primary baseline command for framework
validation, and references per-package `pytest` invocations as the focused
command used while iterating on the Foundry Local device coverage.

### Baseline Snapshot

The WW29 baseline snapshot from the aggregate command:

| Check | Result | Meaning |
|---|---|---|
| `poe test -A` | **8,241 passed, 2 xfail, 286 skipped, 0 failed** | Regression baseline captured in WW29; not an XPU hardware test |

### Existing Ollama Test Coverage

| Tests | Coverage | Execution | XPU Relevance |
|---|---|---|---|
| [`test_cmc_integration_with_chat_completion`](python/packages/ollama/tests/test_ollama_chat_client.py) | Non-streaming chat completion against live Ollama | Integration-marked; skipped unless `OLLAMA_MODEL` is set to a real model. **Passed on XPU (WW33)** | Exercises the full Agent Framework → Ollama LLM path; GPU evidence collected from container logs |
| [`test_cmc_streaming_integration_with_chat_completion`](python/packages/ollama/tests/test_ollama_chat_client.py) | Streaming chat completion against live Ollama | Same guard. **Passed on XPU (WW33)** | Same path, streaming variant |
| [`test_cmc_integration_with_tool_call`](python/packages/ollama/tests/test_ollama_chat_client.py) | Non-streaming chat with function tool invocation | Same guard. **Passed on XPU (WW33)** | Validates tool-calling works on XPU-backed inference |
| [`test_cmc_streaming_integration_with_tool_call`](python/packages/ollama/tests/test_ollama_chat_client.py) | Streaming chat with function tool invocation | Same guard. **Passed on XPU (WW33)** | Validates streaming + tool-calling on XPU |

These tests use `OllamaChatClient()` with no mocks — they hit a real Ollama server.
In CI, Ollama is installed on-the-fly and `qwen2.5:1.5b` is pulled. For XPU
validation, they were run against a Vulkan-backed Ollama on Intel Arc Pro B60 with
`OLLAMA_MODEL=qwen2.5:0.5b`. Container logs confirmed 25/25 layers offloaded to GPU.

### XPU Coverage Status

| Test | Status |
|---|---|
| `FoundryLocalClient(device=DeviceType.CPU / GPU)` configuration compatibility | Existing tests cover the two documented device values with a mocked `FoundryLocalManager` |
| `FoundryLocalClient(device=DeviceType.XPU)` configuration compatibility | **Proposed PR-1** — parameterize the existing device-init test to also cover `DeviceType.XPU` |
| Real `FoundryLocalClient` chat completion on XPU (Foundry Local runtime) | Not implemented; blocked on confirming `DeviceType.XPU` in `foundry-local-sdk 0.5.1` |
| Real Ollama chat completion on XPU (Vulkan backend) | **4/4 passing, GPU confirmed** (WW33) — Intel Arc Pro B60, 25/25 layers offloaded, zero CPU fallback |
| Foundry Local XPU sample | Not implemented |

An XPU test must confirm that the relevant model computation uses the Intel GPU. A
successful HTTP response alone is not sufficient evidence; utilization must be
independently observed and CPU-only fallback must be rejected.

## Contributions

| # | Contribution | Status | Acceptance criteria |
|---|---|---|---|
| PR-1 | Add `DeviceType.XPU` case to `test_foundry_local_client_init_with_device`; parameterize across all documented device values | Proposed | Unit test proves the `FoundryLocalManager` mock receives `device=DeviceType.XPU`; configuration coverage, not hardware support |
| PR-1a | Documentation update listing XPU as a supported `device` option | Proposed | Merged docs page listing XPU alongside CPU / GPU with any prerequisites |
| PR-2 | Add a `python-tests-foundry-local` integration job to CI, targeting an Intel XPU self-hosted runner | Proposed | Job installs Foundry Local, loads a small model with `device=DeviceType.XPU`, runs integration tests, and green on the target runner |
| PR-2a | Add integration test suite under `python/packages/foundry_local/tests/` | Proposed | Tests cover `get_response` (non-streaming), streaming, and tool-calling against a real `FoundryLocalClient(device=DeviceType.XPU)` |
| PR-2b | Add `foundry_local_agent_xpu.py` companion sample | Proposed | Runnable sample end-to-end on an Intel XPU host |
| Issue-1 | Open a tracking issue describing the Intel XPU enablement plan | Proposed | Issue triaged and linked from PR-1 / PR-2 |
| E2E-1 | Validate Ollama integration tests against XPU-backed Ollama | **COMPLETE (WW33)** | 4/4 tests pass, 100% GPU offload confirmed via container logs |

## XPU Test Plan

### E2E-1: Ollama on Intel XPU via Vulkan backend

**Goal:** Prove the framework's local-LLM client path works end-to-end on Intel XPU
with no framework code changes.

- Run native Ollama with Vulkan backend (`OLLAMA_VULKAN=true`) on Intel XPU host
- Load a small model and verify all layers offload to GPU
- Run the 4 existing `@pytest.mark.integration` Ollama tests
- Confirm tests pass and GPU utilization via container logs

**Status: COMPLETE.** All 4 integration tests pass with 100% GPU offload.

| Component | Value |
|---|---|
| GPU | Intel Arc Pro B60 (BMG G21), 23.9 GiB discrete |
| CPU | Intel Xeon 6980P, 237 GiB system RAM |
| Compute backend | Vulkan (`OLLAMA_VULKAN=true`, `GGML_VK_VISIBLE_DEVICES=0`) |
| GPU offload | 25/25 layers (`qwen2.5:0.5b`) / 29/29 layers (`llama3.2:3b`) — 100% |
| Test command | `uv run pytest --import-mode=importlib packages/ollama/tests/test_ollama_chat_client.py -m integration --timeout=120 -v` |
| Result | 4 passed, 20 deselected, 2 warnings in 5.03s |

### Smoke Test: Foundry Local + `DeviceType.XPU` on XPU hardware

**Goal:** Prove the Foundry Local client-side path works on Intel XPU before
adding the CI job.

- Confirm the shipping `foundry-local-sdk` exposes `DeviceType.XPU`; file an
  upstream request if it does not
- Build a `FoundryLocalClient(model=<small-model>, device=DeviceType.XPU)` and
  call `await agent.run("Say Hello World")`
- Confirm the assistant reply contains the expected substring
- Confirm Intel GPU utilization during the call and reject CPU-only fallback

**Status:** Not implemented. Blocked on verifying `DeviceType.XPU` in the
pinned `foundry-local-sdk>=0.5.1,<0.5.2`.

### E2E-2: Foundry Local + `DeviceType.XPU` in CI

**Goal:** Continuous validation that the framework's Foundry Local client
correctly drives Intel XPU inference.

- Add `python-tests-foundry-local` job to `python-merge-tests.yml`, targeting an
  Intel XPU self-hosted runner (following the existing Ollama job pattern)
- Install Foundry Local runtime and pull a small model with `device=DeviceType.XPU`
- Run `pytest packages/foundry_local/tests -m integration`

**Status:** Not implemented; depends on PR-2 and PR-2a.

### Upstreaming

- Submit hardware-independent configuration tests and documentation (PR-1, PR-1a)
- Submit the integration CI job, tests, and companion sample (PR-2, PR-2a, PR-2b)
- File an upstream request against `microsoft/Foundry-Local` if `DeviceType.XPU`
  is not present in the shipping SDK
- Use a standalone repository if `microsoft/agent-framework` has no suitable
  home for hardware-dependent tests

## Next Steps

- [x] Run the Agent Framework Python unit suite: **8,241 passed** (WW29 baseline)
- [x] Analyze the repo for local embedding and LLM provider paths, mocking
      strategy, and existing integration CI patterns
- [x] Identify Foundry Local as the only provider exposing a `device` parameter
      and confirm it has zero existing integration tests and no CI job
- [x] Confirm no in-process ML frameworks (`torch`, `transformers`, IPEX) are
      imported anywhere in the Python packages
- [x] Run Ollama integration tests against XPU-backed Ollama (E2E-1):
      **4/4 passed** (WW33, Vulkan backend, `qwen2.5:0.5b`)
- [x] Independently confirm Intel GPU utilization during Ollama inference:
      **CONFIRMED** — 25/25 layers offloaded to Intel Arc Pro B60 via Vulkan
- [ ] Confirm `DeviceType.XPU` is exposed by the shipping `foundry-local-sdk`;
      file upstream request if not
- [ ] Prepare and submit PR-1 parameterizing the device-init test across all
      documented device values including XPU
- [ ] Prepare and submit PR-1a documenting XPU as a supported device option
- [ ] Open Issue-1 describing the Intel XPU enablement plan on
      `microsoft/agent-framework`
- [ ] Stand up Foundry Local on Intel XPU host and run the smoke test
- [ ] Prepare and submit PR-2a adding integration tests with
      `@skip_if_foundry_local_integration_tests_disabled` guard
- [ ] Prepare and submit PR-2 adding the `python-tests-foundry-local` CI job
- [ ] Prepare and submit PR-2b adding a Foundry Local XPU companion sample
- [ ] Follow up on PR-1 / PR-2 through review and merge

## OSPDT Assessment

All proposed PRs are test infrastructure, CI YAML, and sample code — no algorithms,
no novel IP, no new dependencies. The combined diff would be under 200 lines.

| For | Against |
|---|---|
| Upstreaming makes Intel XPU a first-class tested path in Microsoft's framework | Process overhead is disproportionate to the code size |
| A merged PR is a citable artifact for customers evaluating Intel GPU support | The Foundry Local PRs are blocked until `DeviceType.XPU` is confirmed in the SDK |
| Microsoft benefits from the test coverage regardless of device type | The highest-value result (E2E-1) required zero upstream changes and zero OSPDT |

**Recommendation:** Defer OSPDT until `DeviceType.XPU` is confirmed in
`foundry-local-sdk`. E2E-1 already proves the framework works on Intel XPU today.
Once the SDK is verified, batch all PRs into one OSPDT submission.

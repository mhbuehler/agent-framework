# Microsoft Agent Framework XPU Enablement Report

## Summary

Microsoft Agent Framework is an orchestration and client framework, not a model
runtime. It contains no in-process ML code paths (no `torch`, no `transformers`,
no `intel_extension_for_pytorch`); inference is always delegated to an external
service — hosted (Azure OpenAI, OpenAI, Anthropic, Foundry) or local (Foundry
Local, Ollama). XPU enablement will initially focus on two practical steps:

1. Add config- and factory-level unit tests confirming that
   `FoundryLocalClient(device=DeviceType.XPU)` is accepted and threaded through
   to the underlying `foundry-local-sdk`, plus a documentation update listing
   XPU as a supported device.
2. Add an integration-level test job that runs the existing (and new) Foundry
   Local integration tests against Intel XPU hardware, and contribute a matching
   sample to the framework's Foundry provider examples.

The Foundry Local provider is the natural first target: it is the only client in
the framework with a `device: DeviceType` parameter today, and it currently has
zero integration tests and no CI job — a clean lane to add real hardware
coverage without redesigning existing surfaces.

## Repos Analyzed

| Repository | Role | Recommendation |
|---|---|---|
| [`microsoft/agent-framework`](https://github.com/microsoft/agent-framework) | Core framework: Python + .NET packages, provider clients, samples, tests | Add configuration regression tests for `DeviceType.XPU`; add a Foundry Local integration CI job on Intel XPU hardware |
| [`microsoft/Foundry-Local`](https://github.com/microsoft/Foundry-Local) | External local-inference runtime consumed by `agent-framework-foundry-local` | Confirm `DeviceType.XPU` is exposed by the shipping `foundry-local-sdk`; file upstream request if not |
| [`ollama/ollama`](https://github.com/ollama/ollama) / [`ipex-llm/ipex-llm`](https://github.com/intel-analytics/ipex-llm) | External local-inference runtime consumed by `agent-framework-ollama`; IPEX-LLM ships a SYCL-backed Ollama build for Intel GPUs | Opportunistic — Ollama integration tests are backend-agnostic and can be re-run against an XPU Ollama with no framework code changes |

## Scope Assessment

### Python vs .NET

The device-selection surface (`DeviceType.CPU / GPU / XPU`) exists **exclusively in the
Python packages**. The .NET side of the framework has:

- No `Microsoft.Agents.AI.FoundryLocal` project or NuGet package
- No `DeviceType` enum for hardware accelerator selection
- No device parameter on any .NET client class
- No local-inference sample with device selection

The .NET Foundry packages (`Microsoft.Agents.AI.Foundry`,
`Microsoft.Agents.AI.Foundry.Hosting`) target cloud-hosted Azure AI Foundry only.
The only local-model path in .NET is a basic ONNX Runtime GenAI sample
(`dotnet/samples/02-agents/AgentProviders/onnx/`) which takes a raw model path with
no device-selection parameter — the execution provider is determined by which model
variant the user downloads (cpu-int4, cuda, DirectML).

**Conclusion:** XPU enablement work targets the Python packages only. A .NET ONNX
DirectML-to-SYCL path is a potential stretch goal but is not in scope for this plan.

### Framework-Level vs Runtime-Level Boundaries

| Component | Where device selection happens | Framework code changes needed? |
|---|---|---|
| Foundry Local (`python/packages/foundry_local`) | `FoundryLocalClient(device=DeviceType.XPU)` passes through to `foundry-local-sdk` | Minimal — parameterize test, add sample, add CI job |
| Ollama (`python/packages/ollama`) | `OllamaChatOptions` exposes `num_gpu` / `main_gpu` pass-throughs; actual device is determined by the Ollama binary (CUDA, ROCm, or SYCL via IPEX-LLM) | None — purely backend configuration |
| Lab/Lightning (`python/packages/lab/lightning`) | Uses `agentlightning` (VERL/vLLM) for RL training; samples reference `gpu_memory_utilization`, `n_gpus_per_node` | None in framework — XPU support depends on upstream `agentlightning`/VERL/vLLM adding Intel GPU backends |

### Dependency Risk: `foundry-local-sdk` Version Pin

The `agent-framework-foundry-local` package depends on `foundry-local-sdk>=0.5.1,<0.5.2`.
Whether `DeviceType.XPU` is an enumerated value in this SDK version is the **critical
upstream dependency**. If it is not present:

- Option A: File a feature request on `microsoft/Foundry-Local`
- Option B: Pass a raw string value if the SDK accepts it
- Option C: Wait for a `foundry-local-sdk` release that adds XPU

This must be verified on a system where the SDK is installed.

### CI Architecture for XPU Jobs

The existing CI (`python-merge-tests.yml` / `python-integration-tests.yml`) uses
parallel jobs per provider, all running on `runs-on: ubuntu-latest` (GitHub-hosted).
A Foundry Local XPU job requires:

- A **self-hosted runner** with Intel discrete GPU (e.g., `[self-hosted, linux, intel-xpu]`)
- A new `foundry_local` entry in the `paths-filter` step (currently only `foundry` exists, covering cloud Foundry)
- A new `foundryLocalChanged` output wiring the conditional trigger
- The Foundry Local runtime pre-installed or installed on-the-fly (following the Ollama pattern)

The Ollama integration job (`python-tests-misc-integration`) is the closest existing
pattern: it installs Ollama, pulls models, starts the service, and runs tests —
all within a single job. The Foundry Local XPU job would follow the same structure
but on a self-hosted Intel GPU runner.

## Existing Foundry Local Sample

A Foundry Local sample already exists at
`python/samples/02-agents/providers/foundry/foundry_local_agent.py`. It demonstrates
streaming and non-streaming chat with function tools using `FoundryLocalClient(model="phi-4-mini")`
but does **not** pass a `device=` parameter. The XPU companion sample (PR-2b) would
be a variant that explicitly specifies `device=DeviceType.XPU` and includes a
hardware-verification step (checking `xpu-smi` or similar).

## Stretch Goals (Out of Primary Scope)

| Goal | Dependency | Effort | Value |
|---|---|---|---|
| Lab/Lightning RL training on XPU | `agentlightning` / VERL / vLLM adding Intel GPU backend | High — entirely external | Demonstrates full-stack XPU: training + inference |
| .NET ONNX on Intel GPU | ONNX Runtime adding SYCL execution provider or using OpenVINO EP | Medium — sample-level change | Broader .NET story for Intel hardware |
| Ollama on XPU in CI | Standing up IPEX-LLM container on self-hosted runner | Low — no framework changes | Proves the full local-LLM path on Intel hardware |

## Testing

### Integration Test Census

As of WW33, the Python integration tests across all packages:

| Metric | Count | Notes |
|---|---:|---|
| Files containing `@pytest.mark.integration` | 29 | Across `packages/` directory |
| Total integration-marked test instances | 121 | Requires live provider access (API keys, running services) |
| Remote-provider integration tests | ~117 | OpenAI, Azure OpenAI, Anthropic, Foundry (cloud), Foundry Hosting, GitHub Copilot, Cosmos |
| Local-runtime integration tests (Ollama) | 4 | `test_ollama_chat_client.py` — make **live LLM calls** to a running Ollama server |
| Local-runtime integration tests (Foundry Local) | 0 | No integration tests exist for Foundry Local |

The 4 Ollama integration tests are:

| Test | What it does |
|---|---|
| `test_cmc_integration_with_chat_completion` | Non-streaming chat completion against live Ollama |
| `test_cmc_streaming_integration_with_chat_completion` | Streaming chat completion against live Ollama |
| `test_cmc_integration_with_tool_call` | Non-streaming chat with function tool invocation |
| `test_cmc_streaming_integration_with_tool_call` | Streaming chat with function tool invocation |

These tests use `OllamaChatClient()` with **no mocks** — they hit a real Ollama
server. In CI, Ollama is installed on-the-fly and `qwen2.5:1.5b` is pulled. They
are the **zero-framework-change path to proving XPU inference**: point
`OLLAMA_HOST` at an IPEX-LLM XPU-backed Ollama and re-run.

### Unit Test Analysis (Summary)

As of WW29, the Agent Framework Python unit test status:

| Metric | Count | Notes |
|---|---:|---|
| Total unit tests across run scope | 8,529 | Excludes `devui` and `lab` packages, integration tests (require third-party credentials), and all .NET tests |
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
Provider tests use `unittest.mock` at the SDK client boundary (`MagicMock(spec=AsyncOpenAI)`,
`patch.object(AsyncClient, "chat", ...)`, `patch("...FoundryLocalManager", ...)`)
combined with fake env vars via `monkeypatch.setenv`; no cassette/replay system,
no network-block enforcement. Unit tests provide useful regression signal for
factory and configuration plumbing, but do not prove XPU execution.

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

### XPU coverage status

| Test | Status |
|---|---|
| `FoundryLocalClient(device=DeviceType.CPU / GPU)` configuration compatibility | Existing tests in `test_foundry_local_client_init_with_device` cover the two currently documented device values with a mocked `FoundryLocalManager` |
| `FoundryLocalClient(device=DeviceType.XPU)` configuration compatibility | **Proposed PR-1** — parameterize the existing device-init test to also cover `DeviceType.XPU` |
| Real `FoundryLocalClient` chat completion on XPU (Foundry Local runtime) | Not implemented |
| Real Ollama chat completion on XPU (Vulkan backend) | **4/4 passing, GPU confirmed** (WW33) — `qwen2.5:0.5b` on Intel Arc Pro B60 via Vulkan, 25/25 layers offloaded, zero CPU fallback |
| Foundry Local XPU sample under `python/samples/02-agents/providers/foundry/` | Not implemented |

An XPU test must confirm that the relevant model computation uses the Intel GPU.
A successful HTTP response alone is not sufficient evidence; utilization must be
independently observed (e.g., `xpu-smi dump`, `intel_gpu_top`) and CPU-only
fallback must be rejected.

## Contributions

| # | Contribution | Status | Acceptance criteria |
|---|---|---|---|
| PR-1 | Add `DeviceType.XPU` case to `test_foundry_local_client_init_with_device`; parameterize across all documented device values | Proposed | Unit test proves the `FoundryLocalManager` mock receives `device=DeviceType.XPU` on `get_model_info`, `download_model`, and `load_model`; configuration coverage, not hardware support |
| PR-1a | Documentation update listing XPU as a supported `device` option in the Foundry Local provider README | Proposed | Merged docs page listing XPU alongside CPU / GPU with any prerequisites |
| PR-2 | Add a `python-tests-foundry-local` integration job to `.github/workflows/python-merge-tests.yml`, mirroring the existing `python-tests-misc-integration` (Ollama) pattern, targeting an Intel XPU self-hosted runner | Proposed | Job installs Foundry Local, loads a small model with `device=DeviceType.XPU`, runs the new integration tests, and green on the target runner |
| PR-2a | Add a small integration test suite under `python/packages/foundry_local/tests/` marked `@pytest.mark.integration` + `@pytest.mark.flaky` + a new `@skip_if_foundry_local_integration_tests_disabled` guard | Proposed | Tests cover `get_response` (non-streaming), streaming, and a tool-calling variant against a real `FoundryLocalClient(device=DeviceType.XPU)` |
| PR-2b | Add `foundry_local_agent_xpu.py` companion sample to `python/samples/02-agents/providers/foundry/` | Proposed | Runnable sample end-to-end on an Intel XPU host; referenced from the provider README |
| Issue-1 | Open a tracking issue on `microsoft/agent-framework` describing the Intel XPU enablement plan and inviting feedback | Proposed | Issue triaged and linked from PR-1 / PR-2 |
| Smoke Test | Validate `FoundryLocalClient(device=DeviceType.XPU)` end-to-end on real XPU hardware (outside CI) | Proposed | Correct response text, verified Intel GPU utilization during the call, no CPU-only fallback |
| E2E-1 | Validate `test_cmc_integration_with_chat_completion` against an XPU-backed Ollama (no framework change required) | **Tests passing (WW33)** | 4/4 integration tests pass with `OLLAMA_MODEL=qwen2.5:0.5b` on native Ollama + Intel GPU (Vulkan); GPU utilization independently unconfirmed |

## XPU Test Plan

### Smoke Test: Foundry Local + `DeviceType.XPU` on XPU hardware

**Goal:** Prove the Foundry Local client-side path works on Intel XPU before
adding the CI job.

- Install Foundry Local runtime on an Intel XPU host (GNR VM + B60, or similar)
- Confirm the shipping `foundry-local-sdk` exposes `DeviceType.XPU`; file an
  upstream request if it does not
- Build a `FoundryLocalClient(model=<small-model>, device=DeviceType.XPU)` and
  call `await agent.run("Say Hello World")`
- Confirm the assistant reply contains the expected substring
- Confirm Intel GPU utilization during the call and reject CPU-only fallback

**Status:** Not implemented.

### E2E-1: Ollama on Intel XPU via Vulkan backend

**Use case:** re-use the existing Ollama integration tests to prove the
framework's local-LLM client path works end-to-end when the backend runs on
Intel XPU — no framework code change required.

**Backend:** Native Ollama with Intel GPU support via the **Vulkan** compute
backend. Upstream `llama.cpp` (which Ollama wraps) supports Vulkan as a portable
GPU path that works on Intel discrete GPUs without requiring the IPEX-LLM SYCL
build. Vulkan uses the Mesa/ANV driver stack and works out of the box, but does
not access Intel-specific hardware features (XMX matrix engines) and delivers
lower throughput than the SYCL/Level-Zero path provided by IPEX-LLM.

For the purposes of XPU enablement, the Vulkan backend proves that inference runs
on the Intel GPU (not CPU-only fallback). An IPEX-LLM SYCL backend would provide
better performance but is not required for functional validation.

**Execution (WW33):**

- Ollama running in Docker container (`mbuehle-ollama`) on Intel XPU host with
  native Ollama + Intel GPU support (Vulkan backend)
- Models available: `llama3.2:3b`, `qwen2.5:0.5b`
- From a checkout of `microsoft/agent-framework`, with
  `OLLAMA_HOST=http://localhost:11434` and `OLLAMA_MODEL=qwen2.5:0.5b`:
  ```
  uv run pytest --import-mode=importlib \
      packages/ollama/tests/test_ollama_chat_client.py \
      -m integration --timeout=120 -v
  ```
- Result: **4 passed, 20 deselected, 2 warnings in 5.03s**

**GPU utilization verification — CONFIRMED:**

Docker container logs (`docker logs mbuehle-ollama`) provide definitive evidence:

```
OLLAMA_VULKAN:true
inference compute: id=0 library=Vulkan name=Vulkan0
    description="Intel(R) Arc(tm) Pro B60 Graphics (BMG G21)"
    type=discrete total="23.9 GiB" available="21.5 GiB"
    pci_id=0000:06:10.0

llama_prepare_model_devices: using device Vulkan0
    (Intel(R) Arc(tm) Pro B60 Graphics (BMG G21)) - 21963 MiB free
load_tensors: offloading output layer to GPU
load_tensors: offloading 23 repeating layers to GPU
load_tensors: offloaded 25/25 layers to GPU       ← 100% GPU, zero CPU fallback
```

Hardware summary:

| Component | Value |
|---|---|
| GPU | Intel Arc Pro B60 (BMG G21), 23.9 GiB discrete |
| CPU | Intel Xeon 6980P, 237 GiB system RAM |
| Compute backend | Vulkan (via `OLLAMA_VULKAN=true`, `GGML_VK_VISIBLE_DEVICES=0`) |
| GPU offload | 25/25 layers (`qwen2.5:0.5b`) / 29/29 layers (`llama3.2:3b`) — 100% |
| Model buffer | 373.71 MiB on Vulkan0 (`qwen2.5:0.5b`) |
| KV cache | 384.00 MiB on Vulkan0 |
| Compute buffer | 128.02 MiB on Vulkan0 |

**Status: COMPLETE.** All 4 integration tests pass with 100% GPU offload on
Intel Arc Pro B60 via Vulkan backend. No framework code changes required.

### E2E-2: Foundry Local + `DeviceType.XPU` in CI

**Use case:** continuous validation that the framework's Foundry Local client
correctly drives Intel XPU inference through the Foundry Local runtime.

- Add `python-tests-foundry-local` job to `python-merge-tests.yml`, targeting an
  Intel XPU self-hosted runner
- Install Foundry Local runtime and pull a small model with
  `device=DeviceType.XPU`
- Export `FOUNDRY_LOCAL_MODEL` and any device-selection env var the new
  integration tests read
- Run `pytest packages/foundry_local/tests -m integration`
- Verify green and, out-of-band, verify utilization on the target runner

**Status:** Not implemented; depends on PR-2 landing the job definition and
PR-2a landing the integration tests it runs.

### Upstreaming

- Submit planned hardware-independent configuration tests and documentation to
  `microsoft/agent-framework` (PR-1, PR-1a)
- Submit the integration CI job, integration tests, and companion sample to
  `microsoft/agent-framework` (PR-2, PR-2a, PR-2b)
- File an upstream request against `microsoft/Foundry-Local` if `DeviceType.XPU`
  is not present in the shipping `foundry-local-sdk`
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
- [ ] Confirm `DeviceType.XPU` is exposed by the shipping `foundry-local-sdk`;
      file upstream request if not
- [ ] Prepare and submit PR-1 parameterizing the device-init test across all
      documented device values including XPU
- [ ] Prepare and submit PR-1a documenting XPU as a supported device option
- [ ] Open Issue-1 describing the Intel XPU enablement plan on
      `microsoft/agent-framework`
- [ ] Stand up an Intel XPU host with Foundry Local installed and run the
      Foundry Local smoke test manually
- [x] Run Ollama integration tests against XPU-backed Ollama (E2E-1):
      **4/4 passed** (WW33, Vulkan backend, `qwen2.5:0.5b`)
- [x] Independently confirm Intel GPU utilization during Ollama inference:
      **CONFIRMED** — container logs show 25/25 layers offloaded to
      Intel Arc Pro B60 (BMG G21) via Vulkan, zero CPU fallback
- [ ] Prepare and submit PR-2a adding integration tests under
      `python/packages/foundry_local/tests/` with a
      `@skip_if_foundry_local_integration_tests_disabled` guard
- [ ] Prepare and submit PR-2 adding the `python-tests-foundry-local` CI job
      targeting an Intel XPU self-hosted runner
- [ ] Prepare and submit PR-2b adding a Foundry Local XPU companion sample under
      `python/samples/02-agents/providers/foundry/`
- [ ] Follow up on PR-1 / PR-2 through review and merge

## Conclusions (WIP — Draft)

### Is XPU enablement worth pursuing in this repo?

**Yes, but with clear scope boundaries.** The Microsoft Agent Framework is an
orchestration layer that delegates all inference to external runtimes. This means:

1. **The framework itself needs almost zero code changes for XPU.** There are no
   `torch`, `transformers`, or IPEX imports anywhere in the production Python
   packages. XPU enablement is about proving the configuration plumbing works and
   adding CI infrastructure — not fixing broken GPU code paths.

2. **The quickest proof-of-value requires zero framework changes — and is now
   complete.** The 4 Ollama integration tests pass against a Vulkan-backed Ollama
   on Intel Arc Pro B60 with 100% GPU offload confirmed via container logs. This
   proves the entire local-inference path works on Intel hardware today with no
   code changes.

3. **The Foundry Local path has upstream risk.** Whether `DeviceType.XPU` exists in
   the pinned `foundry-local-sdk>=0.5.1,<0.5.2` is unverified. If it doesn't, the
   Foundry Local enablement plan is blocked on Microsoft shipping that enum value.
   The Ollama path has no such dependency.

### Recommended prioritization

| Priority | Action | Effort | Blocked by |
|---|---|---|---|
| ~~P0~~ | ~~Run Ollama integration tests against XPU-backed Ollama (E2E-1)~~ | ~~Hours~~ | **DONE (WW33)** — 4/4 pass, 100% GPU offload confirmed |
| P1 | Verify `DeviceType.XPU` in `foundry-local-sdk 0.5.1` | Minutes | Access to install the SDK |
| P2 | Submit PR-1 (unit test parameterization) + PR-1a (docs) | Days | P1 confirmation |
| P3 | Submit PR-2/2a/2b (CI job + integration tests + sample) | Weeks | P1 + self-hosted runner infra |

### Decision framework for the team

The effort-to-value tradeoff:

- **Low effort, high signal:** E2E-1 (Ollama on XPU) proves the framework works on
  Intel hardware with zero code changes. Good for a demo, validates the
  architecture claim.
- **Medium effort, upstream-dependent:** Foundry Local PRs add durable CI coverage
  but depend on Microsoft's SDK exposing `DeviceType.XPU`. If the SDK doesn't
  support it yet, we're blocked or need to work with the Foundry Local team.
- **Not worth pursuing now:** Lab/Lightning (VERL/vLLM) and .NET ONNX — both depend
  on large external projects adding Intel GPU support. Monitor only.

### What "XPU enabled" means in this context

Unlike repos with in-process GPU kernels, "XPU enabled" for this framework means:

1. The framework correctly passes device configuration to the local runtime
2. Integration tests prove end-to-end inference on Intel XPU hardware
3. CI continuously validates this path doesn't regress
4. Documentation and samples guide users to the XPU path

It does **not** mean the framework itself runs compute on the XPU — it means the
framework is a verified, tested, documented path to XPU-accelerated inference via
its local-runtime providers.

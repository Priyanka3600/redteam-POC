# Ocular RAG and Builder Agent Security Testing POC

## 1. Purpose

This proof of concept (POC) demonstrates a controlled workflow for assessing
the resilience of Ocular agents:

- The Ocular Knowledge Base Playground (RAG agent).
- The Ocular Builder Agent run endpoint.

The assessment workflow uses a configured simulator, an explicitly authorized
test target, and an evaluator. Keep target-specific configuration and live
execution procedures in access-controlled project materials rather than this
public overview.

Run this POC only against approved test/staging agents with synthetic data.

## 2. High-Level Architecture

```text
Assessment selection
        |
        v
Assessment vulnerability and attack configuration
        |
        +--> DeepTeam simulator model
        |       Generates/enhances attack prompts
        |
        v
Shared asynchronous model callback
        |
        v
Selected OcularClient
        |
        +--> Approved test deployment (RAG or Builder)
                Deployment-specific configuration is kept private
        |
        v
Target response
        |
        v
DeepTeam evaluator model
        Scores the target response
        |
        v
Console report and JSON result under results/
```

The callback is shared; it does not itself select an agent. The selected
`OcularClient` determines which target receives the prompt.

## 3. Components and Code Locations

| Component | Responsibility |
|---|---|
| Assessment orchestrator | Selects an assessment and coordinates the simulator, approved test target, and evaluator |
| Shared client | Handles target communication and response parsing using private deployment configuration |
| Model configuration | Creates separate simulator and evaluator models |
| Assessment configuration | Defines the resilience criteria used by an assessment |
| Mocked test suite | Validates request construction and response parsing without contacting a live target |

## 4. Target Agent Integrations

### 4.1 RAG Agent

The RAG integration is one supported target type. Its endpoint, request
template, stream behavior, and authorization are deployment-specific.

### 4.2 Builder Agent

The Builder integration is another supported target type. Its endpoint,
request template, required headers, and authorization are deployment-specific.

Do not publish target URLs, tenant or workspace identifiers, request templates,
or credentials. Store them in access-controlled configuration and use them
only for explicitly authorized, non-production testing.

## 5. Request Construction and Callback

The application selects a configured target and uses a shared asynchronous
callback to obtain a response for evaluation. Target selection and request
construction are implementation details; deployment-specific values are
intentionally omitted from this overview.

## 6. DeepTeam Simulator and Evaluator

The simulator and evaluator are LiteLLM-backed models. They are distinct from
the Ocular target and from each other:

- **Simulator**: supplies assessment test inputs.
- **Target**: an explicitly authorized RAG or Builder test deployment.
- **Evaluator**: assesses responses against the selected resilience criteria.

`models.py` reads model settings in this order:

1. `DEEPTEAM_SIMULATOR_MODEL` for the simulator, falling back to
   `DEEPTEAM_LITELLM_MODEL`.
2. `DEEPTEAM_EVALUATION_MODEL` for the evaluator, falling back to
   `DEEPTEAM_LITELLM_MODEL`.

The current implementation constructs both LiteLLM models with
`temperature=1`. Keep these model names aligned with model groups that are
configured and healthy on the configured LiteLLM proxy. The LiteLLM API key
and base URL are separate from the Ocular authorization settings.

### Approximate model cost

In an observed run, LiteLLM usage was approximately **$0.02** for generating
4 attacks and testing 2 vulnerability types within one vulnerability group.
Treat this as a run-specific estimate, not a per-vulnerability rate: actual
spend depends on the number of generated attacks and evaluation calls, prompt
and response token sizes, model pricing, and retries. Check the LiteLLM logs
for the actual cost of each run; this estimate is not a quote or spending
limit.

## 7. Configuration

Deployment-specific settings belong in a local, access-controlled
configuration file. Do not publish target URLs, request settings, authorization
details, tokens, cookies, or API keys in documentation or reports.

Any assessment load limits and timeouts must be selected with the target owner
for an isolated, approved test environment.

## 8. Controlled Validation

This public overview does not include live-target commands or connectivity
instructions. Live assessments require an approved, isolated non-production
target, synthetic data, authorized credentials, and the target owner's
procedures. Do not run assessments against systems outside the approved scope.

## 9. Results

On successful completion, the assessment runner writes a timestamped JSON file
to its configured results directory:

- RAG result filenames use the assessment name, for example
  `eu_ai_act_<timestamp>.json`.
- Builder result filenames are prefixed with the agent name, for example
  `builder_eu_ai_act_<timestamp>.json`.

The console prints a detailed test-case table when DeepTeam returns test cases.
Results and LiteLLM spend logs can contain prompts or model responses; handle
them as potentially sensitive test data.

The number of LiteLLM requests is not fixed by the code. It depends on the
assessment's vulnerability types and attacks, generated variations,
multi-turn attacks, provider retries, and fallbacks. Ocular target requests
are separate from simulator/evaluator LiteLLM requests. For actual per-run
counts and spend, filter LiteLLM logs to the run's time window and key, and
consult Ocular service logs for target requests.

## 10. Validation

The test suite includes:

- `tests/test_agent_selection.py`: verifies agent selection, RAG and Builder
  request URLs/bodies, Builder headers, and Builder response extraction.
- `tests/test_response_parser.py`: verifies query parameter handling, auth
  header construction, and response parsing.

Run the focused tests with:

```powershell
.\.venv312\Scripts\python.exe -m pytest tests\test_agent_selection.py tests\test_response_parser.py -q
```

## 11. Troubleshooting

| Symptom | Interpretation and next check |
|---|---|
| HTTP error | Ask the target owner to verify the approved test configuration and consult service-side diagnostics. Do not expose request bodies or credentials in shared reports. |
| Timeout | Stop repeated requests and ask the service owner to review the approved test limits and service-side diagnostics. |
| `LiteLLM` reports no healthy deployment | Confirm the simulator/evaluator model group is correctly named and has a healthy deployment on the configured LiteLLM proxy. |
| `WinError 1231` or `httpx.ConnectError` for LiteLLM | The machine could not connect to the configured LiteLLM endpoint. Check network/VPN, DNS, proxy connectivity, and service availability. |
| Empty assessment results | Check the assessment configuration and the installed framework's expected category format. |
| PostHog multiple-client warning | This is an observability warning. It is separate from the HTTP, LiteLLM, or assessment exception unless another traceback identifies it as the failure. |

Do not treat a client timeout increase as a fix for a server-side gateway
deadline. When reporting a failure, capture the timestamp and error category.
Never include target identifiers, request bodies, authorization headers,
cookies, or API keys.

## 12. POC Boundaries

- The POC sends prompts to the configured RAG or Builder Ocular target; it does
  not provision or modify either service.
- The target agent's model and behavior are configured by the Ocular
  deployment/flow. The DeepTeam simulator and evaluator are configured through
  LiteLLM and `.env`.
- Assessment request volume and cost vary by configuration and retries.
- Successful local/unit tests validate request construction and response
  parsing with mocked responses; they do not prove live endpoint availability,
  permissions, or production readiness.
- Use an approved non-production target, synthetic prompts, and authorized
  credentials.

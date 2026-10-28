---
name: jev-ultrafast
description: "Run jev-ultrafast, the finite-choice browser automation runtime in the bu-J repository: Microsoft Decision 1 (Azure / Microsoft Foundry) or OpenAI Decisions choose one of eight operations and an observed target per step, executed through Browser Harness. Use for the local inspector (jev), the Python jev_ultrafast.Agent, custom goals, or the offline verification suite."
---

# jev-ultrafast (Microsoft Decision 1 / OpenAI Decisions)

jev-ultrafast uses **Microsoft Decision 1** in Azure / Microsoft Foundry (`DECISION_PROVIDER=azure`, default deployment `decision-1`) or **OpenAI Decisions** (`DECISION_PROVIDER=openai`, default model `gpt-6-luna`) to select a finite browser operation and an observed target in one request. A separate OpenAI-compatible text model supplies text for `TYPE_TEXT`. The browser runs through [Browser Harness](https://github.com/browser-use/browser-harness).

## Get it

It lives in `jev-ultrafast/` of the [bu-J](https://github.com/geekjapan/bu-J) repository as a separate uv project (Python >= 3.12). The Decision API work is on the `feat/openai-decisions-api` branch until it is merged into `main`.

```bash
git clone --branch feat/openai-decisions-api https://github.com/geekjapan/bu-J.git
cd bu-J/jev-ultrafast
uv sync
cp .env.example .env   # fill in the variables below; never commit it
```

Run `uv run browser-harness --doctor` if Chrome is not connected, and allow remote debugging when prompted. On Windows, use `powershell -NoProfile -ExecutionPolicy Bypass -File scripts/run-windows.ps1 -Mode demo -Provider azure` (or `-Provider openai`) instead of the inspector command below.

## Operations and loop

Keep the loop small: `page -> indexed elements -> operation + target -> execution`.
Operations: `CLICK`, `TYPE_TEXT`, `SELECT`, `SCROLL_UP`, `SCROLL_DOWN`, `WAIT`, `DONE`, `BLOCKED`.

- The input is one natural-language goal. Do not add site-specific plans or hardcoded field values.
- The decision provider chooses an operation and operation-specific target heads in one request. Consume only the selected operation's target.
- Targets map strictly to observed elements and supported operations. Model output never becomes selectors, coordinates, shell commands, or executable JavaScript.
- `TYPE_TEXT` invokes the text helper model. Cache a stale retry's value only while its entire helper input is identical.
- Never retry a browser mutation. Log execution before observing its result.
- Verify the actual final outcome independently. A `DONE` choice is not proof of success.

## Configure Azure / Microsoft Foundry (default)

```dotenv
DECISION_PROVIDER=azure
AZURE_ENDPOINT=https://your-resource.cognitiveservices.azure.com
AZURE_DEPLOYMENT_NAME=decision-1
AZURE_API_KEY=your-azure-api-key
# Or leave AZURE_API_KEY empty and supply AZURE_ENTRA_TOKEN:
# export AZURE_ENTRA_TOKEN="$(az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv)"
TEXT_MODEL_API_KEY=your-text-model-key
```

`AZURE_ENDPOINT` is the resource root. The client calls `/providers/microsoft/v1/systemone` and authenticates with the `api-key` header.

## Configure OpenAI Decisions

```dotenv
DECISION_PROVIDER=openai
OPENAI_API_KEY=your-openai-key
OPENAI_DECISION_MODEL=gpt-6-luna
TEXT_MODEL_API_KEY=your-text-model-key
```

## Run the local inspector (`jev`)

```bash
cd jev-ultrafast
uv run --env-file .env jev
```

Open `http://127.0.0.1:8766` and click **Start demo -> Run automatically**, or **Choose next** to pause before each execution. The inspector shows numbered elements, operation and target probabilities, and executed actions. Run one inspector at a time; it uses a temporary Chrome profile.

## Use the Python library

```python
from jev_ultrafast import Agent

with Agent(
    "https://www.google.com/travel/flights?hl=en",
    "Find one-way flights from Zurich to London on September 20, 2026, "
    "for one adult in economy. Stop when matching flight options are visible.",
) as agent:
    for state in agent.run():
        print(state["elapsed_ms"], state["status"])
```

Run a custom goal from the command line (repeat `--goal` for an ordered list):

```bash
uv run --env-file .env python examples/run.py \
  --url https://en.wikipedia.org/wiki/Main_Page \
  --goal 'Find and open the Wikipedia article about Gödel’s incompleteness theorems.'
```

## Verify offline

The repository ships the `verify-decision-stack` suite for both the jev runtime and bu-J's `DecisionAgent`. It uses mock HTTP responses and an isolated Chromium; no API key or paid call is needed. Exit code 0 means every stage passed and all deliberately broken variants were detected; skipped checks count as failures.

```bash
cd jev-ultrafast
python3 verification/run.py
```

Read `artifacts/verification/REPORT.md` afterwards. The suite's own skill, `jev-ultrafast/.agents/skills/verify-decision-stack/SKILL.md`, documents its bounds and must stay next to `verification/`.

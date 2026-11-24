# Router + Cascade Gateway

A gateway that sits between an app (for example the Aider coding agent) and AI models.
Each request is routed to the cheapest model likely to handle it, the reply is checked,
and the request escalates to a stronger model only if the check fails. Every step is
recorded as a trace so cost, latency and decisions can be inspected,and the same
requests can be sent through a single strong model for comparison.

```
client (Aider, script, browser)  ->  POST /v1/chat/completions
        model "auto"   : router -> cheap tier -> verify -> (escalate to strong)
        model "single" : strong tier only      (baseline for comparison)
        model "cheap"  : cheap tier only       (baseline)
```

## 1. Run it (offline mock, no keys needed)

```bash
pip install -r requirements-dev.txt
pytest                                   # 29 tests
ROUTER_CONFIG=configs/tiers.mock.yaml uvicorn app.main:app_factory --factory --port 8000
python scripts/compare.py --prompts data/sample_prompts.mock.jsonl
```

**The mock provider is for tests and dashboard development only.** Its spans are tagged
`provider=mock`. Never use it for evaluation numbers or for the teacher demo.

## 2. Run it for real

1. `cp configs/tiers.example.yaml configs/tiers.yaml`
2. Choose one cheap and one strong model your provider currently offers. Check the
   provider's docs for the OpenAI-compatible `base_url`, model IDs, free-tier limits and
   **current list prices**. Fill in `price_in_per_m` and `price_out_per_m` with the list
   prices even on a free tier (shadow pricing), otherwise savings will read as zero.
3. Export keys: `export CHEAP_API_KEY=... STRONG_API_KEY=... ADMIN_KEY=...`
4. `uvicorn app.main:app_factory --factory --port 8000`

Keys stay in environment variables on the server. Never put them in the frontend or commit them.

## 3. Connect Aider

Aider talks to any OpenAI-compatible endpoint. Check Aider's current documentation for the
exact option names; this is the usual form:

```bash
export OPENAI_API_BASE=http://localhost:8000/v1
export OPENAI_API_KEY=dummy
cd your-practice-repo
aider --model openai/auto            # router + cascade
aider --model openai/single          # same agent, strong model only (comparison)
```

Notes:
- Aider may warn that the model name is unknown. That is expected for a gateway model name.
- This gateway supports streaming by buffering the full reply, verifying it, then replaying
  it as a stream. If a client misbehaves, try Aider's no-stream option first.
- Aider uses plain text edit formats, not tool calling, so the gateway rejects `tools`
  requests in this version.
- If the cheap model keeps failing the edit-format check, try Aider's `whole` edit format.
- Smoke test first: run one small edit and confirm a trace appears at `/api/traces`.

## 4. How anyone can verify it is live (not static)

1. Send two very different prompts and compare `x_router.start_tier` and the `route` span's
   `reason` and `features`: the decision changes with the input.
2. `PUT /api/config {"threshold": 0.9}` and resend the same prompt: the routing changes live.
3. Compare `x_router.cost_usd` and token counts with the provider's own usage console.
4. Run `scripts/compare.py` on the same prompts: single vs auto, side by side.
5. Read `pytest` output: tests assert that routing differs by input and that failed checks escalate.

## 5. What this version verifies, and what it does not

The verifier only sees the reply, not your repository. It checks: not truncated, not empty,
SEARCH/REPLACE blocks well formed, fenced Python parses, and valid JSON when JSON was
requested. It cannot tell whether an edit is *correct*. Quality claims for the demo must
come from the bug-fix task suite where real tests run, not from this check alone.

Other limits: rule-based router (version 1), no semantic cache yet, no tool-call support,
cost counterfactual is an estimate (tokens used x strong-tier price).

## 6. Layout

```
app/main.py       FastAPI app, OpenAI-compatible endpoint, observability and config APIs
app/router.py     pricing helper and RuleRouter (swap in a learned Decider later)
app/cascade.py    try a tier, verify, escalate, spans for each step
app/verifiers.py  EditFormatVerifier, JsonVerifier
app/providers.py  OpenAI-compatible client and the mock provider
app/tracing.py    spans, SQLite trace store, metrics aggregation
docs/CONTRACTS.md API and trace schema contract for the dashboard
```

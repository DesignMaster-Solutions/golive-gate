# Go-Live Gate

An OpenMinion 0.1 agent that answers one question before a small-business website goes live:
**is it ready, ready with warnings, or blocked?** Built for Cortensor Hackathon #5 by Design Master Solutions LLC.

We build and host websites for small businesses in the US and Brazil. Every launch has the same floor
(title and description, canonical, Open Graph, JSON-LD, icons, a sitemap, a real 404, `llms.txt`, and a
robots.txt that does not lock out AI search crawlers). We used to check it by hand. Go-Live Gate turns
that checklist into a repeatable agent run.

## Workflow

```
python gate.py example.com [--auto-approve yes|no]
```

1. **Investigator** (`Agent` + `@tool` + `approval_callback` + `session_id`): calls three read-only tools,
   `plugin.golive.inspect_home`, `inspect_crawlers`, `inspect_404` (plain `urllib`, 15 s timeouts, facts only).
   If the model skips a tool, the gap is filled deterministically so the judge never guesses.
2. **Judge** (`Agent(output_type=GoLiveReport)`): gets the raw facts and returns a pydantic-validated report
   (verdict + one row per check with the evidence and the fix). Splitting tools from structured output keeps the judge's
   request small (about 3k tokens) and means the judge cannot fetch anything itself.
3. **Publish** (`@tool(dangerous=True)` + `forced_tools` + `approval_callback`): a second agent must call
   `save_report`, and OpenMinion routes that call through the human approval callback (interactive y/N, or
   `--auto-approve yes|no` for CI). Nothing is written if the human says no.
4. **Diagnostics**: `run_id`, `run_state` and wall time are printed; `out/<site>-<timestamp>/facts.json` keeps the
   raw tool facts next to `report.md`, so a reviewer can check the model's verdict against what the tools saw.

The rules for the verdict live in the agent instructions (`gate.py`); the facts live in `checks.py` and are
covered by offline tests against a local HTTP server.

## Setup

```bash
python3.12 -m venv .venv && .venv/bin/pip install -r requirements.txt
export OPENROUTER_API_KEY=...      # we ran on a free model: nvidia/nemotron-3-ultra-550b-a55b:free
.venv/bin/openminion --home-root .om/home --data-root .om/data setup --provider openrouter \
  --model nvidia/nemotron-3-ultra-550b-a55b:free --no-chat
.venv/bin/python -m unittest test_checks -v
.venv/bin/python gate.py capegraniteconnection.com --auto-approve yes
# Groq free tier: add --tpm 6500 (see finding 3; the main turn alone is ~8.7k tokens, so it still fails there)
```

Cortensor Portal works the same way: `setup --provider cortensor-portal` with `CORTENSOR_API_KEY`.

## Example run

See [`examples/capegraniteconnection.com-20261028T174922Z/`](examples/capegraniteconnection.com-20261028T174922Z/):
`report.md` (verdict **ready**, 11 checks), `facts.json` (what the tools saw) and `run.txt` (approvals, run ids,
timings: investigator 232 s, judge 355 s wall time on a free model, publish approved by `--auto-approve yes`).

## Building Blocks Tested

| block | how we used it | result |
|---|---|---|
| `Agent(output_type=...)` | pydantic `GoLiveReport` with `Literal` statuses | validates once the schema is in the message (finding 4) |
| `@tool` with `idempotent=True` | three read-only network tools | works |
| action policy | read-only tools still hit the confirmation prompt; approved in `approval_callback` | finding 1 |
| tool exposure | the first request of a turn lists only `submit_output`; tools come in later iterations | finding 2 |
| `@tool(dangerous=True)` + `forced_tools` + `approval_callback` | `save_report` behind human approval | works |
| `session_id` | one fresh session per run and agent (`golive-<host>-<ts>`, `-judge`, `-publish`) | a follow-up run in the same session lost the first message (finding 5) |
| `ProviderError` / `TurnTimeoutError` | free-tier 429 retried after 65 s; a timed-out investigator keeps the facts it already collected | works |
| provider pacing | none built in; we wrap the OpenAI-compatible adapter to stay under 6.5k tokens/min | finding 3 |

## What We Learned

1. **Read-only tools still ask for confirmation.** With the default `action_policy`
   (`default_action: require_confirm`, `allow_read_only_without_prompt: true`), a `@tool(idempotent=True,
   tags=("read_only",), capabilities=("read_only", ...))` still returns `"openminion: Policy confirmation required
   ... reply yes"` as the agent's text. With `output_type` set this surfaces as an `AgentOutputValidationError`
   whose input is that policy message, which is confusing. Expected: SDK tools that declare `read_only` run without a
   prompt, or the policy message is raised as its own error type instead of being parsed as model output.
2. **The first request of every turn hides the user tools.** It carries only `submit_output`, with a tool-calling
   contract that lists no tools; replayed alone against `openai/gpt-oss-120b`, the model answers "the inspection tools
   are not available" (`tools/measure_prompt.py` dumps that request). In the full loop the tools are requested in
   later iterations, so one three-tool turn costs several provider round trips (391 s on a paced free tier), which
   is what drives finding 3. We also register the tool manifests (the workaround another builder documented for
   0.1.4); on 0.1.14 our tools were reached with and without it, so we cannot claim it is needed.
3. **A single turn is too big for an 8k tokens/min free tier.** After a first `submit_output`-only call (sent three
   times), the main turn sends ~17k characters of system prompt plus eight built-in tools (`tool.request`, `respond`,
   `coding`, `plan`, `research`, `decompose`, `clarify`, `review.diff`), even for an `Agent` with no tools of ours.
   Groq answered `413 Request too large ... Requested 8683` on `openai/gpt-oss-120b`. Turning off
   `auto_fact_extraction` in the agent profile and `max_tokens` on the provider did not change the request.
   There is also no pacing for free-tier providers. One investigator turn makes several provider calls back to back; on
   Groq's free tier (8k tokens/min for `openai/gpt-oss-120b`) that is a 429 on the second or third call, and the SDK's
   own retry (`max_retries: 2`, `backoff_ms: 300`) is far shorter than the 60 s window. A `tokens_per_minute`
   knob on the connection would make free tiers usable. A single oversized request returns HTTP 413 from Groq.
4. **`output_type` is not enforced on the model.** The judge answered a markdown table prefixed with
   `openminion: `; the SDK's JSON recovery found nothing to recover. A JSON schema placed at the end of `instructions`
   had no effect; the same schema inside the user message worked on the first try.
5. **A second `run()` in the same session did not see the first message.** After a validation failure we asked
   "reply with only the JSON object" in the same `session_id`; the model answered "no investigation data was
   provided". We now resend the facts with the retry.
6. **The freshness guard replaced a grounded answer.** The investigator called all three tools, yet its final text
   was "I couldn't provide a grounded current general answer because Missing exact-date evidence for an exact-date
   freshness answer". The tool facts were fine (the judge used them), but the turn's text was lost. A tool result
   that came from a live fetch could count as fresh evidence.
7. **Our own bug, caught by a real run:** the first live run flagged a 697-character title on
   capegraniteconnection.com. The page title is 63 characters; our parser was concatenating the `<title>` elements of
   inline SVG icons. Fixed in `checks.py` with a regression test. The agent repeats whatever the tools report, which
   is why the facts layer has its own tests.

## License

MIT. Design Master Solutions LLC.

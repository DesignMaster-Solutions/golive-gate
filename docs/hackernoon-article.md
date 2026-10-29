# I Let an Agent Decide If a Website Can Go Live, and Kept the Model Out of the Facts

I run a small web agency. We build and host websites for small businesses in the US and Brazil, and every launch has the same floor: a title and a description, a canonical URL, Open Graph tags, JSON-LD, icons, a sitemap, a 404 page that really returns 404, an `llms.txt`, and a `robots.txt` that does not lock AI search crawlers out. For years we checked that list by hand, one tab per item.

For Cortensor Hackathon #5 I turned the checklist into an agent called **Go-Live Gate**, built on OpenMinion 0.1, the agent SDK from the Cortensor ecosystem. It answers one question: is this site ready, ready with warnings, or blocked? The code is MIT licensed at github.com/DesignMaster-Solutions/golive-gate and there is a 2-minute demo at youtu.be/tVH-x62QV3I.

This post is about the one design decision that made it trustworthy, and about what it looks like to run an agent on open, swappable inference instead of one vendor's API.

## The rule: the model never fetches anything

The tempting version of this agent is one prompt: "here is a URL, tell me if it is ready". That version hallucinates. It will tell you a sitemap exists because sites usually have one.

So Go-Live Gate is split in three agents with different powers:

1. **Investigator.** It can call three read-only tools: `inspect_home`, `inspect_crawlers` and `inspect_404`. They are plain `urllib` with 15-second timeouts and they return facts only (title length, which Open Graph tags exist, which bots `robots.txt` blocks, the HTTP status of a random missing path). If the model skips a tool, the gap is filled deterministically, so the next step never guesses.
2. **Judge.** It gets the raw facts and must return a pydantic-validated `GoLiveReport`: a verdict plus one row per check with the evidence and the fix. The judge has no tools at all. It cannot fetch, so it cannot invent.
3. **Publisher.** A third agent must call `save_report`, a tool marked `dangerous=True`. OpenMinion routes that call through a human approval callback (an interactive y/N, or `--auto-approve yes|no` for CI). If the human says no, nothing is written.

Every run writes `facts.json` next to `report.md`. A reviewer can open both and check the model's verdict against what the tools actually saw. That file is the whole point: the agent is only as honest as the layer under it, and that layer has its own offline tests against a local HTTP server.

```python
class Check(BaseModel):
    name: str
    status: Literal["pass", "warn", "fail"]
    evidence: str   # the fact from a tool result that justifies the status
    fix: str = ""

class GoLiveReport(BaseModel):
    site: str
    verdict: Literal["ready", "ready_with_warnings", "blocked"]
    checks: list[Check]
    summary: str
```

## Why this matters for decentralized inference

When the facts live in code and the model only reasons over them, the model becomes replaceable. That is what makes open inference practical for a business tool.

OpenMinion talks to any OpenAI-compatible provider. Our recorded runs used a free open-weight model (`nvidia/nemotron-3-ultra-550b-a55b:free` through OpenRouter); the same project is set up for the Cortensor Portal with one command (`setup --provider cortensor-portal`). I did not have to rewrite prompts or tools to switch. The judge's output is validated by a schema either way, so a weaker or slower node fails loudly instead of quietly writing a wrong verdict.

If you are building on a decentralized inference network, you cannot assume the same model, the same latency or the same token limits on every call. Designing the agent so that the model is the least trusted part is how you get repeatable results anyway.

## What broke, honestly

The hackathon asked for building blocks tested and lessons learned, and these were the useful ones:

- **Read-only tools still asked for confirmation.** A tool tagged `read_only` and `idempotent=True` still came back as "Policy confirmation required ... reply yes", because the decorator default `min_scope="WRITE_SAFE"` wins over the tags. Declaring `min_scope="READ_ONLY"` fixed it.
- **The first request of every turn was not my agent.** It was the automatic memory extractor, carrying the whole system prompt but exposing only one tool, `submit_output`. A model replaying that request alone said "the inspection tools are not available", and I spent an hour thinking my tools were hidden. A top-level `"brain": {"auto_fact_extraction": {"enabled": false}}` in `agents.json` removes that round trip.
- **One turn did not fit a free tier.** With default tools, the main turn sent about 17k characters of system prompt plus eight built-in tools. Groq answered `413 Request too large` at 8,683 tokens. Free tiers also need pacing the SDK does not have (its retry waits 300 ms; the rate window is 60 s), so I wrapped the adapter to stay under 6.5k tokens per minute.
- **`output_type` is not enforced on the model.** The judge once answered with a markdown table. A JSON schema at the end of the instructions changed nothing; the same schema inside the user message worked on the first try.
- **A second `run()` in the same session lost the first message**, so retries now resend the facts.

And one bug was mine. The first live run flagged a 697-character title on a client site whose real title is 63 characters. My parser was concatenating the `<title>` elements inside inline SVG icons. The agent repeated what the tool reported, which is exactly why the facts layer has regression tests and the model does not get the last word.

## The result

On capegraniteconnection.com, a site we built, the gate returned **ready** with 11 checks: title 63 characters, description 149, canonical present, all four Open Graph tags, 12 JSON-LD types including LocalBusiness and FAQPage, `en-US`, both icons, no AI crawler blocked, a sitemap with 44 URLs, a 3,902-byte `llms.txt`, and a real 404. On a free model the investigator took 232 seconds and the judge 355. Slow, but it runs unattended before every launch, and it is cheaper than the hour of tabs it replaces.

If you want to try it:

```bash
python3.12 -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/python -m unittest test_checks -v
.venv/bin/python gate.py yoursite.com --auto-approve yes
```

The lesson I would carry to any agent on open infrastructure: let code collect the facts, let the model reason over them, put a schema on the answer and a human in front of anything that writes. Then the model can come from anywhere.

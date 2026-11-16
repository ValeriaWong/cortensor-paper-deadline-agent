# What We Learned

Findings from actually building and running Paper Deadline Watch
(`openminion==0.1.21`, free Pollinations endpoint, 2026-11-17). Every item
below was observed in a run, not inferred from docs. Nothing here repeats
another entry's upstream issues — these are this project's own findings.

## 1. The sandbox's `no_proxy` breaks the SDK's HTTP client

**Observed:** every `Agent.run()` failed before any request with
`ProviderError: ... InvalidURL: Invalid port: ':1]'`.

**Root cause (reproduced with plain httpx):** `ProviderHTTPClient` builds
`httpx.Client(follow_redirects=True, trust_env=True)`. This sandbox's
`no_proxy` contains bracketed IPv6 entries (`[::1]`,
`[fd8b:4f84:7d32:99::1]`); httpx mangles one into a URL with port `:1]`.
`trust_env=False` avoids the parse error but then there is no direct
egress (connection timeout) — the egress proxy is mandatory.

**Fix in this repo:** `config.sanitize_no_proxy()` strips bracketed
entries from `no_proxy`/`NO_PROXY` in-process before any runtime is built
(the proxy itself stays). Verified: the same request then returns HTTP 200.

**Reproduction (zero model cost):**

```python
import httpx
httpx.Client(follow_redirects=True, trust_env=True).post(
    "https://text.pollinations.ai/openai/v1/chat/completions",
    json={"model": "openai", "messages": [{"role": "user", "content": "Hi"}]},
)  # raises InvalidURL: Invalid port: ':1]'
```

Upstream note: the SDK could construct its client with an explicit
`proxy=` and a cleaned `no_proxy`, instead of trusting the environment
verbatim.

## 2. Pollinations free tier is 1 reply / 45 s per IP — and the SDK's retry cannot beat it

**Observed:** `HTTP 402: Free limit reached: 1 reply every 45 seconds per
IP. Retry after Ns`. The SDK retries internally at ~300 ms, so an
in-window retry is guaranteed to 402 again.

**Fix in this repo:** `agent.run_agent_with_retry()` catches the 402,
sleeps `Retry-after + 5 s`, and retries up to 3 times; plus
`config.pace_model_call()` keeps 50 s between model calls in-process.
The limit is per egress IP, so other agents sharing the IP still consume
the slot — a retry can legitimately fail, and then the brief falls back
to the labelled template instead of failing the command.

## 3. Free-model replies need defensive JSON parsing

**Observed:** with `output_type=BriefSynthesis`, the SDK raised
`AgentOutputValidationError` on the reply `'paper-watch: 2+2 equals 4.'`
— the model answered in prose, and the SDK prefixes replies with
`agent_id: `.

**Fix in this repo:** run with `output_type=None`, read `result.text`,
strip the `name: ` prefix, extract the first balanced `{...}` object,
then Pydantic-validate. Any failure → template brief, with the raw reply
kept in the run record. The citation table is never model-written, so a
prose reply costs nothing but the synthesis paragraphs.

## 4. The deadline cells need three parsers, not one

**Observed in the verified tables:** `full_deadline` mixes ISO dates,
English month names (`Dec 8, 2026`), `TBD`, `rolling - no fixed
deadline`, and multi-track `;`-separated cells (`2026-10-25 (Long);
2026-11-16 (Short)`). One journal `fixed_deadline` holds two dates
(`2026-11-30; 2026-12-30`). Invalid calendar dates (`2026-02-30`)
appear and must be skipped, never "corrected".

**Fix in this repo:** `dates.py` splits on `;`, parses ISO + both
English month orders per segment, keeps the segment's parenthetical
track label, and returns the earliest date >= today. Covered by
`tests/test_dates.py`.

## 5. The tables have traps a naive loader drops or doubles

**Observed:**
- 18 journal rows have an empty `abbreviation` (full name present, e.g.
  `Neural Networks` with a real 2026-12-01 fixed deadline). Skipping
  empty-abbreviation rows silently loses a deadline that is inside the
  120-day window.
- `CLSR` appears in both the journals and the law table with the
  identical name and date — one venue, listed twice.
- `FSE` / `SEC` are each *two different venues* sharing an abbreviation
  (Foundations of Software Engineering vs Fast Software Encryption;
  Edge Computing vs IFIP Information Security). Dedup by abbreviation
  alone would delete a real venue.

**Fix in this repo:** identifier falls back to `full_name` when the
abbreviation is empty; dedup key is
`(abbreviation, full_name, date, deadline_kind)`.

## 6. Model calls are the slow part; everything else is instant

**Observed:** the deterministic path (`deadlines`, template brief,
`--skip-model` probe) runs in ~1 s; a model brief needs 4 paced calls
(2 subagents + synthesis + reminder) and is dominated by the 45 s free
window and model latency, not by our code. Design consequence: the
default `deadlines` command never touches the model, and the brief
always has a template to fall back to.

## 7. (Live-run outcomes — filled in after the probe run)

- Model brief (Pollinations, topic 人工智能): _pending — see
  `fixtures/live_run_2026-11-17_brief-ai.json`._
- Full probe suite (`paper-watch probe`): _pending — see
  `docs/BUILDING_BLOCKS_TESTED.md`._

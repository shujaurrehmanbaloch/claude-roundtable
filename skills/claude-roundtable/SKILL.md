---
name: claude-roundtable
description: Bring routed frontier models into the Claude Code harness for consult, critique, or implementation via the local CLIProxyAPI proxy — currently nicknamed sol, terra, and luna (fetch /v1/models if the list may have changed; more providers like Gemini/Grok/Kimi show up automatically once logged in). No slash command required — trigger on natural language like "ask terra", "with sol", "use luna", or any name that isn't a Claude model. Typos are fine; match against the live model list. /consult, /critique, /implement exist as explicit shortcuts but are not required.
---

# Claude Roundtable — consult / critique / implement on routed models

You are the ORCHESTRATOR, running on this session's normal Anthropic login.
NEVER set `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` on this session's own
environment — routed env vars go inline on individual headless calls only.
(Anthropic's terms ban routing a Claude subscription login through a proxy;
this flow never does that.)

For Claude-to-Claude mixing (Fable ↔ Sonnet ↔ Opus) no routing is needed:
the user can `/model` mid-session. This skill is for the OTHER subscriptions.

## Config

| Env var | Default | Meaning |
|---|---|---|
| `MODEL_ROUTER_URL` | `http://127.0.0.1:8317` | proxy address |
| `MODEL_ROUTER_KEY` | `my-proxy-key` | key from the proxy's config.yaml |
| `MODEL_ROUTER_MODEL` | `gpt-5.6-sol` | default routed model |
| `MODEL_ROUTER_EFFORT` | `high` | effort for IMPLEMENT only |

CONSULT and CRITIQUE always run at `high` effort (the remote model is doing
the thinking). IMPLEMENT runs at `MODEL_ROUTER_EFFORT` (the spec carries the
thinking).

**Model picking (fuzzy, forgiving):** the user speaks casually — "with terra",
"ask sol", "use luna" — and may typo ("dera", "tera"). Map whatever they said
to the closest ID in the proxy's `/v1/models` list (e.g. "terra" →
`gpt-5.6-terra`). Fetch the list if you haven't this session. Always state
which model you picked ("Routing to gpt-5.6-terra") BEFORE the call, so a
wrong guess is visible immediately. If nothing plausibly matches, use the
default and say so. If the user names no model, use `MODEL_ROUTER_MODEL`
(gpt-5.6-sol) without comment.

## Preflight — once per session, before the first routed call

```bash
curl -s "${MODEL_ROUTER_URL:-http://127.0.0.1:8317}/v1/models" \
  -H "Authorization: Bearer ${MODEL_ROUTER_KEY:-my-proxy-key}"
```

Connection refused → start the proxy (`~/cli-proxy-api/cli-proxy-api.exe
-config ~/cli-proxy-api/config.yaml`, backgrounded) and re-check.
Auth error or empty model list → STOP and tell the user their provider login
needs (re)running: `cd ~/cli-proxy-api && ./cli-proxy-api.exe -config config.yaml -codex-login`.

## CONSULT — a second opinion (read-only)

```bash
ANTHROPIC_BASE_URL="${MODEL_ROUTER_URL:-http://127.0.0.1:8317}" \
ANTHROPIC_AUTH_TOKEN="${MODEL_ROUTER_KEY:-my-proxy-key}" \
claude -p "<question, with relevant file/dir PATHS for it to read>" \
  --model "${MODEL_ROUTER_MODEL:-gpt-5.6-sol}" --effort high \
  --bare --max-turns 15 --output-format json \
  --disallowedTools "Edit,Write,NotebookEdit"
```

Prompt rules:
- Pass PATHS, never file contents — the worker reads them on its own quota.
- End every consult prompt with: "Return your verdict and top points in
  30 lines or fewer."
- The JSON output includes a `session_id` — keep it for follow-ups.

## CRITIQUE — same call as CONSULT, different framing

Prompt shape: "Critique <paths or described change>: strengths, top 5 issues
ranked by severity, one concrete fix each. 40 lines or fewer."

## FOLLOW-UP — cross-model memory (the important trick)

The remote model remembers its own session. To continue a consult/critique
(e.g. hand it Fable's counterpoint), resume — send ONLY the new delta:

```bash
ANTHROPIC_BASE_URL="${MODEL_ROUTER_URL:-http://127.0.0.1:8317}" \
ANTHROPIC_AUTH_TOKEN="${MODEL_ROUTER_KEY:-my-proxy-key}" \
claude -p --resume <session_id> "<only the new information or counterpoint>" \
  --model "${MODEL_ROUTER_MODEL:-gpt-5.6-sol}" --effort high \
  --bare --max-turns 15 --output-format json \
  --disallowedTools "Edit,Write,NotebookEdit"
```

Never re-paste what the remote model already saw.

## IMPLEMENT — build via a routed model (one compound call per task)

```bash
ANTHROPIC_BASE_URL="${MODEL_ROUTER_URL:-http://127.0.0.1:8317}" \
ANTHROPIC_AUTH_TOKEN="${MODEL_ROUTER_KEY:-my-proxy-key}" \
claude -p "<self-contained task spec>" \
  --model "${MODEL_ROUTER_MODEL:-gpt-5.6-sol}" \
  --effort "${MODEL_ROUTER_EFFORT:-high}" \
  --bare --max-turns 30 --permission-mode acceptEdits \
&& ls -la <expected output paths> \
&& grep -c "<acceptance marker>" <built file>
```

The spec must be self-contained: exact file paths, what to create or change,
acceptance criteria, what NOT to touch, and "verify your work, then report
changed paths + how you verified". You make every design call in the spec —
the worker just types. Misses go back as a fix delegation with the failure
evidence attached; you do not fix code yourself.

## TOKEN RULES (non-negotiable — this is why the skill exists)

1. Paths, never contents, in delegate prompts.
2. Cap every response ("≤30 lines") — whatever comes back lives in this
   session's context forever.
3. Independent opinions run in PARALLEL (your own analysis in-session while
   the routed call runs); chain Fable→Sol only when one must critique the other.
4. Follow-ups go through `--resume`, sending only the delta.
5. Never read a whole built file back into context — verify with targeted
   `grep -c` / `head` / `tail` slices.

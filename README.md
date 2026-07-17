# Claude Roundtable

Use Claude Roundtable to bring GPT-5.6 models like Sol and
other frontier models into Claude's working harness.

The point is simple: Claude Code is where the real work already happens.
Your files, terminal output, edits, tools, reasoning thread, and project
memory are already there. Claude Roundtable keeps Claude as that home base,
then invites other models into the same workflow when their perspective
helps.

So instead of leaving Claude to ask another model somewhere else, you bring
that model into Claude's room. Ask Sol for a second opinion or ask Terra to
implement. The answer comes back inline in the same chat window, with the
thread intact.

It feels like a roundtable inside Claude: one chat, one memory, one working
context, and multiple model perspectives. Claude keeps the tools, the flow,
and the final judgment.

## Why this matters

- Stay in one Claude Code session while getting multiple model worldviews.
- Keep follow-ups cheap and useful: each routed model keeps its own session,
  so Claude can resume that model's memory instead of re-sending everything.
- Keep Claude as the orchestrator: it decides what to ask, checks the result,
  and carries the final answer forward in the same chat.
- Use Claude Code as the harness for other frontier models, so the extra
  intelligence shows up inside the environment where your work already lives.

Behind the scenes, Claude Roundtable uses CLIProxyAPI to reach models through
your existing provider subscription login. It does not give free access, and
it does not require copying work into a separate app window.

It is optimized for real back-and-forth work: ask once, get the answer back
inside Claude, then follow up without rebuilding the whole prompt from
scratch.

## How to use it

Just talk naturally:

```text
Sol, is this migration actually safe to ship?
Get Sol to critique this refactor before I merge it.
Let Terra handle this small change, then verify it works.
```

You can also use explicit commands:

```text
/consult ...
/critique ...
/implement ...
```

## Setup option 1: ask Claude

Open Claude Code in this folder and say:

```text
Set up the Claude Roundtable plugin from here. Install what you can, then
tell me the exact login command I need to run myself.
```

Claude can place the skill and commands for you. The only part you usually
need to do yourself is the browser login for the routed provider account.

## Setup option 2: manual

1. Download [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) —
   grab the [latest release](https://github.com/router-for-me/CLIProxyAPI/releases)
   for your OS and unzip it, for example to `~/cli-proxy-api`.

2. Create `config.yaml` from `config.example.yaml`:

   ```yaml
   host: "127.0.0.1"
   port: 8317
   api-keys:
     - "pick-any-string-you-want"
   ```

3. Log in to the routed provider:

   ```bash
   cd ~/cli-proxy-api
   ./cli-proxy-api.exe -config config.yaml -codex-login
   ```

   On macOS or Linux, drop `.exe` if your binary does not use it.

4. Set the router environment variables.

   PowerShell:

   ```powershell
   $env:MODEL_ROUTER_URL="http://127.0.0.1:8317"
   $env:MODEL_ROUTER_KEY="pick-any-string-you-want"
   $env:MODEL_ROUTER_MODEL="gpt-5.6-sol"
   $env:MODEL_ROUTER_EFFORT="high"
   ```

   macOS/Linux:

   ```bash
   export MODEL_ROUTER_URL="http://127.0.0.1:8317"
   export MODEL_ROUTER_KEY="pick-any-string-you-want"
   export MODEL_ROUTER_MODEL="gpt-5.6-sol"
   export MODEL_ROUTER_EFFORT="high"
   ```

5. Install the plugin in Claude Code:

   ```text
   /plugin marketplace add <this folder>
   /plugin install claude-roundtable@claudex
   ```

   Or install by hand: copy `skills/claude-roundtable/` to `~/.claude/skills/`
   and copy `commands/*.md` to `~/.claude/commands/`.

**If a routed call fails right away**, check the proxy is up and you're
logged in:

```bash
curl http://127.0.0.1:8317/v1/models -H "Authorization: Bearer pick-any-string-you-want"
```

Connection refused → the proxy isn't running, start it (step 3). Empty list
or an auth error → the provider login didn't take, re-run the `-codex-login`
command.

## Important rule

Never route Claude's own login through the proxy, and never set
`ANTHROPIC_BASE_URL` or `ANTHROPIC_AUTH_TOKEN` on the main Claude session.
Claude Roundtable only sets those values inline for individual routed calls.

## Other models

The same setup can route to Gemini, Grok, Kimi, or other supported models if
CLIProxyAPI supports that provider and you have the subscription/login for it.

This README is focused on the subscription-backed flow, not a direct API-key
workflow.

## Inspiration

Sparked by the community conversation around using frontier models inside
Claude Code, including [Theo Browne's GPT-5.6 Sol experiment](https://x.com/theo/status/2075757587010929003).

Claude Roundtable turns that idea into a reusable workflow: same Claude Code
harness, any routed model, inline answers, and optimized follow-ups that
resume the routed model's own session.

## License

MIT - see [LICENSE](LICENSE).

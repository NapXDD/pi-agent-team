# pi Agent Team — Plan / Code / Review

Four-role agent team in `pi` — orchestrator, plan, code, review — with no
third-party package. Each role runs as a separate `pi` process via `bin/pi-team`.

| Role | Definition | Provider | Model | Thinking | Access |
|---|---|---|---|---|---|
| Orchestrator | `orchestrator.md` | anthropic | `claude-sonnet-5` | medium | read-only |
| Plan | `researcher.md` | anthropic | `claude-fable-5-1` | high | read-only |
| Code | `implementor.md` | opencode-go | `deepseek-v4.1-flash` | off | **read-write** |
| Review | `reviewer.md` | anthropic | `claude-opus-5` | high | read-only |

All four are launched the same way: `./bin/pi-team <role>`. Only the implementor
has `edit`/`write` — enforced by pi's `--tools` allowlist, not by prompt.

Status: **working.** pi 0.85.1 on Node 22, four role definitions, the `pi-team`
runner and `team-check` verifier. Both providers authenticated; the anthropic
roles reach Claude through a VPN-gated corporate proxy (step 2).

Verified live: the implementor passes its smoke test on `deepseek-v4.1-flash`,
and the anthropic roles work from a normal terminal.

**One operational caveat:** the proxy's quota is shared per key. Running an
interactive Claude Code session and `pi` against it at the same time causes
`429 rate_limit_error` on whichever loses the race. Run one at a time.

---

## Why no orchestrator package

The original design used `@redentor_dev/pi-orchestrator`. Dropped, for two
reasons found during verification:

1. **It is broken against pi 0.74.2.** The package passes `--session-id` on every
   *fresh* delegation (`extensions/orchestrator-workflow.ts:873`), and pi 0.74.2
   has no such flag — `pi --session-id x -p hi` → `Error: Unknown option:
   --session-id`. Only resumed delegations use the documented `--session`. So
   every new delegation would fail.
2. **Trust.** pi packages run with full system access. 58 downloads/month, one
   maintainer, 2,685 lines. Provenance is clean (OIDC-published from the matching
   public repo), which rules out a hijacked publish but says nothing about
   behaviour. Not worth it for what is, in the end, three `pi` invocations.

What is given up: `/team`, the `delegate_*` tools, `review_diff`, the status
widget, and automatic session-resume plumbing. `bin/pi-team` covers the useful
part of that. The agent files keep the package's exact frontmatter format, so
adopting it later (once it fixes the flag) is just `pi install`.

---

## Step 0 — Verification results

Checked against the real install (pi 0.74.2) and package source. Several
specifics in the original draft were wrong.

- [x] **Provider `opencode-go` is built in.** `auth.json` key `opencode-go`,
      env var `OPENCODE_API_KEY`.
- [x] **Model IDs.** `deepseek-v4-flash` is real, and is a **distinct model**
      from `deepseek-v4.1-flash` — both exist independently in the `opencode-go`
      catalog. The draft's claim that it is "temporarily routed to V4.1-Flash for
      backwards compatibility" is unsupported — they are separate models, and the
      implementor now runs on `deepseek-v4.1-flash` by choice (same 1M context and
      384K output, plus image input). `claude-fable-5-1`,
      `claude-opus-5` and `claude-sonnet-5` all exist under `anthropic`.
      Caveat: verified against the OpenCode catalog cached at
      `~/.cache/opencode/models.json`, because pi only enumerates models for
      providers you are authenticated to. Re-confirm with `pi --list-models`
      after step 2.
- [x] **The DeepSeek tool-call bug is real.** When an assistant message carries
      `tool_calls`, the V4 API requires its `reasoning_content` to be replayed
      verbatim; most harnesses drop it and get `400 invalid_request_error: The
      reasoning_content in the thinking mode must be passed back to the API`.
      Filed against opencode (#24190, #25058), codex (#24500), oh-my-pi (#1484).
      **pi already handles it** — `pi-ai` implements
      `compat.requiresReasoningContentOnAssistantMessages` and
      `thinkingFormat: "deepseek"`. Running the implementor at `thinking: off`
      avoids the code path entirely, which is what this plan does.
- [x] **Auth.** Both env vars and `auth.json` work. Resolution order:
      `--api-key` → `auth.json` → environment variable → `models.json`.
      `auth.json` wins over env, so do not set both.
- [x] **No per-command deny list.** There is a *tool* allowlist (`--tools`,
      enforced by the harness), but `bash` is all-or-nothing. `git push` cannot
      be blocked by configuration.
- [x] **No native model fallback.** Switching models is a manual edit.
- [x] **Agents are global.** `~/.pi/agent/agents/` is the only agent directory.
      `.pi/settings.json` project overrides exist for *settings*, not agents.

### Frontmatter schema corrections

The draft was wrong in three ways. Verified against `loadAgent()` in the package
source and its bundled agents:

| Field | Draft had | Actually |
|---|---|---|
| `thinking` | `true` / `false` | `off` \| `minimal` \| `low` \| `medium` \| `high` \| `xhigh` (default `medium`) |
| `execution` | `read-only` / `read-write` | `parallel` \| `sequential` — **concurrency scheduling, not permissions**. An invalid value silently falls back to a default, so it reads like a security control while enforcing nothing. |
| `name` | omitted | falls back to filename; be explicit |
| `tools` | absent | comma-separated allowlist — **this** is what enforces read-only |

`pi-team` ignores `execution` (irrelevant without a delegating parent) and
validates `thinking` strictly rather than silently defaulting.

---

## Step 1 — Install

```bash
nvm use 22                                                        # REQUIRED
npm install -g --ignore-scripts @earendil-works/pi-coding-agent   # → 0.85.1
```

Done. `--ignore-scripts` rather than `curl -fsSL https://pi.dev/install.sh | sh`.
No other packages needed.

### Node 22 is required

**pi >= 0.85 requires Node >= 22.19.0.** This machine's default is still
v20.20.2, and npm installs pi anyway (engines are advisory), so pi crashes at
startup with a 55 KB minified stack trace about `node:fs` not exporting
`globSync`. v22.23.2 is installed under nvm; `nvm use 22` selects it for a shell,
`nvm alias default 22` makes it permanent.

`bin/pi-team` and `bin/team-check` now check the Node major version first and
fail with that message instead of the stack dump.

### Why 0.85.1 and not 0.74.2

The first install landed 0.74.2, which broke two things that only surfaced under
live testing:

- **Its model catalog stopped at `claude-opus-4-7` / `claude-sonnet-4-6`.** None
  of `claude-sonnet-5`, `claude-opus-5` or `claude-fable-5-1` existed, so pi
  warned "Using custom model id" and passed the string through without knowing
  context limits or thinking support. 0.85.1 has all four, at 1M context / 128K
  output with thinking supported.
- **It never sent the `x-opencode-session` header**, which OpenCode Go now
  requires: `400 Request is missing x-opencode-session and cannot be routed
  efficiently`. 0.85.1 sends it.

This is exactly the Step 0 caveat coming true: the model IDs were verified
against the OpenCode catalog at `~/.cache/opencode/models.json`, not pi's own.
Always re-check with `pi --list-models` after auth.

## Step 2 — Auth (corporate proxy)

Claude here is reached through a **VPN-gated proxy**, not `api.anthropic.com`.
The `pk_...` credential is a *virtual key* for that gateway. pi was sending it
straight to Anthropic, hence `401 invalid x-api-key` — the key was fine, the
endpoint was wrong.

Claude Code's own config (`~/.claude/settings.json`) already describes the route:

```
ANTHROPIC_BASE_URL       = https://claude-proxy-<...>.southeastasia-01.azurewebsites.net
ANTHROPIC_CUSTOM_HEADERS = X-Api-Key:pk_...
ANTHROPIC_AUTH_TOKEN     = <placeholder; real auth is the X-Api-Key header>
```

### The fix — `~/.pi/agent/models.json`

pi can route a built-in provider through a proxy with no code and no extension.
Per `docs/models.md`: "All built-in Anthropic models remain available. Existing
OAuth or API key auth continues to work."

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://claude-proxy-<...>.southeastasia-01.azurewebsites.net"
    }
  }
}
```

Nothing else changes: `auth.json` keeps the `pk_...` value, and pi sends it as the
`x-api-key` header — the same header the gateway expects (HTTP header names are
case-insensitive, so `X-Api-Key` and `x-api-key` are the same header).

If the gateway turns out to want the key somewhere else, add an explicit header
instead — `headers` values support `$ENV_VAR` and `!command` interpolation:

```json
{ "providers": { "anthropic": {
    "baseUrl": "https://<proxy>",
    "headers": { "X-Api-Key": "$ANTHROPIC_VIRTUAL_KEY" }
} } }
```

**Note:** writing this file is blocked by the Claude Code sandbox as traffic
redirection, so create it yourself.

### The gateway also pins the client

With the base URL right, the next response was:

```
426 {"error":"Outdated Claude Code version detected.
     Please use version: claude-cli/2.1.269 (external, cli)"}
```

HTTP 426 = Upgrade Required. The gateway inspects the client identity, not just
the credential — it is a Claude Code-specific proxy, so pi is refused on sight.
Confirmed by the owner (the user) as their own proxy, pi is set to identify
accordingly:

```json
{ "providers": { "anthropic": {
    "baseUrl": "https://claude-proxy-<...>.southeastasia-01.azurewebsites.net",
    "headers": { "User-Agent": "claude-cli/2.1.269 (external, cli)" }
} } }
```

### Then a schema mismatch

Past the client gate, the next response was:

```
400 invalid_request_error: messages.1.output_config: Extra inputs are not permitted
```

The proxy relays to an Anthropic API version that predates the `effort` /
adaptive-thinking fields. pi's generated metadata sets
`compat.supportsMidConvoEffort: true` for the Claude 5 models, which makes
`anthropic-messages.js` inject pseudo-messages carrying the unsupported field:

```js
messages.push({ role: "system", content: [], output_config: { effort: activeEffort } });
```

Disable it per model via `modelOverrides` (documented as applying to built-in
provider models):

```json
"modelOverrides": {
  "claude-opus-5": { "compat": {
      "supportsMidConvoEffort": false,
      "forceAdaptiveThinking": false
  } }
}
```

Both flags are needed. `supportsMidConvoEffort: false` removes the message-level
`output_config` and the `thinking: {type: "adaptive", block_binding: ...}` shape;
`forceAdaptiveThinking: false` stops the *top-level* `output_config` set by the
very next branch. Together they fall through to classic budget-based thinking
(`thinking: {type: "enabled", budget_tokens: N}`), which older Anthropic-
compatible endpoints accept.

**Confirmed fixed (2026-09-17):** with both flags set, the
`messages.N.output_config` 400 no longer occurs. The next response was a
`429 rate_limit_error` from the gateway — a different failure class, which is how
we know the schema path is clear.

### Then rate limits — the current blocker

With the schema clear, every anthropic call returns:

```
429 {"type":"error","error":{"type":"rate_limit_error","message":"Error"}}
```

Diagnosis so far:

- **Not model-specific.** Fails identically on `claude-opus-5` and
  `claude-fable-5-1`, so it is not an Opus-only quota.
- **Not transient.** Still 429 after 75s of manual backoff, with fresh
  request_ids each time (so these are real attempts, not cached failures).
- **The proxy itself is healthy.** The Claude Code session doing this setup runs
  through the same gateway with the same key, concurrently, without issue.

**Resolved: it was contention.** pi works normally from a terminal when no
Claude Code session is running against the same key. The gateway is not
singling pi out — the `User-Agent` and `compat` settings are correct — the two
clients were simply competing for one shared per-key budget, and whichever lost
the race got the 429.

The proxy itself is fast and healthy throughout: three unauthenticated probes
returned in 0.574s / 0.634s / 0.665s. A 429 is an immediate, deliberate refusal,
not slowness — worth remembering, because "the proxy is slow" and "the proxy is
refusing" call for completely different fixes.

**Operational rule: run one client at a time.** Do not drive `pi` and an
interactive Claude Code session against this gateway simultaneously.

pi's retry defaults (3 tries at 2s/4s/8s ≈ 14s) are too narrow for a per-minute
limiter, so `~/.pi/agent/settings.json` now carries:

```json
"retry": { "enabled": true, "maxRetries": 5, "baseDelayMs": 5000 }
```

That is 5s/10s/20s/40s/80s ≈ 155s of backoff before giving up.

**Expect more of these.** The gateway is on an older schema, so other pi
features may be rejected as it evolves. The same `compat` mechanism covers the
likely candidates — `supportsCacheControlOnTools`, `supportsStrictTools`,
`supportsTemperature` (Opus 4.7+ rejects non-default temperature). The flags are
documented in `pi-ai/dist/types.d.ts` around line 575.

If the gateway later bumps its pinned version, the 426 message names the new
one — update the `User-Agent` string to match. Worth knowing this is the likely
failure mode after a proxy upgrade, since the symptom (426 on every anthropic
role) looks nothing like a version pin at first glance.

### Common mistake, already hit

The first `auth.json` value was 45 characters beginning `X-Api-Ke` — the whole
header line (`X-Api-Key:` + the 35-char key) pasted into the key field. Only the
value after the colon belongs there.

`bin/team-check` now shape-checks credentials before spending an API call, so a
wrong-field paste is caught for free instead of via a 401.

### Requires VPN

The gateway is only reachable on the VPN. Connect before running any anthropic
role, or every call fails at the network layer.

## Step 2b — Auth status

`~/.pi/agent/auth.json` exists, empty (`{}`), mode `0600`. pi creates it at that
mode, so a `chmod` is unnecessary.

Preferred — `/login` from inside pi, which keeps keys out of shell history:

```bash
pi
/login          # select anthropic, then run again for opencode-go
```

If writing the file by hand, prefer indirection over a literal key on disk:

```json
{
  "anthropic":   { "type": "api_key", "key": "!op read 'op://vault/anthropic/credential'" },
  "opencode-go": { "type": "api_key", "key": "OPENCODE_API_KEY" }
}
```

- `"!cmd"` runs a shell command and uses stdout (cached for the process lifetime)
- a bare name is read as an environment variable
- anything else is literal

Anthropic note: subscription auth (Claude Pro/Max) via `/login` bills
third-party harness usage as **extra usage, per token** — it does not draw on
plan limits.

## Step 3 — Role definitions

**Done.** In `~/.pi/agent/agents/` (global — every repo on this machine):

- `researcher.md` — anthropic / `claude-fable-5-1`, `thinking: high`,
  `tools: read,bash,grep,find,ls`
- `implementor.md` — opencode-go / `deepseek-v4.1-flash`, `thinking: off`,
  `tools: read,bash,edit,write,grep,find,ls`
- `reviewer.md` — anthropic / `claude-opus-5`, `thinking: high`,
  `tools: read,bash,grep,find,ls`

Read-only is genuinely enforced for researcher and reviewer: no `edit`, no
`write`, and `--tools` is an allowlist the harness applies to the subagent
process. For the implementor, `bash` is unrestricted, so "never push / amend /
force-reset / write outside the repo" lives in its body prompt and is enforced
only by your reading the diff.

## Step 4 — The runner

**Done.** `bin/pi-team` reads a role's frontmatter for provider/model/thinking/
tools and its body for the system prompt, then invokes pi.

```bash
./bin/pi-team --list                                  # roles, models, ro/rw
./bin/pi-team researcher  "Plan: add retry to the X client"
./bin/pi-team implementor "<the approved plan>"
./bin/pi-team reviewer    "Review the diff against: <plan>"

./bin/pi-team implementor -c "Fix the two BLOCKING findings: ..."   # continue
./bin/pi-team reviewer --dry-run "..."                # print the pi command
./bin/pi-team researcher - < brief.md                 # task from stdin
./bin/pi-team implementor -i                          # interactive TUI
```

Sessions are per role under `~/.pi/agent/team-sessions/<role>/`; `-c` continues
the most recent, `--session <id>` resumes a specific one. Each role runs with
`--no-extensions` so a subagent cannot recurse into the team setup.

## Step 5 — Orchestrator

**Sonnet 5**, defined like any other role in
`~/.pi/agent/agents/orchestrator.md`, so it is launched the same way:

```bash
./bin/pi-team orchestrator -i        # interactive; it calls pi-team for the rest
```

It has `read,bash,grep,find,ls` — no `edit`/`write`, so all code genuinely goes
through the implementor.

### Sonnet orchestrator + Opus reviewer

This ordering (the reviewer stronger than the orchestrator) is deliberate:

- The two roles fail differently. Orchestration is bounded and structured;
  review is open-ended defect-hunting, where capability converts most directly
  into caught bugs.
- **You** are the orchestrator's backstop — the plan gate puts a human in the
  loop before code exists. Nobody backstops the reviewer; its findings are the
  last thing before merge.
- Acceptance criteria now come from the researcher, which moved the
  judgment-heaviest work off the orchestrator.

The one real risk is not bad decisions but **brief fidelity**: the orchestrator
is upstream of everything the reviewer knows, so a constraint dropped while
paraphrasing the plan is invisible to review. Opus cannot catch what it was never
told. The mitigation is process, not model — the orchestrator prompt requires
acceptance criteria and validation commands be copied **verbatim** into both
downstream briefs. That closes the gap more reliably than an upgrade, since a
bigger model still summarizes when allowed to.

Pin it explicitly and do not drop below this tier. The orchestrator is not a
router: it owns decomposition, acceptance criteria, brief-writing and the
loop-back decision. Brief quality caps every subagent's output — subagents start
with **zero context** and see only the task text, so a vague brief yields a
plausible diff that solves the wrong problem, and the reviewer will not catch it
(it checks the diff against the plan, not the plan against your intent). It is
also the only role that never gets a second pass.

The cost worry is smaller than it looks: the orchestrator should never scan the
codebase itself — that is the researcher's job — so its context is bounded by
report sizes (~150 lines research, ~80 lines implementation), not file dumps.

## Step 6 — Fallback for the code agent

No native fallback exists, so this is a manual switch. Trigger:
**two consecutive failures on the same tool call → switch.**

```bash
sed -i 's/^model: .*/model: deepseek-v4-flash/' ~/.pi/agent/agents/implementor.md
```

Candidates on the same provider, verified present in pi's catalog:
`deepseek-v4-flash` (the previous primary), `deepseek-v4-pro`, `glm-5.1`,
`kimi-k2.6`, `qwen3.6-plus`, `minimax-m2.7`.

## Step 7 — Verify

```bash
pi --list-models | grep -E 'fable-5-1|opus-5|sonnet-5|deepseek-v4.1-flash'
./bin/pi-team --list
```

Confirms the four model IDs against pi's own catalog rather than the OpenCode
cache. Tested so far only to the auth boundary: every role launches pi with
valid flags and stops at "No API key found".

## Step 8 — Smoke test before any real work

```bash
./bin/pi-team implementor "Create /tmp/smoke.txt containing one line: hello."
./bin/pi-team reviewer    "Review /tmp/smoke.txt. Does it contain exactly one line?"
```

Confirms each provider can actually tool-call — especially the OpenCode Go route
in read-write mode, and that the DeepSeek round-trip survives more than one turn.
Finding a broken provider here costs seconds; finding it mid-feature costs a
wasted implementation pass.

**Result (2026-09-17):**

- **implementor / opencode-go / deepseek-v4.1-flash — PASS.** Wrote the file,
  then ran `wc -l` to verify: a multi-turn tool sequence, which is precisely what
  the `reasoning_content` bug would break. It followed the output format and
  checked criterion 1 by number with evidence.
  (Also passed on `deepseek-v4-flash` before the switch — on that run it caught
  its own missing trailing newline, rewrote with `printf`, and reported the
  correction instead of claiming success. Both models honour the prompt.)
- **reviewer / anthropic / claude-opus-5 — FAIL.**
  `401 authentication_error: invalid x-api-key`. The `anthropic` entry in
  `auth.json` is 45 characters beginning `X-Api-Ke` — a header name, not a key.
  Anthropic keys begin `sk-ant-`. Re-run `/login` and select anthropic.

`pi --list-models` writes to **stderr**, not stdout; `team-check` captures with
`2>&1` accordingly.

## Step 9 — Run

```
plan → [STOP, you approve] → implement → review
  ↓  BLOCKING findings? → implementor -c with a focused follow-up
  ↓  re-review; still blocking after 2 rounds? → stop, hand back to human
```

Two rules the original draft left out, both of which belong in the orchestrator's
own instructions:

- **The plan gate.** Nothing gets written until you approve the plan. Fixing a
  wrong sentence is cheaper than reviewing 300 lines of wrong diff.
- **The review→fix loop, capped.** An uncapped fix/re-review cycle is the fastest
  way to burn tokens on an agent team. Two rounds, then a human.

### Why the loop terminates

Open-ended defect-hunting has no natural fixed point — a capable reviewer asked
to find problems will always find problems, and every fix adds lines that give it
somewhere new to look. Four things bound it:

1. **Scope contracts each round.** The first review is the only exhaustive pass.
   A re-review checks exactly two things: were the named findings fixed, and did
   the fix introduce a new BLOCKING defect. A defect that could have been raised
   in round one but was not is NON-BLOCKING from then on. Shrinking scope is the
   actual convergence guarantee; the rest are backstops.
2. **The orchestrator labels the round** (`FIRST REVIEW` / `RE-REVIEW of findings
   N`). An unlabelled re-review gets a fresh sweep, which is exactly the moving-
   goalposts failure.
3. **Only BLOCKING findings re-enter the loop.** NON-BLOCKING ones go to the
   human as advisory and are never auto-delegated.
4. **A hard cap of two fix rounds**, then it stops and hands back.

Findings also require a concrete failure scenario — inputs or state producing a
wrong result. That single rule kills most speculative findings before they can
become a fix round.

Every brief must be self-contained: Goal, Context (file paths with line refs),
Constraints, Acceptance criteria, Validation to run, Out of scope. Never
reference "the plan above" — the subagent cannot see it.

## Who owns acceptance criteria

The **researcher** proposes them; you approve them at the plan gate; the
implementor checks them off by number; the reviewer verifies them against the
diff. Each role's prompt is written to that contract.

The researcher owns the proposal because it is the only role that reads the
codebase before code is written. The orchestrator must not scan the codebase, so
it cannot know the test runner, where tests live, or which fixtures exist — it
would produce criteria like "handles errors gracefully" instead of a runnable
command. Fable reads the repo, so it can name `pytest tests/test_client.py -k
retry` and mean it.

Two things this ordering buys:

- **Criteria are fixed before implementation.** Written afterwards, they get
  shaped to fit whatever was built — the test-to-fit failure, where everything
  passes and nothing was verified.
- **The plan gate gets teeth.** Approving "add retry to the X client" commits you
  to nothing. Approving five numbered criteria and the command that checks them
  is a decision you can actually hold the diff against.

The researcher specifies; it does not write tests — it has no `edit` or `write`
tool. The implementor writes them.

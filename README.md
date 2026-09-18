# pi-agent-team

An agent team — **orchestrator, plan, code, review** — built on
[`pi`](https://pi.dev) with no third-party orchestration package. Each role runs
as its own `pi` process, launched by a single script.

The point is the **separation**: the roles that think about the code cannot edit
it, and the role that edits it is handed a brief rather than a conversation.
Read-only is enforced by `pi`'s `--tools` allowlist, not by asking the model
nicely.

## Two workflows

Pick one per run by invoking its orchestrator. They differ in exactly one place:
who writes the plan.

**Automaton** — the researcher plans, you approve at the plan gate, the loop runs.

```
researcher → [you approve] → implement → review → [≤2 fix rounds] → you
```

**Hybrid** — you plan yourself, in whatever session you like, and hand the
markdown over. There is no researcher and no plan gate: approval already
happened when you wrote it. One command, runs to the end.

```
you write plan.md → [pre-flight] → implement → review → [≤2 fix rounds] → you
```

```bash
./bin/pi-team orchestrator-automaton -i                  # researcher plans
./bin/pi-team orchestrator-hybrid - < plan.md            # you planned
```

Hybrid exists because planning is the step that benefits most from a
conversation — you iterate on scope, argue with the model, change your mind —
and that is exactly what a one-shot subagent cannot do. The rest of the pipeline
is mechanical enough to hand off.

Its one gate is a **pre-flight** on the plan you hand over. A plan written in a
live conversation says "the file we looked at" and "handle errors gracefully",
because you were both there. The implementor sees only the brief text, so the
hybrid orchestrator checks for dangling references and unverifiable criteria and
bounces them back to you rather than guessing. It is told not to fill gaps
itself — an orchestrator that has not read the codebase writes exactly the vague
criteria that make the review chain unfalsifiable.

In both workflows the **first review is the only open-ended pass**: an
exhaustive hunt, no narrowing. Every re-review after it is strictly narrower —
were the named findings fixed, did the fix break something new — which is what
makes the loop terminate instead of drifting.

## The team

| Role | Definition | Provider | Model | Thinking | Access |
|---|---|---|---|---|---|
| Orchestrator (automaton) | `orchestrator-automaton.md` | openai-codex | `gpt-5.6-sol` | medium | read-only |
| Orchestrator (hybrid) | `orchestrator-hybrid.md` | openai-codex | `gpt-5.6-sol` | medium | read-only |
| Plan | `researcher.md` | openai-codex | `gpt-5.6-sol` | medium | read-only |
| Code | `implementor.md` | opencode-go | `deepseek-v4.1-flash` | off | **read-write** |
| Review | `reviewer.md` | openai-codex | `gpt-5.6-sol` | medium | read-only |

Only the implementor has `edit`/`write`. The two orchestrators are alternatives,
never both in one run — nothing stops you starting one from the other, so don't.

> **Note.** The thinking roles were previously differentiated by model — Sonnet
> orchestrating, Fable planning, Opus reviewing, on the reasoning that review is
> open-ended defect-hunting and deserved the strongest model.
> [PLAN.md](PLAN.md#sonnet-orchestrator--opus-reviewer) still argues that case.
> They now all run `gpt-5.6-sol` at `medium`, so that differentiation is gone:
> the reviewer is no longer stronger than the roles it checks. The role prompts,
> not the model tier, are doing the separating.

## Requirements

- **Node >= 22.19** — pi 0.85+ crashes on older Node with a 55 KB minified stack
  trace about `node:fs` not exporting `globSync`. Both scripts check the version
  first and fail readably instead.
- **pi 0.85.1+** — 0.74.x lacks the Claude 5 model IDs and does not send the
  `x-opencode-session` header OpenCode Go now requires.
- Credentials for `openai-codex` and `opencode-go`. (`anthropic` too, only if
  you move a role back onto a Claude model.)
- **VPN**, if your Anthropic route is a gated corporate proxy (see below).

## Install

```bash
nvm use 22                                                        # required
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

Role definitions live in `~/.pi/agent/agents/` — pi only reads agents from that
one global directory, so they apply to every repo on the machine.

## Auth

Preferred, keeps keys out of shell history:

```bash
pi
/login          # select openai-codex, then run again for opencode-go
```

Resolution order is `--api-key` → `auth.json` → environment variable →
`models.json`. `auth.json` wins over the environment, so don't set both.

If you write `~/.pi/agent/auth.json` by hand, prefer indirection over a literal
key on disk — `!cmd` runs a shell command and uses stdout, a bare uppercase name
is read as an environment variable:

```json
{
  "anthropic":   { "type": "api_key", "key": "!op read 'op://vault/anthropic/credential'" },
  "opencode-go": { "type": "api_key", "key": "OPENCODE_API_KEY" }
}
```

> **Paste the value, not the header.** A `key` of 45 characters beginning
> `X-Api-Ke` is the whole header line pasted into the key field. `team-check`
> shape-checks for this before spending an API call on a 401.

### Routing Anthropic through a proxy

If Claude is reached through a gateway rather than `api.anthropic.com`, point pi
at it in `~/.pi/agent/models.json` — no code, no extension:

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://<proxy-host>",
      "headers": { "User-Agent": "claude-cli/2.1.269 (external, cli)" }
    }
  },
  "modelOverrides": {
    "claude-opus-5": {
      "compat": { "supportsMidConvoEffort": false, "forceAdaptiveThinking": false }
    }
  }
}
```

Three things that setup is working around, each with a distinctive symptom:

| Symptom | Cause | Fix |
|---|---|---|
| `401 invalid x-api-key` | key sent to the wrong endpoint | `baseUrl` |
| `426 Upgrade Required` | gateway pins a client version | `User-Agent` (the error names the required string) |
| `400 messages.N.output_config: Extra inputs are not permitted` | gateway relays to an API version predating adaptive thinking | both `compat` flags |

Both `compat` flags are needed: `supportsMidConvoEffort: false` removes the
message-level `output_config`, `forceAdaptiveThinking: false` stops the
top-level one set by the next branch. Together they fall through to classic
budget-based thinking, which older Anthropic-compatible endpoints accept.

## Usage

```bash
./bin/pi-team --list                                  # roles, models, ro/rw

# Pick a workflow (see Two workflows above)
./bin/pi-team orchestrator-automaton -i               # researcher plans; interactive
./bin/pi-team orchestrator-hybrid - < plan.md         # you planned; runs to the end

# Or drive the roles yourself
./bin/pi-team researcher  "Plan: add retry to the X client"
./bin/pi-team implementor "<the approved plan>"
./bin/pi-team reviewer    "FIRST REVIEW. Review the diff against: <plan>"

./bin/pi-team implementor -c "Fix the two BLOCKING findings: ..."   # continue
./bin/pi-team reviewer --dry-run "..."                # print the pi command
./bin/pi-team researcher - < brief.md                 # task from stdin
```

Sessions are per role under `~/.pi/agent/team-sessions/<role>/`; `-c` continues
the most recent, `--session <id>` resumes a specific one. Every role runs with
`--no-extensions` so a subagent cannot recurse back into the team setup.

## Verify

```bash
./bin/team-check
```

Checks Node and pi versions, credential presence and shape, then every model ID
against pi's own catalog (not a cached one), then runs a live two-role smoke
test: the implementor creates a file, the reviewer checks it. The provider and
model lists are read out of the role files at run time, so adding a role or
switching a model cannot leave a stale assertion behind. Finding a broken
provider here costs seconds; finding it mid-feature costs a wasted
implementation pass.

## Working with the team

Every brief must be **self-contained**: Goal, Context (file paths with line
refs), Constraints, Acceptance criteria, Validation to run, Out of scope. Never
write "the plan above" — subagents start with zero context and see only the task
text.

**Acceptance criteria** are checked off by number by the implementor and verified
against the diff by the reviewer. Who *proposes* them is the difference between
the two workflows: the researcher does in automaton, you do in hybrid.

Either way the proposer must be someone who has read the codebase before the
code exists — that is what lets a criterion name
`pytest tests/test_client.py -k retry` and mean it. Neither orchestrator is
allowed to invent one, because neither has read the codebase; left to it, an
orchestrator produces "handles errors gracefully", which no later role can
falsify. In hybrid the orchestrator will stop and ask you for a missing
criterion rather than write one.

**The review loop is capped at two fix rounds.** Open-ended defect-hunting has
no natural fixed point, so scope contracts each round: the first review is the
only exhaustive pass, and a re-review checks exactly two things — were the named
findings fixed, and did the fix introduce a new BLOCKING defect. Label the round
(`FIRST REVIEW` / `RE-REVIEW of findings N`); an unlabelled re-review gets a
fresh sweep, which is the moving-goalposts failure. Only BLOCKING findings
re-enter the loop.

## Changing a role's model

A role's model lives in the `model:` line of its frontmatter, in
`~/.pi/agent/agents/<role>.md` — **not** in this repo, and not in
`~/.claude/settings.json` (that configures Claude Code, which is a different
harness and does not read these files).

**1. See what's available.** Remember this writes to stderr:

```bash
pi --list-models 2>&1                      # whole catalog
pi --list-models 2>&1 | grep opencode-go   # one provider
```

The columns are `provider · model · context · max-out · thinking · images`, so
you can check a candidate actually supports what the role needs before switching
to it — the implementor handles images, for instance, which rules out about half
the `opencode-go` catalog.

**2. Edit the role.** `provider:` and `model:` must agree — the model has to be
one the listed provider serves:

```bash
$EDITOR ~/.pi/agent/agents/reviewer.md
```

```yaml
provider: anthropic
model: claude-opus-5      # ← change this
thinking: high
```

Or in place, when you just want the model swapped:

```bash
sed -i 's/^model: .*/model: claude-opus-4-8/' ~/.pi/agent/agents/reviewer.md
```

**3. Verify.** `pi-team --list` re-parses every role file, so it catches a typo
or a bad `thinking:` value before you spend an API call:

```bash
./bin/pi-team --list
./bin/pi-team reviewer --dry-run "test"    # see the exact pi command
```

### Changing thinking level

`thinking:` accepts `off`, `minimal`, `low`, `medium`, `high`, `xhigh`. The
runner validates it strictly rather than silently defaulting, so a typo is a
clean error. To try a level without editing the file:

```bash
./bin/pi-team researcher --thinking xhigh "Plan: ..."
```

Leave the implementor at `thinking: off`. That is deliberate — the DeepSeek V4
API requires `reasoning_content` to be replayed verbatim on assistant messages
carrying `tool_calls`, and `off` avoids the code path entirely. pi does handle
it correctly (`compat.requiresReasoningContentOnAssistantMessages`,
`thinkingFormat: "deepseek"`), so this is belt-and-braces, not a workaround for
a pi bug.

### Switching provider

Changing `provider:` as well as `model:` needs a credential for the new provider
in `auth.json` — run `pi` then `/login`. Then re-run `./bin/team-check`, which
smoke-tests the route rather than just checking the key is present.

If you move an anthropic role onto a model not already covered by
`modelOverrides` in `models.json`, and you are behind an older gateway, expect
the `400 messages.N.output_config` error — add the same two `compat` flags for
the new model ID. `team-check` reads the provider and model lists out of the
role files at run time, so there is no hardcoded list to update alongside.

### Falling back when a model misbehaves

pi has no native fallback, so this is a manual switch. Trigger: **two
consecutive failures on the same tool call.**

```bash
sed -i 's/^model: .*/model: deepseek-v4-flash/' ~/.pi/agent/agents/implementor.md
```

Same-provider candidates for the implementor, all confirmed present and all with
1M context:

| Model | Max out | Images | Note |
|---|---|---|---|
| `deepseek-v4-flash` | 384K | no | the previous primary; known-good here |
| `deepseek-v4-pro` | 384K | no | stronger, slower |
| `kimi-k2.7-code` | 262K | yes | code-specialised |
| `qwen3.8-max` | 131K | yes | |
| `glm-5.3` | 131K | no | |
| `minimax-m3` | 131K | yes | |

PLAN.md step 6 lists an older set (`glm-5.1`, `kimi-k2.6`, `qwen3.6-plus`,
`minimax-m2.7`) — those still exist, but the catalog has moved on.

## Known limits

- **`bash` is all-or-nothing.** The `--tools` allowlist is per *tool*, not per
  command, so `git push` cannot be blocked by configuration. For the implementor,
  "never push / amend / force-reset / write outside the repo" lives in its body
  prompt and is enforced only by your reading the diff.
- **The workflow split is prompt-enforced.** `orchestrator-hybrid` has no
  researcher in the sense that its prompt omits the role and forbids calling it
  — but `researcher.md` is still on disk and still invokable via `bash`, because
  the automaton workflow needs it. Same for the two orchestrators starting each
  other. `tools:` cannot express "this role but not that one"; only `bash` is
  gated, and it is all-or-nothing.
- **`execution: parallel|sequential` is concurrency scheduling, not
  permissions.** It reads like a security control and enforces nothing; an
  invalid value silently falls back to a default. `tools:` is what enforces
  read-only.
- **No native model fallback.** Switching is a manual edit — see
  [Changing a role's model](#changing-a-roles-model).
- **Role definitions are global.** `~/.pi/agent/agents/` is the only directory pi
  reads agents from; `.pi/settings.json` project overrides exist for *settings*,
  not agents. A model change affects every repo on this machine.
- **`pi --list-models` writes to stderr**, not stdout. Capture with `2>&1`.

## Troubleshooting

**`429 rate_limit_error` on every anthropic role.** The gateway budget is shared
per key. Measured directly: **100 requests per rolling 60 seconds**, keyed to the
credential and reported in the response headers:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 97
X-RateLimit-Reset: <now + 60s>
```

The counter lives on the gateway, so **closing terminals does not reset it** — it
only decays with time. Run one client at a time: driving `pi` and an interactive
Claude Code session against the same key concurrently means whichever loses the
race gets the 429.

pi's retry defaults (3 tries at 2s/4s/8s ≈ 14s) are too narrow for a per-minute
limiter. `~/.pi/agent/settings.json`:

```json
"retry": { "enabled": true, "maxRetries": 5, "baseDelayMs": 5000 }
```

That is 5s/10s/20s/40s/80s ≈ 155s of backoff before giving up.

**A 429 is a refusal, not slowness.** The gateway answers unauthenticated probes
in ~0.6s throughout. The two call for completely different fixes.

**`426` after a gateway upgrade.** The pinned client version moved; the error
message names the new one. Update the `User-Agent` in `models.json` to match.
Worth knowing, because the symptom — every anthropic role failing at once —
looks nothing like a version pin at first glance.

**Expect more schema rejections.** The gateway runs an older API version, so
other pi features may be refused as pi evolves. The same `compat` mechanism
covers the likely candidates: `supportsCacheControlOnTools`,
`supportsStrictTools`, `supportsTemperature` (Opus 4.7+ rejects non-default
temperature). Flags are documented in `pi-ai/dist/types.d.ts` around line 575.

## Layout

```
bin/pi-team       role runner — parses frontmatter, invokes pi
bin/team-check    post-login verifier (PLAN.md steps 7–8)
PLAN.md           design doc: decisions, verification results, rejected options
~/.pi/agent/agents/*.md    role definitions (global, not in this repo)
```

`bin/pi-team` reads a role's frontmatter for provider/model/thinking/tools and
its body for the system prompt. The files keep the `@redentor_dev/pi-orchestrator`
frontmatter format, so adopting that package later is just `pi install`.

It was dropped for two reasons. The first no longer applies: it passed
`--session-id` on every fresh delegation, which pi 0.74.2 did not have — pi
0.85.1 does, so that incompatibility is gone. The second still stands: pi
packages run with full system access, and 2,685 lines from a single maintainer
at ~58 downloads/month is a lot of trust for what is, in the end, three `pi`
invocations. Provenance is clean (OIDC-published from the matching public repo),
which rules out a hijacked publish but says nothing about behaviour.

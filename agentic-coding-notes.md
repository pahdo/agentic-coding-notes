## Optimize your agentic coding setup - March 5, 2026

Any questions? Please ask Daniel Zou

### Intro
- These notes are for power users of agentic coding tools that prefer to squeeze out a bit of extra performance.
- On a weekly basis, the Claude Code defaults and built-in become better and better, so I don’t expect the benefits to be durable. Constant iteration is needed to stay ahead of the curve.

### The basics
- CLAUDE.md
	- Global / Project Level
		- Global: `~/.claude/CLAUDE.md`: loaded every session, every project. BE VERY JUDICIOUS with this file.
		- Project: `./CLAUDE.md` in project root - loaded every session in that project only.
		- As the file grows, figure out what you can extract to project-level CLAUDE.md and skills.
		- Example rule — `simplicity`:
		```markdown
		**Core Test**: Can you explain this architecture in 2 minutes? NO → simplify. YES → ship it.
		**5-Line Rule**: If core logic needs >5 lines of pseudocode, it's over-engineered.
		**Banned Patterns**: Abstract interfaces for single implementations. Config systems for hardcoded values. Vendor abstraction until 2+ vendors exist.
		**Auto-Reject**: Timeline >2 weeks | Dependencies saving <4 hours | Features for <80% users | Abstractions for <3 use cases
		**Stop Coding When**: Core workflow works | Happy path smooth | Data safe | Basic error handling exists | Tests pass
		```
	- XML Priority Tags
		- `critical` rules get followed more reliably (but still not guaranteed…)
		```markdown
		<rules>
		<rule priority="critical">Research → Plan → Branch → Code → Test → Document → Merge</rule>
		<rule priority="critical">ALWAYS run REAL tests - NEVER use mocks or simulations</rule>
		<rule priority="high">Stuck >5min? Launch parallel Gemini + Grok analysis</rule>
		</rules>
		```
- Constant improvement: Auto-memory
	- `~/.claude/projects/<project>/memory/MEMORY.md`
	- Project-level context
	- This is automatic now, feel free to force the agent to commit something to memory by simply asking it to not make the same mistake again
	- /memory toggles this on or off (default: ON)
- Post tool call hooks
	- Three event types: `PreToolUse`, `PostToolUse`, `UserPromptSubmit`
	- Wired in `settings.json`:
	```json
	"hooks": {
	  "PreToolUse": [{"matcher": "Edit|Write", "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/detect-secrets.sh"}]}],
	  "PostToolUse": [{"matcher": "", "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/context-usage.sh"}]}],
	  "UserPromptSubmit": [{"matcher": "", "hooks": [{"type": "command", "command": "bash ~/.claude/hooks/register-context.sh"}]}]
	}
	```
- Skills
	- Can be invoked by `/skill-name` or by the agent when it matches the request. 
	- Use skills to store procedures that are useful but don’t need to be in CLAUDE.md
		- For my Obsidian, I have multi-agent debate and propagate insights to the rest of the folder as skills
	- `/audit <scope>` — multi-model audit skill (133 lines):
	```markdown
	# Modes
	- `/audit <scope>` — Full audit: scan → issues → user answers → execute → verify
	- `/audit scan <scope>` — Scan only: produce issues doc, stop before execution

	# Rules:
	- Grok MUST NOT be trusted with fact checking by Claude Opus on factual claims.
	- Consensus = confidence, disagreement = investigate. Don’t majority-vote.
	- "Is this actionable?" is the quality bar.
	```
	- `/propagate <source-file>` — propagate findings into all related working docs:
	```markdown
	# Three phases: extract → rewrite claims → review for contradictions
	# Prime directive: REWRITE, don’t annotate.
	# If a finding confirms a claim: add evidence citation inline. Don’t restate.
	# If a finding challenges a claim: rewrite the claim. Update severity labels, verdicts.
	# If a finding refines a claim: rewrite with nuance. Don’t leave original + footnote.
	# One statement of each finding. Never restate 3+ times across sections.
	```
- Subagents / Tasks
	- Task tool spawns parallel sub-agents. Useful for both parallelization and context window management.
- Custom Agents
	- `~/.claude/agents/` available to Task tool as subagent types.
	- Use when you think you need a skill but you don’t need or want to add to the main agent context!
	- code-reviewer.md:
	```markdown
	---
	name: code-reviewer
	description: Expert code review specialist. Use proactively after writing or modifying code.
	tools: Read, Grep, Glob, Bash
	---
	When invoked:
	1. Run `git diff` to see recent changes
	2. Focus on modified files
	3. Begin review immediately
	Feedback by priority:
	- **Critical** (must fix): crashes, security, data loss
	- **Warning** (should fix): bugs, bad patterns
	- **Suggestion** (consider): style, readability
	```
	- security-auditor.md:
	```markdown
	---
	name: security-auditor
	description: Security vulnerability scanner. Use proactively when reviewing auth, data handling, API code.
	tools: Read, Grep, Glob, Bash
	---
	Focus areas: SQL injection, XSS, CSRF, insecure deserialization, broken access control,
	SSRF, path traversal, command injection, cryptographic failures
	For each finding: Severity | File and line | Issue | Fix
	Only report real issues, not theoretical concerns.
	```
### Interesting concepts
- Context window management is everything
	- High context window usage (even without reaching limits) may affect reasoning performance
	- ["Context Length Alone Hurts LLM Performance Despite Perfect Retrieval"](https://arxiv.org/abs/2510.05381) 
	- Compaction is horrible!!
	- Use /clear when old conversation context is not needed
		- Note: typing “/clear” into a remote control session currently doesn’t work…
- Persona model suggests garbage in garbage out
	- Idea: input token “intelligence” affects output token “intelligence”.
	- Paper: Small prompt variations makes a big difference for Llama (https://arxiv.org/abs/2310.11324)
	- Idea: Prompts that invoke “intelligent” personas produce more “intelligent” output because they influence persona selection.
	- Anthropic's [Persona Selection Model](https://www.anthropic.com/research/persona-selection-model): LLMs learn to simulate diverse "personas" during pretraining. Post-training refines personas.
	- Maybe: Does prompt engineering matter more for Gemini than Opus?
- Models are not better but different
	- Using the same model for multi-agent debate may have consistent blind spots
	- My observations:
		- Claude Opus is best at avoiding hallucinations, but is often “lazy”
		- Grok will doggedly look for disconfirming info, even at the risk of hallucination
		- Gemini is extremely powerful but lacks common sense
		- Codex is amazing for fixing thorny bugs that other models will never figure out
### The techniques
- Custom "plan mode" via CLAUDE.md
	- From my CLAUDE.md:
	```markdown
	<rule priority="critical">Research → Plan → Branch → Code → Test → Document → Merge</rule>
	```

	```markdown
	1. **Research**: Use Task tool for 2+ aspects before coding
	2. **Plan**: Create PRD for large tasks; decompose for parallel work
	3. **Branch**: Create feature branch before code changes
	4. **Code**: Implement with simplicity principles
	5. **Test**: Run REAL tests locally
	6. **Document**: Update relevant docs
	7. **Merge**: Commit with "why" not "what"
	```
- Multi-agent research and planning
	- From my CLAUDE.md:
	```markdown
	<rule priority="critical">When launching subagents, use the Task Tool, Codex CLI, Grok API, and Gemini CLI. You must follow the model selection rules below. If unspecified, evenly distribute between Claude, Codex, Grok, and Gemini for N subagents.</rule>
	<rule priority="critical">Codex: ALWAYS pass `-m gpt-5.3-codex` flag. NEVER use bare `codex exec` without -m. CLI defaults can change between versions.</rule>
	<rule priority="critical">Grok: Use grok-4-1-fast-reasoning (2M tokens) for analysis/review, grok-code-fast-1 for coding</rule>
	<rule priority="critical">Gemini: ALWAYS use gemini-3.1-pro-preview — ALWAYS pass `-m gemini-3.1-pro-preview` flag. NEVER use bare `gemini -p` without -m. The CLI defaults to an old model if -m is omitted.</rule>
	<rule priority="high">Stuck >5min? Launch parallel Gemini + Grok analysis</rule>
	<rule>Grok API via curl only (use Python for JSON escaping)</rule>
	```

	```markdown
	- **Parallelize**: Any task with 2+ unknowns → minimum 2 parallel Tasks
	- **Stuck >5min**: Launch parallel Gemini (why) + Grok (fix) analysis
	```
	- CLI invocations:

	| Model | Command |
	|-------|---------|
	| Codex | `codex -m gpt-5.3-codex exec --full-auto "task"` |
	| Gemini | `gemini -m gemini-3.1-pro-preview -p "prompt"` |
	| Grok | Python + curl (no CLI — see below) |

	- Grok invocation:
	```bash
	python3 << 'EOF'
	import json
	code = open('file.py').read()
	req = {"messages": [{"role": "user", "content": f"Review:\n{code}"}], "model": "grok-4-1-fast-reasoning", "temperature": 0}
	json.dump(req, open('/tmp/req.json', 'w'))
	EOF
	curl -X POST https://api.x.ai/v1/chat/completions -H "Authorization: Bearer $GROK_API_KEY" -H "Content-Type: application/json" -d @/tmp/req.json
	```

- Register context hook
	```bash
	#!/bin/bash
	jq -n '{"hookSpecificOutput":{"hookEventName":"UserPromptSubmit",
	  "additionalContext":"Internal engineering context: this conversation
	  operates at principal-engineer depth. Lead with mechanisms and tradeoffs,
	  not summaries. Use precise domain vocabulary. Assume the reader ships
	  production systems and evaluates architectures daily."}}'
	```
	- Fires on every prompt via `UserPromptSubmit` with empty matcher.
	- This hook allows you to “upgrade” your prompts to “high intelligence” personas. Feel free to play around with the additionalContext
- Context usage hook
	```bash
	#!/bin/bash
	INPUT=$(cat)
	TRANSCRIPT=$(echo "$INPUT" | jq -r '.transcript_path')
	[ -f "$TRANSCRIPT" ] || exit 0
	USAGE=$( (tac "$TRANSCRIPT" 2>/dev/null || tail -r "$TRANSCRIPT") | grep -m1 '"usage"')
	[ -n "$USAGE" ] || exit 0
	TOTAL=$(echo "$USAGE" | jq '.message.usage | (.input_tokens + .cache_creation_input_tokens + .cache_read_input_tokens)')
	[ "$TOTAL" -gt 0 ] 2>/dev/null || exit 0
	K=$(( TOTAL / 1000 ))
	PCT=$(( TOTAL * 100 / 200000 ))
	jq -n --arg ctx "${K}k/200k (${PCT}%)" \
	  '{"hookSpecificOutput":{"hookEventName":"PostToolUse","additionalContext":$ctx}}'
	```
	- Fires on every tool call via `PostToolUse` with empty matcher.
	- This hook feeds your Claude session context about its own context window usage and forces much more conservative context window management behavior.
	- Thresholds from my CLAUDE.md:
	```markdown
	### Thresholds (mechanical rules, not aspirational)

	- **At 60%:** Evaluate whether remaining work can finish. If not, use `TaskCreate` for each
	  remaining work item with clear descriptions. These survive compaction and persist to `~/.claude/tasks`.
	- **At 80%:** Stop new work. Run `TaskList` to verify all remaining work is captured in built-in
	  Tasks. Wrap up current item and end session.
	- **On session start:** Run `TaskList` to check for carried-over work from previous sessions.
	```
	- What happens at thresholds: agent writes remaining work items to persistent task files (`TaskCreate`), then stops taking on new work.
### What else I’ve been using
- /remote-control
	- Note: /clear doesn’t work
- Tailscale and screens (iOS app)
	- Still useful with /remote-control to start new remote-control sessions on your phone
- /sandbox
	- Like dangerously-skip-permissions, but safer. It can’t call out to other model CLIs, though
- Obsidian

### Thanks for reading! Please feel free to suggest any improvements

---

## Building gmail-sync overnight: LRA + health-check pattern in practice - May 12, 2026

A two-day case study: built a Gmail → workbook sync daemon (`~/myprojects/gmail-sync`) end-to-end using the long-running-agent (LRA) harness and the health-check module, mostly autonomously, mostly while blocked on a single external dependency (GCP OAuth credentials). The interesting parts are the patterns, not the project.

### The fixture-gate pattern

The whole project gated on one external action: download OAuth credentials from GCP console. Without `~/.config/gmail-sync/credentials.json`, every Gmail API call exits 1. Naively this blocks the LRA loop at task 2.

What worked instead: implement each module against **synthetic fixtures**, treat the live verify step as a separate gate. Each `complete` call carries explicit "live verify deferred — gates on auth" in the feedback string, so the .progress.json log preserves the deferred-verifiability of every claim. Fixtures lived in `tests/fixtures/*.json` modeled on real Gmail API message shapes (plaintext, multipart-with-attachment, html-only).

Net result: 67/67 fixture-driven tests covering converter, router, writer, state, sync orchestrator, daemon plist generator, retry logic. When credentials land, only the live verify needs to run — the structural soundness is already proved.

**Transferable rule:** when an LRA task's `Verify` block needs an external dependency you don't have, encode the verification at the structural level (fixtures, schema checks, behavioral regression tests) and mark the live step as deferred-verifiable in the feedback. Do NOT mark `failed` — that stalls the loop. Do NOT mark `pass` without acknowledging the deferral — that produces silent under-implementation.

### Watchdog state machines for asynchronous human handoffs

Cron was blocked (macOS Sequoia FDA-requirement on Terminal modifying crontab — `Killed: 9` with no useful error). Switched to `launchd` `LaunchAgent` plists. Two agents at staggered cadence:

- `com.pahdolabs.gmail-sync.overnight` — state-machine driver. Polls `~/.config/gmail-sync/credentials.json`, then `token.json`, then `~/Library/LaunchAgents/com.pahdolabs.gmail-sync.plist`. Each state transition triggers the next bootstrap step automatically.
- `com.pahdolabs.gmail-sync.health` — runs `lra-checks run` every 30 min, writes JSON report + URGENT.md on blocker failures.

The state machine means: the human places ONE file (`credentials.json`), runs ONE interactive command (`gmail-sync auth` — browser-required), and the system finishes bootstrapping itself within 30 minutes. No further human action.

Concurrency safety: `fcntl.flock` at `~/.config/gmail-sync/sync.lock` for the Python sync process; atomic `mkdir`-based lock for the shell watchdog (`.overnight.lock.d/` with 1h stale-eviction). macOS does **not** ship `flock(1)` — the shell binary you'd reach for in Linux scripts is absent. `mkdir` is atomic on POSIX and works everywhere.

### Health-check module: deterministic + LLM, with loop closure

The LRA's `lra-checks` module mixes two check kinds:

1. **Deterministic** (`.checks/*.yaml`): bash exit-code, fast (<1s), replayable. Codifies known failure modes.
2. **LLM-graded** (`.checks/*.llm.md`): `claude -p` reads recent context (HANDOFF, PREFLIGHT, DESIGN, `.progress.json`), emits `RESULT: pass|fail`. Adapts to current risk surface; can spot drift a hardcoded list would miss.

The promotion path: when the LLM check finds a real issue, *codify it as a deterministic check* so future ticks catch it without LLM cost. Example from this build:
- LLM noticed HANDOFF.md claiming task 2 BLOCKED while `.progress.json` showed all 8 done → built `handoff-matches-progress.yaml` to grep the curated doc against the durable task log
- LLM noticed `DESIGN.md` referencing `src/gmail_sync/lock.py` → built `design-references-real-modules.yaml` to assert every module DESIGN claims exists actually exists on disk

**Transferable rule:** the LLM is for *discovering* drift modes; deterministic checks are for *defending against* known drift modes. Don't pay LLM cost forever for things you can codify.

### Loop closure: the system catching its own staleness

Highest-value moment of the session: the LLM coherence check found a real bug in MY code. Two CLI commands called `build_service()` without passing `state=`, so the `AUTH_EXPIRED` sentinel could never fire in production despite passing isolated unit tests. The check produced the diagnosis verbatim:

> "auth.py implements `_record_failure`/`_record_success` ... but every CLI command (`auth`, `pull`, `labels`, `sync`, `status`, `doctor`) calls `build_service()` without passing `state=`, so the failure counter is never incremented in production paths."

Fix: module-level `gmail.set_state()` setter wired via a `_build_service_with_state()` CLI helper. Plus `_retry` now catches 401/403 and increments the counter. Plus two new regression tests covering the 3x401→sentinel flow and the success→counter-reset flow.

**The integration-vs-unit gap is exactly what LLM coherence checks are good for.** Unit tests pass against the *isolated helper*. LLM reads the call graph and spots that production code paths bypass the helper. This is silent-correctness-failure territory — the most expensive bug class.

### The argv-ordering bug (parallel-agent coordination)

A parallel agent landed a new `lra-checks` CLI mid-session, moving the LLM-grading machinery from `long-running-agent health-check` to its own binary. The new code had a subtle bug: `subprocess.run(["claude", "-p", "--add-dir", path, prompt])` — but `--add-dir` consumes the next positional, leaving `prompt` orphaned. Claude CLI rejects with "Input must be provided either through stdin or as a prompt argument". Confirmed by direct repro: swap to `claude --add-dir <path> -p <prompt>` and it works.

Workaround without touching upstream code: a deterministic shim (`coherence-llm-shim.yaml`) that calls `claude` with correct argv ordering inside its own bash body, parses `RESULT:` itself, exits 0/1. Bypasses the broken LLM path entirely, gets the value back, retires when upstream fixes the bug.

**Transferable rule:** when a parallel agent's surface has a bug you can route around with a shim, prefer the shim over blocking. Mark the upstream as known-broken in a HANDOFF/commit message so the other agent's session picks it up.

### macOS-specific landmines

- `crontab -` requires Full Disk Access on Sequoia+. Terminal/your shell needs FDA. Solution: launchd LaunchAgents, native to macOS, no permission grant required.
- `flock(1)` doesn't exist. Use `mkdir` as atomic lock or write the lock script in Python with `fcntl.flock`.
- `timeout(1)` doesn't exist by default. Either install coreutils (`gtimeout`) or rely on the outer caller's timeout (lra-checks enforces 180s on deterministic checks, 300s on LLM).
- `host(1)` for DNS is BSD-flavored; piping to `dscacheutil -q host` is the macOS fallback.

### What I'd skip next time

- The codex adversarial-review step. First attempt hung 30+ min on stdin (heredoc didn't survive subprocess), second I never retried — the LLM coherence health check turned out to subsume the value (continuous adversarial loop vs one-shot review). Codex-as-debate makes sense for *design phase*; LLM-coherence-as-watchdog makes sense for *implementation+post*.

- Pre-emptive `.gitignore` editing for runtime artifacts. The first time `.checks/history.jsonl` showed dirty, `.gitignore` didn't help because the file was already tracked. `git rm --cached <path>` is the unblock — the .gitignore takes effect once it's untracked. Cheap habit: any file the harness *writes during runtime* gets added to .gitignore the moment you commit its parent dir.

### Concrete files for reference

- `~/myprojects/gmail-sync/DESIGN.md` — locked decisions in their own subsections (P5 forward-only, P7.6 verbatim threads, P8.4 flock, P4.4 NFKD+ASCII)
- `~/myprojects/gmail-sync/PREFLIGHT.md` — 10-category pre-execution gates
- `~/myprojects/gmail-sync/.checks/` — 13 health checks (5 blocker, 7 warn, 1 info)
- `~/myprojects/gmail-sync/scripts/overnight-tick.sh` — state-machine watchdog
- `~/myprojects/long-running-agent/skill/HEALTHCHECK-MODULE.md` — full spec for the health-check pattern
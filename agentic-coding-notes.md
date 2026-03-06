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
*Slide deck version: [github.com/pahdo/spc-dotfiles-demo](https://github.com/pahdo/spc-dotfiles-demo)*
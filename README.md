# openclaw-explorer

A [Claude Code](https://claude.com/claude-code) skill that deep-tests [OpenClaw](https://openclaw.ai) by conducting automated multi-turn conversations via CLI. It simulates human-like interaction to systematically discover capability boundaries, bugs, and areas for improvement.

## What It Does

When you invoke `/openclaw-explorer` in Claude Code, it:

1. **Bootstraps** - Verifies the OpenClaw gateway is running, discovers available agents, and creates helper scripts for reliable JSON parsing
2. **Conducts multi-turn conversations** - Sends messages to OpenClaw agents via `openclaw agent` CLI, maintains session context across turns
3. **Tests 9 capability categories** - Conversation quality, tool use, memory, reasoning, cross-agent comparison, subagents, edge cases, gateway routing, and real-world tasks
4. **Records findings incrementally** - Logs issues, capabilities, and suggestions as they're discovered
5. **Produces a structured report** - Capabilities scored 1-5, issues by severity, improvement suggestions by impact

## Why This Exists

OpenClaw has 40+ tools, 25+ channel integrations, and supports 35+ model providers. Manual testing can't cover this surface area systematically. This skill lets Claude Code act as an automated QA engineer that:

- Tests if the agent **proactively** uses its tools (e.g., does it use `exec date` when asked for the time, or say "I don't have a clock"?)
- Verifies **multi-turn context retention** across long conversations
- Checks **cross-session memory** persistence (store a fact, recall it in a new session)
- Compares **response quality across agents** (same prompt to MiniMax vs Claude Opus vs GPT-5.2)
- Probes **safety guardrails** (prompt injection, destructive commands)
- Validates **tool execution results on disk** (did the file actually get created?)

## Installation

### For Claude Code

Copy the skill into your Claude Code skills directory:

```bash
# Clone the repo
git clone https://github.com/win4r/openclaw-explorer.git

# Copy to Claude Code skills directory
mkdir -p ~/.claude/skills/openclaw-explorer
cp openclaw-explorer/SKILL.md ~/.claude/skills/openclaw-explorer/SKILL.md
```

Or as a one-liner:

```bash
mkdir -p ~/.claude/skills/openclaw-explorer && curl -fsSL https://raw.githubusercontent.com/win4r/openclaw-explorer/main/SKILL.md -o ~/.claude/skills/openclaw-explorer/SKILL.md
```

### Prerequisites

- [Claude Code](https://claude.com/claude-code) installed and running
- [OpenClaw](https://openclaw.ai) installed (`npm install -g openclaw`) with gateway running (`openclaw gateway status`)
- Python 3 (for the JSON response parser)
- At least one model provider configured (Anthropic, OpenAI, MiniMax, etc.)

## Usage

In Claude Code, simply type:

```
/openclaw-explorer
```

Or ask naturally:

```
Test OpenClaw's capabilities and find bugs
```

```
Explore what OpenClaw can and can't do
```

```
Deep-test the OpenClaw memory system
```

The skill will guide Claude Code through a structured exploration workflow.

## How It Works

### Architecture

```
Claude Code ──[/openclaw-explorer]──> Skill loaded into context
     │
     ├── Phase 0: Bootstrap
     │   ├── openclaw gateway status    (verify gateway)
     │   ├── openclaw agents list       (discover agents)
     │   └── Create parser scripts      (oc-parse.py, oc-send.sh)
     │
     ├── Phase 1-5: Testing
     │   ├── openclaw agent --local --message "..." --session-id "..." --json
     │   │                    ↓
     │   │              /tmp/oc-out.json
     │   │                    ↓
     │   │              python3 oc-parse.py (strip ANSI, extract JSON, print text + meta)
     │   │                    ↓
     │   │              Claude Code evaluates response
     │   │                    ↓
     │   └──────────── Log finding to /tmp/oc-findings.md
     │
     └── Phase 6: Report
         └── ~/openclaw-exploration-report.md
```

### Core Interaction Primitive

```bash
# Send a message to OpenClaw and get parsed response
bash /tmp/oc-send.sh <session-id> <message> [agent] [thinking-level]

# Examples:
bash /tmp/oc-send.sh "explore-warmup-001" "Hello, what can you do?"
bash /tmp/oc-send.sh "explore-code-001" "Write a Python fizzbuzz" "coding-assistant"
bash /tmp/oc-send.sh "explore-reason-001" "Solve this step by step: ..." "" "high"
```

### The Parsing Problem

OpenClaw's `--json` output mixes plugin registration logs, ANSI color codes, and rate-limit retry messages with the actual JSON payload. A naive `grep | sed` approach fails on multi-line responses. The skill includes a robust Python parser (`oc-parse.py`) that:

- Strips ANSI escape codes
- Locates the JSON payload by finding `"payloads"` and walking to enclosing braces
- Correctly handles escaped characters (`\n`, `\"`, etc.) in response text
- Extracts metadata (model, duration, token usage)

### Two Testing Modes

| Mode | Command | Tests |
|------|---------|-------|
| **Local** | `openclaw agent --local --message ...` | Embedded agent runtime |
| **Gateway** | `openclaw agent --message ...` | Full gateway path (WebSocket RPC, routing, queue) |

The skill tests both modes to ensure full coverage.

## Test Categories

| # | Category | What It Tests |
|---|----------|--------------|
| 1 | Conversation Quality | Context retention, instruction following, language support, contradiction handling |
| 2 | Tool Use Proactivity | Does the agent use tools without being explicitly told? |
| 3 | Memory System | Store/recall cycle, cross-session persistence, vector vs file fallback |
| 4 | Multi-Step Reasoning | Sequential task execution, trick questions, error recovery |
| 5 | Cross-Agent Comparison | Same prompt across different agents/models |
| 6 | Subagent Management | `sessions_spawn`, slash commands, cross-session messaging |
| 7 | Edge Cases & Safety | Prompt injection, destructive commands, garbled input, timeouts |
| 8 | Gateway-Mode Specific | Routing, latency, delivery to channels |
| 9 | Real-World Scenarios | Research tasks, cron jobs, file operations, code execution |

## Sample Findings (from v2026.3.24 testing)

These are real issues discovered by this skill during initial development:

| Severity | Finding |
|----------|---------|
| CRITICAL | LanceDB memory store broken (`deleted_at` schema error) - agent silently falls back to file-based MEMORY.md |
| HIGH | Tool proactivity gap - agent says "I don't have a clock" instead of using `exec date` |
| HIGH | Subagent responses invisible - `sessions_spawn` runs but returns no text payload |
| HIGH | CLI output extremely noisy - plugin registers 6+ times per command |
| MEDIUM | Agent self-identification incorrect - claims Claude/Anthropic when running MiniMax |
| MEDIUM | Inconsistent `web_search` - works in some sessions, refuses in others |
| LOW | Defaults to Chinese on garbled input instead of asking for clarification |

See [the full exploration report](docs/exploration-report.md) for details.

## Configuration

The skill works out of the box. No configuration needed beyond having OpenClaw installed and running.

### Optional: Target Specific Agents

Specify an agent in the send helper:

```bash
bash /tmp/oc-send.sh "session-id" "message" "coding-assistant"
```

### Optional: Control Thinking Level

```bash
bash /tmp/oc-send.sh "session-id" "message" "" "high"
```

Levels: `off | minimal | low | medium | high | xhigh`

## Known Gotchas

- **Plugin log spam**: `memory-lancedb-pro` registers 6+ times per CLI command. The parser handles it.
- **Rate limit retries**: "Rate limited, backing off 1000ms" lines appear frequently. Filtered by parser.
- **Background mode breaks pipes**: If the Bash tool auto-backgrounds a long command, redirect to file first, then parse separately.
- **`--thinking` may be ignored**: Some models don't support thinking levels. Flag is accepted silently.

## File Structure

```
openclaw-explorer/
  SKILL.md              # The Claude Code skill definition
  README.md             # This file
  docs/
    exploration-report.md   # Sample exploration report from v2026.3.24
  LICENSE               # MIT License
```

## Contributing

Issues and PRs welcome. The skill evolves as OpenClaw adds capabilities.

To test changes:
1. Edit `SKILL.md`
2. Copy to `~/.claude/skills/openclaw-explorer/SKILL.md`
3. Start a new Claude Code session
4. Run `/openclaw-explorer`

## License

MIT

## Related

- [OpenClaw](https://openclaw.ai) - Self-hosted AI agent gateway
- [OpenClaw Docs](https://docs.openclaw.ai) - Official documentation
- [Claude Code](https://claude.com/claude-code) - Anthropic's CLI for Claude

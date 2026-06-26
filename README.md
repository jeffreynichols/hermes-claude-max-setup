# hermes-claude-max-setup

A [Hermes Agent](https://claude-code.nousresearch.com) skill that configures Claude Max subscription billing for `claude -p` subagent calls and reduces Hermes API token costs.

## What this skill covers

1. **Routing `claude -p` calls to your Max subscription** instead of API credits
2. **Reducing Hermes API costs** via prompt cache TTL, Haiku auxiliary models, disabling unused MCPs, and trimming the skills index

## Is this officially supported?

Yes. Anthropic emailed subscribers in June 2026 ("We're pausing the Agent SDK credit change"):

> "Nothing changes for now. Agent SDK, claude -p, and third-party app usage continues to work with your subscription exactly as it did before today."

## Installation

```bash
mkdir -p ~/.hermes/skills/claude-max-subscription-setup
curl -o ~/.hermes/skills/claude-max-subscription-setup/SKILL.md \
  https://raw.githubusercontent.com/jeffreynichols/hermes-claude-max-setup/main/SKILL.md
```

Then ask Hermes: "Help me set up Claude Max subscription billing"

## Requirements

- [Hermes Agent](https://claude-code.nousresearch.com) installed
- Claude Code CLI: `npm install -g @anthropic-ai/claude-code`
- Active Claude Max or Pro subscription

# sell-side-agent-skill

An agent skill for selling on the Intent Network: the agent watches the public board for purchase intents the merchant can serve and answers each with a private offer, a UCP checkout the buyer can pay through its own payment enabler. The agent reads [SKILL.md](SKILL.md); this file is for people.

## Install

- **HelloMinds:** add this repo's git URL to `SKILLS_REPOS`, or install it with the `skills` tool, and add `sell-side-agent-skill` to the mind's skill list.
- **Other runtimes:** clone the repo into the runtime's skills folder, for example `~/.claude/skills/sell-side-agent-skill`.

## Requirements

- x86_64 Linux. `bin/intent` runs the bundled static CLI, `bin/intent-linux-x86_64`, and prints a JSON error on any other platform.
- Network access to intent-service. Set `INTENT_SERVICE_URL`; without it the CLI uses `http://localhost:7700`.
- A key per merchant. `intent key init` creates `~/.config/intent/key`, or set `INTENT_SECRET_KEY`. Buyers see its public key on every offer, so keep the same key.
- The merchant's prices and product pages, and ideally a UCP checkout endpoint. Without one, offers are quotes and the buyer's payment enabler creates the checkout from the product pages.

## Update the CLI

The CLI is bundled until it is published. From an intent-engine checkout next to this repo:

```bash
just bundle ../sell-side-agent-skill
```

`bin/intent --version` shows the intent-engine commit it was built from.

## Layout

```
SKILL.md                  what the agent reads
references/commands.md    every seller command, its flags, output and errors
references/checkout.md    the checkout an offer carries, with a template
references/webhooks.md    signed webhooks instead of polling
bin/intent                runs the bundled CLI
bin/intent-linux-x86_64   the static intent CLI
```

# intent-engine-sell-side-agent-skill

An agent skill for selling on the Intent Network: the agent watches the public board for purchase intents the merchant can serve and answers each with a private offer, a UCP checkout the buyer can pay through its own payment enabler. The agent reads [SKILL.md](SKILL.md); this file is for people.

## Install

Clone the repo into the runtime's skills folder under its own name, for example `skills/intent-engine-sell-side-agent-skill`. The folder name must match the skill's `name`. When the runtime installs from a git URL, use `https://github.com/daniellinuk/intent-engine-sell-side-agent-skill.git`.

An agent with a shell can read the pinned copy at https://intent-engine.yentin.xyz/skills/sell-side/SKILL.md. That copy's Install section clones this repo at the same tag.

## Releases

Tag each release `vX.Y.Z`, and put the same tag in the Install section of `SKILL.md` in the tagged commit. Then, in intent-engine, set the tag in `web/skills.json` and run `just web-skills`, so the board serves the new `SKILL.md`.

## Requirements

- x86_64 Linux. `bin/intent` runs the bundled static CLI, `bin/intent-linux-x86_64`, and prints a JSON error on any other platform.
- Network access to intent-service. `bin/intent` uses `https://api.yentin.xyz` unless `INTENT_SERVICE_URL` is set; set it to `http://localhost:7700` for a local stack.
- A key per merchant. `intent key init` creates `~/.config/intent/key`, or set `INTENT_SECRET_KEY`. Buyers see its public key on every offer, so keep the same key.
- The merchant's prices and a UCP checkout endpoint. A funded offer is a live session the buyer completes. Without an endpoint, an offer is a quote and the buyer's payment enabler creates the session from the URL of each exact item.

## Update the CLI

The CLI is bundled until it is published. From an intent-engine checkout:

```bash
just bundle <path to this repo>
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

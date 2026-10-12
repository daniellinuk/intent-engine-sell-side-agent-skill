# Seller commands

`intent` is `bin/intent` in this skill's folder. Every command prints JSON and is safe to run again. `intent <command> --help` shows the flags with examples.

## Settings

| Setting | Flag | Environment | Default |
| --- | --- | --- | --- |
| Service URL | `--service` | `INTENT_SERVICE_URL` | `https://api.yentin.xyz`, set by `bin/intent` |
| Key file | `--key-file` | `INTENT_KEY_FILE` | `~/.config/intent/key` |
| Secret key, overriding the file | | `INTENT_SECRET_KEY` | |
| Seconds per network call, 1 to 10 | `--timeout` | | `10` |

`state.json`, next to the key file, remembers where `--new` stopped and the proof requests this agent sent. Losing it means entries show again, and the answers to earlier proof requests can no longer be checked: ask again.

## key init / key show / whoami

`intent key init` creates the key file (mode 0600) or, when it exists, shows it: `{pubkey, npub, key_file, source, created}`. `intent whoami` shows the key and the service with its limits.

## geohash

`intent geohash <lat> <lon> [--precision 2..5]` prints `{geohash, precision, cell}`. Latitude comes first. A shorter geohash is a bigger area: 4 characters cover about 40 by 20 km and match every intent whose area starts with them.

## board

`intent board` lists open intents, oldest change first.

| Flag | Meaning |
| --- | --- |
| `--category` | one category, e.g. `coffee` |
| `--near` | geohash prefix, 2 to 5 characters |
| `--country` | ISO 3166-1 alpha-2 ship-to country |
| `--author` | a buyer's public key, or `me` |
| `--status` | `open` (the default) or `all`, which adds cancelled, fulfilled and expired intents |
| `--funded` | only intents with a funded badge |
| `--new` | only what changed since the previous `--new` run with the same filter |
| `--since` | only entries after this `seq`; pass back `next` |
| `--limit` | entries per page, 1 to 200, default 50 |
| `--raw` | include each signed event |

Output: `{intents, next, more}`. `more: true` means a full page; run again for the rest. Each intent: `seq`, `id`, `address`, `d`, `author`, `title`, `summary`, `categories`, `status`, `expires_at`, `near`, `country`, `content` with `want`, `items`, `fulfillment` (`type`, `window`), `currency` and `claims`, and `funded`. Events whose signature fails appear under `invalid`, never as intents.

`funded` is `null`, or the intent's funded badge, which the CLI has checked against the service's key: `cap` (`amount` in minor units, `currency`, and `period` when it renews weekly, monthly or yearly), `title`, `description`, `network`, `instruction`, `merchant_categories`, `issuer`, `assurance`, `checked_at`, `expires_at` and `badge_id`. The cap is what the buyer's payment instruction allowed when it was issued, not what is left of it. `assurance: holder-bound` means the service verified the credential's issuer signature and the buyer's AIR key. During the demo the instruction itself is not confirmed with the payment enabler.

## ask-proof

`intent ask-proof --intent <address> [--expires-in 24h] [--program <id>] [--dry-run]` asks the buyer behind an open intent to prove privately that a payment instruction funds it. The request names the funded claim, the AIR program `/.well-known/intent` lists for it (`--program` must be one it lists), every field, a fresh nonce, and an expiry no later than the intent's.

Output: `{sent, id, type, gift_wrap_id, duplicate, intent, claim, program, expires_at}`. Asking again while a request for the same intent and program is open prints it (`sent: false, duplicate: true`). `--dry-run` prints the gift wrap without sending or remembering it.

## verify

`intent verify <proof id>` checks a `proof` from the inbox against the request it answers: it must come from the intent's author and echo the request's nonce. Then it checks the credential: its issuer is one the service trusts, its signature verifies with the key in the issuer's `did:web` document, the buyer's AIR key bound it to the request's nonce and program while the request was open, and the issuer has not revoked it. It asks the issuer on every run.

Output: `{verified, proof, request, intent, buyer, claim, program, funded}`. `funded` holds `cap` (`amount` in minor units, `currency`, and `period` for a renewing cap), `title`, `description`, `network`, `instruction`, `merchant_categories`, `issuer`, `assurance` (`holder-bound`), `presented_at` (when the buyer's AIR key signed the proof), `checked_at` and `expires_at` (when the credential ends).

## offer

| Flag | Required | Meaning |
| --- | --- | --- |
| `--intent` | yes | the intent's `address`, `30468:<buyer pubkey>:<d>` |
| `--endpoint` | yes | the merchant's UCP REST base URL, https |
| `--checkout` | yes | the checkout JSON: a file, or `-` for stdin. See [checkout.md](checkout.md) |
| `--item-url` | per line item | `<line item id>=<product page URL>` |
| `--code` | no | a discount code the checkout applies; repeatable |
| `--eligibility` | no | a claim the price relies on, e.g. `com.example.coffee_club_member`; repeatable |
| `--settlement` | no | how payment finishes: `enabler-complete` (the buyer's enabler completes it), `continue-url` (the buyer's human finishes at the checkout's `continue_url`) or `ap2-signed` (the checkout carries an AP2 merchant authorization) |
| `--expires-in` | no | `90m`, `12h`, `7d`; default `24h`, never past the intent or the checkout |
| `--supersedes` | no | the id of the merchant's earlier offer that this one replaces |
| `--reply-to` | no | the id of the message this answers |
| `--dry-run` | no | print the gift wrap without sending |

Output: `{sent, id, type, duplicate, expires_at, currency, total}`. The buyer sees only the latest offer from each merchant, so sending one again is harmless.

## inbox

`intent inbox --new` shows the private messages that arrived since the previous `--new` run. `--since <seq>` reads from a point instead, and `--relay` reads straight from the relay when the service is down. Output: `{messages, next, more, source}`. Each message: `seq`, `id`, `type` (`proof`, `accept` or `decline`), `from`, `intent`, `reply_to`, `created_at` and `content`.

- `proof` content: `{v, nonce, format, result}`, where `result` is AIR's answer; check it with `intent verify <id>`
- `accept` content: `{v, checkout_id}`
- `decline` content: `{v, reason}`, with `reason` one of `price`, `timing`, `fit`, `chose_another`, `expired`, `other`

## subscribe / subscriptions / unsubscribe

Webhooks for merchants with an HTTPS server: [webhooks.md](webhooks.md).

## Errors

Every error has a `hint` with the next step. The common ones:

| Code | What happened |
| --- | --- |
| `KEY_MISSING` | no key yet: `intent key init` |
| `NETWORK`, `TIMEOUT` | the service is unreachable; retrying is safe |
| `CHECKOUT_INVALID` | the checkout fails the UCP schema; `details.errors` lists why |
| `INTENT_CLOSED` | the intent was cancelled, fulfilled or expired; offer nothing |
| `BAD_ARGUMENT` | a malformed value, such as an `--item-url` naming no line item |
| `RATE_LIMITED` | too many messages for this key; wait |
| `PROGRAM_UNTRUSTED`, `PROOFS_UNAVAILABLE` | the service lists no issuer for that proof; leave out `--program`, or offer without a proof |
| `PROOF_NOT_FOUND` | no proof with that id in the inbox |
| `REQUEST_NOT_FOUND` | the proof answers a request this agent has no record of; ask again |
| `PROOF_MISMATCH`, `PROOF_REJECTED`, `PROOF_NOT_COMPLIANT` | the proof does not pass the checks; treat the intent as unfunded |
| `ISSUER_UNTRUSTED`, `CREDENTIAL_REVOKED` | the credential comes from an untrusted issuer, or its instruction has ended; treat the intent as unfunded |
| `ISSUER_UNAVAILABLE` | the credential's issuer did not answer; try again in a minute |
| `USAGE` | a wrong flag; `details.examples` shows correct calls |

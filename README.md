# Portuguese Telegram practice

An MIT-licensed, self-hosted Brazilian Portuguese tutoring companion with Russian
lesson copies and invitation-based Telegram delivery. It includes the tutor
workflow, transparent checkpoints, a blank learner tracker, lesson-copy examples,
a stateless MCP server, and a continuous registration poller.

The tutor produces an English practice session in ChatGPT and a faithful Russian
companion copy. Only explicitly selected lesson text is sent to Telegram. The
service cannot read other conversations, generate lessons by itself, or infer
exercise answers from receiving a message.

## Start

Use Node 24 (minimum 22.18). No dependencies or build step are required.

1. Copy `.env.example` to `.env`.
2. Generate two separate 32-byte hex secrets with
   `node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"`.
   Set `ADMIN_API_TOKEN` and `TOKEN_ENCRYPTION_KEY` to different generated values.
3. Run `npm start` and keep it running for registration and stop/resume commands.
4. Create a dedicated Telegram bot with @BotFather and `/newbot`.
5. Run `node --env-file=.env scripts/admin.ts save`, enter
   `{"token":"YOUR_BOTFATHER_TOKEN"}` on stdin, and end with Ctrl+D.
   Keep real credentials out of command arguments, source control and chat.
6. Open the returned pairing link, press Start, and run
   `node --env-file=.env scripts/admin.ts pair`.
7. Create a personal invitation:
   `node --env-file=.env scripts/admin.ts invite "Companion" ru`.
8. Send that link privately to the intended person. They open it and press Start.
   The running poller verifies their private Telegram chat and acknowledges it.
9. Run `node --env-file=.env scripts/admin.ts status`, then
   `node --env-file=.env scripts/admin.ts test RECIPIENT_ID` to verify delivery.

Invitations are single use and expire in seven days. Names on invitations are
labels, not recipient identity verification. Recipients need no ChatGPT account.
They can send `/stop` to pause and `/start` to resume. The owner can pause or
remove recipients and revoke unused invitations. Removed recipients need a new
invitation. `/language ru`, `/language en`, and `/language pt` choose explanation
language; the tutor must actually provide a matching copy before it can be sent.

## Tutor workflow and progress

Use [docs/TUTOR_WORKFLOW.md](docs/TUTOR_WORKFLOW.md) as the instruction source for
your tutor conversation or existing scheduled practice. Start from a blank
[examples/learner-state.json](examples/learner-state.json), then update it only
from actual learner attempts. Nothing in this release includes another learner's
private history or progress. The sample companion in
[examples/russian-companion.json](examples/russian-companion.json) is an example,
not a record of a learner's completed session.

Keep the existing practice schedule if you already have one. Do not create a
duplicate daily practice automation. After real recipient registration and
successful delivery verification, update that schedule to create the English
session and send a faithful Russian companion through the delivery tool.

MCP tools at authenticated `POST /mcp`:

- `telegram_connection_status`
- `telegram_create_invite`
- `telegram_sync_subscribers`
- `telegram_test_subscriber`
- `send_portuguese_lesson(lesson_key, text)`

The lesson tool sends Russian text to the verified owner and active invited
Russian-language subscribers. It processes invitations and stop commands first.
It returns separate delivery outcomes. For language variants or subscriber-only
sending, `POST /api/publish` accepts `content_key`, `copies` keyed by ru/en/pt, and
`include_owner`. The command-line publisher uses that endpoint:

```sh
node --env-file=.env scripts/admin.ts publish examples/russian-companion.json
```

Use a stable key for each lesson and examine `allSent` and per-recipient results.
Repeated completed deliveries are deduplicated. Failed or uncertain attempts are
never silently retried. Telegram confirmation means the API accepted the message
for the destination; it does not prove reading or learning.

## Hosting and security

The standalone Node listener defaults to loopback. Admin API and MCP requests
require `Authorization: Bearer <ADMIN_API_TOKEN>`; use an MCP client supporting
bearer auth. A ChatGPT-hosted plugin needs its hosting provider's supported
identity/OAuth integration; this repository does not implement an OAuth server.
For remote hosting, use an HTTPS reverse proxy and configure `PUBLIC_ORIGIN` and
`ADMIN_URL` without a trailing slash. Supervise the process and persist the
ignored `data/` folder. The bot token is encrypted with AES-GCM; retain the key
with your backup. Do not share admin credentials with lesson recipients.

The poller runs continuously. If the service stops, registration and commands
wait; Telegram keeps pending updates for at most 24 hours. This differs from a
private Sites deployment, which checks while its setup page is open and before
lesson sends. Do not make a private setup page public to work around webhook
access. No second scheduler or automatic daily generation is installed here.

## Tests

Run `npm test`. SQLite-backed tests cover encryption, invitation checks, private
chat registration, recipient receipts, fan-out, deduplication, opt-out, revocation,
language routing, durable migrations and local HTTP authentication. Telegram
responses in tests are simulated. Live rollout requires a real invited person to
press Start and a successful Telegram delivery receipt.

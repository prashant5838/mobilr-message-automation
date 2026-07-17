# WhatsApp Messaging Automation Service

Event-driven WhatsApp Business Cloud API automation for order lifecycle
notifications, cart-abandonment nudges, segmented broadcast campaigns, and
two-way conversation handling — with opt-in/opt-out compliance and an
audit trail throughout.

## Architecture at a glance

```
 Order/Cart events ─┐                                   ┌─> Meta WhatsApp
 (internal service) │                                   │   Cloud API
                     ▼                                   │
  API server (Express) ──> outbound-messages queue ──────┘
  /api/events/*            (BullMQ, rate-limited,
                             exp. backoff retries)
                     ▲
  campaigns ─────────┤
  /api/campaigns      broadcast-campaigns queue
                       (segments audience, fans out
                        into outbound-messages, throttled)

  cart-updated ──────> scheduled-triggers queue (delayed 2h job,
                        cancels on cart-converted)

  Meta ──> POST /webhooks/whatsapp ──> inbound-events queue ──> conversation
            (delivery receipts +                                parser/router
             inbound replies)                                   (TRACK/STOP/
                                                                   CANCEL/human)

  SQLite: contacts, messages, templates, campaigns, consent_audit_log,
          conversation_events
  Redis: BullMQ job storage/coordination for all four queues above
```

Two processes:
- `npm start` — the Express API + webhook receiver (`server.js`)
- `npm run worker` — the BullMQ workers that actually call the WhatsApp API
  (`src/queues/startWorkers.js`)

Running them separately lets you scale send throughput (more worker
instances) independently of API/webhook traffic, and means a slow Graph API
call never blocks webhook acknowledgement.

**Mock mode.** If `WA_ACCESS_TOKEN` / `WA_PHONE_NUMBER_ID` are not set, the
WhatsApp client logs what it *would* send instead of calling Meta. This lets
you exercise the entire pipeline — triggers, queueing, retries, the
dashboard, the two-way parser — without live credentials. Swap in real
credentials in `.env` when ready to go live.

## Setup

```bash
npm install
cp .env.example .env        # fill in WA_* credentials when available
npm run migrate             # creates SQLite schema
npm run seed                # sample contacts/templates/messages for the dashboard
npm start                   # terminal 1: API + webhook server
npm run worker              # terminal 2: queue workers
```

Requires a local Redis instance (`redis-server`, or `docker run -p 6379:6379 redis`).

Point WhatsApp's webhook configuration (Meta App Dashboard → WhatsApp →
Configuration) at `https://<your-domain>/webhooks/whatsapp`, using
`WA_WEBHOOK_VERIFY_TOKEN` as the verify token, and subscribe to the
`messages` webhook field.

## Event triggers

| Event | Endpoint | Template used |
|---|---|---|
| Order placed | `POST /api/events/order-placed` | `order_confirmation` |
| Order shipped | `POST /api/events/order-shipped` | `shipping_update` |
| Out for delivery | `POST /api/events/out-for-delivery` | `shipping_update` |
| Delivered | `POST /api/events/delivered` | `shipping_update` |
| Payment failed | `POST /api/events/payment-failed` | `payment_failed` (register via Template API) |
| Cart updated (schedules a >2h abandonment check) | `POST /api/events/cart-updated` | `cart_abandonment` |
| Cart converted (cancels pending nudge) | `POST /api/events/cart-converted` | — |

Example:
```bash
curl -X POST localhost:3000/api/events/order-placed \
  -H "x-api-key: $ADMIN_API_KEY" -H "Content-Type: application/json" \
  -d '{"phone":"+919876500001","name":"Ananya","orderId":"10234","amount":"₹2,499"}'
```

All admin/internal routes (`/api/*`) require an `x-api-key` header matching
`ADMIN_API_KEY`. The webhook path (`/webhooks/whatsapp`) is public but
signature-verified with `WA_APP_SECRET` when set.

In production, these endpoints are called by your order-management /
checkout services (directly, or via an internal event bus/SQS topic that a
small adapter forwards here) — not by a human.

## Template management

Templates go through: **draft → submitted (pending) → approved/rejected**,
support **versioning** (`POST /api/templates/:id/new-version`) and
**localization** (`POST /api/templates/:id/localize`, adds a new
language variant tied to the same template name).

```bash
# 1. Create a draft
curl -X POST localhost:3000/api/templates -H "x-api-key: $ADMIN_API_KEY" \
  -H "Content-Type: application/json" -d @templates/order_confirmation.json

# 2. Submit for Meta review
curl -X POST localhost:3000/api/templates/1/submit -H "x-api-key: $ADMIN_API_KEY"

# 3. Poll/refresh approval status (Meta reviews async, minutes-hours)
curl -X POST localhost:3000/api/templates/1/refresh-status -H "x-api-key: $ADMIN_API_KEY"
```

The `templates/` folder has 3 ready-to-submit examples: `order_confirmation`
(UTILITY), `shipping_update` (UTILITY), `cart_abandonment` (MARKETING).

## Broadcast campaigns

```bash
curl -X POST localhost:3000/api/campaigns -H "x-api-key: $ADMIN_API_KEY" \
  -H "Content-Type: application/json" -d '{
    "name": "Diwali Sale",
    "templateName": "cart_abandonment",
    "segmentFilter": {"tags": ["vip","repeat_buyer"], "opt_status": "opted_in"},
    "throttlePerSecond": 15
  }'

curl -X POST localhost:3000/api/campaigns/1/launch -H "x-api-key: $ADMIN_API_KEY"
```

The campaign worker resolves the audience (opt-in + tag filters), then
enqueues one job per recipient into `outbound-messages`, spaced to respect
`throttlePerSecond`. The outbound worker additionally enforces a **global**
rate cap (`WA_MAX_MSGS_PER_SECOND`) via BullMQ's `limiter`, so concurrent
campaigns and transactional traffic never collectively exceed WhatsApp's
per-second messaging limit. Marketing-category sends are also gated to
business hours (`BUSINESS_HOURS_START/END`); transactional sends are exempt
since they're user-expected service notifications.

## Two-way conversations

Inbound replies are parsed by `src/conversation/parser.js`:

- `TRACK <order id>` → bot looks up status and replies inline
- `STOP` / `UNSUBSCRIBE` → opt-out recorded + audit log entry + confirmation reply
- `START` / `SUBSCRIBE` → opt back in
- `CANCEL [order id]` → flagged for a human agent (cancellations aren't auto-approved by the bot)
- anything else → flagged `needs_human_agent = 1` on the contact and a holding reply is sent

`GET /api/contacts?needs_human_agent=1` lists conversations waiting on a
human. `POST /api/contacts/:id/resolve-human-agent` clears the flag once an
agent has handled it.

## Compliance & audit

- `contacts.opt_status` is the single source of truth checked before every
  send (checked again at send-time in the outbound worker, not just at
  enqueue-time, in case status changed while a job was queued).
- Every opt-in/opt-out change — from an inbound `STOP`/`START`, an admin
  action, or checkout — is appended to `consent_audit_log` (immutable,
  never overwritten) with source and raw message.
- Marketing sends respect configured business hours; utility sends don't.

## Reliability

- Retries: outbound sends use BullMQ exponential backoff (5 attempts:
  2s/4s/8s/16s/32s). Non-retryable failures (e.g. template not approved)
  throw `UnrecoverableError` immediately instead of burning the retry budget.
- Delivery receipts (`sent`/`delivered`/`read`/`failed`) from the webhook
  update the corresponding `messages` row and roll up into campaign counters.
- Webhook responds `200` immediately and defers processing to a queue, so a
  slow downstream handler never causes Meta to consider the webhook down
  (Meta disables webhooks after repeated timeouts/non-2xx).
- `jobId: cart-${cartId}` dedupes cart-abandonment checks — a new cart
  update replaces the pending delayed job rather than stacking duplicates.

## Dashboard

`GET /api/dashboard/metrics` — delivery/read/failure rates, per-trigger
breakdown, campaign summaries, opt-in/out counts, human-agent queue depth,
inbound intent breakdown.
`GET /api/dashboard/messages?limit=50` — recent message log.

See `dashboard/admin-dashboard-mockup.html` for a visual mockup consuming
this API (mock data pre-loaded; point `API_BASE`/`API_KEY` at a running
instance to go live).

## Project layout

```
src/
  api/          admin REST endpoints (dashboard, templates, campaigns, events, contacts)
  compliance/   opt-in/opt-out + business-hours logic
  conversation/ inbound text parsing + intent routing
  db/           SQLite schema, migration, seed
  events/       business-event -> template-send mapping
  queues/       BullMQ queue defs + the 4 workers
  utils/        logger
  webhooks/     WhatsApp webhook verify + receive
  whatsapp/     Graph API client + template lifecycle + 3 pre-built templates
templates/       exported JSON for the 3 pre-built templates
docs/            architecture notes + per-trigger message-flow diagrams
dashboard/       admin dashboard mockup (static HTML)
```

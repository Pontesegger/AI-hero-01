# Notification services against Cadence infrastructure

Research for [Evaluate notification services against Cadence infrastructure](https://github.com/Pontesegger/AI-hero-01/issues/3), part of [Specify actionable in-app and financial phone notifications](https://github.com/Pontesegger/AI-hero-01/issues/1). Checked **2026-11-10** against public first-party documentation; no accounts provisioned, SDKs installed, or live delivery/SSR tests performed. This is a throwaway research artifact, **not a vendor decision or implementation plan approved by the human**.

## Bottom line

Knock, Novu Cloud, and Courier all offer managed orchestration plus a persisted, React-compatible notification feed that can live in a bell popover, without a dedicated inbox page. App-owned SQLite notifications plus a direct phone provider are also credible: less external feed infrastructure, more application-owned reliability and UI work. An orchestrator does **not** replace the phone transport or its onboarding, consent obligations, and charges. [K1][N1][C1][T1]

The differentiators are feed ownership/retention, operational responsibility, actual connector receipt support, retries, and billing units—not simply “supports WhatsApp.” Particularly:

- Knock's **direct WhatsApp** channel explicitly cannot track Delivered; its logs stop at Sent. Do not generalize that limitation to its separate Twilio route without testing. [K4]
- Novu's current delivery policy explicitly says **one provider send attempt**, including for 429/503 failures; webhook retries are a different layer. [N4]
- Courier documents retries and error failover, but provider/channel **slow-response timeouts** require Enterprise. [C4]
- Saved user history and operational logs have different retention windows. None of those windows decides the product's bounded-history policy automatically. [K3][N3][C2]

## Existing constraints, verified in this checkout

Baseline: commit `913111d`. [README](../../README.md) suggests a free Knock account, not an integration; it identifies React Router SSR, SQLite/Drizzle and Vitest. [Database initialization](../../app/db/index.ts) opens local `data.db` with WAL. [Mailer](../../app/lib/mailer.server.ts) calls Postmark directly and writes JSON to `.outbox/` without a token; that development capture is **not a durable production dispatch queue**. [Refund services](../../app/lib/refunds.server.ts) return email payloads alongside business results. Preserve that existing email behavior unless separately changed.

The map limits scope to actionable lesson Q&A and Refund Requests, with important money-related events the initial phone priority. No phone provider, budget, history length, automatic popup, cross-channel duplication, escalation delay, or Q&A phone delivery has been agreed. A managed feed called “Inbox” does not require adopting a separate inbox page.

## Capability and infrastructure comparison

| Concern | Knock | Novu Cloud (self-host caveat below) | Courier | App-owned SQLite + direct phone provider |
| --- | --- | --- | --- | --- |
| Orchestration | Hosted workflows, channel steps, preferences; retry pipeline. [K1][K5] | Hosted workflows, subscribers, channel steps; dashboard or code-first workflows. [N1][N4] | Templates, routing, multi-step Journeys; error failover. [C4][C9] | Application owns recipient rules, templates and channel dispatch; provider only transports phone messages. **Design option**, not existing functionality. |
| Bell/popover and state | `NotificationIconButton` + `NotificationFeedPopover`, headless hooks; seen/read/archive states. Default badge is **unseen**, not unread; opening popover marks seen, not task completion. [K1][K2] | Bell + menu, notification list, unread counts, synchronized read changes and preferences; composable/headless alternatives. [N2] | React component can be inline or popup; hosted feed, unread/read/open/archive state and cross-device synchronization; custom UI possible. [C1][C2] | Build accessible popover and read/unread endpoints, per-recipient authorization, pagination and counts. Can reuse app styling; no vendor UI required. |
| Saved history / offline catch-up | Feed items remain available indefinitely even after message logs expire; paginated API with date/status filtering. A UI date filter is not deletion. [K3][K10] | Cloud Inbox retention: Free **30 days**, Pro/Team **90 days**, Enterprise custom; separate activity logs 1/7/90 days. [N3] | Feed persists until archive or message expiry; configurable `expiresAt`; read/archive survive sessions. [C2] | Choose age/count bounds and pruning explicitly. Fetch stored rows on return; polling/open/focus refresh can avoid a socket service. **Proposed design**. |
| React / SSR | React and Next.js client-component examples; server SDK for triggers. Secure browser feed with signed user token/enhanced security. React Router v8 hydration/import behavior **not tested**. [K1][K9] | Official React/Remix quickstarts and React Router routing; React 18+ including 19 documented. Enable subscriber HMAC for production. Exact Cadence SSR behavior **not tested**. [N2][N6] | Next.js guide explicitly says server import is safe, browser custom element hydrates, no `ssr: false` needed. Backend mints user-scoped expiring JWTs; SDK does not refresh tokens. React Router integration **not tested**. [C3] | Normal authenticated loaders/actions can serve history/counts at SSR; later browser refresh for new items. No third-party browser identity. **Proposed design**. |
| SMS connectors | Twilio, SNS, Plivo, Telnyx, Vonage and others. [K11] | Twilio, SNS, Plivo, Telnyx, Vonage and others. [N9] | Twilio, SNS, Plivo, Telnyx, Vonage and others. [C10] | One direct API integration, e.g. Twilio Messages for SMS and WhatsApp; no in-app feed provided by that transport. Example, **not provider selection**. [T1] |
| WhatsApp connector | Direct WhatsApp Business API, plus separate Twilio route; direct route has no Delivered tracking. [K4] | WhatsApp Business chat connector with phone-number ID, token and template payload; workflow notifications are distinct from agent conversations. Receipt mapping for this connector remains **unverified**. [N5] | Documented built-in WhatsApp integration uses **Twilio**, requires Content SID. Native direct-to-Meta connector **not verified**. [C5] | Direct Twilio covers both channels; a different transport can be evaluated independently. Transport-specific receipt handling belongs to app. [T1] |
| Persistence / operations | Knock holds feed/state, recipient metadata, preferences and workflow state; app keeps business data. [K1][K3][K7] | Novu holds subscribers/feed/preferences/workflows. Self-hosting moves these into **separate MongoDB**, plus Redis/worker/API/socket/dashboard—not Cadence SQLite. [N2][N7] | Courier hosts feed/state, profiles, preferences and templates. App keeps authoritative Refund Request and Q&A data. [C1][C2][C6] | SQLite owns notification history, read state, preferences, dispatch ledger and consent evidence; provider still holds phone-processing data. App owns backups, pruning, support tools and recovery. **Proposed design**. |

**Catch-up is not an offline client:** stored feed retrieval after reconnection can show notifications generated while a user was away; this is not evidence that any SDK works without network access. Re-fetch after reconnect and after authentication renewal should be acceptance tests. No automatic toast/popup or escalation follows from these capabilities.

## Preferences are not proof of phone consent

- **Knock:** channel/channel-type/category/workflow preferences and hosted/custom preference centers. Unset preferences default to **true**; environment defaults can change that, and workflows can override preferences. [K7]
- **Novu:** workflow, subscriber-global, and per-workflow channel preferences with Inbox UI. Channels default enabled; “critical” workflows discard subscriber preference layers. [N8]
- **Courier:** topic defaults can be opted-in, opted-out or required, with hosted/embedded UI and tenant/user overrides. `REQUIRED` bypasses user opt-out; custom channel-routing API availability can be gated. [C6]

**Architecture implication, not a legal conclusion:** none of these preference APIs by itself establishes verified phone ownership or a sufficiently evidenced permission to send. Keep transport-specific consent/suppression checks independent of a “critical/required” switch. Decide where consent provenance, time, wording/version, destination changes and revocation are authoritative, and how provider opt-outs are synchronized. The parallel transport research owns detailed WhatsApp/Meta/Twilio policies and regional onboarding; those are intentionally not repeated here.

## Reliability and webhooks: separate the three boundaries

1. **Business transaction → orchestration/transport:** even a managed service cannot atomically commit Cadence's SQLite Refund Request and its remote API call. A transactionally recorded dispatch intent/outbox is a design option for every candidate, not an existing queue. SQLite supports atomic transactions, but a dispatcher and recovery policy still have to be built. [S1]
2. **Orchestrator → provider:** an accepted trigger is not handset delivery. Provider receipts have channel-specific coverage. A read receipt is not proof the human completed the underlying action. [K5][N4][T1]
3. **Service → Cadence status webhook:** retryable HTTP delivery can produce duplicates; handlers need raw-body signature verification, replay/timestamp checks, durable receipt before successful acknowledgement, idempotent application and tolerance for late/out-of-order updates. These are engineering implications, not a universal exactly-once guarantee.

| Service | API duplicate protection | Provider dispatch | Outbound status/preference webhook semantics |
| --- | --- | --- | --- |
| Knock | `Idempotency-Key` on workflow trigger only; **24 hours**, matching payload, successful responses cached. [K6] | Up to **8 retries / 30 minutes**, exponential backoff+jitter for retryable failures. Receipt checks via supported provider webhook or polling; polling up to 10 checks / 30 minutes. [K5] | HMAC-SHA256 `x-knock-signature`, millisecond timestamp; selected event types, per-environment. Retries “a handful” over a few hours; selected 3xx/4xx are never retried, other non-2xx generally retried. Exact schedule not guaranteed. [K8] |
| Novu | Trigger supports API idempotency, but docs say **not enabled for all organizations; contact support**. `transactionId` is not equivalent atomic API protection. Confirm enablement/window before relying on it. [N10] | Current policy: **one attempt**, no automatic retry/failover even for provider 429/503; failed channel step continues workflow. Framework bridge retries are separate. [N4] | Svix signature/timestamp verification, 2xx within 15s; **8 attempts**, immediate then 5s/5m/30m/2h/5h/10h/10h. Manual replay/recovery; endpoint can disable after 5 days of failures. [N11] |
| Courier | Send API supports `Idempotency-Key`; default **25 hours**, custom expiration up to a year. Docs say cached original errors are replayed too. [C7] | Transient errors retried about **24h / 104 attempts**; error failover across configured providers/channels. Overall timeout default/cap 72h; provider/channel timeouts Enterprise-only. This is not permission to enable phone fallback. [C4] | Signed `courier-signature` over timestamp+raw body, 2xx within **10s**. Retries network/timeouts, 408/429/5xx about a day, then drops; no auto-disable. Every destination receives all event types. Message `data.id` repeats across legitimate status updates—do not deduplicate all updates by message ID alone. [C8] |
| Direct transport | App deduplication ledger required; remote send idempotency/reconciliation must be verified for chosen provider. | Twilio exposes message ID, queued/sent/delivered/undelivered and channel-dependent read states plus StatusCallback; app owns pre-acceptance retry and ambiguous-timeout recovery. [T1] | Transport-specific verification and recovery; reconcile against message resource where possible. Do not assume one transport's callbacks behave like an orchestrator's events. [T1] |

**Unresolved operational design:** dispatcher hosting (supervised loop/cron/process), restart recovery, retry deadline, stale actionable-message handling, manual redrive, alerts and deduplication after vendor key expiry. A SQLite outbox need not imply Redis, but it does imply a continuously/repeatedly running supervised dispatcher. Do not implement fire-and-forget promises inside request handlers. Resending an entire workflow to recover one failed phone step can duplicate its in-app item; recovery must be channel-aware.

**Source caveat:** Novu's retry page says `messages.failed`, while its webhook catalog lists `message.failed`. Confirm the actual event type/payload before writing a recovery consumer. The published retry policy is evidence of advertised behavior, not a fault-injection test. [N4][N11]

## Local development and verification effort

- **Knock:** isolated environments, dashboard/CLI tests, API sandbox mode and skip-delay settings. Sandbox previews do not prove provider delivery, and test runner trigger-frequency behavior differs from production API. No fully local Knock server was established. [K12]
- **Novu:** Remix quickstart/test trigger; development/production environments; local self-hosting is possible, but adds MongoDB, Redis and service containers. Code-first Framework bridge development can use a tunnel; it is not required merely to adopt dashboard-managed Cloud workflows. [N6][N7][N12]
- **Courier:** Test environment and built-in test provider; in-app needs no third-party transport. Test sends count toward usage. Webhook development requires a reachable endpoint/tunnel or recorded-payload fixtures. [C1][C8][C11]
- **App-owned option:** deterministic SQLite fixtures + fake phone adapter can exercise bounded history, unread state, outbox leases/restarts, duplicate callbacks and expired actions entirely locally; small real-provider sandbox tests remain necessary. Cadence's existing mail transport seam is a local-testing precedent, not a phone implementation.

No app changes were made, so `app/` typecheck/test completion checks do not apply. This research verified documentation, not working integrations.

## Pricing snapshot (USD, public list prices; recheck before purchase)

| Option | Billing unit and current entry tiers | Important caveats |
| --- | --- | --- |
| Knock | Free **10,000 messages/month**; Starter **$250/month includes 50,000**, then **$0.005/message**; Enterprise custom. One recipient on two channels = two messages. [K13] | Failed/bounced sends excluded. Phone/email provider fees separate. Free retains Knock branding; feed retention unlimited versus 30-day non-Enterprise operational logs. |
| Novu Cloud | Free **10,000 workflow runs/month**; Pro from **$30 / 30,000**, Team from **$250 / 250,000**; paid additional runs **$1.20/1,000** per pricing page. One execution per subscriber, independent of channel count. [N12][N13] | Dev/test runs count; Free hard-stops at cap (402). Transport fees separate. Free Inbox 30 days, Pro/Team 90 days; Pro removes branding. Self-host core has no Novu license fee, but infrastructure/operations and Enterprise features are separate costs. |
| Courier | Developer free **10,000 sends/month**; Business **$0.005 per send**; Enterprise custom. Multi-channel template to one recipient counts once. [C11][C12] | Billing docs explicitly say Business charges **every send, including first 10,000**, not only overage; failures, preference-blocked and Test sends count. Provider fees separate. Confirm any UI/tenant preference gates in account. |
| App-owned + direct transport | No orchestrator subscription; pay provider transport/number/registration charges and hosting/engineering/operations. | Not “free notifications.” No geographic/volume assumptions or comparative phone quote made; transport cost investigation is separate. |

Estimate from **recipient fan-out and channel mix**, not user count alone. Without monthly event volume, phone destinations, consent uptake, history bound and service-level expectations, a cost winner cannot be established.

## Remaining human decisions / gates

1. **Ownership:** managed feed + orchestration, app-owned feed + direct transport, or a hybrid? Avoid silently creating two competing read-state authorities.
2. **History and unread semantics:** age/count bound; whether pruning affects unread counts; seen versus read; whether an item remains after the underlying Refund Request/Q&A action is resolved. Knock feed content is immutable after generation, so stale items require archive/new message or app-derived rendering. [K2]
3. **Reliability contract:** acceptable delay/outage, duplicate tolerance, stale-phone cutoff, recovery operator, and whether carrier delivery receipts are required. No cross-channel escalation or automatic duplication is approved.
4. **Phone and consent:** integrate the sibling transport findings, supported recipient countries, required receipt support, sender/account onboarding and consent/suppression source of truth. Do not infer this from vendor preference UI.
5. **Budget/data boundaries:** volumes and paid-tier tolerance; profile/content minimization, residency, retention/deletion/export requirements and support-log retention. For financial events, keep business records authoritative in Cadence regardless of vendor feed retention.
6. **Before implementation:** confirm Novu idempotency enablement and failed-event spelling; test selected connector receipt mapping, SSR/hydration, session/user switching, offline catch-up, history bounds, opt-out races, ambiguous API timeouts, retries after crash, duplicate/out-of-order webhooks and preservation of existing Postmark emails. These are future verification gates, not blockers to completing this research.

## Primary sources

All links below were read on 2026-11-10. Documentation capabilities and list prices can change; “unverified” means no reliable conclusion was established here, not that the vendor lacks the feature.

- [K1] [Knock React feed/popover](https://docs.knock.app/in-app-ui/react/feed)
- [K2] [Knock in-app state, API exposure and immutable content](https://docs.knock.app/integrations/in-app/feeds)
- [K3] [Knock data retention: operational logs versus indefinite feed items](https://docs.knock.app/manage-your-account/data-retention)
- [K4] [Knock WhatsApp integration and Delivered limitation](https://docs.knock.app/integrations/chat/whatsapp)
- [K5] [Knock delivery retry/status pipeline](https://docs.knock.app/send-notifications/delivering-notifications)
- [K6] [Knock idempotency](https://docs.knock.app/api-reference/overview/idempotent-requests)
- [K7] [Knock preferences/defaults/overrides](https://docs.knock.app/preferences/overview)
- [K8] [Knock outbound webhook signing and retries](https://docs.knock.app/developer-tools/outbound-webhooks/overview)
- [K9] [Knock Next.js server/client integration](https://docs.knock.app/getting-started/quick-start/nextjs)
- [K10] [Knock paginated/date-filtered feed endpoint](https://docs.knock.app/api-reference/users/feeds/list_items)
- [K11] [Knock integration index, including SMS connectors](https://docs.knock.app/llms.txt)
- [K12] [Knock workflow testing/sandbox](https://docs.knock.app/send-notifications/testing-workflows)
- [K13] [Knock pricing and metering](https://knock.app/pricing)
- [N1] [Novu workflow/integration documentation index](https://docs.novu.co/_llms/notify.txt)
- [N2] [Novu Inbox and React support](https://docs.novu.co/platform/inbox)
- [N3] [Novu limits, including Inbox versus activity retention](https://docs.novu.co/platform/developer/limits)
- [N4] [Novu delivery retry policy](https://docs.novu.co/platform/developer/delivery-retries)
- [N5] [Novu WhatsApp Business workflow connector](https://docs.novu.co/platform/integrations/chat/whats-app)
- [N6] [Novu Remix quickstart](https://docs.novu.co/platform/quickstart/remix)
- [N7] [Novu self-host infrastructure, persistence and local development](https://docs.novu.co/community/self-hosting-novu/deploy-with-docker)
- [N8] [Novu preference precedence and critical workflows](https://docs.novu.co/platform/concepts/preferences)
- [N9] [Novu SMS connector index](https://docs.novu.co/_llms/notify.txt)
- [N10] [Novu trigger idempotency and enablement caveat](https://docs.novu.co/api-reference/events/trigger-event)
- [N11] [Novu outbound webhook events/signing/retries](https://docs.novu.co/platform/developer/webhooks/webhooks)
- [N12] [Novu billing, caps and provider costs](https://docs.novu.co/platform/account/billing)
- [N13] [Novu pricing](https://novu.co/pricing/)
- [C1] [Courier integration/SDK index, including popup and hosted feed](https://www.courier.com/docs/llms.txt)
- [C2] [Courier Inbox persistence, expiry and device synchronization](https://www.courier.com/docs/in-app/how-the-inbox-behaves)
- [C3] [Courier SSR import safety, JWT scope and renewal](https://www.courier.com/docs/guides/add-an-inbox-to-nextjs)
- [C4] [Courier retry, routing, failover and timeout gates](https://www.courier.com/docs/send/routing)
- [C5] [Courier WhatsApp via Twilio and required Content SID](https://www.courier.com/docs/integrations/direct-message/whatsapp)
- [C6] [Courier preference defaults, required topics and tenant caveats](https://www.courier.com/docs/recipients/preferences/overview)
- [C7] [Courier Send API and idempotency headers](https://www.courier.com/docs/api-reference/send/send-a-message)
- [C8] [Courier webhook event/retry/drop semantics](https://www.courier.com/docs/monitor/webhooks/outbound) and [signature verification](https://www.courier.com/docs/guides/track-delivery)
- [C9] [Courier Journeys documentation index](https://www.courier.com/docs/_llms/docs.txt)
- [C10] [Courier SMS connector index](https://www.courier.com/docs/llms.txt)
- [C11] [Courier billing: multi-channel unit, all sends and Test usage](https://www.courier.com/docs/workspaces/billing)
- [C12] [Courier pricing](https://www.courier.com/pricing)
- [T1] [Twilio Messages API: transport channels, statuses and callbacks](https://www.twilio.com/docs/messaging/api/message-resource)
- [S1] [SQLite atomic transactions](https://www.sqlite.org/transactional.html)

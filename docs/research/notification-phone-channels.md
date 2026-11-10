# WhatsApp and SMS for financial notifications

**Checked:** 10 November 2026 against public first-party documentation. **Status:** research, not a product decision or implementation specification. Resolves [Evaluate WhatsApp and SMS for financial notifications](https://github.com/Pontesegger/AI-hero-01/issues/2), under [Specify actionable in-app and financial phone notifications](https://github.com/Pontesegger/AI-hero-01/issues/1). Throwaway branch; do not merge. No accounts created, services provisioned, messages sent, or live delivery tests performed.

## Bottom line

- Both channels are plausible for a **Refund Request** notification, but neither establishes that a recipient acted, that a **Refund** occurred, or even that a person read the message. Keep provider acceptance, delivery, reading, and application action distinct. [S4, S12]
- WhatsApp requires recipient permission, business onboarding, and generally an approved template for unsolicited-in-time notifications; the 24-hour window is opened/reset by a user message, not by using Cadence or receiving a business message. SMS has no equivalent universal WhatsApp window, but is subject to provider consent rules and country/carrier registration and content requirements. [S1, S2, S5, S6]
- **Countries, sender business location, budget, volume, and intended recipients remain human decisions.** Without these, a global cost winner, legal conclusion, and production launch date would be invented. No channel/vendor, escalation, automatic duplication, or Q&A phone delivery is selected here.
- **Pricing changed in October 2026:** Meta now charges utility templates even inside the service window and charges service messages beyond 1,000 free service messages/month/business number. Some Twilio pages still state the previous free-utility-window rule. Use Meta's current pricing as the upstream rule and confirm actual provider billing before purchase. [S3, S7, S8]

## Consent and opt-out

| Area | Verified constraint | Consequence for later specification (not an agreed product rule) |
| --- | --- | --- |
| WhatsApp | Recipient must supply their number/username and opt in to subsequent contact. Honor opt-outs made **on or off** WhatsApp. Published privacy policy and applicable notices/permissions are required. [S1] | An existing phone number, purchase, or staff role is not automatically WhatsApp permission. Define the named sender, notification subjects, consent evidence, withdrawal UI, and suppression behavior. |
| SMS through Twilio | Prior express consent is required even for non-marketing transactional/informational messages; retain evidence, date, and method. Consent is sender/subject-specific, not transferable; conversational consent is not recurring-notification consent. Some downstream-platform scenarios require express written consent. [S5] | Obtain permission from the actual Student/Instructor/Platform Admin/Team Admin recipient, rather than assuming someone else's Refund Request authorizes contact. Check whether Cadence or each organization owns the sender. |
| Opt-out through Twilio | First message must include “Reply STOP to unsubscribe” or equivalent standard keyword. Withdrawal must be easy; after one confirmation, no further messages absent renewed consent. [S5] | Budget copy/segments for opt-out text. Verify the chosen sender supports replies and provider suppression scope. Do not assume blocking a WhatsApp business also suppresses SMS, or that a provider handles every off-channel withdrawal. |

Provider policy is not a country-specific legal opinion. Human/legal review must settle applicable transactional-message rules, quiet hours, minors, evidence retention, and re-consent when numbers change. Suggestion: record consent per recipient/channel/sender/subject with notice version and time; recheck it before any queued send or retry. This is a risk-control recommendation, not authorization to build it. [S1, S5, S6]

## WhatsApp: onboarding, content, reach

- **Direct Meta Cloud API:** business portfolio, WhatsApp account, Messaging account, business app, tokens/permissions, registered business number and webhook endpoint. Current Meta terminology separates WhatsApp account (presence/numbers) from Messaging account (templates/billing); older provider docs still use WABA. Verification affects capabilities/limits. [S2]
- **Twilio:** upgraded account, portfolio administrator, sender number that can receive an ownership OTP by SMS/voice, profile/display-name review and Meta business verification before production under Twilio's documented flow. Verification can take weeks. Existing personal/Business app numbers need migration planning; do not delete an existing account merely to experiment. Provider-specific coexistence differs: Telnyx documents a separate coexistence flow. [S9, S11]
- **Templates/window:** outside 24 hours after the last user message, use an approved template; approval can be rejected, paused or withdrawn. Automation needs a clear escalation path (email/web support can qualify). A Refund Request tied to an agreed transaction is a **utility candidate**, not guaranteed to be categorized that way. Adding promotional copy can make a template marketing. Platform Admin/Team Admin internal alerts require explicit category/use-case confirmation rather than assuming that a customer refund-update example covers them. [S1, S7]
- **Text/links:** free-form text supports 4,096 characters and optional URL previews, but that does **not** make free-form sending valid outside the window. Standard template body: 1,024 characters; header/footer: 60 each. URL buttons support a URL up to 2,000 characters with a variable appended at the end; parameter values need correct encoding. Submit samples and localized templates for review. [S13, S14]
- **Reach:** recipient needs an accessible WhatsApp account; valid phone syntax alone is insufficient. Twilio documents blocked destinations including Cuba, Iran, North Korea, Syria, Crimea/Donetsk/Luhansk; business/use-case restrictions can add others. Do not interpret a pricing-country dropdown as a supported-route guarantee. [S11, S15]
- **Identifiers:** current Meta webhooks can omit phone numbers for username users and instead include business-scoped user IDs (BSUIDs); map replies/opt-outs to the correct recipient without assuming every inbound identifier is E.164. Provider/region rollout support still needs account verification. [S18, S7]
- **Financial/privacy limits:** WhatsApp prohibits sharing/requesting full payment-card, financial-account, or sensitive-ID numbers. Its restricted financial verticals include payday loans, debt collection, etc.; an education refund alert is not blanket permission for financial-services messaging. Validate actual copy and business model. [S1]
- **Capacity:** Meta has portfolio-level messaging limits, per-number throughput and per-recipient rate limits, with quality-related enforcement. Current public defaults are not an account's approved production entitlement; inspect actual limits after onboarding. [S2]

## SMS: content, geography, onboarding

- SMS is carrier-routed text; unlike WhatsApp it does not require the recipient to install WhatsApp. Sender types, two-way replies, registration, throughput, supported routes, and filtering vary by country/provider. Twilio geo permissions are an additional account gate, not permission to ignore local rules. [S6, S16]
- Normal single-segment limits: **160 GSM-7** or **70 UCS-2** units; concatenated messages normally allow **153/67** per segment. GSM extension characters and some Unicode characters consume extra units; emoji/smart quotes can change encoding. US/Canada toll-free concatenated limits differ (152/66). Charges are per segment, not per notification. Count the final localized copy, URL, sender identity and opt-out instructions. [S10]
- There is no single worldwide SMS template-approval rule. For example, US long-code A2P traffic has registration requirements; India distinguishes international routes from domestic DLT-registered senders/use cases and disallows shortened URLs. India's public guide also broadly prohibits “money/loan” content: whether a Refund Request alert is accepted requires provider clarification if India is selected, not an assumption either way. These are examples, **not chosen target countries**. [S16, S17]
- Links are text and consume segment space. A short link may reduce cost but worsen filtering or violate destination rules; use an owned, recognizable HTTPS domain as a recommendation, with destination-specific validation. A reply-incompatible sender also needs a compliant opt-out design agreed with the provider. [S5, S17]

## Delivery, reading, retries and duplicates

| Evidence | What it establishes | What it does not establish |
| --- | --- | --- |
| Successful create response + provider message ID / queued / accepted | Provider created/accepted the send request. Store this response; Twilio does not emit a callback for the initial status. [S4] | Handset delivery or reading. |
| Sent | Handoff to downstream messaging network; Telnyx explicitly describes carrier acceptance for SMS / WhatsApp acceptance for WhatsApp. [S11, S12] | Recipient device delivery. |
| Delivered | Delivery receipt reported by channel/carrier. SMS confirmation availability/precision is route-dependent; Telnyx also exposes `delivery_unconfirmed`. [S4, S12] | Actual human reading, correct identity, or action. |
| WhatsApp read | Read receipt, **if enabled** by recipient; SMS has no corresponding read status in these APIs. [S4, S11] | Comprehension, acknowledgement of responsibility, Refund processing, or in-app read state. Absence of receipt is not proof of unread. |

**Recommended reliability questions for the later spec:**

1. Define event/recipient/channel deduplication and separate send attempts from logical notifications. Callback retry is **not** permission to send another message. Meta retries failed webhook delivery for up to seven days and warns of duplicate notifications; Telnyx documents duplicate/out-of-order callbacks and persistent event-ID deduplication. Authenticate callbacks, durably record them, then acknowledge promptly. [S18, S12]
2. A create timeout can mean “accepted but response lost.” Reconcile provider IDs/logs rather than blindly creating another send. Twilio explicitly documents reconciliation for messages created without a returned SID. Recheck permission and relevance before a retry; terminal consent/template/route errors need correction, not infinite retry. [S19, S1, S5]
3. Decide expiry, transient-error backoff, retry cap, stale-event cancellation and monitoring. Sending and webhook retries are separate mechanisms; the reviewed sources do not establish an end-to-end exactly-once delivery guarantee. Missing delivery/read signals must not automatically trigger SMS fallback: that would add a new product policy, cost and consent requirement.
4. Decide how late/out-of-order delivery affects the UX. Reading a WhatsApp message must not silently mark the bell notification read or resolve the underlying Refund Request; this is a proposed separation for human approval, not a glossary change.

## Small provider comparison and cost model

Public list-price examples in USD, retrieved on the checked date; not a quote, budget, worldwide price, or claim of measured deliverability.

| Option | Useful distinction | Published charging structure / caveat |
| --- | --- | --- |
| **Meta Cloud API direct (WhatsApp only)** | No BSP messaging layer; Cadence owns Meta credentials, templates, billing and webhook integration. SMS requires another provider. [S2] | Meta charges **delivered business-to-user messages** by category and recipient number market, with utility/authentication volume tiers. Current service free tier is 1,000/month/business number; utility-in-window is charged since Oct 1, 2026. Direct integration avoids a BSP's per-message handling fee, not operating/support costs. [S3] |
| **Twilio (WhatsApp + SMS)** | Common messaging API, sender onboarding, status callbacks, WhatsApp sandbox. [S4, S8, S9, S20] | WhatsApp: **$0.005 inbound or outbound** handling plus Meta charges; failed-message processing/add-ons can add fees. US SMS example: **$0.0083/segment** inbound/outbound plus carrier fees, numbers and registration. The US example is not an assumed market. Documentation has conflicting free-window descriptions; confirm billing. [S8, S21] |
| **Telnyx (WhatsApp + SMS)** | Embedded Meta signup, WhatsApp templates/statuses, SMS delivery reports including unconfirmed and signed callbacks. [S11, S12] | WhatsApp detailed table/calculator: **$0.004/message + Meta**. Same page's hero still says “conversation” and “$0 markup”; use detailed per-message breakdown pending a written quote. US local SMS example: **$0.004/part + carrier**; numbers/10DLC are separate. Availability, exact inbound WhatsApp fee scope and country pricing require confirmation. [S22, S23] |

Estimate only after country/volume/copy choices:

- **WhatsApp:** delivered messages × current Meta market/category/tier rate, less applicable free allowance + BSP handling for applicable inbound/outbound traffic + number/optional-feature/tax costs. Do **not** assume all replies or in-window utility templates are free. [S3, S8, S22]
- **SMS:** sum of billable segments × destination/carrier/sender rate + inbound replies + recurring number/campaign costs + registration/setup + optional features/taxes/failed-send fees. Compare the same route, localized copy and expected delivery evidence, not headline “per-message” rates. [S10, S21, S23]

## Privacy and testing boundaries

- Cloud API documents Signal-protocol protection plus TLS/at-rest encryption, but that does not make message content inaccessible to the business integration. A BSP adds another processor; Twilio documents message-body access and retention/redaction controls, including limited-access compliance copies. Redaction is **not** a universal immediate-deletion guarantee; eligibility/configuration needs checking (the page specifies Twilio Editions), and Twilio has limited control over WhatsApp retention. [S2, S24]
- Recommendation: phone copy should be minimal, with a link to an authenticated, authorized Cadence page; no payment credentials, refund reasons or sensitive identifiers in the body/URL/logs. Avoid bearer action links or “approve Refund” side effects on GET: previews/scanners/forwarding are a disclosure/action risk. Treat SMS as unsuitable for secrets; app-to-provider HTTPS alone does not establish end-to-end confidentiality through carriers. Review DPA/subprocessors, regions/transfers, retention/access/deletion, consent logs, shared/recycled numbers and lock-screen exposure before selection. These are security-review items, not verified account configurations. [S1, S13, S24]
- **Without signup:** later local tests can mock provider responses and replay documented webhook fixtures (accepted vs delivered, disabled read receipts, duplicates/out-of-order events, consent withdrawn in queue, create timeout, Unicode segmentation, stale Refund Request, rejected template). This research added no code.
- **With authorized accounts later:** Meta creates test accounts/numbers without a payment method for template testing. Twilio's WhatsApp sandbox uses joined recipients, preapproved templates only, one message/three seconds, and three-day join sessions; it cannot validate custom refund templates or production throughput and messages are normally billable. Console migration may put new trials in “Try out WhatsApp” instead. [S2, S20]
- Twilio documents legacy test credentials/magic SMS numbers: no real delivery or charges, **no status callbacks**, and no general WhatsApp simulation. Availability depends on console/account generation. Actual carrier acceptance, local sender rules, custom template approval and prices need a later authorized pilot using consented test devices, HTTPS callbacks and verified signatures. No sandbox result should be treated as production delivery evidence. [S25, S12, S20]

## Human decisions / account-dependent follow-up

1. Which recipient countries and languages, sender legal entity/location, and recipient roles? Is the sender Cadence or individual organizations? This determines consent scope, registration, Meta onboarding and whether internal financial alerts fit utility templates.
2. WhatsApp, SMS, recipient choice, or neither when permission/reach is unavailable? What happens on opt-out or failed delivery? Existing email behavior, escalation and channel duplication remain unchanged unless separately decided.
3. Monthly/peak send volume, acceptable setup/recurring spend, hard spend cap, urgency/quiet hours, expiry/retry budget and tolerance for unknown outcomes? No exact budget was supplied.
4. Exact Refund Request trigger/action/copy, acceptable lock-screen disclosure, and whether an amount may appear? Do not conflate requesting a Refund with completing one.
5. Who owns onboarding/support/replies/opt-outs, business verification, sender number, template languages and privacy review? Which account limits and route-specific registrations apply? Confirm quotes and the documented pricing discrepancies in a live account **only after authorization**.

## Sources

All below are first-party pages accessed 10 November 2026. Public docs establish advertised behavior, not tested account eligibility or a legal opinion.

- **S1** [WhatsApp Business Messaging Policy](https://business.whatsapp.com/policy) — updated Sep 23, 2026; consent, window, privacy, restricted verticals.
- **S2** [Meta: About the WhatsApp Business Platform](https://developers.facebook.com/docs/whatsapp/cloud-api/overview) — accounts, test resources, encryption, limits.
- **S3** [Meta: Pricing](https://developers.facebook.com/docs/whatsapp/pricing/) — explicitly effective Oct 1, 2026, including utility/service changes.
- **S4** [Twilio: outbound status lifecycle](https://www.twilio.com/docs/messaging/guides/outbound-message-status-in-status-callbacks).
- **S5** [Twilio Messaging Policy](https://www.twilio.com/en-us/legal/messaging-policy) — updated Apr 13, 2026.
- **S6** [Twilio: SMS Geo Permissions](https://www.twilio.com/docs/messaging/guides/sms-geo-permissions).
- **S7** [Twilio: WhatsApp key concepts](https://www.twilio.com/docs/whatsapp/key-concepts) — categories, windows, WAME; includes outdated free-utility-window claim despite Nov 2026 page modification.
- **S8** [Twilio: WhatsApp pricing](https://www.twilio.com/en-us/whatsapp/pricing) — handling fees; prose/calculator disagree about in-window charging.
- **S9** [Twilio: WhatsApp Self Sign-up](https://www.twilio.com/docs/whatsapp/self-sign-up).
- **S10** [Twilio: character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit).
- **S11** [Telnyx: WhatsApp quickstart](https://developers.telnyx.com/docs/messaging/whatsapp/overview).
- **S12** [Telnyx: receiving messaging webhooks](https://developers.telnyx.com/docs/messaging/messages/receiving-webhooks).
- **S13** [Meta: text messages](https://developers.facebook.com/documentation/business-messaging/whatsapp/messages/text-messages).
- **S14** [Meta: template components](https://developers.facebook.com/documentation/business-messaging/whatsapp/templates/components).
- **S15** [Twilio: geographic restriction error 63058](https://www.twilio.com/docs/api/errors/63058).
- **S16** [Twilio: United States SMS guidelines](https://www.twilio.com/en-us/guidelines/us/sms).
- **S17** [Twilio: India SMS guidelines](https://www.twilio.com/en-us/guidelines/in/sms).
- **S18** [Meta: webhooks overview](https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/overview).
- **S19** [Twilio: delivery logging and reconciliation](https://www.twilio.com/docs/messaging/guides/outbound-message-logging).
- **S20** [Twilio: WhatsApp sandbox](https://www.twilio.com/docs/whatsapp/sandbox).
- **S21** [Twilio: United States SMS pricing](https://www.twilio.com/en-us/sms/pricing/us.html).
- **S22** [Telnyx: WhatsApp pricing](https://telnyx.com/pricing/whatsapp) — detailed rates refreshed Nov 9, 2026; hero text conflicts with table.
- **S23** [Telnyx: SMS pricing](https://telnyx.com/pricing/messaging).
- **S24** [Twilio: Message Redaction](https://www.twilio.com/docs/messaging/guides/privacy-message-redaction).
- **S25** [Twilio: test credentials](https://www.twilio.com/docs/iam/test-credentials) — legacy-console caveat.

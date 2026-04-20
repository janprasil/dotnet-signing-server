# Stripe Webhook Events — dotnet-signing-server

Endpoint: `POST /api/webhooks/stripe` (`Controllers/StripeWebhookController.cs`)

When configuring the webhook endpoint in Stripe Dashboard (Developers → Webhooks → Add endpoint), enable the following event types. Local development via `stripe listen --forward-to localhost:5000/api/webhooks/stripe` forwards all events by default — prod must explicitly subscribe.

## Subscribed events

| Event | Handler | Purpose |
|-------|---------|---------|
| `checkout.session.completed` | `HandleCheckoutSessionCompletedAsync` | Safety net for crediting after one-time credit pack purchase. Primary credit path is the `/Billing/Checkout/Confirm` client redirect (`BillingController.ConfirmCheckout`); webhook handles the case where the user closes the browser before redirect. |
| `payment_intent.succeeded` | `HandlePaymentIntentSucceededAsync` | Credits auto-recharge purchases (off-session `PaymentIntent` with `metadata.type = "auto_recharge"`). Other payment intents are ignored. |
| `payment_intent.payment_failed` | `HandlePaymentIntentFailedAsync` | Sends failure email to user when an off-session auto-recharge payment is declined. |
| `payment_method.detached` | `HandlePaymentMethodDetachedAsync` | Disables auto-recharge for the affected user when their **last** saved payment method is removed (e.g. via the Stripe Billing Portal). Prevents auto-recharge from failing at runtime. |

## Idempotency

All events are recorded in `WebhookEvents` table (schema `dotnet_signing`). The outer handler (`HandleWebhook`) stores `EventId = stripeEvent.Id` with a unique constraint — duplicate webhook deliveries are skipped.

For `checkout.session.completed`, a second idempotency row is written with `EventId = session.Id, EventType = "checkout.confirm"` to prevent double-grant race with `ConfirmCheckout`.

## Auto-recharge flow summary

1. User enables auto-recharge during checkout (`saveCard`/`autoRecharge` checkbox → `PaymentIntentData.SetupFutureUsage = "off_session"` → Stripe attaches PaymentMethod to Customer).
2. When the user's balance falls below `AutoRechargeService.ThresholdCredits` (10), the next successful sign request fires `TryAutoRechargeAsync` (fire-and-forget in `ApiController.DebitUserAsync`).
3. Stripe confirms the PaymentIntent off-session → `payment_intent.succeeded` webhook → credits added (with `auto_recharge_{pi_id}` idempotency key).
4. If the payment is declined → `payment_intent.payment_failed` webhook → email notification, 15-minute cooldown before retry.
5. If the user removes their last card → `payment_method.detached` webhook → auto-recharge disabled automatically.

## Non-subscribed (intentionally ignored)

Stripe sends many events during a purchase flow (`invoice.created`, `invoice.finalized`, `charge.succeeded`, `mandate.updated`, etc.). These are not needed for the one-time-purchase model — the platform uses **usage credits**, not subscriptions. If the model later adds subscriptions, revisit this list.

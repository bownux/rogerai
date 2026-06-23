# RogerAI billing (Stripe wallet)

Prepaid credits: users buy credits via Stripe Checkout; a webhook credits their wallet.
**1 credit = $1** by default (`ROGERAI_CREDIT_USD`). SDK-free (raw Stripe API + stdlib HMAC).
Inert until `STRIPE_SECRET_KEY` is set - the broker logs `billing: disabled` and `topup`/webhook
return 503 until then.

## What's built
- `POST /billing/checkout` `{ "usd": 10 }` → `{ "url": "<stripe checkout>", "credits": 10 }`
- `POST /billing/webhook` → verifies `Stripe-Signature` → on `checkout.session.completed`
  calls `store.AddCredits(user, credits)`
- CLI: `rogerai topup [usd]` → prints the checkout link
- (TUI `/topup` and Stripe **Connect** payouts to owners are the next step)

## Activate it (your steps)
1. **Stripe → Developers → API keys**: copy the **Secret key** (`sk_test_…` to start, `sk_live_…` later).
2. **Stripe → Developers → Webhooks → Add endpoint**:
   - URL: `https://broker.rogerai.fyi/billing/webhook`
   - Event: `checkout.session.completed`
   - copy the **Signing secret** (`whsec_…`).
3. Add to `~/ai/RogerAI/.env` **and** as DO app secrets (encrypted):
   ```
   STRIPE_SECRET_KEY=sk_test_…
   STRIPE_WEBHOOK_SECRET=whsec_…
   # optional:
   STRIPE_SUCCESS_URL=https://rogerai.fyi/topup/success
   STRIPE_CANCEL_URL=https://rogerai.fyi/topup/cancel
   ROGERAI_CREDIT_USD=1.0
   ```
   (I can push these to the DO app via `doctl apps update` once you've added them.)
4. **Test**: `rogerai topup 10` → open the link → pay with Stripe test card `4242 4242 4242 4242` →
   the webhook credits your wallet → `rogerai balance` shows +10.

## Payouts to owners (follow-up)
Owners accrue credits per request (broker already tracks `rogerai.earnings`). Payouts will use
**Stripe Connect Express** - onboarding link per owner, then batched weekly transfers above a
threshold (~$25), platform take = the broker `--fee` (25%). Needs Connect enabled on the account.

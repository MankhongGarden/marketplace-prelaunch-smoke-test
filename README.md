# Sixteen Production Blockers ฿125 Found That Four Rounds of Static QA Missed
## A Pre-Launch Smoke Test Runbook for Next.js + Supabase + Stripe Marketplaces

This is a runbook, framed around a real number: **฿125** of live PromptPay charges, **16 production blockers** found, all of which would have hit my first real user. None of them showed up in the four prior Tester rounds of static review (TypeScript build · ESLint · static query checks · manual code walks).

The thesis: static review catches the easy 80%. The hard 20% — the bugs that hit your first user — only show up under live click-through with real money. This document is the structure I now use before any marketplace launch on Next.js + Supabase + Stripe.

---

## What the 16 bugs looked like (taxonomy)

The bugs sorted into 5 categories:

| Category | Count | Example |
|---|---|---|
| Schema / code drift | 4 | Code referenced `bookings.confirmed_at` · column didn't exist in DB |
| RLS policy gaps | 4 | Admin-only INSERT policy silently rejected buyer/team writes |
| Stripe live-mode regulatory | 2 | PromptPay live charge required `billing_details.email`; test mode tolerated missing |
| State-label collisions | 2 | `booking.status = 'confirmed'` meant 2 different things per actor |
| UX / missing-link | 4 | No nav to the new feature · no status separation between actors |

**All 16 were silent.** TypeScript built clean. Postgres returned 400s that the Supabase client swallowed. The user just saw a button that did nothing. This is the failure mode static review can't catch — it requires a real payment, a real DB write, a real RLS evaluation under real auth context.

The five categories below are the screen I now run *before* burning live money, plus the five-step ฿30 PromptPay flow that actually finds the bugs.

---

## Pre-flight (run before live test)

Eight cheap checks. If any fail, the live test will waste money on bugs you could have caught with a SQL query.

| # | Check | Expected | Why |
|---|---|---|---|
| 1 | Stripe KYC approved | `details_submitted=true · charges_enabled=true` | Live charges fail without |
| 2 | All 3 Stripe env vars flipped to live | `sk_live_*` · `pk_live_*` · `whsec_*` (live) | sk/pk mode mismatch = 401 |
| 3 | Pre-launch banner removed (if applicable) | banner gone | UX confusion if live but UI still says "soft launch" |
| 4 | Webhook live endpoint registered | `we_*` in Stripe dashboard with correct events | Without it, webhook gets zero deliveries |
| 5 | Admin role assigned to test user | `users.role='admin'` | Many flows need admin-gated routes |
| 6 | Phone OTP verified for test user | `auth.users.phone_verified_at IS NOT NULL` | First-action gate may block |
| 7 | At least one active supplier in DB | `count(*) WHERE is_active=true AND is_accepting=true` | No supply = no transaction possible |
| 8 | `auth.users` NULL string check | `count(*) WHERE phone_change IS NULL OR phone_change_token IS NULL OR ...` = 0 | gotrue crashes login when any string column is NULL |

The last one specifically: Supabase's gotrue (the auth backend) has a long-standing quirk where it `string.split()` on columns assumed to be strings. If a row has `NULL` in `phone_change` or `phone_change_token` (or six other columns), login crashes for that user. Easy fix with a `COALESCE` UPDATE — but if you don't know about it, login works on test mode (clean fixtures) and breaks in prod (real users with partial profiles).

The COALESCE fix:

```sql
UPDATE auth.users SET
  phone_change             = COALESCE(phone_change,             ''),
  phone_change_token       = COALESCE(phone_change_token,       ''),
  email_change             = COALESCE(email_change,             ''),
  email_change_token_new   = COALESCE(email_change_token_new,   ''),
  email_change_token_current = COALESCE(email_change_token_current, ''),
  reauthentication_token   = COALESCE(reauthentication_token,   ''),
  confirmation_token       = COALESCE(confirmation_token,       ''),
  recovery_token           = COALESCE(recovery_token,           '');
```

Run this once before the smoke test. Make it part of your pre-launch checklist.

---

## Screen 1: schema / code drift checks

The cheapest bug class to catch: code references a column that doesn't exist. The TypeScript compiler can't help because the database client is `any`-typed at the boundary.

Grep `src/` for column accessors on a few high-traffic tables, then verify each in `information_schema`:

```sql
SELECT column_name
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'bookings'
  AND column_name IN ('paid_at', 'confirmed_at', 'contact_revealed_at');

SELECT column_name
FROM information_schema.columns
WHERE table_schema = 'public'
  AND table_name = 'users'
  AND column_name IN ('full_name', 'display_name', 'name');
  -- 'name' does NOT exist in standard Supabase auth.users
```

Any row missing from the result is a drift. Two common patterns:

- **Migration shipped, code wasn't refactored.** Column was renamed or dropped; some leaf component still queries the old name. PostgREST returns 400 with `column does not exist`. Supabase JS client swallows it into an empty result. The user sees "no data" silently.
- **Code shipped, migration didn't.** Column added in a new feature's component; the migration adding it never went out, or only went out to staging. Same symptom, opposite root cause.

The fix is mechanical (rename the code or ship the migration), but the *detection* is what static review misses.

---

## Screen 2: RLS policy gaps

For every user flow that INSERT / UPDATE / DELETE-s on tables owned by buyer or seller, confirm the policies exist for *that role*:

```sql
SELECT polname, polcmd, pg_get_expr(polqual, polrelid) AS using_expr
FROM pg_policy
WHERE polrelid = 'public.bookings'::regclass
ORDER BY polcmd, polname;
```

Look for `polcmd = 'a'` (INSERT) and `polcmd = 'w'` (UPDATE) policies. If only `polcmd = 'r'` (SELECT) and admin-named policies exist, the buyer-side write path will silently fail.

Common patterns this catches:

- **Admin-only INSERT policy** on a table both admin and buyer need to write. Buyer's INSERT returns 401 from PostgREST; Supabase client returns `{error: {...}, data: null}`; UI logs the error to a `console.error` no one reads.
- **Policy gating on a phased-out enum value.** E.g., a SELECT policy `WHERE verification_status = 'verified'` after the brand renamed `'verified'` to `'enrolled'`. The policy still evaluates; it just never matches, so users see empty lists.
- **Missing `polcmd = 'w'`** when migration added INSERT but forgot UPDATE. INSERT works; UPDATE silently fails the moment the user tries to edit.

Grep extension for catching the second pattern after vocab changes:

```bash
grep -rE "verified|vetted|certified|guaranteed" supabase/migrations/
```

Plus query the policies for vocab references:

```sql
SELECT polname, polrelid::regclass
FROM pg_policy
WHERE pg_get_expr(polqual, polrelid)   ILIKE '%verified%'
   OR pg_get_expr(polwithcheck, polrelid) ILIKE '%verified%';
```

Any hit is a policy that needs to be re-evaluated against current vocabulary.

---

## Screen 3: state-label collisions

This was the most expensive category to debug. Look for enum values that *look* the same across actors but mean different things.

Example from my project: `booking.status = 'confirmed'` had two meanings:

- **Buyer side:** "I paid my fee; waiting on the seller to also pay."
- **Seller side:** "Both fees paid; contact revealed; engagement is live."

The same enum value. Two different UI states. Each side's component checked `status === 'confirmed'` and showed completely different copy. The buyer thought everything was done; the seller thought everything was pending. Neither realized the other had a different view of the same row.

Detect by grepping:

```bash
grep -rn 'booking.status\|status === "confirmed"' src/app/dashboard/
```

If the same status check appears in both buyer-side and seller-side components, **they need separate state.** Add explicit columns (`contact_revealed_at`, `seller_acknowledged_at`) that each side can read distinctly — don't overload the state machine column with role-specific meaning.

---

## Screen 4: Stripe live-mode regulatory quirks

Two bugs hit during live testing that test mode tolerated. The regulator's view of live charges is stricter than Stripe's test-mode validator.

| Parameter | Test mode | Live mode (TH PromptPay) |
|---|---|---|
| `payment_method_data.billing_details.email` | optional · works without | **required** |
| `payment_method_data.billing_details.name` | optional | recommended (improves dashboard receipts) |
| `description` | optional | recommended (regulatory paper trail) |

The fix is one line per Stripe call:

```ts
const payment = await stripe.paymentIntents.create({
  amount: 3000, // ฿30
  currency: 'thb',
  payment_method_types: ['promptpay'],
  payment_method_data: {
    type: 'promptpay',
    billing_details: {
      email: user.email,        // REQUIRED live
      name:  user.display_name, // recommended
    },
  },
  description: `Buyer fee · request ${request.number}`, // recommended
});
```

Apply to *both* the buyer-fee path *and* the seller-fee path. I caught the buyer side first, missed the seller side, and the seller's first live charge failed silently — the PI never reached `succeeded`, the webhook never fired the "engagement live" handler.

---

## The 5-step ฿30 PromptPay smoke test

This is the actual click-through. ฿30 per real charge × 4 charges (buyer + seller × 2 retries) ≈ ฿120-150 total. Plan to refund all of them after.

### Step 1 — Buyer creates a request

Login as buyer (Google / LINE / email). Navigate to the new-request page. Fill the smallest-tier form (cheapest district, smallest size). Set the preferred-date field to at least `now + 5 days` (gives the state machine cron buffer). Submit.

Verify (run after submit):

```sql
SELECT id, request_number, status, flow_status_v2, preferred_dates_arr
FROM inspection_requests
WHERE buyer_id = '<test_buyer_uuid>'
ORDER BY created_at DESC LIMIT 1;
```

Expected: `status='open'`, `flow_status_v2='bidding'`.

### Step 2 — Seed a fake bid (skip the seller-side join flow for speed)

Inserting directly into `request_matches` is faster than going through the seller's join UI, which often has its own onboarding hurdles. You're testing the buyer's pick + pay flow here; the seller's onboarding is a separate smoke test.

```sql
INSERT INTO request_matches (
  request_id, team_id, status, match_score,
  bid_price_thb, bid_team_size, bid_scope, bid_message,
  bid_submitted_at, anonymous_alias,
  estimated_duration_hours, includes_pdf_report, report_delivery_days
) VALUES (
  '<request_id>', '<test_team_id>', 'bid_submitted', 92,
  2500, 2,
  '["scope_a","scope_b","scope_c"]'::jsonb,
  'smoke test bid',
  now(), 'ทีม A1',
  3, true, 2
);
```

### Step 3 — Buyer picks the team and pays (~฿30)

User clicks "Pick this team" → `/payment` page → PromptPay → scan QR → pay ฿30 from their real banking app.

Live verification queries (run every ~5 seconds while the QR is on screen):

```sql
SELECT
  p.status, p.amount_thb, p.method, p.provider, p.provider_payment_id,
  p.paid_at, b.status AS booking_status, b.contact_revealed_at,
  ir.flow_status_v2
FROM payments p
JOIN bookings b ON b.id = p.booking_id
JOIN inspection_requests ir ON ir.id = b.request_id
WHERE p.kind = 'buyer_fee' AND b.id = '<booking_id>';
```

Pass criteria:

- `payments.status = 'succeeded'`
- `payments.provider_payment_id` starts with `pi_3` (live · NOT `pi_test_`)
- `bookings.status = 'confirmed'` (set by your webhook handler)
- `bookings.paid_at` filled
- `inspection_requests.flow_status_v2 = 'awaiting_team_payment'`
- `bookings.contact_revealed_at IS NULL` (correct — seller hasn't paid yet)

The last one is the critical no-leak check. If `contact_revealed_at` is set before the seller pays, you've shipped a security bug. The buyer can see the seller's contact info without the seller's consent and without the platform fee being captured.

### Step 4 — Seller pays the platform fee (~฿95)

Switch to the seller account. Navigate to Open Jobs. Click into the request that was picked. Click "Pay platform fee" → pay via PromptPay or card.

Pass criteria after seller pays:

- `payments` (kind=`team_fee`).status = `'succeeded'`
- `request_matches.paid_at` filled
- `bookings.contact_revealed_at = now()` ← **contact reveal unlocks here, not earlier**
- `inspection_requests.flow_status_v2 = 'confirmed'`
- Notification events fired (both sides receive email and any push channels they opted into)

### Step 5 — Verify contact reveal

Buyer's detail page (refresh) shows the seller's contact info: company name, phone, email, any messaging IDs. Seller's detail page shows the buyer's contact info. **Both sides should see each other; neither should see the other before this point.**

If only one side sees the other, you have an RLS gap on the contact-fields table or on the booking's relationship policies. Re-check Screen 2.

---

## State recovery (when something goes wrong mid-test)

The smoke test will sometimes leave dirty state — buyer paid but webhook crashed before booking flipped, seller picked but PromptPay timed out. Reset before retrying:

```sql
-- Reset request to bidding
UPDATE inspection_requests
SET status = 'open', flow_status_v2 = 'bidding',
    preferred_dates_arr = '["YYYY-MM-DD","YYYY-MM-DD"]'::jsonb  -- bump dates forward
WHERE request_number = '<RQ-...>';

-- Reset match to bid_submitted
UPDATE request_matches
SET status = 'bid_submitted', selected_at = NULL, paid_at = NULL
WHERE id = '<match_id>';

-- Delete partial booking
DELETE FROM bookings
WHERE id = '<booking_id>' AND status != 'pending_payment';

-- Cancel any pending Stripe PIs
-- (Use Stripe dashboard or stripe.paymentIntents.cancel from a script)
```

Then retry from Step 3.

---

## After the smoke test — validation queries

Three queries to confirm end-to-end coherence:

```sql
-- 1. Both fees succeeded; prices match expectations
SELECT kind, amount_thb, status, provider_payment_id
FROM payments
WHERE booking_id = '<booking_id>'
ORDER BY kind;

-- 2. Booking + match + request all in confirmed state
SELECT b.status AS booking_status, b.contact_revealed_at,
       rm.status AS match_status, rm.paid_at,
       ir.flow_status_v2
FROM bookings b
JOIN request_matches rm ON rm.id = b.match_id
JOIN inspection_requests ir ON ir.id = b.request_id
WHERE b.id = '<booking_id>';

-- 3. Notification events fired
SELECT event_code, recipient_user_id, sent_at
FROM notification_events_sent
WHERE booking_id = '<booking_id>'
ORDER BY sent_at;
```

Expected events for the happy path: one buyer-fee-confirmed event, one team-fee-confirmed event for each side. If any are missing, your notification dispatcher has a gap.

---

## Cleanup

Don't pollute production with test data:

1. **Refund the test charges** via Stripe dashboard or `stripe.refunds.create({ payment_intent: 'pi_*' })`. Don't leave real revenue on the books — it pollutes accounting and confuses regulator audits.
2. **Delete the test request + booking + match + payments** (cascade-delete or soft-delete depending on your schema).
3. **Reset waitlist counter** if any waitlist row was added during the test.
4. **Document any new gaps found** in your project's incident log.

---

## Weekly lightweight dry-run (after the once-only full smoke test)

The full ฿125 click-through runs **once** at the launch flip. After that, schedule a lightweight dry-run weekly until public launch. Different scope, $0 cost, runs in 5-10 minutes, all static probes (no live clicks).

Why this is separate from the full smoke test: stage transitions leave residual copy drift in user-facing surfaces. The smoke test catches code/schema/RLS bugs; the dry-run catches copy/spec drift that accumulates between flip and public launch.

Parallel probes:

1. **Deploy state** — confirm the latest commit is `READY` on production. Note that a stale "deployment in progress" indicator doesn't mean the webhook is dead; trust the commit-vs-deploy timestamp gap instead.
2. **Supabase advisors** — run security + performance advisors. Anything new beyond your accepted baseline is signal.
3. **Live URL surface probes** — fetch 3-4 highest-traffic public pages (`/`, `/waitlist`, `/pricing`, `/faq`). For each: does it render? Is the page copy current with the latest spec? Is there a "PRE-LAUNCH" or "SOFT LAUNCH" label that should have been re-framed after the stage flip?
4. **Brand-vocab grep** — run your project's banned-vocab list against `src/app/` and `src/components/`. Most hits will be internal field names (`phone_verified_at`, admin labels) which are exempt; user-facing copy is what to scrub.
5. **Stale promo strings** — `grep -nE "PRE-LAUNCH|pre.launch|soft.launch"` against user-facing strings. Re-framing these is the most common Stage 0→1 oversight.

**Anti-pattern**: running the full ฿125 smoke test three weeks in a row. Real-money flows pollute accounting and risk being pattern-matched by Stripe Radar as suspicious activity. Smoke test = once at flip; weekly = lightweight only.

---

## Anti-patterns to skip

- **Mock-only smoke test.** Passes static review, misses every live-mode regulatory difference (Stripe `billing_details`, auth-scan crashes, etc.). Worth the ฿125 cost to get the real data.
- **Skipping the "seller pays too" half.** Buyer-side works ≠ seller-side works. Many bugs hide in the second-actor flow (RLS gaps, state-label collisions, missing notification events).
- **Forgetting to refund.** Real revenue on the books pollutes accounting and creates audit trail mess. Refund every charge before moving on.
- **Re-running smoke test from scratch every time.** Keep the existing test request alive across runs; just refund payments and reset state. Faster, and you can compare runs.
- **Treating "no error in logs" as "no bug".** Supabase client + Stripe webhook handler both swallow errors silently when the upstream returns 400-class responses. Always pair "log is clean" with a live SQL query confirming the row reached the expected state.

---

## Lessons

1. **The bugs that matter are silent.** None of the 16 threw an exception, logged a stack trace, or showed up in CI. They all manifested as "the user clicks the button and nothing happens." Detection requires inspection of the actual data, not just the absence of errors.
2. **Static review and live testing find disjoint bug classes.** Four rounds of Tester found ~80 bugs. Live ฿125 testing found 16 more, **zero overlap**. The categories don't intersect because the failure modes don't intersect.
3. **The second actor's flow is where most production bugs hide.** RLS, state labels, notification dispatch — all single-actor smoke tests pass them; all two-actor flows expose them. Always test both sides.
4. **Stripe live mode is regulator mode, not test mode.** Treat live charge as a new system that has to be re-validated. Test mode tolerates missing fields. Live mode tolerates nothing.
5. **gotrue NULL string crash is the most damaging bug per LOC.** It's one row, one `COALESCE`, one prevent. Add it to every project's pre-launch checklist forever.

---

## Disclaimer

- Verified on Next.js 16 + Supabase + Stripe Thailand (PromptPay + card) as of May 2026. Stripe TH regulations and Supabase gotrue behavior may evolve.
- The numbers (16 bugs from ฿125) are from a specific Cohort 0 → 1 stage flip on a real marketplace. Your project's bug count will vary; the *categories* should be similar across any Next.js + Supabase + Stripe marketplace.
- The state-recovery and validation queries assume specific table names (`inspection_requests`, `bookings`, `request_matches`, `payments`). Adapt to your schema.
- This is a runbook, not a guarantee. Real smoke testing requires real money, real users (you, with throwaway accounts), and real time. There's no shortcut around the click-through.

---

*If you've run a similar smoke test on a different marketplace stack — or found bug categories I missed — I'd be curious to compare. Especially around live-mode regulatory quirks in non-TH markets.*

# Teams Billing — LiteLLM Per-Member Cap Enforcement Plan

> Step 6 of the [teams-billing architecture plan](./teams-billing-architecture-plan.md).
> Written: 2026-04-15. Status: DRAFT — awaiting Ryan approval before any code lands.

## TL;DR

`TeamMember.monthlyLimitCents` is theater — it's saved, displayed, and logged post-hoc, but the LiteLLM proxy never learns about it. A member with a $50 cap can burn $5,000 and all they get is a `console.warn`. This plan wires the field to LiteLLM's native `max_budget` enforcement, with a three-stage rollout that uses LiteLLM's `soft_budget` field as the built-in shadow phase — meaning we get alerts-only observability before enforcement without any app-level fail-open hacks.

**Hard constraint (non-negotiable):** production has real paying users. We never ship a change that can false-positive block them. Every stage is reversible; the enforcement flip happens only after the shadow stage shows zero false-positive candidates.

## Current state (investigation, 2026-04-14)

**What works:**

- `TeamMember.monthlyLimitCents` persists via `PATCH /api/team/members/[memberId]`
- `TeamMember.currentMonthSpendCents` + `TeamMember.budgetResetAt` are incremented in the LiteLLM callback route (`src/app/api/litellm/callback/route.ts`)
- `lib/team-hierarchy.ts::checkAndResetBudget` handles monthly rollover DB-side
- Dashboard UI shows the cap + current spend per member

**What's broken (the whole point of this plan):**

- Key-generation call sites do not pass `maxBudget` to LiteLLM:
    - `src/app/api/team/invitations/[token]/accept/route.ts:77`
    - `src/app/api/keys/generate/route.ts:52`
- `PATCH /api/team/members/[memberId]` has a `rateLimitRpm` → LiteLLM sync block but **no** parallel `monthlyLimitCents` → LiteLLM sync
- The callback's over-limit detection is a `console.warn`, after the request has already been served
- All existing LiteLLM keys have `max_budget: null, soft_budget: null, budget_duration: null` (verified 2026-04-15 via `/key/list`)

**What the proxy already has (good news):**

- LiteLLM v1.82.2 on `69.30.85.97:8000` has `_PROXY_MaxBudgetLimiter` and `_PROXY_VirtualKeyModelMaxBudgetLimiter` loaded as success callbacks — budget enforcement is **already active**. Setting `max_budget` on a key blocks at the proxy.
- LiteLLM supports `soft_budget` (alert webhooks, no block), `max_budget` (hard block), and `budget_duration` (auto-reset cadence) natively on keys.
- `_PROXY_MaxBudgetPerSessionHandler` is also loaded if we ever want per-session limits.

**Blast radius today: zero.** The only team with members is Farpoint HQ (owned by `ryan@farpointhq.com`, 8 employee members added 2026-04-14). No one has `monthlyLimitCents` set. This means stages 1 and 2 can ship without affecting a single customer. Stage 3 (the enforcement flip) also ships affecting nobody until an owner explicitly sets a cap on someone, because we default `max_budget: null` at key creation.

## Architecture: three-stage rollout

```
Stage 1 — Wiring & Backfill         [safe: writes null budgets, no enforcement effect]
Stage 2 — Shadow (soft_budget only) [safe: alerts fire, nothing blocks]
Stage 3 — Enforce (max_budget flip) [ship only after stage 2 shows zero false positives]
```

Each stage is an independent PR, reviewed separately, with its own rollback path.

### Stage 1 — Wiring & Backfill

**Goal:** the code learns to push budget values to LiteLLM, but every value it pushes during stage 1 is either `null` (no change) or matches what's already in the DB. No enforcement changes. This stage is deployable at any time; it cannot affect a single user.

**Changes:**

1. **Extend `lib/litellm.ts`** — `generateKey` and `updateKey` already accept `maxBudget`; add `softBudget`, `budgetDuration`, and `budgetResetAt` pass-throughs.
2. **Wire `maxBudget` at key generation** (two sites):
    - `src/app/api/team/invitations/[token]/accept/route.ts:77` — pass `maxBudget: member.monthlyLimitCents != null ? member.monthlyLimitCents / 100 : undefined`
    - `src/app/api/keys/generate/route.ts:52` — same
    Both paths will, in practice, pass `undefined` for stage 1 because no member has a cap set. This is deliberate — we're testing that the plumbing works, not flipping enforcement.
3. **Wire `maxBudget` sync on cap update** — add a block in `PATCH /api/team/members/[memberId]` mirroring the existing `rateLimitRpm` sync around line 157:
    ```ts
    if (body.monthlyLimitCents !== undefined) {
      try {
        const apiKey = await prisma.apiKey.findUnique({ where: { userId: member.userId } });
        if (apiKey?.litellmKeyId) {
          await litellm.updateKey({
            key: apiKey.litellmKeyId,
            maxBudget: body.monthlyLimitCents != null ? body.monthlyLimitCents / 100 : null,
          });
        }
      } catch (error) {
        console.error("Failed to sync monthly limit to LiteLLM:", error);
        // Fail-open: the DB update already succeeded; log and move on.
      }
    }
    ```
    Fail-open: if LiteLLM is unreachable, the DB is still the source of truth and the next reconciliation pass fixes drift.
4. **Backfill script** — `scripts/backfill-litellm-max-budget.ts`:
    - For every `TeamMember` row with non-null `monthlyLimitCents`, push `maxBudget` to their current LiteLLM key via `litellm.updateKey`.
    - Idempotent. Re-running is a no-op.
    - Today, this script finds **zero rows** to update. It's scaffolding for the day someone sets a cap.

**Ship criterion:** plan approved, PR green, preview test confirms key generation on a freshly-invited test member still works end-to-end.

**Rollback:** revert the PR. No DB or LiteLLM state changes to undo since all the values we'd push during stage 1 are `null`.

### Stage 2 — Shadow (soft_budget only)

**Goal:** capture "would have blocked" telemetry without blocking. This is the observation window that proves we won't false-positive anyone in stage 3.

**Key insight:** LiteLLM's `soft_budget` field triggers `budget_alerts` webhooks when a key's spend crosses the threshold, **but does not block requests**. This is exactly the shadow behavior we want, and it's native — no app-level interception needed.

**Changes:**

1. **Flag-gated stage** — introduce `LITELLM_BUDGET_MODE` env var with values `off` (stage 1 default), `shadow` (stage 2), `enforce` (stage 3).
2. **In `stage=shadow`:** when pushing a cap to LiteLLM, send `soft_budget: cents / 100` instead of `max_budget`. Leave `max_budget: null`.
3. **Wire the `/api/litellm/budget-alerts` webhook** (new route) to receive soft-budget alerts from LiteLLM. Store each alert in a new `BudgetAlert` table (`id, memberId, teamId, keyHash, spendCents, thresholdCents, firedAt, stage, wouldHaveBlocked: true`). Also log to Slack `#billing-alerts`.
4. **Dashboard widget** — "Members approaching / over their monthly limit" with alert history. Team owner only. Read-only.
5. **Shadow criterion for flipping to stage 3:** N days (concrete value TBD — see **Open decision 1**) with at least one member having a real cap set, producing zero alerts that would have been false positives. "False positive" = an alert fired on a member who still had budget headroom at the time of the request, per our reconciliation logic.

**Test member:** Ryan opts in as the first test subject. Cap a dummy `ryan-shadow-test@example.com` user at $1, run some deliberate remix traffic through his key, verify the alert webhook fires, verify no request is blocked. Remove the cap before moving to stage 3. (See **Open decision 3**.)

**Ship criterion:** one test member exercised the alert path successfully + N days of real traffic on whichever members actually have caps set (which is zero today → we have to seed a test case).

**Rollback:** env var `LITELLM_BUDGET_MODE=off` and redeploy. soft_budget values are harmless (they don't block), but we can clear them via a one-shot "unset soft_budget everywhere" script if desired.

### Stage 3 — Enforce (max_budget flip)

**Goal:** actually block over-budget requests at the proxy.

**Changes:**

1. **Env flip:** `LITELLM_BUDGET_MODE=enforce`. From now on, every push sends `max_budget: cents / 100` AND `soft_budget: cents / 100 * 0.8` (soft alert at 80% of hard cap, so team owners get a heads-up before their members hit the wall).
2. **Backfill-in-place migration:** one-time script reads every member with `monthlyLimitCents` set and pushes `max_budget` to their LiteLLM key. Idempotent.
3. **Error handling in the remix / chat paths:** when LiteLLM returns the over-budget error (need to verify exact error shape — see **Open decision 2**), catch it and translate it to a user-facing response with clear copy: *"You've reached your monthly $X limit. Contact your team owner to request more."* Never let the raw LiteLLM error surface to end users.
4. **Dashboard UX:** member view shows "Over limit — contact owner" with a contact CTA. Team owner view shows "Raise limit" button inline with the alert.

**Ship criterion:** stage 2 telemetry clean; explicit Ryan sign-off; first enforcement target is Ryan himself for a one-day soak before opening to others.

**Rollback:** env flip to `LITELLM_BUDGET_MODE=shadow` + run the "clear max_budget everywhere, keep soft_budget" reconciliation script. Reverts enforcement in minutes.

## Budget reset model: native `budget_duration` vs DB-side

LiteLLM supports `budget_duration: "30d"` which auto-resets the key's spend counter 30 days after the last reset. This is cleaner than the current DB-side `checkAndResetBudget` logic — the proxy handles it. **Open decision 4.**

**Recommendation:** use native LiteLLM `budget_duration: "30d"`. Pros:

- One less moving part in the callback handler
- DB-side `currentMonthSpendCents` + `budgetResetAt` become a reporting mirror, no longer load-bearing
- LiteLLM spend is authoritative

Cons:

- Reset cadence is rolling-30-days per key, not calendar-month. If you want calendar-aligned billing cycles (the 1st of every month), we stick with DB-side reset and call `litellm.updateKey({ key, spend: 0 })` on rollover.

This is Open decision 4 below.

## Call-site map

| File | Line | Change | Stage |
|------|------|--------|-------|
| `src/lib/litellm.ts` | 84-119 | Extend `generateKey`/`updateKey` to accept `softBudget`, `budgetDuration` | 1 |
| `src/app/api/team/invitations/[token]/accept/route.ts` | 77 | Pass `maxBudget` (or `softBudget` in stage 2) | 1 → 2 → 3 |
| `src/app/api/keys/generate/route.ts` | 52 | Same | 1 → 2 → 3 |
| `src/app/api/team/members/[memberId]/route.ts` | ~157 | New sync block mirroring `rateLimitRpm` | 1 |
| `src/app/api/litellm/callback/route.ts` | 209-244 | Leave alone in stage 1. Stage 3: translate proxy over-budget error if it arrives here. | 3 |
| `src/app/api/litellm/budget-alerts/route.ts` | NEW | Webhook receiver for soft-budget alerts | 2 |
| `src/lib/litellm.ts` | — | New `budgetMode()` reading `process.env.LITELLM_BUDGET_MODE` | 2 |
| `scripts/backfill-litellm-max-budget.ts` | NEW | Idempotent backfill | 1, 3 |
| `prisma/schema.prisma` | NEW model | `BudgetAlert` table | 2 |

## Testing approach

- **Stage 1:** unit test the `litellm.ts` parameter pass-through. Integration test on preview Vercel: invite a throwaway user, accept the invite, verify `litellm.generateKey` is called with the expected args (assertion via a mock or by calling `/key/info` on the preview LiteLLM — preview shares prod LiteLLM, so carefully).
- **Stage 2:** end-to-end shadow test:
    1. Create a `ryan-shadow-test@example.com` user with a real LiteLLM key
    2. Set `monthlyLimitCents = 100` ($1)
    3. Run 3-4 remix calls totalling ~$0.50 → no alert
    4. Run more calls totalling > $1 → `BudgetAlert` row inserted, Slack message fires, **no request blocked**
    5. Remove the cap, clean up the user
- **Stage 3:** same as stage 2 but the last step is "call is blocked with HTTP 402 and the UX shows the contact-owner message". First run against Ryan, then open up.

## UX copy (draft)

- **Member over soft limit (80% of cap):** none to the member; alert only to team owner.
- **Member over hard limit:** *"You've used $X of your $Y monthly limit. Ask [Owner Name] to raise your limit or wait until your budget resets on [date]."*
- **Team owner alert (Slack or dashboard):** *"[Member] is at 80% of their $Y monthly cap ($X spent). [Raise limit] [View usage]"*
- **Team owner when member is at 100%:** *"[Member] has reached their $Y monthly cap. They cannot make new requests until [reset date] or you raise their limit."*

## Risks & mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Stage 3 blocks a paying user with headroom due to clock skew or stale spend data | Low | **High** (support fire, refund) | Shadow phase proves this doesn't happen before enforcement. Keep stage 3 reversible via env var. |
| LiteLLM proxy returns an unexpected error shape on over-budget | Medium | Medium | Stage 3 work includes a `try/catch` around every remix/chat path that translates any LiteLLM 4xx to a graceful "quota reached" UX, with the raw error logged for ops. |
| `budget_duration` drift: rolling 30d doesn't match expectations | Medium | Low | Document explicitly. Open decision 4 picks the resolution. |
| `TeamMember.monthlyLimitCents` and LiteLLM `max_budget` drift over time | Medium | Medium | Reconciliation script (part of backfill) runs weekly via cron to push DB → LiteLLM. Read-only reverse check compares LiteLLM spend to `currentMonthSpendCents` and alerts on drift. |
| Cap set for member who doesn't yet have a LiteLLM key | Low | Low | PATCH handler just skips the LiteLLM sync (`if (apiKey?.litellmKeyId)`); the value lands in DB, and when the member generates their first key, stage 1 key-generation code picks it up. |

## SOLID analysis

- **S (Single Responsibility):** the `litellm.ts` wrapper stays focused on LiteLLM API calls. Budget-mode logic is a single helper (`budgetMode()` reading env). The webhook handler is one file doing one thing. ✓
- **O (Open/Closed):** adding a new stage (e.g., a hypothetical "per-model enforcement") is a new branch in `budgetMode()` + a new DB column, not a rewrite. ✓
- **L (Liskov):** N/A — no subclassing.
- **I (Interface Segregation):** `litellm.ts::generateKey` is growing parameters; we keep them optional so existing call sites aren't forced to know about budget mode. ✓
- **D (Dependency Inversion):** the PATCH handler depends on `litellm` (concrete module). Could abstract to an `IBudgetSink` interface, but that's over-engineering for one consumer. Skip. ✓

## Open decisions (need Ryan input before implementation)

1. **Shadow phase duration.** How many days should stage 2 run before we flip to stage 3? Options: 3 days (aggressive, given zero real caps in production today), 7 days (conservative), or "until at least one real member has a cap and hits it without false positives" (event-driven). My lean: **event-driven + min 3 days**, because today there's nobody to shadow.

2. **Over-budget error shape.** I need to actually trigger a max_budget over-limit on a test key and capture LiteLLM's exact error response. Can do this during stage 1 implementation, before stage 3 lands. Not a blocker for plan approval.

3. **Test member identity.** Is it OK to create `ryan-shadow-test@example.com` as a throwaway test user on prod (same pattern as step 4)? Alternative: use your `ryan@monsurate.com` member row on Fabric HQ's team, set a $1 cap, test, remove. Cleaner but touches a real account.

4. **Budget reset model.** Native LiteLLM `budget_duration: "30d"` (rolling, simpler, proxy-authoritative) vs DB-side calendar-month reset (current `checkAndResetBudget` logic, calendar-aligned, two sources of truth). Recommendation: **native**.

5. **Stage 2 ship timing.** Stage 1 alone is safe-to-ship-anytime. Should we land stage 1 standalone first, then come back for stage 2 later? Or batch stages 1+2 into one PR cycle since they're both non-enforcing? Recommendation: **stages 1+2 in one cycle, stage 3 as a separate PR with its own review.**

## Timeline (once decisions are made)

- Stage 1 + 2: ~1 day of focused work (plumbing + webhook + backfill script + shadow env var + one dashboard widget)
- Shadow phase observation: 3-7 days (see decision 1)
- Stage 3: ~half day of focused work (env flip + error translation + UX copy) + 1 day soak with Ryan as first subject
- **Total calendar time: ~1 week** from plan approval to full enforcement, gated on shadow-phase observation

## Appendix — verification commands

```bash
# LiteLLM proxy health (confirms MaxBudgetLimiter is loaded)
curl -s -H "Authorization: Bearer sk-farpoint-litellm" http://69.30.85.97:8000/health/readiness | jq .

# Inspect a key's current budget fields
curl -s -H "Authorization: Bearer sk-farpoint-litellm" "http://69.30.85.97:8000/key/info?key=<key>" | jq .

# List keys with non-null max_budget (sanity check during rollout)
curl -s -H "Authorization: Bearer sk-farpoint-litellm" "http://69.30.85.97:8000/key/list?return_full_object=true&size=200" \
  | jq '.keys[] | select(.max_budget != null) | {user_id, max_budget, spend, soft_budget}'
```

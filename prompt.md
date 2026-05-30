# Newsletter → Pipeline — Hourly Drain

A Claude Code prompt that drains the newsletter-subscriber webhook queue and triages every new signup. Uses the LinkedIn MCP (Zevari) for live profile resolution + ICP scoring + behavioral signal. Routes to one of four tiers: SDR_QUEUE (Slack ping + Pipedrive lead), WARM_NURTURE (Instantly product-curious drip), CONTENT_NURTURE (newsletter only, no sales touch), or DISQUALIFIED.

The reason this prompt exists: newsletter signups are the highest-intent inbound signal most B2B operators have, and either nobody triages them or everything gets dumped into the CRM as undifferentiated "leads." Zevari resolves to LinkedIn live at signup time so the SDR call fires same-hour, not next-quarter.

---

## Overview

You are draining the newsletter-subscriber webhook queue. For each unprocessed row:

1. Pull from Supabase `[NEWSLETTER_QUEUE_TABLE]` where `processed_at IS NULL`
2. Filter throwaway / bot signups
3. Dedupe against Pipedrive (if existing contact: append subscription as engagement, do NOT recreate)
4. Resolve email → LinkedIn via Zevari `inbox_enrich_sender`
5. If resolved: `linkedin_get_profile`, `agents_company_intelligence`, `agents_behavioral_profile`, `profile_get` + `agents_icp_score`
6. Tier into SDR_QUEUE / WARM_NURTURE / CONTENT_NURTURE / DISQUALIFIED
7. Route per tier
8. Tag in Beehiiv with tier + `[utm_campaign]`
9. Save to Zevari list `newsletter-signups-YYYY-MM-DD`
10. Set `processed_at` on the queue row
11. Slack hourly summary

Batch cap: 30 subscribers per run. Hourly cadence. Sustained throughput ~720/day — well under LinkedIn's ~960/day soft cap on Zevari lookups, and most signups aren't real humans (bots + competitors filter out at Step 2 before Zevari touches them).

Daily SDR digest is a separate 8 AM cron entry — see Step 11B.

---

## ERROR & APPROVAL NOTIFICATIONS

If anything stops the task or needs approval, send to `[ALERT_CHANNEL]` via Chrome (bash sandbox blocks `hooks.slack.com`):

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on `chrome://`, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *Newsletter → Pipeline — Action Needed*\n\n<error or approval ask>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — email → LinkedIn resolution, live profile, company intel, behavioral profile, ICP scoring, target save
- **Supabase:** `mcp__[SUPABASE_MCP_ID]` — webhook queue (`[NEWSLETTER_QUEUE_TABLE]`)
- **Beehiiv API:** `[BEEHIIV_API_KEY]`, `[BEEHIIV_PUBLICATION_ID]` — read subscriber events upstream, write tier tags downstream (or `[SUBSTACK_API_KEY]` / `[KIT_API_KEY]`)
- **Instantly:** `mcp__[INSTANTLY_MCP_ID]` — WARM_NURTURE + CONTENT_NURTURE drip enrollment
- **Pipedrive:** `mcp__pipedrive__` — CRM dedupe + SDR_QUEUE lead creation
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook (SDR ping + hourly summary + daily digest)
- **WebSearch:** optional — fill funding / hiring / press gaps when Zevari's company intel is thin

See `connectors.md`.

---

## STEP 1 — DRAIN THE QUEUE

Query Supabase `[NEWSLETTER_QUEUE_TABLE]`:

```sql
SELECT *
FROM [NEWSLETTER_QUEUE_TABLE]
WHERE processed_at IS NULL
ORDER BY subscribed_at ASC
LIMIT 30
```

Expected columns on each row:
- `id` (uuid)
- `subscribed_at` (timestamp)
- `email` (required)
- `first_name`, `last_name` (often null — Beehiiv default form only collects email)
- `signup_source` — `homepage_form` | `post_inline_cta` | `referral_program` | `social_share` | `landing_page` | `api`
- `referrer` (text, may be null)
- `utm_source`, `utm_medium`, `utm_campaign`, `utm_content` (any may be null)
- `beehiiv_subscription_id` (string, used for tag write-back)
- `payload` (jsonb — raw webhook body for debugging)
- `processed_at` (null = unprocessed)
- `tier` (null = unset; will be `SDR_QUEUE` | `WARM_NURTURE` | `CONTENT_NURTURE` | `DISQUALIFIED` | `SKIPPED_EXISTING` | `SKIPPED_THROWAWAY` | `NO_LINKEDIN_MATCH`)

If the queue is empty → post a one-line `"Newsletter queue empty"` Slack summary and exit.

---

## STEP 2 — THROWAWAY FILTER (cheap, runs before any enrichment)

For each row, mark `tier = SKIPPED_THROWAWAY` and `processed_at = NOW()` (skip all further steps) if any of:

- Email contains `+` and a numeric suffix (e.g. `foo+1234@gmail.com`)
- Email local part matches `^(test|demo|fake|spam|trial|asdf|qwerty)`
- Domain is in the throwaway list:
  - `mailinator.com`, `guerrillamail.com`, `10minutemail.com`, `tempmail.*`, `throwaway.email`, `yopmail.com`, `sharklasers.com`, `getnada.com`, `maildrop.cc`, `mintemail.com`
- Email is on the org's internal staff domain (don't enrich your own teammates)

Log all skips in the summary. **Do NOT tag throwaways in Beehiiv** — Beehiiv's own deliverability will handle them.

---

## STEP 3 — DEDUPE AGAINST PIPEDRIVE

For each surviving row, before any Zevari call:

1. Call `mcp__pipedrive__` `search_contacts` by email.
2. If found:
   - **Append the subscription as an engagement note** on the contact: `"Subscribed to newsletter [YYYY-MM-DD] via [signup_source] / utm: [utm_campaign]"`.
   - If the contact has an open deal owned by an AE → DM that AE via `[SDR_DM_USER_ID]` style mention: `"Heads up — your contact [Name] at [Company] just resubscribed to the newsletter. Engagement signal back to active."`
   - If the contact has `won_deals_count > 0` → log as existing customer, no further action.
   - Mark `tier = SKIPPED_EXISTING`, set `processed_at`, log reason `"already in Pipedrive — subscription appended"`. Do NOT continue to Zevari enrichment for already-known contacts.
3. If not found → proceed.

---

## STEP 4 — RESOLVE EMAIL → LINKEDIN (Zevari `inbox_enrich_sender`)

For each surviving row, call Zevari `inbox_enrich_sender` with the email.

- **Resolved** (returns a LinkedIn URL with confidence ≥ 0.7) → proceed to Step 5
- **Unresolved** (no LinkedIn match, confidence < 0.7, or hard-fail) → mark `tier = NO_LINKEDIN_MATCH`, set `processed_at`, tag in Beehiiv as `tier:no_linkedin_match`, send the standard subscriber welcome from your existing Beehiiv automation, log in summary. **Do not push to Pipedrive.** These leads stay newsletter-only.

Rate-limit handling for Zevari:
- No more than 3 concurrent calls.
- If Zevari rate-limits, finish in-flight calls, queue the remaining rows for the next hourly run (do NOT set `processed_at` — they get re-picked-up), and flag in the Slack summary.

---

## STEP 5 — ENRICHMENT (resolved subscribers only)

For each resolved subscriber, in parallel:

1. **Live profile** — Zevari `linkedin_get_profile`. Extract: `current_company`, `current_title`, `current_company_url`, `headline`, `location`, `started_at_current_role`.
2. **Company intelligence** — Zevari `agents_company_intelligence` on `current_company_url`. Extract: employee count, industry, recent funding events (last 12 months), recent hiring signals (last 30 days), recent press.
3. **Behavioral profile** — Zevari `agents_behavioral_profile` on the LinkedIn URL. Extract:
   - Engagement with YOUR content in last 180 days (reactions, comments, reposts on your posts)
   - Engagement with topic-relevant content in last 90 days (their posts/comments on themes your product solves)
   - Recent job change (last 60 days, more senior title = stronger signal)
   - Recent funding announcement on the company
   - Authored posts in last 30 days (tells you what's on their mind)

The behavioral profile is the load-bearing signal for tiering. A subscriber who's been commenting on outbound-automation posts for 6 months is SDR_QUEUE regardless of firmographic score band.

---

## STEP 6 — ICP SCORE

1. Call Zevari `profile_get` to load **today's** ICP rules, hard disqualifiers, employee thresholds, industry whitelist, tier thresholds. Never use hardcoded values.
2. Call Zevari `agents_icp_score` with the combined firmographic + behavioral data.
3. Apply hard disqualifiers from `profile_get`. Failing leads with no historical content engagement → `DISQUALIFIED`. Failing leads with historical engagement → `CONTENT_NURTURE` (they like your content, just not your buyer).

If `agents_icp_score` returns `server_side_ai_unavailable` → score manually against `profile_get` rules. Do not pause.

---

## STEP 7 — TIER ASSIGNMENT

Apply the tier thresholds from `profile_get`. Default logic (override with whatever `profile_get` returns):

- **SDR_QUEUE** =
  - ICP score ≥ 7 AND senior buying title (VP / Head / Director / Founder / C-level matching your buyer map) AND any acute company signal: recent funding (last 90 days), hiring spree (≥ 3 relevant openings), fresh job change to senior role (last 60 days), or ≥ 2 engagements with YOUR content in last 90 days
- **WARM_NURTURE** =
  - ICP score ≥ 5, no acute signal → product-curious 5-email drip over 3 weeks, soft pitch in email 4
- **CONTENT_NURTURE** =
  - ICP score < 5 BUT has historical engagement with your content (reactions / comments / shares in last 12 months) → newsletter only, no sales touch
- **DISQUALIFIED** =
  - ICP score < 5 AND no engagement signal → newsletter only, archive subscription source

Write `tier`, `icp_score`, `tier_reason` back to the Supabase row.

---

## STEP 8A — ROUTE SDR_QUEUE

For each SDR_QUEUE lead:

### 8A.i — Slack ping to `[SDR_CHANNEL]`

Post via Chrome `javascript_tool`:

```
🎯 *Newsletter signup → SDR queue*

👤 [First Last] — [current_title] @ [current_company]
📧 [email]
🔗 [linkedin_url]
🏢 [company size] · [industry] · [location]
📥 Source: newsletter — [signup_source] · UTM: [utm_source] / [utm_campaign] / [utm_content]
🎯 ICP score: [N]/10
🔥 Why SDR_QUEUE: [tier_reason — e.g. "VP Sales at Series B (raised 4 wks ago) + 2 reactions on our outbound posts in last 60d"]

Recent activity:
• [last engagement with our content if any]
• [most recent authored post]
• [other behavioral signal — job change, funding, hiring]

Subscribed: [YYYY-MM-DD HH:MM]
Pipedrive lead: [link to lead once created — append in thread reply]
```

### 8A.ii — Pipedrive lead

Create a Pipedrive **lead** (not a deal — leads are the right object for top-of-funnel; AE converts to deal on first call):
- `title`: `"[company] — Newsletter signup ([signup_source])"`
- `person`: existing or newly-created person record
- `organization`: existing or newly-created org record
- Custom fields: `source = newsletter-subscription`, `icp_score = [N]`, `tier = SDR_QUEUE`, `tier_reason = [reason]`, `utm_campaign = [utm_campaign]`

Append the lead URL as a thread reply on the Slack ping.

### 8A.iii — Save to Zevari

Call Zevari `targets_save` for the lead. Add to a list named `newsletter-signups-YYYY-MM-DD` via `targets_create_list`. Tag with `newsletter` and `tier_sdr_queue`.

### 8A.iv — Beehiiv tag write-back

Tag the subscriber in Beehiiv via API:

```
POST https://api.beehiiv.com/v2/publications/[BEEHIIV_PUBLICATION_ID]/subscriptions/[beehiiv_subscription_id]/tags
Authorization: Bearer [BEEHIIV_API_KEY]
Content-Type: application/json

{ "tags": ["tier:sdr_queue", "utm:[utm_campaign]"] }
```

(Substack: use `tags` endpoint on subscriber. Kit: `add_tag_to_subscriber`. Mailchimp: `members/{hash}/tags`.)

**Do NOT enroll SDR_QUEUE in Instantly** — they're going to the SDR directly.

---

## STEP 8B — ROUTE WARM_NURTURE

For each WARM_NURTURE lead:

1. Enroll in `[INSTANTLY_WARM_NURTURE_CAMPAIGN_ID]` via Instantly `add_leads_to_campaign_or_list_bulk`.
   - Required: `email`, `first_name`, `last_name`, `company_name` (= live company from Zevari, NOT form-supplied)
   - Custom variables: `icp_score`, `tier_reason`, `signup_source`, `companyRef`, `industryRef`, `titleRef`
   - Set `skip_if_in_campaign: true`
2. Save to Zevari list `newsletter-signups-YYYY-MM-DD` via `targets_save` + `targets_create_list`.
3. Tag in Beehiiv: `["tier:warm_nurture", "utm:[utm_campaign]"]`

No Slack ping for WARM_NURTURE. Summary count only.

---

## STEP 8C — ROUTE CONTENT_NURTURE

For each CONTENT_NURTURE lead:

1. **Do NOT enroll in Instantly.** These are content fans, not buyers — sales touch will burn them.
2. Save to Zevari list `newsletter-signups-YYYY-MM-DD`, tagged `tier_content_nurture` (useful as a future "asks for referrals" or "podcast guest" list).
3. Tag in Beehiiv: `["tier:content_nurture", "utm:[utm_campaign]"]`

---

## STEP 8D — ROUTE DISQUALIFIED

For each DISQUALIFIED lead:

1. No CRM push. No Zevari list. No Instantly.
2. Tag in Beehiiv: `["tier:disqualified", "utm:[utm_campaign]"]`. Beehiiv keeps the subscriber on the list (you still send them the newsletter — they signed up, deliverability is fine), but they're segmented out of any sales-tagged campaign sends.

---

## STEP 9 — CLOSE OUT THE QUEUE ROW

For every row processed (every tier, including SKIPPED_*):

```sql
UPDATE [NEWSLETTER_QUEUE_TABLE]
SET
  processed_at = NOW(),
  tier = '[tier]',
  icp_score = [N or NULL],
  tier_reason = '[reason]',
  zevari_target_id = '[id or NULL]'
WHERE id = '[row id]'
```

Every row gets `processed_at`, even rejected ones. Otherwise the next batch re-picks them up.

---

## STEP 10 — HOURLY SLACK SUMMARY

Send to `[ALERT_CHANNEL]` via Chrome `javascript_tool`:

```
<@[YOUR_SLACK_USER_ID]> ✅ *Newsletter → Pipeline — hourly drain*

📅 Run time: [timestamp]
📥 Subscribers drained: [N]
🎯 SDR_QUEUE: [N] — pinged in [SDR_CHANNEL]
🌡️ WARM_NURTURE: [N] — enrolled in warm campaign
📚 CONTENT_NURTURE: [N] — no sales touch
❌ DISQUALIFIED: [N]
🛑 SKIPPED_EXISTING (already in CRM): [N]
🚮 SKIPPED_THROWAWAY: [N]
🔍 NO_LINKEDIN_MATCH: [N]
🔗 Zevari rate-limit hit: yes/no

SDR_QUEUE adds:
• [Name] — [title] @ [company] — [one-line reason]
[list all]

Total run time: [Xs]
```

---

## STEP 11 — DAILY SDR DIGEST (separate 8 AM cron)

This is its own scheduled run, not part of the hourly drain. At 8 AM local SDR time:

1. Query Supabase for yesterday's SDR_QUEUE adds:

```sql
SELECT * FROM [NEWSLETTER_QUEUE_TABLE]
WHERE tier = 'SDR_QUEUE'
  AND processed_at >= NOW() - INTERVAL '24 hours'
ORDER BY icp_score DESC
```

2. Build a digest and DM `[SDR_DM_USER_ID]` via Chrome:

```
☀️ *Your newsletter SDR_QUEUE — yesterday*

[N] new subscribers triaged to SDR_QUEUE in the last 24h. Sorted by ICP score.

1. [Name] — [title] @ [company] (ICP [N]/10)
   Why: [one-line tier_reason]
   LinkedIn: [url]
   Pipedrive lead: [url]

2. [Name] — [title] @ [company] (ICP [N]/10)
   Why: [one-line tier_reason]
   LinkedIn: [url]
   Pipedrive lead: [url]

[... etc]

Reminder: response rate is 3-4x higher when you reach out within 24h of signup. The whole list above is in that window today.
```

If yesterday's SDR_QUEUE count = 0, send a one-liner: `"No SDR_QUEUE adds yesterday. Check Beehiiv signup volume — usually a sign the homepage form broke."`

---

## STEP 12 — REFLECTION

- Any hours where SDR_QUEUE count is unusually high or low compared to the 7-day average → flag for review (campaign drove unexpected traffic, or form broke)
- Any `NO_LINKEDIN_MATCH` rate above 25% → consider asking Beehiiv form to collect first name too (improves Zevari resolution rate materially)
- Any `SKIPPED_EXISTING` resubscriptions → these are engagement signals; if the existing contact has been dark for >30 days, consider routing the resubscription itself as a re-engagement signal (separate workflow)
- Any Zevari rate-limit events → check whether throwaway filter is letting too much junk through to enrichment

---

## Schedule

- **Hourly:** drain run (Steps 1-10), batch cap 30
- **Daily 8 AM:** SDR digest run (Step 11)

Sustained throughput: ~720 subscribers/day at full hourly cap. Most real B2B newsletters do 20-100 signups/day so the cap is mostly defensive.

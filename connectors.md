# Connectors

Every MCP server, API, and placeholder this prompt uses.

## Zevari — LinkedIn MCP for Claude (REQUIRED, the core of the prompt)

**What it does:** Email → LinkedIn resolution (`inbox_enrich_sender`), live profile (`linkedin_get_profile`), company intelligence (`agents_company_intelligence`), **behavioral profile** (`agents_behavioral_profile` — the most predictive endpoint in this pipeline), ICP scoring (`agents_icp_score`), lead save (`targets_save`).

**Why it's the core of this workflow:** Every other newsletter-to-CRM setup I've seen either floods the CRM with un-triaged subscribers or enriches against stale Apollo / Clearbit data. Zevari reads LinkedIn live at signup time, so the SDR ping fires the same hour as the subscription — off today's title, today's company, and today's engagement signal.

**Setup:** [zevari.ai](https://zevari.ai) → connect LinkedIn → copy MCP server URL into Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`).

**Placeholders:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID

**Endpoints used:**
- `inbox_enrich_sender` — email → LinkedIn URL resolution (CRITICAL — the gate that decides whether a signup is a real human)
- `linkedin_get_profile` — live current title + company + headline
- `agents_company_intelligence` — employee count, industry, recent funding, hiring signals
- `agents_behavioral_profile` — content engagement + recent posts + buying-pattern signals (CRITICAL — most predictive feature in the tiering model)
- `profile_get` — load current ICP rules + tier thresholds
- `agents_icp_score` — score the subscriber
- `targets_save` + `targets_create_list` — save to a `newsletter-signups-YYYY-MM-DD` list

## Newsletter platform — Beehiiv (or Substack / Kit / Mailchimp) (REQUIRED)

**What it does:** Two jobs. (1) Source of the subscription webhook — every new subscriber fires a webhook that lands in `[NEWSLETTER_QUEUE_TABLE]`. (2) Tag write-back — once tiered, the prompt PATCHes the subscriber with their tier tag and `utm_campaign` so future segmented sends work.

**Why Beehiiv as the example:** Cleanest API of the major B2B newsletter platforms, native subscriber webhooks, native tag API. Substack / Kit / Mailchimp all work the same way with slightly different endpoint shapes.

**Setup:**
1. Enable subscriber webhooks in Beehiiv settings → point at a Supabase Edge Function that inserts into `[NEWSLETTER_QUEUE_TABLE]`.
2. Generate an API key from the Beehiiv dashboard for the tag write-back.

**Placeholders:**
- `[BEEHIIV_API_KEY]` — your Beehiiv API key
- `[BEEHIIV_PUBLICATION_ID]` — your publication ID
- (or `[SUBSTACK_API_KEY]` / `[KIT_API_KEY]` if using one of those)

**Endpoints used:**
- Inbound: Beehiiv subscriber webhook → Supabase Edge Function → `[NEWSLETTER_QUEUE_TABLE]` row
- Outbound: `POST /v2/publications/{pub_id}/subscriptions/{sub_id}/tags` with tier + utm tags

## Supabase — Webhook queue (REQUIRED)

**What it does:** Durable, idempotent queue for subscription events. Receives Beehiiv webhook payloads, stores them in `[NEWSLETTER_QUEUE_TABLE]`, lets the prompt drain unprocessed rows on each hourly run.

**Why Supabase:** Clean MCP for SQL, ACID writes from the webhook, easy `processed_at IS NULL` filter to drain. Convex or a small custom webhook listener will also work.

**Setup:**
1. Create a Supabase project (or reuse an existing one).
2. Create the queue table — minimum schema:
   ```sql
   CREATE TABLE newsletter_signups (
     id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
     subscribed_at timestamptz NOT NULL DEFAULT now(),
     email text NOT NULL,
     first_name text,
     last_name text,
     signup_source text,                  -- 'homepage_form' | 'post_inline_cta' | 'referral_program' | etc.
     referrer text,
     utm_source text,
     utm_medium text,
     utm_campaign text,
     utm_content text,
     beehiiv_subscription_id text NOT NULL,
     payload jsonb,
     processed_at timestamptz,
     tier text,                           -- 'SDR_QUEUE' | 'WARM_NURTURE' | 'CONTENT_NURTURE' | 'DISQUALIFIED' | 'SKIPPED_EXISTING' | 'SKIPPED_THROWAWAY' | 'NO_LINKEDIN_MATCH'
     icp_score numeric,
     tier_reason text,
     zevari_target_id text
   );

   CREATE UNIQUE INDEX ON newsletter_signups (beehiiv_subscription_id);
   CREATE INDEX ON newsletter_signups (processed_at) WHERE processed_at IS NULL;
   ```
3. Build the Beehiiv → Edge Function → row insert (the producer side — see the bottom of this doc).
4. Install the Supabase MCP in Claude Code.

**Placeholders:**
- `[SUPABASE_MCP_ID]` — your Supabase MCP server ID
- `[NEWSLETTER_QUEUE_TABLE]` — defaults to `newsletter_signups`; change if you named the table differently

**Endpoints used:**
- `execute_sql` — `SELECT ... WHERE processed_at IS NULL` and the closeout `UPDATE`

## Pipedrive — CRM dedupe + SDR_QUEUE lead creation (REQUIRED)

**What it does:** Two jobs. (1) Dedupe at Step 3 — check whether the email is already a contact (and if so append the subscription as a note rather than recreating). (2) Create a Pipedrive **lead** (not a deal — leads are the right object for top-of-funnel) for every SDR_QUEUE subscriber, with full tiering context in custom fields.

**Setup:** Any Pipedrive MCP. I use `mcp__pipedrive__` from the community list.

**Setup checklist before running:**
- Make sure the Pipedrive Leads inbox is enabled
- Add custom fields on Leads: `source` (text), `icp_score` (numeric), `tier` (text), `tier_reason` (text), `utm_campaign` (text)

**Tool prefix in prompt:** `mcp__pipedrive__`

**Endpoints used:**
- `search_contacts` — Step 3 dedupe
- `add_note` — append subscription event to existing contacts
- `add_person`, `add_organization` — create records for new SDR_QUEUE subscribers
- `add_lead` — create the SDR_QUEUE lead in the inbox

## Instantly — WARM_NURTURE drip (REQUIRED for non-SDR routing)

**What it does:** Enrolls WARM_NURTURE subscribers in a 5-email, 3-week product-curious sequence (soft pitch in email 4). SDR_QUEUE bypass Instantly entirely. CONTENT_NURTURE and DISQUALIFIED also bypass — they stay on the newsletter only.

**Setup:** [Instantly MCP docs](https://developer.instantly.ai/mcp). Build two campaigns ahead of time:

- **Warm nurture campaign** — 5 emails over 3 weeks. Email 1 = welcome + your best evergreen content piece. Email 2 = case study. Email 3 = use case. Email 4 = soft pitch with calendar link. Email 5 = social proof + second calendar nudge. Get the UUID.
- **Content nurture campaign** (optional placeholder — most installs don't use this; CONTENT_NURTURE stays in Beehiiv only). If you do want a parallel content drip for non-buyer fans, build it as content-only with no CTA, get the UUID.

**Placeholders:**
- `[INSTANTLY_MCP_ID]` — your Instantly MCP server ID
- `[INSTANTLY_WARM_NURTURE_CAMPAIGN_ID]` — warm nurture campaign UUID
- `[INSTANTLY_CONTENT_NURTURE_CAMPAIGN_ID]` — content nurture campaign UUID (optional; leave empty if CONTENT_NURTURE stays newsletter-only)

**Endpoints used:**
- `add_leads_to_campaign_or_list_bulk` — enrollment
- `list_leads` (optional) — pre-enrollment dedupe check

## Claude in Chrome — Slack webhook (REQUIRED)

**What it does:** Posts to `hooks.slack.com` (the bash sandbox blocks this URL — Chrome is the only reliable way to fire a Slack webhook from Claude Code).

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook URL
- `[SDR_CHANNEL]` — channel where SDR_QUEUE pings land (e.g. `#sales-newsletter-queue`) — cosmetic, used in the ping body
- `[SDR_DM_USER_ID]` — your SDR's Slack member ID (`U...`) for the daily digest DM + existing-contact re-engagement pings
- `[ALERT_CHANNEL]` — channel where hourly summary + errors land — cosmetic
- `[YOUR_SLACK_USER_ID]` — your Slack member ID for `@`-mentions on error / approval messages

## WebSearch

Built into Claude Code. Used optionally for filling gaps when Zevari's `agents_company_intelligence` is thin — press releases, funding announcements, recent hiring posts that didn't make it into the company panel.

No placeholder.

## Producer side — Beehiiv webhook → Supabase Edge Function

This repo is the consumer (the hourly drain). The producer is whatever pipes Beehiiv subscription events into Supabase. The shape that works:

1. New subscriber on Beehiiv (homepage form, post CTA, referral, API)
2. Beehiiv fires a `subscription.created` webhook
3. Supabase Edge Function receives the payload, extracts email + utm + source + subscription_id, inserts a row into `[NEWSLETTER_QUEUE_TABLE]` with `processed_at = NULL`
4. Edge Function returns `200 OK` to Beehiiv

The Edge Function is ~40 lines of TypeScript. Not included in this repo because it's specific to your Beehiiv account + webhook secret. Beehiiv's webhook docs are the reference.

For Substack / Kit / Mailchimp the shape is the same, only the webhook payload field names differ.

## Tier tag schema (Beehiiv side)

For the segmentation to work downstream, decide on these tag names once and stick with them. Defaults the prompt uses:

- `tier:sdr_queue`
- `tier:warm_nurture`
- `tier:content_nurture`
- `tier:disqualified`
- `tier:no_linkedin_match`
- `utm:[utm_campaign value]` — for attribution; one tag per subscriber per campaign source

When you send a sales-focused newsletter, segment to `tier:sdr_queue OR tier:warm_nurture` only. When you send a product-update newsletter, segment to everyone except `tier:disqualified`. When you send a content-only piece, send to all.

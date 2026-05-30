# linkedin-mcp-newsletter-to-pipeline

A Claude Code prompt that turns every newsletter subscriber into a triaged GTM signal. Beehiiv / Substack / Kit signup → live LinkedIn MCP resolution → ICP + behavioral score → SDR_QUEUE / WARM_NURTURE / CONTENT_NURTURE / DISQUALIFIED.

Sharing it because newsletter signups are the highest-intent inbound signal most B2B operators have, and 99% of them waste it.

## The problem with most newsletter-to-CRM setups

I've watched this play out at every B2B company I've worked with. Pick one:

- **Sync everything as "leads"** — Beehiiv → Pipedrive zap. Now your AEs have 4,000 "leads" with no titles, no companies, no scoring. They ignore the whole list.
- **Sync nothing** — newsletter lives in its own silo. The CRO can't see that the CFO of a $50M ICP account just subscribed last Tuesday. SDR never calls.
- **Enrich with Apollo / Clearbit** — get back a title and a company that are 30-90 days stale. Same problem I wrote about in [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger). The "Senior Manager" who signed up got promoted to VP three weeks ago and your scoring missed it.
- **Use anonymous-visitor identification** (RB2B, Warmly, Clearbit Reveal) — fine for *unknown* visitors but newsletter signup is **the opposite of anonymous**. You have an email. The job is to resolve it to a human, not guess from an IP.

The newsletter signup is the highest-intent inbound signal a content reader will ever give you. They liked your writing enough to want more in their inbox every week. That's a real preference. And almost every operator either floods the CRM with junk or treats subscribers as a separate untriaged list that sales never sees.

## What this does instead

The prompt runs on every new subscriber event (Beehiiv webhook is the primary example — easy MCP, but Substack/Kit/Mailchimp work the same way). For each new signup:

1. **Queue the event** — Supabase row, durable, idempotent
2. **Skip throwaways** — `+test` addresses, mailinator-style domains, obvious bot signatures
3. **Skip-but-update existing CRM contacts** — if they're already in Pipedrive, append the subscription as an engagement event, don't recreate
4. **Resolve to LinkedIn live** — Zevari `inbox_enrich_sender` (email → LinkedIn URL), then `linkedin_get_profile` for today's title + company
5. **Company intelligence** — Zevari `agents_company_intelligence` (size, industry, recent funding, hiring signals)
6. **Behavioral profile** — Zevari `agents_behavioral_profile` (do they pattern-match someone who buys what you sell? Recent posts, recent job change, recent funding, comments on relevant outbound topics)
7. **ICP score** — Zevari `profile_get` (live rules) + `agents_icp_score`
8. **Route to one of four tiers:**
   - **SDR_QUEUE** — ICP_PASS + senior buying title + active company signal → Slack ping to SDR, Pipedrive lead with `source=newsletter-subscription`, Zevari hot-leads list
   - **WARM_NURTURE** — ICP_PASS but no acute signal → 5-email product-curious nurture in Instantly, soft pitch in email 4
   - **CONTENT_NURTURE** — out of ICP but historically engaged with your content → newsletter only, no sales touch
   - **DISQUALIFIED** — out of ICP and no engagement signal → newsletter only, archive
9. **Tag the subscriber in Beehiiv** with their tier + `[utm_campaign]` for attribution
10. **Save to Zevari** `newsletter-signups-YYYY-MM-DD`
11. **Slack** hourly drain summary + daily 8 AM SDR digest

Latency target: SDR_QUEUE Slack ping within the same hour as signup. Apollo/Clearbit enrichment runs on a weekly cron at best. Zevari hits LinkedIn live the moment the webhook fires.

## Why Zevari (and not just Apollo / Clearbit / RB2B)

This is the load-bearing part of the workflow.

- **Apollo / Clearbit / ZoomInfo** = fine firmographic seed (employee count, industry, revenue band) but title is 30-90 days behind LinkedIn and intent is invisible to them
- **RB2B / Warmly / Clearbit Reveal** = solve a *different* problem (anonymous-visitor identification). They aggregate from IP. Newsletter signups aren't anonymous — you already have the email — and aggregated intent isn't usable per-person
- **HubSpot built-in scoring** = scores against whatever HubSpot has on the contact, which means it scores against Apollo's stale enrichment plus form-fill data, with no behavioral read
- **[Zevari](https://zevari.ai)** = live LinkedIn resolution at signup time. Today's title. Today's company. Today's posts. Whether they engaged with your content in the last 90 days

The single most predictive feature in my newsletter-to-pipeline model: `agents_behavioral_profile` returning "this person has commented on outbound automation posts 4 times in the last 6 months." That's a stronger signal than "this person works at a Series B in your ICP." Both together is the SDR_QUEUE call. Neither comes out of Apollo.

If you're evaluating LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one — his README is honest about ToS risk. For client work where I need this running on a schedule against real subscriber volume, I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Live LinkedIn resolution + ICP + behavioral signal | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The whole point of the prompt — today's title, today's company, today's engagement signal |
| Newsletter platform + subscriber events + tagging | [Beehiiv](https://beehiiv.com) (or Substack / Kit / Mailchimp) | Source of subscriber webhook + write back tier tags for attribution |
| Webhook queue | [Supabase](https://supabase.com) | Durable queue, ACID writes from the webhook, easy `processed_at` flag to drain |
| Drip campaigns (WARM + CONTENT nurture) | [Instantly](https://instantly.ai) | Two pre-built campaigns, prompt just enrolls |
| CRM (SDR_QUEUE lead creation + dedupe) | [Pipedrive](https://pipedrive.com) | Lead drops straight into the SDR's view with full context |
| Notifications | Slack | Hourly drain summary + SDR ping + daily 8 AM digest |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook (bash sandbox blocks `hooks.slack.com`) |

## How to use this

1. Clone the repo
2. Open `prompt.md`
3. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[NEWSLETTER_QUEUE_TABLE]` — your Supabase queue table (default: `newsletter_signups`)
   - `[BEEHIIV_API_KEY]`, `[BEEHIIV_PUBLICATION_ID]` (or `[SUBSTACK_API_KEY]` / `[KIT_API_KEY]`) — newsletter platform
   - `[INSTANTLY_WARM_NURTURE_CAMPAIGN_ID]`, `[INSTANTLY_CONTENT_NURTURE_CAMPAIGN_ID]` — Instantly campaign UUIDs
   - `[ZEVARI_MCP_ID]`, `[SUPABASE_MCP_ID]`, `[INSTANTLY_MCP_ID]` — MCP server IDs
   - `[SDR_CHANNEL]`, `[SDR_DM_USER_ID]`, `[ALERT_CHANNEL]`, `[SLACK_WEBHOOK_URL]`, `[YOUR_SLACK_USER_ID]` — Slack
4. Paste the prompt into Claude Code (I save it as a slash command)
5. Run hourly via cron for the drain, plus a daily 8 AM run for the SDR digest

See `connectors.md` for full setup, including the Beehiiv webhook → Supabase Edge Function piece.

## What gets generated (sample)

See `examples/sample-run.md` for a real hourly drain pulling 24 new Beehiiv signups: 3 SDR_QUEUE, 7 WARM_NURTURE, 6 CONTENT_NURTURE, 3 DISQUALIFIED, 5 unresolved.

## Things I learned building this

- **The 24-hour window matters more than people think.** Newsletter signup → SDR contact within a day = 3-4x higher response rate than the same outreach a week later. The subscriber still remembers signing up. After a week they don't.
- **`agents_behavioral_profile` catches what firmographics miss.** "This CTO has been commenting on outbound automation posts for 6 months" is a stronger conversion signal than "this person works at a Series B in your ICP." Both together is the SDR_QUEUE call. The firmographic-only score on its own routes way too many people into the warm bucket who never convert.
- **Always update existing Pipedrive contacts when they sign up.** Newsletter subscription is an engagement event even if they're already a known contact. Append it to their record — don't recreate, don't dedupe-and-forget. Future you needs to see "first touch was a demo six months ago, but they just resubscribed to the newsletter, so they're back in the buying window."
- **Throwaway filters should be aggressive.** `+plus` addresses, mailinator / guerrillamail / 10minutemail domains, and any email that no LinkedIn resolver can match — collectively about 90% of them are bots, content scrapers, or competitors snooping. DISQUALIFIED is fine for those, no SDR cycles wasted.
- **Tag subscribers in Beehiiv with their tier.** When you send a sales-focused newsletter (product update, webinar invite, case study drop), you want to target SDR_QUEUE + WARM_NURTURE only. Without the tag write-back you can't segment, and you end up either spamming everyone or sending nothing.
- **ICP rules live in `profile_get`, not in the prompt.** Same lesson as my other pipelines — RevOps changes the rules weekly. Read live, never hardcode.
- **Daily SDR digest at 8 AM, not in-the-moment pings for every signup.** Newsletter signups are high-intent but they're not "drop everything" the way demo requests are. Hourly drain runs into Slack get tuned out. The 8 AM digest of yesterday's SDR_QUEUE adds with one-line context per lead is what the SDR actually reads with coffee.

## Adapting this

This is built around Beehiiv + Supabase + Pipedrive + Instantly. Swap in:

- **Newsletter platform:** Substack, Kit (ConvertKit), Mailchimp, Customer.io — anything with a "new subscriber" webhook + a tag-write API
- **Webhook queue:** Supabase / Convex / a tiny webhook listener — anything with durable rows + a processed flag
- **CRM:** HubSpot (`mcp__hubspot__`), Salesforce, Attio
- **Drip:** Smartlead, HubSpot Sequences, Customer.io
- **Notifications:** Discord / Teams / email — anywhere you can fire a webhook from Claude in Chrome

The shape (subscriber webhook → durable queue → live LinkedIn resolve → tier → route → tag back) generalizes to any content-led inbound funnel.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-inbound-lead-triage](https://github.com/jpeslar1/linkedin-mcp-inbound-lead-triage) — Real-time webhook → live ICP score → HOT/WARM/COLD routing
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — Morning sales briefing with live LinkedIn signals on every open opportunity
- [linkedin-mcp-inbox-zero-triage](https://github.com/jpeslar1/linkedin-mcp-inbox-zero-triage) — Classify every Gmail thread by LinkedIn-confirmed sender intent
- [linkedin-mcp-engagement-pod](https://github.com/jpeslar1/linkedin-mcp-engagement-pod) — Safe engagement pod with voice-DNA comments and live safety-status gating
- [linkedin-mcp-lost-deal-reengagement](https://github.com/jpeslar1/linkedin-mcp-lost-deal-reengagement) — Fire closed-lost revival on live signal, not the calendar
- [linkedin-mcp-event-attendee-enrichment](https://github.com/jpeslar1/linkedin-mcp-event-attendee-enrichment) — Resolve event attendees live on LinkedIn for accurate titles + tiering
- [linkedin-mcp-webinar-followup](https://github.com/jpeslar1/linkedin-mcp-webinar-followup) — Tier webinar attendees by ICP × behavioral signal, not just attendance
- [linkedin-mcp-trade-show-pipeline](https://github.com/jpeslar1/linkedin-mcp-trade-show-pipeline) — 3-phase trade show pipeline with live LinkedIn enrichment
- [linkedin-mcp-icp-discovery](https://github.com/jpeslar1/linkedin-mcp-icp-discovery) — Seed best customers → behavioral lookalike scoring → live-confirmed top-tier list

Or browse them all in [awesome-linkedin-mcp](https://github.com/jpeslar1/awesome-linkedin-mcp) — the curated index of LinkedIn MCP workflows, servers, and adjacent tools.

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT.

## Who I am

John Peslar — solo founder, build outbound and inbound automations for B2B clients. [johnpeslar.com](https://johnpeslar.com).

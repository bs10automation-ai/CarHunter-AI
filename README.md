# CarHunter-AI
An autonomous AI agent that monitors AutoScout24 daily, analyzes used car listings across Italy, and sends email alerts only when it finds a genuinely good deal — with full reasoning visible at every step.
Built entirely in n8n, combining Apify for real-time listing scraping, Google Gemini for market-aware deal analysis, Airtable for persistent deduplication state, and Gmail for delivery. Zero manual interaction required after setup.

What it does

Every day at midnight (Europe/Rome), Apify scrapes fresh AutoScout24 listings matching the configured criteria (Italy, used, under €8,000). After each run, Apify fires a webhook directly into n8n. The workflow then:


Extracts individual listings from the Apify payload
Checks Airtable to filter out any listing already seen in a previous run
Passes each new listing to a Gemini-powered AI Agent
The agent calls a custom tool (analyze_listing_signals) to assess mileage patterns, price-age signals, and negotiation potential
Using both the tool output and its own knowledge of the Italian used car market, the agent produces a structured verdict: deal score (0–100), market estimate, reasoning, and a contact recommendation
If deal score ≥ 60 and recommendContact: true, the listing is saved to Airtable and a formatted email alert is sent via Gmail
Listings that don't meet the threshold are silently skipped — no noise, no false positives


Why this is different from a simple price alert

Most price-alert tools just apply a static filter ("under €X"). CarHunter AI reasons about value: a Land Rover Evoque at €6,900 with 300k km is analyzed differently than a Renault Clio at €6,900 with 60k km, even though both pass the same price filter. The agent understands brand depreciation curves, mileage-to-age ratios, and seller type signals — and explains its reasoning in plain language in every alert.
Apify Schedule (daily, Europe/Rome)
        │
        │ webhook POST after each run
        ▼
Webhook Trigger (n8n, Production URL)
        │
Extract Listings (Code node)
  → splits Apify items array into individual n8n items
        │
        ├─────────────────────────────────────┐
        │                                     │
Check If Seen (Airtable Search)         [parallel branch]
  → filter: {listing_id} = "..."
        │
Is New Listing (IF node)
  → $json.id is empty → True (new) / False (already seen)
        │ True
Get Listing Data (Code node)
  → restores full listing fields from Extract Listings context
        │
AI Agent (Gemini 2.5 Flash)
  → Tool: analyze_listing_signals (Code Tool)
       - mileage signal (very_low / normal / high / very_high)
       - price-age signal
       - negotiation potential (0-100)
       - motivation signals array
       - recommendContact boolean
  → Agent combines tool output with market knowledge
  → Returns: dealScore, marketEstimate, verdict, reasoning, recommendContact
        │
Parse Agent Output (Code node)
  → strips markdown fences, JSON.parse, restores sellerPhone from listing data
        │
Is Worth Notifying (IF node)
  → recommendContact is true AND dealScore >= 60
        │ True
Save to Airtable (Create record)
  → listing_id, URL, Title, Price, seen_at, deal_score, Notified
        │
Format Email (Code node)
  → builds subject and body strings with all listing fields
        │
Send Gmail Alert
  → to: configured recipient
  → full AI reasoning included in body
  Tech stack


n8n — workflow orchestration (n8n Cloud)
Apify — AutoScout24 scraper (blackfalcondata/autoscout24-scraper, pay-per-result)
Google Gemini 2.5 Flash — market-aware deal analysis via n8n native Gemini node
Airtable — persistent state for deduplication across daily runs
Gmail — alert delivery via OAuth2
Monthly cost

ServiceUsageCostApify~300 results/month (10/day)~$0.24Apify actor starts30 starts/month~$0.15Gemini API~300 analyses/month~$0.50–1.00AirtableFree tier$0GmailFree$0Total~$1–2/month
Key engineering decisions

Webhook-driven, not schedule-driven. Rather than having n8n poll Apify on a timer, Apify fires a webhook into n8n after each run completes. This means n8n only activates when there is actually new data to process — cleaner, more event-driven, and demonstrates a real-time trigger architecture rather than a polling pattern.

Persistent deduplication via Airtable. Each listingId (a stable UUID embedded in the AutoScout24 listing URL) is stored in Airtable after processing. On every subsequent run, the workflow queries Airtable before passing any listing to the AI Agent — ensuring the agent never analyzes the same listing twice, even across days. This is proper cross-run state management, not in-memory deduplication that resets on each execution.

AI Agent with explicit tool use, not a single LLM call. The deal assessment uses n8n's AI Agent node rather than a direct "Message a Model" call. The agent is given one tool (analyze_listing_signals) and decides autonomously when and how to call it. This means the reasoning trail is visible: the agent's output explicitly references what the tool returned and explains where it agreed or disagreed with the tool's assessment — making the decision process auditable rather than opaque.

Market knowledge over formulas. An early version included a calculate_deal_score Code Tool with a universal depreciation formula. Testing revealed that the formula produced nonsensical estimates for premium brands (Land Rover, BMW, Alfa Romeo) because brand retention curves vary enormously. The formula was removed in favor of letting Gemini apply its own market knowledge — which correctly assessed that a 2012 Range Rover Evoque holds significantly more residual value than a 2012 Fiat of similar mileage, without any brand-specific calibration in the code.

Tool output as a signal, not a verdict. The analyze_listing_signals tool returns recommendContact: true/false and a negotiationPotential score, but the AI Agent prompt explicitly instructs the model to treat these as inputs to consider rather than final verdicts. In one live test, the tool flagged a listing as overpriced_for_age while the agent correctly overrode this with its own market reasoning — demonstrating that the agent architecture adds genuine judgment on top of deterministic tool outputs.

Explicit field restoration after Airtable check. After the Check If Seen Airtable node, the workflow passes through an IF node and then a Code node that explicitly re-fetches the original listing data from Extract Listings using $('Extract Listings').first().json. This is necessary because the Airtable search node returns Airtable record fields (or an empty object), not the original listing fields — a common source of "undefined" errors in multi-branch n8n workflows.

Structured output via prompt instruction, not response schema. Unlike the Football Intelligence Agent V2 (which used Gemini's native responseSchema parameter via HTTP Request), this project uses n8n's native AI Agent node, which does not expose response schema configuration. Instead, the prompt explicitly instructs the model to return a valid JSON object with specific field names, and the Parse Agent Output Code node strips markdown fences before parsing. This demonstrates prompt-based output structuring as an alternative to API-level schema enforcement.
🚗 CAR DEAL SCOUT ALERT

Land Rover Range Rover Evoque 2.2 TD4 Prestige — Unicoproprietario

💰 Price: €7,900
📊 Deal Score: 75/100
🎯 Verdict: good_deal
📍 Location: Torino - TO
🛣️ Mileage: 229,000 km
💹 Market Estimate: €9,000

🤖 AI Analysis:
Based on my deep knowledge of the Italian used car market, a 2015 Land Rover
Range Rover Evoque 2.2 TD4 Prestige with 229,000 km has a fair market value
of approximately €9,000. The asking price of €7,900 is about 12.2% below
market estimate. The analyze_listing_signals tool indicated 'overpriced_for_age'
but this reflects a general model that doesn't account for premium brand
pricing. Average km/year of 20,818 is within normal limits for a diesel of
this age. Negotiation potential: 50%.

📞 Seller: MS Automobili Srl
☎️ Phone: +393409097048
🔗 Link: https://www.autoscout24.com/offers/...
What I'd improve next


Add a yearFrom filter in Apify Input to exclude very old vehicles automatically rather than relying on the agent's score to filter them out
Store the agent's full reasoning in Airtable (not just the score) to build a historical record of what was analyzed and why
Add a second email digest mode: a weekly summary of all listings seen that week, not just immediate alerts for top scores


About this project

Built as a portfolio project while developing practical AI automation skills — with a deliberate focus on problems that matter for real-world deployment: deduplication across stateful runs, event-driven rather than polling architectures, and making AI reasoning visible and auditable rather than treating the model as a black box that returns a number.

The domain (Italian used car market) was chosen because it's a space with genuine domain knowledge embedded in the workflow — the agent's market estimates are meaningfully better than a formula precisely because it has internalized real pricing patterns, and that difference is visible in the output.

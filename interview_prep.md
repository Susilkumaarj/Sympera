# Sympera Interview Prep — Home Assignment Walkthrough

> Keep this file local — add to `.gitignore` if you don't want it in the submission repo.

## Context — what Sympera actually does

This isn't just a take-home; it mirrors Sympera's product:

- **Client Intelligence** — interpret transaction patterns, monitor business health, identify opportunities (your Task 1)
- **Sales Enablement** — give RMs product recommendations, conversational hooks, objection handling (your Task 3)
- **Domain-specific LLMs** — customized open-source models for financial services (your Task 2 system prompt)
- **Portfolio scale** — RMs managing larger portfolios at the right moment
- **Seed-stage Israeli startup**, $10M raised, targeting the $150B SMB banking opportunity, integrates with Salesforce / Dynamics / Teams

Expect questions about how your assignment would generalise to **1000s of customers**, not just Elite Builds.

---

## The 90-second walkthrough (opening pitch)

> "The brief was to turn 20 transactions plus a market snippet into something a Bank RM could actually use. I treated it as three connected problems: extract growth signals, package them for retrieval, and convert them into a sales conversation.
>
> For Task 1, I joined the transactions against the market benchmark — 15% material inflation, 4% industry margin, the High Expense Velocity credit flag — and engineered 8 signals split into Resilience Alpha (how the firm is beating the cost surge) and Growth Reinvestment (where they're deploying capital).
>
> For Task 2, I produced a sub-300-token JSON profile across behavioural, demographic, psychographic, and financial-signal segments — structured for a RAG system to retrieve the right slice without parsing the full report.
>
> For Task 3, the system prompt enforces JSON-only output, an analyst persona, and grounding rules so the model omits weak signals instead of inventing them. The product recommendation is an Asset-Backed Equipment Term Loan — the $10K downpayment is a live acquisition signal — and the hook leads with client behaviour, not a pitch."

---

## Likely Questions — by category

### 1. Approach & framing

**Q: Walk me through your overall approach.**
Three layers: raw data → structured signals → RAG-retrievable profile → RM action. Each layer is independent so the pipeline degrades gracefully — even if the LLM step fails, the signals JSON is still useful to a human RM.

**Q: What was the hardest part?**
Picking which signals were genuinely informative vs. noise. With only 20 transactions and 1 month of data, it's tempting to over-claim. I kept signals tied to either an explicit market benchmark or a clear behavioural pattern — anything weaker I left out.

**Q: Why split signals into Resilience Alpha and Growth Reinvestment?**
The brief uses those exact labels. Resilience Alpha answers "why is this firm beating the market?" — useful for underwriting. Growth Reinvestment answers "where are they spending strategically?" — useful for product cross-sell. The split maps directly to RM use cases.

---

### 2. Task 1 — Signal Engineering

**Q: Why those 8 signals?**
Each ties to either the market snippet (margin, inflation, expense velocity) or a behavioural pattern (collect-first, savings discipline, capex rate, pipeline diversification). I deliberately avoided generic metrics like "total spend" that don't differentiate this firm.

**Q: Walk me through "Controlled Expense Velocity."**
The market snippet flags High Expense Velocity as a credit risk. I defined velocity two ways: opex as % of revenue (46.8%), and timing — what % of debits land within 7 days *after* a credit. 100% of Elite Builds' debits follow a credit, average 3.2-day lag. So they're not just lean; their spending is *funded by* incoming revenue, not preceding it.

**Q: Operating margin 53.2% vs. all-in margin 39.8% — which is the "true" margin?**
Both are valid for different audiences. Operating margin is the standard accounting view (capex and savings transfers are capital allocation, not opex). All-in margin treats every outflow as a cost — useful for stress-testing cash conversion. I report both. Either way, the firm is 10x the industry average.

**Q: You assumed 3.5% savings APY to back-calculate reserves. What if that's wrong?**
The implied reserves figure is directional, not exact. At 2% APY you'd infer ~$72K; at 5% APY ~$28K. Both still confirm the firm has meaningful idle cash, which is the actual signal. I flagged this as an assumption in the README and in the signal's `assumption` field.

**Q: What if Elite Builds had only one big client at 80% of revenue?**
Signal 8 (Diversified Pipeline) would flip from positive to a concentration risk flag. The credit conversation changes — instead of a growth product, the RM should be probing pipeline durability before extending credit.

**Q: How would you handle 200 transactions or multiple months?**
Move from totals to rolling metrics: 3-month moving margin, vendor concentration trend, project win rate. Time-series patterns reveal acceleration/deceleration that single-month totals hide. I'd also add a confidence score per signal based on sample size.

**Q: Why no pandas? The requirements file lists it.**
For 20 rows, pure Python was clearer and faster. Pandas pays off at scale or with complex joins. I'd switch to pandas (or polars) the moment we hit multiple customers or rolling-window aggregation.

---

### 3. Task 2 — RAG dataset

**Q: Why those four segments — behavioural, demographic, psychographic, financial?**
The brief required behavioural, demographic, psychographic. I added `financial_signals` separately because mixing raw numbers into prose strings hurts retrieval — the RM AI should be able to look up `op_margin_pct` as a number, not parse it from "53.2% vs 4% industry average."

**Q: Why is the 300-token limit important?**
Two reasons. First, retrieval — smaller chunks mean the RAG layer can fit more customer profiles into context per query. Second, cost — at production scale, every retrieved chunk multiplies by every query. Token discipline scales.

**Q: How did you measure tokens?**
chars/4 is the cl100k_base heuristic for English. For compact JSON with numerics and underscored keys it lands closer to chars/4.3–4.5. I aimed for ~280 tokens to leave headroom.

**Q: Why short factual strings instead of full sentences?**
RAG retrieval is similarity-based, not grammatical. "Collect-first: revenue precedes outflows" embeds nearly identically to a sentence-form version but uses half the tokens. Density beats prose.

**Q: How would the RM AI actually retrieve this?**
Embed each segment block separately (or even each line) so a query like "is this firm a credit risk?" pulls the `credit_risk` and `expense_velocity` fields without dragging in psychographics. The top-level structure is the chunking unit.

**Q: What embedding model would you use?**
For finance-specific work, a domain-tuned model — Sympera mentions customized open-source LLMs, so likely a fine-tuned BGE or E5 on finance corpora. For a baseline, `text-embedding-3-small` or `BAAI/bge-large-en-v1.5` works.

**Q: What if you had 10,000 customers — how do you scale this?**
Pipeline: nightly transaction sync → signal extraction job → segment generation (LLM or rules) → embed → upsert into vector store. Versioning matters — segments change as new transactions arrive, so each customer needs a `period` field and the vector store needs upserts keyed by `(customer_id, period)`.

---

### 4. Task 3 — Prompt engineering

**Q: Walk me through your system prompt choices.**
Persona first — "senior commercial banking intelligence analyst" anchors the tone and vocabulary. Then hard constraints (JSON only, sub-300 tokens, no markdown). Then grounding rules (omit weak segments, no marketing language). The grounding rule is the most important — without it the model invents psychographic claims with no transaction-level evidence.

**Q: Why specify "If evidence is weak, omit the segment instead of inventing information"?**
Hallucination control. LLMs prefer to fill structure than leave fields empty. This rule explicitly reverses that bias. In a regulated banking context, omission is safer than confabulation.

**Q: What if the model returns invalid JSON?**
Two layers: (1) tighten the prompt with `response_format={"type": "json_object"}` on OpenAI or schema-constrained decoding on local models, (2) wrap the call in a validator + retry loop with the JSON parse error fed back as a correction message.

**Q: How would you evaluate prompt quality at scale?**
Build a small gold-set of customer profiles a human analyst has labelled. Measure: schema compliance rate, retrieval recall on test queries, hallucination rate (claims not grounded in transactions), and downstream — did the RM use the hook, did the meeting convert.

**Q: Why "institutional banking language" and no marketing tone?**
RM audience. The downstream consumer is a human banker who needs to *trust* the output. Marketing language reads as a pitch and erodes trust; institutional language reads as analysis.

---

### 5. Bank product recommendation & RM hook

**Q: Why Asset-Backed Equipment Term Loan over SBA 7(a)?**
SBA 7(a) is government-backed, longer approval, lower rate but more paperwork. Asset-backed term loan is faster, the equipment itself is the collateral, and the firm's profile (53% margin, $41K reserves) makes underwriting clean. For an already-creditworthy firm, asset-backed wins on speed.

**Q: Why not a working capital line?**
The signals say they don't *need* working capital — the collect-first model eliminates the cash-flow gap that a credit line solves. Pitching a credit line to a firm that already manages liquidity well reads as a generic pitch and erodes RM credibility.

**Q: Walk me through the hook.**
> "Most contractors delay equipment ownership because they're protecting liquidity — your business is already preserving reserves while investing in expansion simultaneously..."

Three moves: (1) name the common objection ("protecting liquidity") to neutralise it; (2) demonstrate the RM has read the account ("preserving reserves while investing"); (3) end on an observation, not a close — the client is invited to confirm, which lowers resistance.

**Q: What would a *bad* hook look like?**
"Hi, we noticed you bought equipment last month. We have great rates on equipment loans." Generic, transactional, no signal that the RM understands the business. Triggers the standard objection ("we're managing it ourselves") instead of pre-empting it.

**Q: How would you A/B test hooks in production?**
Track at the funnel: open rate (if email) → meeting accepted → product conversation engaged → product sold. Hook A vs. Hook B across matched-pair customer segments. Sympera could host this as a hook library with per-segment performance.

---

### 6. Production / scale (high-value questions)

**Q: How would you turn this into a production pipeline at Sympera?**
1. **Ingest** — transaction feed per bank customer (API or batch)
2. **Enrich** — categorise transactions (this PDF gave them pre-categorised; in real life this is a classifier)
3. **Signal engine** — rules + ML for signal extraction, versioned
4. **Profile generator** — LLM produces `customer_segments.json` with the system prompt I wrote
5. **Vector store** — embed and upsert per customer/period
6. **RM-facing layer** — Salesforce/Dynamics widget that queries the store and renders signals, recommendations, hooks
7. **Feedback loop** — track which hooks RMs used and whether they converted; feed back into prompt iteration and signal weighting

**Q: How does this generalise to other industries?**
The framework (market context + transactions → signals → segments → hooks) is industry-agnostic. The specific signals are not. Each vertical (restaurants, retail, professional services) needs its own benchmark snippet and signal library. The system prompt is reusable with a vertical injection.

**Q: How would you detect when a customer profile becomes stale?**
Two triggers: (1) a transaction event that materially changes a signal (large new credit, missed payroll, etc.), (2) periodic refresh on a cadence. The profile JSON should carry a `freshness` field and the RM UI should surface staleness.

**Q: What's the biggest risk in shipping this?**
LLM hallucination in regulated context. A made-up psychographic claim that influences a credit decision is a compliance issue. Mitigations: schema-constrained outputs, grounding rules in the prompt, post-generation validation that every claim maps to a transaction-level fact, and human-in-the-loop for any underwriting use.

**Q: How do you measure if this is actually helping RMs?**
- Adoption: % of RM customer interactions where the AI profile was opened
- Conversion: meeting-booked rate after using an AI hook vs. baseline
- Quality: RM thumbs-up/down on each generated insight
- Outcome: net new revenue attributable to AI-surfaced opportunities
- Time saved: avg prep time per RM call

---

### 7. Tricky / curveball questions

**Q: What's the weakest part of your submission?**
Single month of data. Most of my behavioural signals (collect-first, expense cadence) would be much stronger across 3–6 months. The capex rate especially — 8.9% in one month could be 0% the next.

**Q: If you had one more day, what would you add?**
A counter-signals section. Right now everything is positive — Resilience Alpha and Growth Reinvestment. In production, the RM also needs the bear case: what's the strongest argument *against* extending credit? Forcing the model to articulate the downside makes the upside more credible.

**Q: Did you use an LLM to write the segmentation JSON, or did you write it by hand?**
[Be honest with your real answer.] If you wrote it manually then iterated the prompt to reproduce it — say so. The point of Task 3 is the prompt is the artifact; whether the JSON came from the prompt or vice versa, the prompt should be able to regenerate it.

**Q: How would you handle a customer whose transactions show genuine distress?**
Same pipeline, opposite output. The signals flip to risk flags, the product recommendation shifts to consultative (restructuring, advisory) rather than growth credit, and the hook becomes a check-in rather than a pitch. The framework holds; only the interpretation flips.

**Q: What if the market snippet was outdated?**
The benchmark is the most volatile input. I'd want it pulled from a live data source (industry reports API, FRED, sector indices) and timestamped. Stale benchmarks produce misleading signals — if inflation has dropped to 5% but my snippet still says 15%, I'm overstating Resilience Alpha.

**Q: Why did you choose to keep two output files (customer_segments.json AND signals_report.json)?**
Different audiences. `signals_report.json` is the full analyst view — every metric, every assumption, every RM action. `customer_segments.json` is the RAG-ready slice — sub-300 tokens, retrieval-optimised. Same underlying data, two access patterns.

---

## Numbers to have memorised cold

- **53.2%** operating margin (vs. **4%** industry)
- **39.8%** all-in margin
- **15%** material inflation; firm absorbed **$4,017** of that premium
- **$112,000** monthly revenue
- **$52,400** operating costs (46.8% of revenue — controlled expense velocity)
- **$41,143** implied liquid reserves (back-calc from $120 interest at 3.5% APY)
- **$5,000** to savings + **$10,000** equipment downpayment in same month
- **3.2 days** average gap between credit and following debit
- **4** projects; largest **40.2%** of revenue (no concentration)
- **8.9%** capex/revenue (mid of 5–10% healthy range)
- **20** transactions, March 2026

---

## Questions to ask THEM

Strong candidates ask. Pick 2–3:

1. "How does Sympera handle transaction categorisation upstream of the signal layer — is it rule-based, a classifier, or LLM-based?"
2. "What's the current production model for the segmentation step — fine-tuned open-source or a frontier API call?"
3. "How do RMs interact with the output today — in Salesforce, a separate UI, or pushed via Teams?"
4. "What does the feedback loop from RM behaviour back into model training look like?"
5. "Which vertical was hardest to bring up — and what made it hard?"
6. "How do you think about the boundary between automated insights and the RM's own judgement?"

---

## Frame the conversation

The assignment is small but the role is about scaling this. When in doubt, pivot from "what I did" to "what I'd build at Sympera scale" — every answer is better if it ends with "and here's how it generalises."

# Sympera AI Home Assignment

Analysis of **Elite Builds LLC**, a small construction firm in Utah, using their last 20 banking transactions and a Q1 2026 industry market snippet. The goal is to surface positive growth signals a Bank Relationship Manager can act on.

## Project Structure

```
.
├── scripts/
│   ├── Extraction.py         # Main script signal engineering (Task 1)
│   └── Exploratory.ipynb     # Notebook used while exploring the data
├── output/
│   ├── customer_segments.json   # RAG-ready segmentation JSON (Task 2)
│   └── signals_report.json      # Full signals output from Extraction.py
├── transactions.json         # Input 20 banking transactions
├── requirements.txt
└── README.md
```

## How to Run

```bash
pip install -r requirements.txt
python scripts/Extraction.py
```

The script reads `transactions.json`, prints a signal report to the console, and writes `output/signals_report.json`.

## Approach

The work is split across the three tasks from the brief.

**Task 1 Positive Signal Engineering.**
`Extraction.py` joins the 20 transactions against the market snippet (15% material inflation, 4% industry net margin, "High Expense Velocity" credit flag). It groups transactions by category, computes operating margin, expense velocity, inventory inflation absorbed, capex ratio, savings rate, and timing gaps between credits and debits. Each metric is checked against the market benchmark to produce a signal. Eight signals are generated, tagged as either *Resilience Alpha* (how Elite Builds is beating the 15% surge) or *Growth Reinvestment* (strategic outflows vs. operational noise).

**Task 2 RAG Dataset.**
`output/customer_segments.json` captures the firm in a compact, retrieval-friendly format under 300 tokens. It contains four blocks `customer_profile`, `behavioral_segments`, `demographic_segments`, `psychographic_segments`, and `financial_signals` so a downstream RAG system can retrieve the right slice without parsing the full signal report.

**Task 3 Prompt and RM Action.**
The system prompt that produced the segmentation JSON is below, along with the bank product recommendation and the RM hook.

## Key Findings

- **Operating margin 53.2% vs. 4% industry average** a 49 percentage point gap. Even the all-in margin (including capex and savings as costs) is 39.8%, still ~10x peers.
- **Inflation absorbed:** $4,017 of the 15% material premium absorbed without hurting margin. Two repeat vendors (Intermountain Lumber, Steel Supply Co) likely on negotiated rates.
- **Expense velocity is controlled.** 100% of debits occur within 7 days *after* a credit (avg lag 3.2 days). Spending follows revenue. This is the opposite of the "High Expense Velocity" flag banks are using to tighten credit.
- **Collect-first model.** First client payment lands before the first inventory purchase. No reliance on vendor credit.
- **Active capital allocation.** $5,000 transferred to savings *and* $10,000 equipment downpayment in the same month building reserves while investing in capacity.
- **Implied liquid reserves ~$41,143** back-calculated from $120 interest at 3.5% p.a.
- **Diversified pipeline.** 4 client payments in a single month, largest at 40.2% of revenue no dangerous concentration.

## Assumptions

- Savings APY assumed at **3.5% p.a.** for back-calculating the implied liquid balance from interest earned. Treat the implied balance as directional, not exact.
- The collect-first pattern is inferred from transaction timing not confirmed contractually.
- Capex ratio is from a single-month snapshot; ideally tracked across a full year for a reliable annualised rate.
- "Operating margin" in this report excludes capex and savings transfers (standard practice). The all-in margin is reported alongside for transparency.
- Industry benchmarks (15% inflation, 4% net margin, 5–10% healthy capex range) come directly from the provided market snippet.

## Exact System Prompt (used to generate `customer_segments.json`)

```
You are a senior commercial banking intelligence analyst specializing in SMB construction businesses.
Your task is to generate a compact RAG-ready customer segmentation profile using ONLY the provided
transaction data and market context.

The output must:
- Be valid JSON only
- Stay under 300 tokens
- Avoid markdown, explanations, or commentary
- Avoid hallucinations or assumptions not grounded in the data
- Focus on retrieval efficiency and signal density

Required JSON structure:
{
  "customer_profile": {},
  "behavioral_segments": [],
  "demographic_segments": [],
  "psychographic_segments": [],
  "financial_signals": {}
}

Segmentation Rules:
- Behavioral segments must describe financial and spending behavior patterns
- Demographic segments must describe company type, industry, scale, and operating model
- Psychographic segments must infer business mindset, growth orientation, technology adoption,
  liquidity discipline, or operational philosophy from transaction behavior
- Financial signals must summarize key benchmark comparisons and operational resilience indicators
- Use concise institutional banking language
- Do not generate marketing language
- Do not repeat transaction-level details unless necessary for segmentation quality
- If evidence is weak, omit the segment instead of inventing information
```

## Bank Product Recommendation

**Primary product: Asset-Backed Equipment Term Loan.**

The $10,000 equipment downpayment in March is a live acquisition signal the deal is already moving. Elite Builds holds ~$41,143 in implied liquid reserves and is transferring $5,000/month to savings, so financing the equipment preserves that buffer instead of depleting it. At a 53.2% operating margin, debt service is low-friction, and the equipment itself acts as collateral. An asset-backed structure gives a faster approval path and lower rate than an SBA 7(a) while keeping underwriting clean.

## RM Hook

> "Most contractors delay equipment ownership because they're protecting liquidity your business is already preserving reserves while investing in expansion simultaneously. That usually indicates a company ready to transition from project-by-project growth into capacity-driven scaling."

It leads with client behaviour instead of a product pitch, neutralises the most common objection ("we're protecting cash") before it comes up, and ends on an observation rather than a close so the client is invited to engage rather than pushed.

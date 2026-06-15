# Chapter 3: How To Pick Your Price

## Core Idea
"What's the best price? Simple: **The price that makes you the most money.**" Pricing affects both your **conversion rate** and your **churn rate** — both usually suffer as price goes up, **but not always proportionally**. "You may lose some — but often — not as much as you gain." The only way to find the best price is to **test, test, test**.

## Frameworks Introduced

- **The Price-Testing Table** (Price → Clicks → Conv Rate → Sales → Churn → LTV → Total Return → Difference)
  - When to use: deciding what to charge, or how far to raise.
  - How: hold clicks constant; change price and watch conversion, churn, and LTV move together. Compute **Total Return = Sales × LTV** per 100 clicks and compare the Difference vs. baseline. Pick the highest Total Return you have data on, then test prices around it.

- **Test on new customers first.**
  - How: "The easiest way to test is to try it out on new customers. Then — provided your churn doesn't go up like crazy and your sales don't go down like crazy — raise it on the rest of your customers."

## Worked Example: The $10 → $20 → $100 Walk-Through (per 100 clicks)

| Price | Clicks | Conv Rate | Sales | Churn | LTV | Total Return | Difference |
|---|---|---|---|---|---|---|---|
| $10 | 100 | 5% | 5 | 10% | $100 | $500 | — (baseline) |
| $20 | 100 | 4% | 4 | 10% | $200 | $800 | +60% |
| $100 | 100 | 2% | 33% | $300 | $600 | +20% |

- **$10 baseline.** 100 clicks × 5% = 5 sales. 10% churn (1 of 10 cancels per month) ⇒ $100 LTV each. Total = 5 × $100 = **$500**.
- **Double to $20 (+100%).** Conversion drops 20% (5% → 4%), churn unchanged ⇒ 4 sales. LTV doubles to $200. Total = 4 × $200 = **$800**. "Choosing to lose one customer to raise the price 100% nets us an extra 60% more revenue." At 30% margins on the first model, this would **triple your profit**.
- **10x to $100 (+1000%).** Conversion drops 60% (5% → 2%) ⇒ 2 sales. Churn triples to 33% (paying ~3 months instead of ~10). LTV = $300 (3x the original, +50% vs. $20). Total = 2 × $300 = **$600**. Still beats $10, but **less than the $20 price**.
- **Verdict:** "We don't know [the best price] yet. But we do know the best price of the ones we have data on — the **$20 price point**." Next, test between $20 and $100 — Hormozi would look at **$39–$59**, "because if churn stays lowish, these may be even more profitable options."

## Key Concepts
- **Conversion rate** — share of clicks that buy; tends to fall as price rises (but slower than price rises).
- **Churn rate** — share who cancel per period; tends to rise at much higher prices (the $100 jump tripled it).
- **LTV (lifetime value)** — per-customer revenue over their lifetime; rises with price even as count falls.
- **Total Return** — Sales × LTV from a fixed number of clicks; the number you actually optimize.
- **The decision rule** — "If you can double your price and close 20% fewer deals, with all else staying the same… you should do it!"

## Mental Models
- Don't assume linearity: "If doubling raised my profits… 10x'ing my price could raise it by 800%!" — sometimes true, **not always**. The math, not the wish, decides.
- Think in **Total Return per 100 clicks**, not per-sale price or per-customer LTV alone — they trade off.
- "Test, test, test" beats guessing: new customers are your free pricing lab.

## Anti-patterns
- **Assuming higher always wins** — the $100 case proves a too-high price can sink Total Return below a moderate raise.
- **Raising on the whole base before testing on new customers** — you can't see whether churn/conversion blew up.
- **Optimizing a single column** (just price, just LTV, just conversion) instead of Total Return.

## Key Takeaways
1. The best price is the one that makes the most money — found only by testing, not by copying the market.
2. Price moves conversion and churn together, usually downward, **but not proportionally** — so a raise often nets more.
3. Decision rule: double price and lose ≤20% of deals ⇒ do it.
4. Map several price points on the table, pick the best you have data on, then test nearby (e.g., $39–$59).
5. Test on **new customers** first; roll out to the base only if churn/sales hold.

## Connects To
- **Ch 2 (Rules)**: "test on new customers first" and "do the math" are operationalized here.
- **Ch 4 (RAISE letter)**: once the tested price is chosen, the letter rolls it out to the existing base.
- **Churn** → the $100 column shows why churn (covered further in Hormozi's *Churn Checklist Playbook*) gates how high you can go.

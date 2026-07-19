# Amazon Product Research & Competition Scoring Automation

An n8n workflow that automates Amazon product research for resellers — collecting product, pricing, and competitor data via SerpAPI, scoring competition level in Google Sheets, and generating AI-written recommendations delivered directly to Telegram.

---

## Overview

Manually researching Amazon products for resale means checking prices, ratings, reviews, and competitor counts one product at a time — slow, repetitive, and hard to scale across product categories.

This workflow automates that process end-to-end: it pulls product and seller data on a schedule, scores each product's competition level using a rules-based formula, has an AI agent generate plain-language reasoning behind each score, and sends a batched, readable report straight to Telegram.

---

## Problem

Product researchers need a fast way to evaluate whether an Amazon product is worth reselling — based on pricing, ratings, and how many competing sellers are already in that market. Doing this by hand across dozens of ASINs doesn't scale.

This workflow solves that by:
- Automatically collecting ASIN-level data (price, ratings, reviews, last month's purchase history) and seller-level data (competitor pricing, ratings, seller count) into a centralized Google Sheet
- Scoring each product's competition level using a rules-based formula (seller count tiers: Low — Enter, Medium — Consider, High — Caution, Pass)
- Using an AI agent to generate a written justification for each score, grounded strictly in the collected data
- Delivering the final report directly to Telegram in readable, batched messages

---

## Architecture

<img width="1320" height="606" alt="image" src="https://github.com/user-attachments/assets/cc4394e6-bb05-463e-94e9-bc4c8c06727d" />


**High-level flow:**
1. Scheduled trigger (every 7 days) reads product titles/categories from Google Sheets
2. SerpAPI request searches Amazon by title/category, returns a list of matching products
3. Data is split, filtered, and limited to control API usage during testing
4. A second SerpAPI request pulls full product details per ASIN (price, ratings, seller data, purchase history)
5. Two parallel paths process product details and seller data separately, then merge into one dataset
6. Google Sheets logs both product and seller data using `ARRAYFORMULA`-driven scoring sheets
7. An Aggregate node consolidates all scored ASINs into a single dataset
8. An AI Agent (Groq / Llama 3.3 70B) generates a competition-verdict justification per product
9. A Code node parses and batches the AI's output to stay within Telegram's message length limit
10. Telegram delivers the final report

---

## Tools Used

| Tool | Purpose |
|---|---|
| [n8n](https://n8n.io/) (self-hosted, Docker) | Workflow orchestration |
| [ngrok](https://ngrok.com/) | Public webhook exposure for local n8n instance |
| [SerpAPI](https://serpapi.com/) | Amazon product & seller data retrieval |
| Google Sheets | Data storage, scoring formulas, ARRAYFORMULA-based dynamic ranges |
| JavaScript (n8n Code node) | Parsing and batching AI output |
| [Groq](https://groq.com/) — Meta Llama 3.3 70B | AI-generated competition reasoning |
| Telegram | Report delivery |

---

## Problems Solved

**Nested seller data wasn't usable in its raw form.**
SerpAPI's `other_sellers` field returns nested objects per product rather than a flat list. Used a Split Out node to break this into individual fields (seller name, rating, reviews, price-per-unit) for clean tabular storage in Google Sheets.

**Two parallel data paths needed to merge before writing to Sheets.**
After the second SerpAPI request, the workflow splits into a product-details path and a seller-data path. Without merging, the downstream Google Sheets node (set to "Execute Once") received mismatched inputs and looped incorrectly. A Merge node unified both paths into a single array before the write step.

**Referencing individual Sheet rows in the AI prompt caused fragmented output.**
Feeding rows into the AI Agent one at a time produced a separate response per row instead of one combined report. An Aggregate node consolidated all scored ASINs into a single item, so the AI processed the full dataset in one call and returned one unified output.

**Uncontrolled SerpAPI credit usage during development.**
Testing against the full product list burned credits unnecessarily while debugging. Added a Limit node capped at 5 ASINs during development, reducing each test run's cost.

**AI output exceeded Telegram's message length limit.**
A single message containing all scored ASINs and reasoning was too long to send. A Code node parses the AI's text output, splits it into fixed-size batches, and formats each batch into its own message — avoiding the length error while keeping the report readable.

**Google Sheets formulas dragged down manually created false "empty" rows.**
Manually dragging formulas down created blank-but-formula-populated cells, which the Sheets API reported as real data — causing n8n's Get Row(s) node to pull dozens of empty rows. Replacing dragged formulas with `ARRAYFORMULA` fixed this: output now dynamically expands only as far as real source data exists.

---

## Limitations

- SerpAPI's purchase history data is limited to the last 30 days; no longer-term historical trend data is available through the API.
- Testing was capped at 20 ASINs per run to manage SerpAPI credit usage, so scaling behavior beyond that hasn't been fully verified.
- No error handling yet for incomplete or failed SerpAPI responses (e.g. validating returned row count against expected input).

## Next Steps

- Add error handling to validate SerpAPI's returned data against expected input counts.
- ~~Replace manually dragged Google Sheets formulas with `ARRAYFORMULA`~~ — **Completed.**
- Add a logistics data layer to pull shipping partner and fulfillment cost information, extending the workflow into landed-cost estimation.

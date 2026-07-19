1. PROBLEM
Researching Amazon products for resale typically means manually checking prices, ratings, reviews, and competitor counts one product at a time — a slow process that doesn't scale well when evaluating multiple product categories.
This workflow automates that research. It collects ASIN-level data — ratings, reviews, last month's purchase history, and listed prices — along with competitor seller data (pricing, ratings, seller count) into a centralized database. Researchers can then review this data in one place instead of gathering it manually per product.
A scoring mechanism built on top of this data flags which ASINs are worth pursuing and which aren't, based on competition level and market signals. This turns raw data into a direct decision-making input, rather than leaving researchers to interpret spreadsheet numbers on their own.

2. ARCHITECTURE

<img width="1325" height="607" alt="image" src="https://github.com/user-attachments/assets/20c3e8d7-e4e1-4dba-a4a9-1b895794377e" />
  
3. TOOLS USED

- n8n (self-hosted via Docker, exposed publicly through ngrok)
- SerpAPI
- Google Sheets
- JavaScript (Code node)
- Groq (Meta Llama 3.3 70B)

4. PROBLEMS SOLVED

- Seller data wasn't usable in its raw form. SerpAPI's other_sellers field returns nested data per product, not a flat list. I used a Split Out node to break this into individual fields (seller name, rating, reviews, price-per-unit), producing a clean tabular structure suitable for storing in Google Sheets.
- Two parallel data paths were producing separate outputs that needed to merge into one. After the second HTTP request, the workflow splits into a product-details path and a seller-data path. Without merging these, the downstream Google Sheets node — set to "Execute Once" — would receive mismatched inputs and loop incorrectly. Adding a Merge node unified both paths into a single array before writing to Sheets, resolving the mismatch.
- Referencing individual Sheet rows directly in the AI prompt caused fragmented, per-row outputs instead of one combined report. I used an Aggregate node to consolidate all scored ASIN rows into a single item before it reached the AI Agent. This meant the agent processed one complete dataset and returned one unified text output, rather than generating a separate response per row.
- Uncontrolled SerpAPI credit usage during development. Each test run against the full product list consumed credits unnecessarily while I was still debugging the workflow. I added a Limit node capped at 5 ASINs during development, reducing each test run to 5 credits instead of the full batch.
- The AI's combined output exceeded Telegram's message length limit. A single message containing all scored ASINs and their reasoning was too long to send in one Telegram message. I used a Code node to parse the AI's text output, split it into fixed-size batches, and format each batch into its own message — avoiding the length error while keeping the report readable.

5. LIMITATIONS AND NEXT STEPS
Known constraints:

- SerpAPI's purchase history data is limited to the last 30 days; no longer-term historical trend data is available through the API.
- Testing was capped at 20 ASINs per run to manage SerpAPI credit usage, so scaling behavior beyond that hasn't been verified.

Planned improvements:

- Add error handling to validate SerpAPI's returned row count against the expected input, to catch incomplete or failed API responses.
- Replace manually dragged Google Sheets formulas with ARRAYFORMULA in the Product Scoring sheet, so scoring ranges auto-expand as new ASINs are added without manual intervention. (Completed — see update below.)
- Add a logistics data layer to pull shipping partner and fulfillment cost information, extending the workflow beyond pure product research into landed-cost estimation.

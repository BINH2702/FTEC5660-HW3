# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

### Multi-agent workflow

The implementation uses a ReAct-style tool workflow: specialists retrieve evidence
and calculate or interview, then independent reviewers assess the evidence. The
private reasoning trace is not part of the output.

1. A quantitative analyst retrieves database evidence through `sql_query`,
   writes and runs a small Python calculation with the restricted calculator
   before reporting its findings. Point 1 uses the stated industry-median rule.
   For point 3, Python reports new-product sales per R&D and its relation to the
   industry median; no fixed multiplier is used to decide PASS or FAIL.
2. A scuttlebutt interviewer uses the interview tool for the chosen
   interview-based point. For point 7, a current employee is an appropriate
   respondent for `working_conditions`; one interview is sufficient and stays
   within the per-company limit.
3. Five separately prompted reviewer generations assess all three points through
   inductive, causal/abductive, analogical, probabilistic, and counterfactual
   lenses. Python aggregates each point by majority vote and reports the winning
   vote share as a consensus measure, not a calibrated probability; each review
   provides only a concise evidence-based rationale, not private chain-of-thought.
4. A deterministic Python selector chooses the target with the most PASS
   verdicts; ties favor more valid verdicts, then PASS on financial points 1 and
   3, then the alphabetically first folder. The selected target is noted in its
   results summary; all companies are still analyzed for grading. This avoids
   asking the language model to count or rank companies.

The analyst and interviewer work in parallel for each company; reviewer calls
are sequential within that company, and at most two companies are processed
concurrently. The model client retries transient request failures up to five
times. The calculator accepts only restricted Python expressions over JSON
data and disallows imports and arbitrary function calls.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
echo "DEEPSEEK_BASE_URL=https://www.moyu.cn/v1" >> .env
python3 hw3.py --data-folder public_test
```

The assignment specifies `deepseek-v4-flash`. To match the configured gateway,
`hw3.py` currently requests `DeepSeek-V4.1-Flash-1017`. This model differs from
the assignment's specified model and may not meet its model restriction;
confirm that this substitution is permitted before submission.
`DEEPSEEK_BASE_URL` defaults to `https://www.moyu.cn/v1` and can be overridden
in `.env`. Use only an API key authorized for the configured endpoint; do not
send a credential to an untrusted proxy. Never put a key in your code; `.env`
must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution

### My 3 points and rules

I chose points 1, 3, and 7 to assess growth potential, the productivity of
research investment, and employee relations. Together they use both quantitative
company data and first-hand scuttlebutt evidence.

| Fisher point | Why I chose it and PASS rule | Evidence |
| --- | --- | --- |
| **1 — Market potential for sales growth** | Fisher asks whether a company's products have the potential to support sizable sales growth. PASS when the company's 2023-to-2025 revenue growth is greater than its industry's `revenue_growth_2y_pct` median; otherwise FAIL. | SQL aggregates order revenue for 2023 and 2025 and retrieves the company's industry median. A restricted Python calculation computes the percentage growth and comparison. |
| **3 — Effectiveness of R&D** | R&D should translate into commercially meaningful products. Reviewers assess FY2025 sales from products launched in 2023–2025 relative to FY2025 R&D spending and the industry's `new_product_sales_per_rd` median. They consider the magnitude and context of the evidence rather than applying a fixed multiplier. | SQL aggregates FY2025 sales for qualifying products and R&D expense and retrieves the industry median. Python computes the ratio and comparison before the reviewer generations. |
| **7 — Labor and personnel relations** | Employee relations affect the company's ability to retain and motivate people. PASS when a current employee provides specific positive evidence, such as fair pay, low turnover, a good workplace, or concerns being addressed; no particular word such as “outstanding” is required. Clear negative evidence or no positive evidence results in FAIL. | One `current_employee` interview on `working_conditions`; this role is permitted to speak to the topic and avoids spending interviews on uninformed respondents. |

### Design

```text
                         per-company tools
                     ┌──────────┴──────────┐
                     │                     │
            SQL query tool          Interview tool
                     │                     │
        Quantitative analyst     Scuttlebutt interviewer
        points 1 and 3           point 7: current employee
        SQL → Python calculator  working_conditions
                     └──────────┬──────────┘
                                │ evidence package
                     5 reviewer generations
                 varied reasoning lenses per judgment
                                │
                    Python majority aggregation
                    verdicts + vote-share metric
                                │
            all companies → deterministic Python selector
```

For each company, the quantitative analyst retrieves database aggregates for
points 1 and 3 and must use the restricted Python calculator before reporting
the results. In parallel, the interviewer makes exactly one topic-matched
interview for point 7. Five reviewer generations receive these same verified facts but use different
reasoning lenses. Python aggregates their verdicts for each point by majority
and reports the winning-vote share as a simple consensus metric, not a
probability estimate. Since all generations use the same model and evidence,
their errors may be correlated. A rationale from a majority review is retained
for each point. After all company analyses finish, Python selects the target by PASS count; ties are
broken by valid verdict count, then PASS on points 1 and 3, then the
alphabetically first folder. This makes aggregation and portfolio selection
reproducible while keeping calculations out of the language model.

### Results

The latest complete public-test run, before adding reviewer aggregation, scored
54/60 (balanced accuracy: 83.3%). Point 1 was 20/20 and point 7 was 20/20.
Point 3 was 14/20: Aurel Systems, Driftwood Systems, Dunmore Holdings, Halcyon
Works, Ironbark Technologies, and Lindale Holdings were incorrectly marked
PASS. The previous point-3 rule over-relied on a single numeric comparison.
The current implementation removes that fixed multiplier and aggregates five
independent evidence-based judgments. This ensemble has not yet been run on the
public set; rerun `python3 hw3.py --data-folder public_test` to measure its
accuracy and consensus.

## Task 2: Reflection on AI Sycophancy and AI Psychosis

AI sycophancy is a tendency for a model to agree with a user's framing or apparent
preferences instead of evaluating evidence independently. In an investment
analysis, this could turn an optimistic prompt or management's confident claims
into an unjustifiably positive verdict. The agent in this homework reduces that
risk by using explicit source-specific rules, retrieving company evidence through
the provided tools, calculating financial comparisons in Python, and using an
employee interview for labor relations rather than relying on management's
narrative alone. These safeguards do not eliminate sycophancy: the interviewer
answer can be ambiguous, and reviewers can still overstate what the specialists
found. A stronger system would preserve source excerpts, label
uncertainty, and test conclusions against counter-evidence before presenting
them.

“AI psychosis” is sometimes used informally to describe harmful interactions in
which AI-generated claims reinforce a person's delusions or paranoia. It is not
a clinical diagnosis of an AI system, and it should not be used to dismiss or
stigmatize people experiencing mental-health difficulties. The relevant design
lesson is that fluent, consistent text is not proof that a claim is true. In
this homework, several agents could accidentally repeat the same unsupported
assumption and make it appear corroborated. Restricting factual claims to
retrieved evidence, keeping interview and database findings distinguishable,
and returning an explicit failure when evidence is missing help limit that
failure mode. For real investment decisions, the output should remain a
screening aid—not a substitute for independent research or professional
judgment—and should not imply certainty beyond the evidence.

Overall, these risks are not solved by adding more agents or making answers
sound more confident. They require clear evidence provenance, calibrated
uncertainty, independent checks, and human oversight proportionate to the
decision's impact.

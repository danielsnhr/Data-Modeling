
# Data Modeling

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Power Query](https://img.shields.io/badge/Power%20Query-217346?style=for-the-badge)
![DAX](https://img.shields.io/badge/DAX-0078D4?style=for-the-badge)

**A dashboard is only as good as the data model behind it.**  
If the model is wrong, the reports are wrong, and the numbers stakeholders rely on stop making sense.

This project takes a messy 23-table business data model and rebuilds it into one that delivers numbers people can trust.

---
## Repository Structure

```text
data-modeling/
│
├── README.md                       # Project overview and technical documentation
│   ├── The Problem                 # Challenges with the original 23-table model
│   ├── The Solution                # Kimball-style dimensional modeling approach
│   ├── Before vs. After            # Comparison of original and redesigned models
│   ├── Model at a Glance           # Overview of fact and dimension tables
│   ├── How I Approached It         # Methodology and 21 key design decisions
│   ├── How I Kept the Numbers Right # Data validation and troubleshooting
│   ├── Results and Limitations     # Analytical capabilities and limitations
│   └── Source and Credit           # Dataset attribution and learning resources
│
├── 1) data-model                   # Power BI project with the redesigned data model
│ 
│
└── images/
    ├── before_model.png            # Original 23-table operational data model
    ├── after_model.png             # Final fact constellation (galaxy schema)
    └── table_list.png              # Final fact, dimension, and support tables
```
---

## The Problem
<img width="806" height="502" alt="Screenshot 2026-11-20 155859" src="https://github.com/user-attachments/assets/bda39d44-a1e2-4790-9076-d850bfc4620d" />

The model I started with, at a glance:

- **Tangled connections:** tables linked in many-to-many relationships, with data filtering in every direction
- **No clear structure:** it was hard to tell how the data fit together, and names and formats were inconsistent
- **Untrustworthy numbers:** duplicated and mismatched records quietly inflate totals
- **Wrong reports:** stakeholders would be making decisions on figures that don't add up
- **No security:** every user could see all of the data

---

## The Solution
<img width="661" height="367" alt="Screenshot 2026-11-20 155917" src="https://github.com/user-attachments/assets/c367603d-5ddc-4cd2-9bdf-c753a1b6870a" />


The rebuilt model:

I rebuilt the model as a **star schema** using **Kimball-style dimensional modeling**, with **Power Query** for data transformation and **DAX** for the measures.

- **Clean connections (star schema / fact constellation):** 23 tables became 6 dimension and 6 fact tables, plus security and measures tables. All relationships are one-to-many with a single filter direction, and fact tables connect only through shared (conformed) dimensions
- **Clear structure (dimensional modeling):** every table has a defined role (dimensions describe, facts record events), with surrogate keys and consistent naming (`dim_` / `fact_` prefixes, snake_case)
- **Trustworthy numbers (data validation):** total sales was checked after every change, and an inflated total was traced to duplicate product records and fixed at the source
- **Reliable reports (centralized DAX measures):** one agreed set of measures that report builders can reuse
- **Secure by design (row-level security, RLS):** dynamic RLS by region, tested with "View as", so each user sees only their own data

**Techniques used:** Dimensional modeling · Star schema · Fact constellation · Surrogate keys · Junk dimension · Role-playing dimension · Bridge (factless fact) table · Accumulating snapshot fact · Header/detail resolution · Data cleansing · Power Query · DAX · Row-level security


---

## Explore the Project

1. [Before vs. After](#1-before-vs-after): what changed, side by side
2. [Model at a Glance](#2-model-at-a-glance): every table and its role
3. [How I Approached It](#3-how-i-approached-it): the four phases and key design decisions
4. [How I Kept the Numbers Right](#4-how-i-kept-the-numbers-right): validation and the problems it caught
5. [Results](#5-results): what the model answers and its limitations


---

## 1. Before vs. After

| Area | Before | After |
|---|---|---|
| Naming | Mixed styles: `ORDERS_2025`, `CUST_MASTER`, `Sheet1`, generic `Column1` headers | snake_case, `dim_` / `fact_` prefixes, `_key` suffix for keys created in the model |
| Total tables | 23 raw tables | 6 dimensions + 6 facts, plus `_measures` and `security` |
| Relationships | Many-to-many, bidirectional, overlapping paths | One-to-many, single direction; facts connect only through shared dimensions |
| Customer data | Spread across 6 tables (master, contacts, user details, addresses, cities, regions) | One `dim_customer` with 60 customers |
| Orders | Split by year, with header and detail tables separate | Appended into one table; `fact_sales` at order-line level |
| Join keys | Text names (customer name, product name) | Surrogate keys (`product_key`, `geo_key`, `flag_key`, `campaign_key`) plus `customer_id` |
| Order lifecycle | Orders, shipments, invoices and payments in separate tables | One `fact_order_process` row per order with five milestone dates |
| Inventory | One column per month | One row per product per month |
| Dates | No date table, mixed formats | `dim_date` shared across facts, one date format |
| Measures | None | Central `_measures` table |
| Security | None | Row-level security by region |
| Data quality | Test records, duplicates, empty and redundant columns | Removed or fixed (see [How I Kept the Numbers Right](#4-how-i-kept-the-numbers-right)) |

<details>
<summary><b>See all 23 changes</b></summary>

| # | Before | After | Concept |
|---|---|---|---|
| 1 | 23 raw tables, many-to-many and bidirectional relationships | 6 dimensions + 6 facts, all one-to-many with a single filter direction, no fact-to-fact links | Star schema / fact constellation, conformed dimensions |
| 2 | Customer data spread over 6 tables (master, contacts, user details, addresses, cities, regions) | One `dim_customer` with 60 rows | Dimension consolidation, snowflake flattened |
| 3 | Several contacts per customer (merging them multiplied customer rows) | Primary contact only, one row per customer | Grain alignment before joining |
| 4 | Category and subcategory stored together in a pipe-delimited column, no header; no product ID, only a product code | Column split, text case matched, merged into the product table; `product_key` created | Parsing, surrogate key |
| 5 | Duplicate products (same name, two codes, some with missing brand, price and supplier) | Incomplete duplicates filtered out, one row per product | Deduplication, referential integrity |
| 6 | Orders split into `ORDERS_2025` and `ORDERS_2026` | Appended into one table; empty and legacy columns removed | Append |
| 7 | Separate order header and order detail tables | `fact_sales` at order-line grain, with header context left-joined in | Header/detail (parent-child) pattern |
| 8 | Joins on text names (customer name, product name) | Surrogate keys (`product_key`, `flag_key`, `geo_key`, `campaign_key`) plus `customer_id` | Surrogate keys, lookups |
| 9 | Channel code, status and priority repeated on every order; channel stored as an unexplained number | `dim_order_flags` (14 combinations), channel codes decoded through a mapping table | Junk dimension, data enrichment |
| 10 | Ship-to and bill-to city stored as text on the order | `dim_geo` with two relationships to `fact_sales` (one active, one inactive) | Role-playing dimension |
| 11 | Pre-aggregated order total, hash keys, source IDs, product descriptions, free-text notes, customer city and region repeated on orders | Removed | Single source of truth, column pruning |
| 12 | Wide inventory sheet: one column per month, headers stored as the first row | Header promoted, unpivoted, dates typed, product key added: `fact_inventory` | Unpivot |
| 13 | `CAMPAIGN_LOG` mixed campaign attributes with daily metrics (budget repeated on every day) | `dim_campaign` (6 rows) + `fact_campaign_spend` | Splitting a mixed-grain table |
| 14 | Promoted products stored as a comma-separated list in one cell | Exploded to rows, whitespace trimmed, keys added: `fact_promotion_coverage` | Bridge / factless fact |
| 15 | Orders, shipments, invoices and payments as separate tables with the same revenue repeated | `fact_order_process`: one row per order with five milestone dates | Accumulating snapshot fact |
| 16 | `sales_targets` standalone (period and target revenue) | `fact_sales_targets` connected to `dim_date` | Plan-vs-actual |
| 17 | `Sheet1` (duplicate of shipments), `dim_order` (one meaningless column), `exchange_rates` (nothing to connect it to) | Deleted, deleted, disabled; invoices, payments, shipments and invoice lines disabled once their dates were captured | Scope discipline |
| 18 | Test records (customer 999, a dummy product) | Removed (61 → 60 customers) | Data cleansing |
| 19 | No date table | `dim_date` via `CALENDARAUTO()`, shared by the facts; extra date relationships only where needed | Shared dimension, self-extending calendar |
| 20 | Mixed naming, text case, date formats and summarization defaults | snake_case, `dim_` / `fact_` prefixes, `_key` suffix, capitalized text values, one date format, "Don't summarize" on columns that shouldn't be summed | Modeling standards |
| 21 | No measures | `_measures` table: total sales, distinct count of orders, active vs. total customers, order-to-pay days and its average | Centralized DAX, correct grain handling |
| 22 | No security | Dynamic RLS: `security` → `dim_customer` → `fact_sales` and `fact_order_process`, using `USERPRINCIPALNAME()`, tested with "View as" | Row-level security |
| 23 | All raw queries in one flat list | Power Query layers: Stage → Dimensions → Facts → Support; raw tables kept as references with load disabled | Lineage, staging layer |

</details>

[↑ Back to top](#data-modeling)

---

## 2. Model at a Glance

| Table | Type | One row per |
|---|---|---|
| `dim_customer` | Dimension | Customer (60), merged from 6 source tables |
| `dim_product` | Dimension | Product, with brand, category, subcategory and supplier |
| `dim_geo` | Dimension | City, with region |
| `dim_campaign` | Dimension | Marketing campaign (6) |
| `dim_order_flags` | Dimension (junk) | Combination of order channel, status and priority (14) |
| `dim_date` | Dimension | Day; built with `CALENDARAUTO()` so it extends itself as data grows |
| `fact_sales` | Fact | Order line |
| `fact_order_process` | Fact (accumulating snapshot) | Order, with order, ship, delivery, invoice and payment dates |
| `fact_inventory` | Fact | Product per month |
| `fact_campaign_spend` | Fact | Campaign per day (impressions, clicks, spend) |
| `fact_sales_targets` | Fact | Month (target revenue) |
| `fact_promotion_coverage` | Fact (factless / bridge) | Campaign and product pair |
| `_measures` | Measures | Reusable DAX measures |
| `security` | Support | User and region, driving row-level security |

[↑ Back to top](#data-modeling)

---

## 3. How I Approached It

1. **Explore:** read every table, asked what each row represents (its grain), and sorted tables into descriptive data (dimensions) and events (facts)
2. **Build the dimensions:** merged related tables into clean dimensions, fixing grain, keys and duplicates along the way
3. **Build the facts:** turned event tables into fact tables with surrogate keys, keeping only what reporting needs
4. **Polish and secure:** enforced naming and format standards, added the date dimension and DAX measures, applied row-level security and tested the final numbers

<details>
<summary><b>See the 21 key design decisions</b></summary>

| # | Decision | Alternative | Why |
|---|---|---|---|
| 1 | Star schema; facts never connect to each other, only through shared dimensions | Link fact tables directly on a common key such as `order_id` | Direct fact-to-fact links cause fan-out and wrong totals; shared dimensions keep filters predictable |
| 2 | Merge customer, contacts, user details, addresses and cities into one `dim_customer` | Keep a separate address dimension (snowflake) | Keeps a true star schema; address data is small and one-to-one with the customer |
| 3 | Keep only the primary contact per customer | Aggregate contacts into a list, or build a separate contact dimension | Keeps the grain at one row per customer; verified by count (still 60 rows) |
| 4 | Call the entity "customer", not "user" | Rename around the user-details table | Most source tables already use "customer"; one name for one entity |
| 5 | Build `fact_sales` from the order lines, using the order header only for context | Two facts (header and lines) linked together | The numbers live in the lines, and lines can always be rolled up to the order; avoids linking two facts |
| 6 | Add `product_key` as a surrogate key | Join on product code or name | The source has no product ID; text keys caused fan-out |
| 7 | Remove the incomplete duplicate products by source ID | Ask the source owner to clean the data, or add a data-quality pipeline | Dataset is small and the incomplete rows were clearly the bad copies; in production the source fix is preferable |
| 8 | Combine channel, status and priority into a junk dimension (`dim_order_flags`) | One dimension per attribute, or leave them on the fact | These attributes are unrelated and low-cardinality; one small dimension keeps the fact narrow |
| 9 | Decode channel codes with a hand-entered mapping table | Show raw codes (10, 20, 30, 40) | Numbers can be mistaken for measures; a mapping gives meaningful names, at the cost of manual upkeep |
| 10 | Keep ship-to and bill-to city on the sales fact, as keys to `dim_geo` | Remove them like the customer's master city | They change per order (transactional), unlike the customer's master location |
| 11 | One `dim_geo` used twice (ship-to active, bill-to inactive) | Duplicate the dimension | Avoids duplicate data; inactive relationships are activated in DAX when needed |
| 12 | Drop pre-aggregated totals, hash keys, source IDs, descriptions and free text | Keep them "just in case" | Smaller model, faster refresh, fewer ways to report the wrong number; anything dropped can be brought back |
| 13 | Split the campaign log into `dim_campaign` and `fact_campaign_spend` | Keep one table | Budget and dates describe the campaign (static); impressions, clicks and spend are daily events |
| 14 | `fact_promotion_coverage` as a bridge between campaigns and products | A third "campaign-product" dimension | It describes a relationship between two entities, not an entity of its own |
| 15 | One accumulating-snapshot fact for the order lifecycle | One fact per step (orders, shipments, invoices, payments) | The same revenue appeared in each step; the business question is about timing between steps |
| 16 | Disable invoices, payments, shipments and invoice lines after taking their dates | Load them as separate facts | Keeps the core model focused; they can be re-enabled for a specific need |
| 17 | `sales_targets` as a small fact connected to `dim_date` | Leave it standalone | Enables target vs. actual through the date dimension |
| 18 | Build `dim_date` with `CALENDARAUTO()` and connect it only where needed | Hand-maintained calendar table; connect every date column | The calendar extends itself as data grows; extra date relationships need extra DAX |
| 19 | Centralize measures in `_measures`; count orders with `DISTINCTCOUNT` | Measures scattered across tables; plain count | One definition per measure; the fact is at line grain, so a plain count would overstate orders |
| 20 | Apply RLS on `dim_customer` | Put it on `dim_geo`, or add region to every fact | `dim_customer` filters two facts (sales and order process); avoids merging region into facts and breaking their grain |
| 21 | Disable `exchange_rates` and drop the unused tables | Load everything | It had nothing to connect to in the model; scope kept to what reporting needs |

</details>

[↑ Back to top](#data-modeling)

---

## 4. How I Kept the Numbers Right

Before touching the fact table I recorded the total sales figure as a control total, and re-checked it after every merge. Joins were always left joins, and uniqueness was verified by counting rows per key.

Two problems were caught this way:

- **Customer contacts:** merging all contacts into the customer table multiplied the rows (60 customers became far more) because contacts and customers have different grains. Fixed by keeping only the primary contact, which restored exactly 60 rows.
- **Duplicate products:** joining sales to products by name inflated total sales. The same product existed twice under different codes, one copy with missing brand, price and supplier. I removed the incomplete duplicates and re-verified uniqueness, and the total returned to its original value.

[↑ Back to top](#data-modeling)

---

## 5. Results 

### What the model can answer

- Sales vs. monthly targets vs. inventory levels, side by side, through the shared date dimension
- How long orders take from order to ship, delivery, invoice and payment
- Campaign spend, impressions and clicks per campaign, and which products each campaign promoted
- Total and active customers, and distinct order counts at the correct grain
- The same report for every user, but each user sees only their own region

### Limitations

- `exchange_rates` was not loaded into the model, so there is no currency conversion
- Invoice and payment amounts, shipment mode and invoice line details were not carried over, because the same revenue is already in `fact_sales`; only their milestone dates are kept
- `fact_promotion_coverage` has no date or measure, so it works as a bridge between campaigns and products
- The channel code mapping was entered by hand and needs updating if the source adds new codes
- `dim_date` is minimal (year and month)

[↑ Back to top](#data-modeling)


Built as a guided project from [Built with Data With Baraa]

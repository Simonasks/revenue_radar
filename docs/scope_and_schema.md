# Revenue Radar - Scope & Schema

# Table of Contents

- [1. Business Scope](#business-scope-v1)
- [2. KPIs (Definitions and Formulas)](#kpis-definitions-and-formulas)
- [3. Star Schema (Grains, Keys, Attributes)](#star-schema-grains-keys-attributes)
- [4. Data Contract (v1)](#data-contract-v1)

*This document defines the analytical foundation of the Revenue Radar project — 
covering its business scope, key metrics, data model, and delivery contract.*

## Business scope (v1)

- **Domain:** Consumer e-commerce (fictional shop).
- **Unit of value:** Customer orders paid in **SEK**.
- **Reporting timezone:** Europe/Stockholm (timestamps stored in UTC; displayed in local time).
- **Primary time grain:** Daily; standard window = last 28 days (plus WoW comparisons).
- **Geography:** Sweden (field `country` on customers/orders; can expand later).
- **Sales channels:** Web and Mobile App (field `channel`).
- **Product model:** SKU-level items with categories and unit **COGS** (cost of goods sold).
- **Commercial rules we must support:**
  - Discounts: item-level and order-level (order-level allocated to items pro-rata by gross line value).
  - Promotions: `promo_code` with start/end validity windows.
  - Returns: full or partial; reduce revenue and margin; tracked at item level via `returns_qty`.
  - Order statuses (normalized): `pending`, `paid`, `shipped`, `delivered`, `returned`, `cancelled`.

**Synthetic source tables (daily drops):**
- `orders` — one row per order.
- `order_items` — one row per SKU in an order.
- `customers` — one row per customer.
- *(later)* `products`, `promotions`, `returns` as their own feeds if we split logic.

## KPIs (definitions and formulas)

### 1. Revenue
**Definition:** Total value of goods sold, after discounts and returns.  
**Formula logic:**
- `revenue_gross = Σ(qty * price_unit)`
- `discount_total = Σ(qty * discount_item) + discount_order_allocated`
- `revenue_net_pre_returns = revenue_gross - discount_total`
- `returns_value = Σ(returns_qty * (price_unit - discount_item - alloc_order_disc_per_unit))`
- `revenue_net = revenue_net_pre_returns - returns_value`

**Business note:**  
Revenue is only recognized for orders with status in `paid`, `shipped`, or `delivered`.

---

### 2. Orders
**Definition:** Count of completed orders (not cancelled).  
**Formula logic:**
- `orders = COUNT(DISTINCT order_id)`  
  where `status ∈ {paid, shipped, delivered}`

---

### 3. AOV (Average Order Value)
**Definition:** Average revenue per order.  
**Formula logic:**  
- `AOV = revenue_net / orders`

**Business note:**  
Shows purchase value per transaction; used to track impact of promos and pricing.

---

### 4. Margin
**Definition:** Profit after subtracting cost of goods sold (COGS).  
**Formula logic:**
- `margin_item = qty * (price_unit - discount_item - alloc_order_disc_per_unit - cogs_unit)`
- `margin = Σ(margin_item) - returns_margin_adjustment`
- `margin_pct = margin / revenue_net`

**Business note:**  
Reflects efficiency of pricing and promotions.

---

### 5. Repeat Rate
**Definition:** Percentage of customers who placed more than one order in the period.  
**Formula logic:**
- `repeat_rate = customers_with_≥2_orders / customers_with_≥1_order`

**Business note:**  
Used to measure customer loyalty and retention.

---

### General KPI conventions
- **Currency:** SEK  
- **Time grain:** Daily, aggregated into rolling 28-day windows for trends.  
- **Negative or zero values:** Clamp at 0 (e.g., no negative revenue).  
- **Returns:** Always reduce revenue and margin.  
- **Discount allocation:** Order-level discounts distributed to items pro-rata by line value.

## Star schema (grains, keys, attributes)

### Facts (measures live here)

**gold.fct_orders**  
- **Grain:** 1 row per `order_id` (completed orders only: paid/shipped/delivered)  
- **Keys:** `order_sk`, `customer_sk`, `order_date_sk`, `channel_sk`, `promo_sk`  
- **Natural keys kept for lineage:** `order_id`, `customer_id`  
- **Measures:**  
  - `orders = 1`  
  - `revenue_gross`, `discount_total`, `revenue_net_pre_returns`, `returns_value`, `revenue_net`  
  - `margin`, `margin_pct`, `aov_at_order` (optional)  
- **Timestamps:** `order_ts`, `paid_ts` (UTC)  
- **Filters:** `status IN ('paid','shipped','delivered')`

**gold.fct_order_items**  
- **Grain:** 1 row per (`order_id`, `item_id`)  
- **Keys:** `order_item_sk`, `order_sk`, `product_sk`, `customer_sk`, `order_date_sk`, `promo_sk`  
- **Natural keys:** `order_id`, `item_id`, `sku`, `customer_id`  
- **Measures:**  
  - `qty`, `price_unit`, `discount_item`, `alloc_order_disc_per_unit`  
  - `revenue_net_item`, `cogs_unit`, `margin_item`  
  - `returns_qty`, `returns_value_item`  
- **Derivations:** order-level discount allocated **pro-rata** by gross line value.

---

### Dimensions (context lives here)

**gold.dim_date**  
- **Grain:** 1 row per calendar date  
- **Key:** `date_sk` (YYYYMMDD int)  
- **Attrs:** `date`, `year`, `quarter`, `month`, `day`, `dow`, `iso_week`, `is_weekend`

**gold.dim_customers** *(SCD Type 2)*  
- **Grain:** 1 row per customer **version**  
- **Keys:** `customer_sk` (surrogate), `customer_id` (natural)  
- **Attrs:** `country`, `signup_date_sk`, optional demographics  
- **SCD:** `is_current`, `valid_from`, `valid_to`

**gold.dim_products** *(SCD Type 1 is fine for this project)*  
- **Keys:** `product_sk`, `sku`  
- **Attrs:** `name`, `category`, `cogs_unit`

**gold.dim_channel**  
- **Keys:** `channel_sk`  
- **Attrs:** `channel` ∈ {web, app}

**gold.dim_promo**  
- **Keys:** `promo_sk`  
- **Attrs:** `promo_code`, `type`, `start_date_sk`, `end_date_sk`

*(Optional)* **gold.dim_status** for normalized order status codes.

---

### Keys & conventions

- **Surrogate keys (`*_sk`)** generated in **Silver v2**:  
  - deterministic hash (e.g., `sha2(lower(trim(natural_key)),256)` → bigint) or incremental ID  
- **Natural keys (`*_id`, `sku`)** kept in facts for lineage and troubleshooting  
- **Naming:** snake_case; measures are decimals with 2 dp (currency) or integers (qty)

---

### Partitions & performance (later in the plan)

- Partition facts by `order_date_sk` (daily)  
- Consider Z-ORDER on `customer_sk`, `product_sk` for item fact  
- Add table comments/owners via Unity Catalog for governance

---

### Integrity & expectations (enforced in Silver/DLT)

- **Not null:** all IDs, timestamps, and SKs  
- **Positive:** `qty > 0`, `price_unit ≥ 0`, `cogs_unit ≥ 0`, discounts ≥ 0  
- **Enums:** `status ∈ ('pending','paid','shipped','delivered','returned','cancelled')`  
- **RI:** every `order_items.order_id` must exist in `orders`

---

### Quick ER sketch (Mermaid)

```mermaid
erDiagram
  dim_date ||--o{ fct_orders : "order_date_sk"
  dim_date ||--o{ fct_order_items : "order_date_sk"
  dim_customers ||--o{ fct_orders : "customer_sk"
  dim_customers ||--o{ fct_order_items : "customer_sk"
  dim_products ||--o{ fct_order_items : "product_sk"
  dim_channel ||--o{ fct_orders : "channel_sk"
  dim_promo ||--o{ fct_orders : "promo_sk"
  fct_orders ||--o{ fct_order_items : "order_sk"

  ## Data Contract (v1)

### Scope
This contract defines the structure and expectations for daily synthetic data deliveries
used in the *Revenue Radar* project. It applies to the tables:
`orders`, `order_items`, and `customers`.

### Delivery guarantees
- **Frequency:** Daily, by 06:00 local time.
- **Destination path:** `/raw/ingest_date=YYYY-MM-DD/`
- **Format:** CSV, UTF-8, with a header row.
- **Delimiter:** Comma (`,`).
- **File naming convention:** `{table}_{YYYYMMDD}.csv`
- **Timestamps:** Stored in UTC; reported in `Europe/Stockholm`.
- **Currency:** SEK (3-letter ISO code, dot decimal, 2 decimal places).

---

### Schemas

**orders**
| Column | Type | Description | Constraints |
|---------|------|-------------|--------------|
| order_id | string | Unique order ID | Unique, non-null |
| customer_id | string | Customer reference | FK → `customers.customer_id` |
| order_ts | timestamp | Order created time (UTC) | Non-null |
| status | string | Order status | Enum: pending / paid / shipped / delivered / returned / cancelled |
| discount_order | decimal(10,2) | Total order-level discount | ≥ 0 |
| promo_code | string | Applied promo code | Nullable |
| country | string | ISO country code | Non-null |

**order_items**
| Column | Type | Description | Constraints |
|---------|------|-------------|--------------|
| order_id | string | Order reference | FK → `orders.order_id` |
| item_id | string | Unique line item ID per order | Unique per order, non-null |
| sku | string | Product SKU | Non-null |
| qty | int | Quantity ordered | > 0 |
| price_unit | decimal(10,2) | Unit price before discount | ≥ 0 |
| discount_item | decimal(10,2) | Per-item discount | ≥ 0 |
| cogs_unit | decimal(10,2) | Cost of goods sold per unit | ≥ 0 |
| returns_qty | int | Quantity returned | ≥ 0 and ≤ qty |

**customers**
| Column | Type | Description | Constraints |
|---------|------|-------------|--------------|
| customer_id | string | Unique customer ID | Unique, non-null |
| signup_ts | timestamp | Signup timestamp (UTC) | Non-null |
| country | string | ISO country code | Non-null |
| channel | string | Acquisition channel (web/app/other) | Nullable |

---

### Constraints & expectations
- **Uniqueness:**  
  - `orders.order_id` unique  
  - (`order_items.order_id`, `item_id`) unique  
  - `customers.customer_id` unique
- **Null handling:**  
  IDs and timestamps must be non-null.
- **Enums:**  
  `status` ∈ {pending, paid, shipped, delivered, returned, cancelled}
- **Numeric ranges:**  
  `qty > 0`; all prices/discounts/COGS ≥ 0; `returns_qty ≤ qty`
- **Referential integrity:**  
  Every `order_items.order_id` must exist in `orders.order_id`.

---

### Error handling
- Rows violating constraints are quarantined to `silver.rejections` with columns:
  `error_reason`, `source_file`, `ingest_date`, `raw_payload`.
- The pipeline is **idempotent**: re-delivery of the same file overwrites existing data for that date.

---

### Notes
This contract will evolve as the project introduces additional entities
(e.g., `products`, `promotions`, or `returns` as separate feeds).
Version each major change as `v2`, `v3`, etc., and update downstream expectations accordingly.

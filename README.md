# Insurance — SQL Practice & Migration

**What This Does**
This project builds a healthcare insurance schema in SQLite, seeds sample data, runs analytical queries, and demonstrates migration to PostgreSQL using Python.

**Files**
- **`sql.ipynb`**: Complete implementation (schema → data → queries → migration)
- **`requirements.txt`**: Python dependencies (`pandas`, `sqlalchemy`, `psycopg2-binary`, `ipython-sql`)
- **`myenv/`**: Virtual environment (optional)



**How It Works (Detailed)**

1. **Schema Creation**
   - **Action:** `CREATE TABLE` for `plans`, `members`, `providers`, `enrollments`, `claims`, `claim_lines`.
   - **How:** Defines a relational model linking members to plans, providers to enrollments, and claims to line items.
   - **Why:** Establishes a realistic healthcare structure for queries and migration testing.

2. **Data Seeding**
   - **Action:** Multi-row `INSERT` statements populate all tables with representative rows (paid/denied claims, in/out-of-network providers).
   - **How:** Each table gets example data so queries return realistic results.
   - **Why:** Enables testing of aggregations, joins, and migration accuracy.

3. **Indexing**
   - **Action:** `CREATE INDEX` on frequently-filtered columns (`member_id`, `claim_status`, `provider_id`).
   - **How:** Indexes speed up WHERE and JOIN conditions in queries.
   - **Why:** Demonstrates query optimization and shows how index syntax differs between SQLite and Postgres.

4. **Example Queries**
   - **Action:** SELECT queries for member lookups, claim histories, denied claims, aggregations (totals by member/provider/month).
   - **How:** Uses JOINs, GROUP BY, and WHERE to extract analytical insights.
   - **Why:** Validates schema and provides pre-migration baseline for comparison.

5. **Migration to PostgreSQL**
   - **Action:** 
     - Use `sqlalchemy.inspect()` to enumerate SQLite tables.
     - Read each table: `pd.read_sql_table(table_name, sqlite_engine)` → DataFrame.
     - Write to Postgres: `df.to_sql(table_name, pg_engine, if_exists='replace')`.
   - **How:** Pandas automatically handles type conversion; SQLAlchemy manages connection pooling and dialect differences.
   - **Why:** Simple, readable, and handles small-to-medium datasets efficiently.

6. **Post-Migration Validation**
   - **Action:** Compare row counts and key aggregates (sums, counts) between SQLite and Postgres.
   - **How:** Run identical SELECT queries on both databases and compare results.
   - **Why:** Catches missing data, type truncation, or transformation errors.

**Quick Start**
```bat
python -m venv myenv
myenv\Scripts\activate.bat
pip install -r requirements.txt
jupyter lab
```
Open `sql.ipynb`, then run cells in order.

---

## Database Schema & Tables (Detailed)

### 1. **plans** — Insurance Products
**Structure:**
- `plan_id` (INT, PK): Unique plan identifier.
- `plan_name` (TEXT): Product name (e.g., "Blue Advantage Silver 70").
- `insurer` (TEXT): Insurance company (Blue Cross, UnitedHealthcare, etc.).
- `plan_type` (TEXT): HMO, PPO, POS.
- `metal_level` (TEXT): Bronze, Silver, Gold (cost-sharing tier).
- `monthly_premium` (DECIMAL): Monthly cost to member.

**Purpose:** Catalog of insurance products members can enroll in.

**Data Flow:** One row per plan. Members choose a plan during enrollment → links to `enrollments` table.

**Key Queries:**
- `SELECT * FROM plans;` — Show all available plans (helps members choose).
- `SELECT plan_name, monthly_premium FROM plans ORDER BY monthly_premium;` — Compare costs.

**Insights:** 
- Which plans are most affordable (Bronze tier = lowest premium).
- Premium differences between insurers (UnitedHealthcare Gold vs. Anthem Bronze).

---

### 2. **members** — Insured Individuals
**Structure:**
- `member_id` (INT, PK): Unique person identifier.
- `first_name`, `last_name` (TEXT): Name.
- `dob` (DATE): Date of birth (age calculation, pediatrics flags).
- `gender` (TEXT): M, F, O (used for specialty matching, e.g., OB/GYN).
- `phone`, `email` (TEXT): Contact info.
- `join_date` (DATE): When they joined the plan.

**Purpose:** Represents insured people (subscribers + dependents).

**Data Flow:** One row per unique person. Linked to `enrollments` (which plan they're on) and `claims` (their medical bills).

**Key Queries:**
- `SELECT member_id, first_name, last_name FROM members WHERE first_name = 'Sarah';` — Find member.
- `SELECT * FROM members WHERE gender = 'F';` — Segment by gender.

**Insights:**
- Member demographics (age, gender distribution).
- Contact information for outreach/notifications.
- Identify high-risk age groups (e.g., members over 65).

---

### 3. **providers** — Doctors & Hospitals
**Structure:**
- `provider_id` (INT, PK): Unique provider identifier.
- `provider_name` (TEXT): Doctor or hospital name.
- `npi` (TEXT, UNIQUE): National Provider Identifier (medical credential).
- `specialty` (TEXT): Family Medicine, Cardiology, OB/GYN, etc.
- `city` (TEXT): Location.
- `in_network` (INT): 1 = covered by insurance, 0 = out-of-network (likely denied).

**Purpose:** Directory of healthcare providers and their coverage status.

**Data Flow:** One row per provider. Linked to `claims` (which provider did the service) and `enrollments` (provider network varies by plan).

**Key Queries:**
- `SELECT provider_name, specialty FROM providers WHERE city = 'Chicago' AND in_network = 1;` — Find covered doctors nearby.
- `SELECT * FROM providers WHERE in_network = 0;` — Out-of-network providers (common denial reason).

**Insights:**
- Network adequacy (how many in-network specialists available).
- Out-of-network leakage (members using out-of-network providers → higher denials).
- Geographic coverage gaps (few providers in rural areas).

---

### 4. **enrollments** — Member-to-Plan Mapping
**Structure:**
- `enrollment_id` (INT, PK): Unique enrollment record.
- `member_id` (FK): Which member.
- `plan_id` (FK): Which plan they're enrolled in.
- `start_date` (DATE): Coverage begins.
- `end_date` (DATE, nullable): Coverage ends (NULL = still active).
- `status` (TEXT): Active, Terminated, Pending.

**Purpose:** Links members to plans; tracks coverage periods (enables "who was covered when?" queries).

**Data Flow:** Many-to-many relationship. Member 1 can enroll in Plan A (Jan–Aug 2024), then Plan B (Sep 2024–now). Eligibility checks use this table.

**Key Queries:**
- `SELECT m.first_name, p.plan_name, e.start_date, e.status FROM members m JOIN enrollments e ON m.member_id = e.member_id JOIN plans p ON e.plan_id = p.plan_id WHERE e.status = 'Active';` — Active members + their plan.
- `SELECT * FROM enrollments WHERE member_id = 1 ORDER BY start_date;` — Member's enrollment history.

**Insights:**
- Plan switching patterns (members leaving Bronze for Gold, indicating dissatisfaction).
- Churn rate (terminated enrollments).
- Coverage verification (critical for claim eligibility; was member covered on claim date?).

---

### 5. **claims** — Medical Bills (Header)
**Structure:**
- `claim_id` (INT, PK): Unique claim.
- `member_id` (FK): Who got the service.
- `provider_id` (FK): Who provided it.
- `claim_date` (DATE): When submitted.
- `service_date` (DATE): When service occurred.
- `submitted_amount` (DECIMAL): What provider charged.
- `allowed_amount` (DECIMAL): What insurance plan allows (provider discount).
- `paid_amount` (DECIMAL): What insurance actually paid.
- `status` (TEXT): Pending, Paid, Denied, Partial.
- `denial_reason` (TEXT): Why denied (Out of Network, Prior Auth, etc.).

**Purpose:** Records medical claims. One row per claim (many details per claim live in `claim_lines`).

**Data Flow:** When a member visits a provider → provider submits a claim → claim enters system → insurer evaluates → status becomes Paid/Denied.

**Key Queries:**
- `SELECT * FROM claims WHERE status = 'Denied';` — All denials (appeals team view).
- `SELECT member_id, COUNT(*) AS num_claims, SUM(paid_amount) FROM claims WHERE status = 'Paid' GROUP BY member_id;` — High-cost members.
- `SELECT SUM(submitted_amount) - SUM(paid_amount) AS savings FROM claims;` — How much insurer negotiated down (network discount).

**Insights:**
- Claims approval rate (% Paid vs Denied).
- Average claim value (submitted vs allowed vs paid → shows negotiation strength).
- Denial patterns (Out of Network = biggest reason → improve network; Prior Auth = process improvement needed).
- Member cost burden (submitted - paid = member responsibility, but usually in/out-of-pocket depends on plan).

---

### 6. **claim_lines** — Claim Details (Line Items)
**Structure:**
- `line_id` (INT, PK): Unique line within a claim.
- `claim_id` (FK): Which claim.
- `procedure_code` (TEXT): CPT code (99213 = office visit, 99291 = ER critical, 92920 = angioplasty).
- `diagnosis_code` (TEXT): ICD code (J45.909 = Asthma, I21.9 = Heart Attack, E11.9 = Diabetes).
- `charge` (DECIMAL): Line item charge.
- `paid` (DECIMAL): Amount paid for this line.

**Purpose:** Medical detail—one row per service/procedure on a claim. Enables diagnosis-based reporting.

**Data Flow:** Many lines per claim. A single ER visit might have 5 line items: admission, ECG, medication, physician time, imaging.

**Key Queries:**
- `SELECT procedure_code, SUM(charge) AS total_charges, SUM(paid) AS total_paid FROM claim_lines GROUP BY procedure_code;` — Which procedures cost most.
- `SELECT diagnosis_code, COUNT(*) FROM claim_lines GROUP BY diagnosis_code;` — Most common diagnoses (disease prevalence).
- `SELECT c.claim_id, SUM(cl.charge) FROM claim_lines cl JOIN claims c ON cl.claim_id = c.claim_id WHERE c.status = 'Paid' GROUP BY c.claim_id;` — Claim totals by summing lines.

**Insights:**
- High-cost procedures (angioplasty, ER visits) → negotiate rates.
- Chronic disease prevalence (asthma, diabetes → design wellness programs).
- Coding patterns (some diagnosis/procedure combos don't make sense → fraud detection).

---

## Data Flow Diagram (Text)
```
MEMBERS
   ↓ (enroll in)
ENROLLMENTS ← (choose) ← PLANS
   ↓
CLAIMS (member used provider)
   ↓ (detail of)
CLAIM_LINES (procedures/diagnoses)
   
PROVIDERS ← (used in) ← CLAIMS
```

**Example Flow:**
1. Sarah Johnson joins Blue Cross Silver plan (PLANS + ENROLLMENTS).
2. Sarah visits Dr. Lisa Thompson (PROVIDERS).
3. Provider submits claim for office visit → Asthma diagnosis (CLAIMS + CLAIM_LINES).
4. Insurer evaluates: allowed = $180, paid = $180, status = Paid.
5. Reports: "Asthma office visit → cost trend," "Dr. Thompson → 50% discount negotiated."

---

## Key SQL Queries & Purpose

## Key SQL Queries & Purpose

| Query | Purpose | Insight |
|-------|---------|---------|
| `SELECT * FROM plans;` | Show all available plans | Plan selection for enrollment |
| `SELECT member_id, first_name FROM members WHERE first_name = 'Sarah';` | Find member by name | Member lookup (member portal) |
| `SELECT provider_name FROM providers WHERE in_network = 1 AND city = 'Chicago';` | In-network doctors nearby | Avoid out-of-network denials |
| `SELECT m.first_name, p.plan_name, e.status FROM members m JOIN enrollments e ON m.member_id = e.member_id JOIN plans p ON e.plan_id = p.plan_id WHERE e.status = 'Active';` | Active members + plans | Eligibility verification |
| `SELECT claim_id, member_id, denial_reason FROM claims WHERE status = 'Denied';` | All denials with reasons | Appeals & process improvement |
| `SELECT m.first_name, COUNT(c.claim_id) AS claims, SUM(c.paid_amount) FROM claims c JOIN members m ON c.member_id = m.member_id GROUP BY m.member_id ORDER BY SUM(c.paid_amount) DESC;` | High-cost members | Risk identification, care management |
| `SELECT pr.in_network, SUM(c.submitted_amount) FROM claims c JOIN providers pr ON c.provider_id = pr.provider_id GROUP BY pr.in_network;` | In-network vs out-of-network spend | Network leakage, contracting focus |
| `SELECT diagnosis_code, COUNT(*) FROM claim_lines GROUP BY diagnosis_code ORDER BY COUNT(*) DESC;` | Most common diagnoses | Disease prevalence, wellness programs |
| `SELECT procedure_code, SUM(paid) FROM claim_lines GROUP BY procedure_code ORDER BY SUM(paid) DESC;` | Highest-cost procedures | Negotiation targets |

**Business Use Cases:**

1. **Eligibility Verification** → Join `members` → `enrollments` → `plans` + check coverage dates before paying a claim.
2. **Denial Analysis** → Count & group denials by reason → identify process gaps (Prior Auth, out-of-network, etc.).
3. **High-Cost Member Management** → Sum paid claims by member → prioritize for care coordination.
4. **Network Adequacy** → Count in-network providers by specialty/city → identify gaps.
5. **Financial Reporting** → SUM(submitted_amount) - SUM(paid_amount) = insurer's negotiated savings.
6. **Member Communication** → Show individual claim history, outstanding balances, coverage details.
7. **Compliance** → Track enrollment dates, termination dates, claims submitted dates for audit trails.

---

## Indexing (Why & What's Indexed)

**Why Indexes:** Speed up WHERE and JOIN conditions. Without indexes, SQLite/Postgres must scan every row.

**Indexes in this schema:**
- `idx_claims_member`: `claims(member_id)` — fast lookup of "show all claims for member X."
- `idx_claims_date`: `claims(service_date)` — fast filtering by date range (e.g., "claims from Jan 2024").
- `idx_claims_status`: `claims(status)` — fast filtering "WHERE status = 'Denied'".
- `idx_enrollments_member`: `enrollments(member_id)` — fast "show active plan for member X."
- `idx_enrollments_dates`: `enrollments(start_date, end_date)` — composite index for "coverage between dates?"
- `idx_providers_network`: `providers(in_network)` — fast filter "show in-network doctors."
- `idx_claim_lines_claim`: `claim_lines(claim_id)` — fast "show all lines on a claim."

---

## Configuration
- **SQLite database**: Stored as `insurancedb.db` in the workspace (ephemeral, easily recreated).
- **PostgreSQL variables** (set in notebook): `pg_user`, `pg_password`, `pg_host`, `pg_port`, `pg_database`. Store in environment variables or `.env` to avoid exposing secrets.


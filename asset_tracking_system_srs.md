# SOFTWARE REQUIREMENTS SPECIFICATION (SRS)
**Project:** Personal Asset Depreciation and TCO Tracking System  
**Author:** Architecture & Engineering  

---

## 1. Introduction

### 1.1 Purpose
This document specifies the complete software requirements for a highly automated, self-correcting Personal Asset Depreciation and Total Cost of Ownership (TCO) Tracking System. It defines the system's architecture, functional behaviors, data structures, and edge-case handling protocols independent of any specific programming language or framework.

### 1.2 Scope
The system is a double-entry financial ledger tailored for physical hardware and asset lifecycle management. It manages virtual accounts, depreciating assets, batched identical items, and automated amortization schedules. It supports multi-ticker valuation (fiat, crypto, commodities), dynamically pegging asset targets to live market rates. It features a non-linear state machine for asset lifecycles, complex transaction distribution algorithms, and hierarchical data tagging. 
* **Out of Scope:** Direct integration with real-world banking APIs; all accounts and funds tracked are virtual ledger representations.

### 1.3 Related Work
While traditional accounting software (e.g., QuickBooks, Xero) handles corporate asset depreciation, they lack automated, dynamic multi-ticker pegging and predictive shortfall recovery for personal replacement cycles. Standard personal finance trackers (e.g., YNAB) track liquidity but fail to calculate true TCO, fixed-cost capitalization for batches, or partial asset upgrades. This system bridges personal sinking-fund mechanics with corporate TCO logic.

### 1.4 Product overview

#### 1.4.1 Product perspective
The system operates as an independent Client-Server application. It consists of a responsive graphical user interface (GUI) for the presentation layer, an API gateway for business logic, an asynchronous task runner for background automation (market polling, debounce backups), and a relational database enforcing ACID compliance.

#### 1.4.2 Product functions
* **Taxonomy & Automation:** Hierarchical Super Tags, reusable Time Periods, and Polymorphic Funding Rules.
* **Valuation Engine:** Static historical snapshotting and dynamic live-market pegging across multiple financial tickers.
* **Ledger Manager:** Immutable, append-only double-entry transaction tracking with exact exchange-rate snapshots.
* **Amortization Engine:** Predictive, self-correcting arrears generation featuring shortfall recovery protocols.
* **Lifecycle State Machine:** Non-linear asset progression handling upgrades, partial batch splits, and premature write-offs.

#### 1.4.3 User characteristics
The target user possesses advanced technical and financial literacy. They understand double-entry accounting concepts, market ticker volatility, and amortization. They require high-density data visualization over simplified, gamified UI elements.

#### 1.4.4 Limitations
* Requires uninterrupted access to third-party market data APIs to maintain dynamic pegging accuracy.
* System integrity relies entirely on the user faithfully mirroring real-world financial transactions into the virtual ledger.

#### 1.4.5 Assumptions and dependencies
* External market APIs (e.g., Yahoo Finance, CoinGecko, Frankfurter) will remain available or have fallback alternatives.
* The deployment environment supports asynchronous task queues (e.g., Celery, Redis) to manage the 15-minute debounced backup protocol.

### 1.5 Definitions
* **TCO (Total Cost of Ownership):** The comprehensive financial burden of an asset, defined as: 
  $$\text{TCO} = (\text{Purchase\_Price} + \text{Capitalized\_Fixed\_Costs} + \text{Maintenance\_Expenses}) - \text{Salvage\_Value}$$
* **Ticker:** Any tradable financial unit, including fiat currencies, cryptocurrencies, commodities, and securities.
* **Pegging:** Tying the target replacement value of an asset to a shifting external market ticker rather than a static localized fiat amount.

---

## 2. Specific requirements

### 2.1 External interfaces
* **User Interface (UI):** A web-based or desktop client featuring a sidebar navigation architecture and complex data-grid dashboards.
* **Market Data API Interfaces:** Outbound REST connections to fetch live and historical exchange rates for fiat, crypto, and equities.

### 2.2 Functions

#### 2.2.1 UML Use Case Diagram
*(Textual Representation)*
* **Actor:** System User
  * **Primary Use Cases:** Manage Taxonomy, Configure Automation (Rules/Presets), Manage Virtual Accounts, Manage Assets/Batches, Execute Transactions, Execute Lifecycle Shifts (Upgrade/Retire).
* **Actor:** Background Worker (System)
  * **System Use Cases:** Poll Market Exchange Rates, Recalculate Arrears, Execute Scheduled Backups.

#### 2.2.2 Detailed Use Case Descriptions

##### UC-1: Taxonomy & Configuration Management
* **Tags & Super Tags:** User creates a hierarchical structure.
  * *Inputs:* Name, Parent Tag ID (optional).
  * *Behavior:* Applying a child tag to an entity rolls its data up to the parent tag for aggregate reporting. Deleting a parent tag safely orphans child tags without cascading deletion.
  * *Polymorphic Mapping:* Tags must be attachable to the following entities: `Assets`, `Batches`, `Virtual Accounts`, `Transactions`, and `Funding_Rules`.
* **Presets:** User defines creation templates.
  * *Inputs:* Preset Name, Default Ticker, Default Life Period, Default Tags, Default Funding Rules.

##### UC-2: Account & Rule Management
* **Accounts:** User creates virtual holding buckets.
  * *Inputs:* Name, Description (optional), Ticker, Type, Tags.
* **Time Periods:** User defines scheduling intervals.
  * *Inputs:* Name (e.g., "5-Year Tech Refresh"), Duration Magnitude/Unit (e.g., 5 Years), Recurrence Magnitude/Unit (e.g., 1 Month).
* **Funding Rules:** User creates calculation formulas.
  * *Inputs:* Rule Type (Percentage, Fixed Cost), Value, Target Ticker, Time Period ID, Linked Entity (Account or Asset).
  * *Behavior:* Account-linked rules automate deposits (e.g., interest). Asset-linked rules dictate required amortization savings.

##### UC-3: Asset & Batch Lifecycle Management
* **Asset Creation:**
  * *Inputs:* Name, Unit Purchase Price, Purchase Ticker, Pegged Ticker, Life Period, Manual Buffer (optional).
  * *Behavior:* System locks the exact exchange rate between Purchase and Pegged tickers at the creation timestamp (or a user-supplied custom historical timestamp).
* **Batch Containers:**
  * *Inputs:* Batch Name, Asset IDs, Fixed Costs.
  * *Behavior:* System capitalizes the input Fixed Costs evenly across all child assets, permanently altering their individual baseline costs.
* **State Machine Transitions:**
  * *Nodes:* Active, Fully Funded, Pre-Retired, Retired.
  * *Behavior:* Any state can transition directly to Retired (explicitly supporting premature upgrades and forced write-offs of Active assets). Active automatically shifts to Fully Funded when $\text{Saved} \ge \text{Target}$. If an expense is logged against a Fully Funded asset causing a shortfall, it reverts to Active and generates recovery arrears based on user configuration (Immediate, Even Split, or Waterfall).
  * *Pre-Retirement:* Asset ceases financial tracking (removed from arrears) but remains in active inventory tracking.
* **Retirement & Salvage:**
  * *Inputs:* Salvage Value (supports zero or negative numbers to account for disposal fees), Received Ticker, Timestamp.
  * *Behavior:* User may execute a salvage transaction at any time during or after the asset's transition to Retired. The system logs this as an inflow/outflow transaction. If the system detects a replacement asset linked via `Asset_Lineage`, positive salvage funds are automatically injected into the new asset's saved pool. The system dynamically updates the final TCO of the retired asset.

##### UC-4: Ledger & Transaction Execution
* **Standard Transaction:**
  * *Inputs:* Type (Funding/Expense), Timestamp, Account ID, Asset/Batch ID, Amount in Account Ticker.
  * *Behavior:* System dynamically queries exchange rate at the given timestamp, calculates the `amount_pegged_ticker`, and writes an immutable ledger row.
* **Grouped Transaction (Bulk Funding/Expense):**
  * *Inputs:* Transaction Type (Funding or Expense), Source/Destination Virtual Account, Target Asset IDs (or Arrear IDs), Total Lump Sum.
  * *Behavior:* System executes a composite ledger entry, drawing from or depositing into the selected Virtual Account, and dividing the lump sum across the targeted assets using a parameter-driven algorithm:
    * *Even:* Total amount divided equally ($\text{Sum} / N_{\text{assets}}$).
    * *Proportional:* Weighted distribution. The UI must prompt the user to select a **Weighting Factor** and a **Proportionality Mode** (Direct or Inverse). Selectable factors must include: Arrear Amount, Asset Base Cost, Asset Total Target, Asset Completion Percentage, Asset Amount Left, and Remaining Lifespan (Days to EOL). *Note: Utilizing Inverse mode ($1/w_i$) allows users to mathematically prioritize assets with lower values, such as routing emergency liquidity to assets with the fewest days remaining to EOL.*
    * *Waterfall:* Sequential cascade. The UI must prompt the user to select a **Sorting Index** and **Direction** (Ascending/Descending). Selectable indexes must include: Asset Creation Date, Arrear Due Date, Arrear Amount, Asset Total Target, Asset Completion Percentage, Asset Amount Left, and Remaining Lifespan (Days to EOL). Remaining surplus is buffered into the final asset; shortfalls leave the final asset partially funded/expensed.
* **Upgrade / Rollover:**
  * *Behavior:* Old asset (whether Active or Fully Funded) is forcefully transitioned to the inactive (Retired) state, freezing its current incomplete or complete saved progress. The user links a New Asset as the child via `Asset_Lineage`. The system evaluates the cost differential:
    * *Deficit/Upgrade ($\text{New\_Cost} > \text{Old\_Accumulated}$):* The system blocks the upgrade and triggers a **Shortfall Resolution Prompt** in the UI. To fulfill the "1-click" UX, the user selects a funding vector, and the API executes a composite, atomic database transaction. The vectors are:
      1. *External Injection:* System auto-generates a bridging Funding inflow transaction for the deficit amount, immediately followed by the Purchase expense.
      2. *Internal Draw:* System deducts the deficit from unallocated general liquidity within a specified Virtual Account.
      3. *Liquidity Reallocation ("Stealing"):* System extracts saved liquidity from a *different* active asset, routing it to the New Asset. This automatically triggers a Shortfall state on the victim asset, restarting its amortization schedule.
    * *Surplus ($\text{New\_Cost} \le \text{Old\_Accumulated}$):* The purchase consumes a portion of the Old Asset's liquidity. The remaining difference (surplus) is carried over as the starting saved balance for the New Asset.
  * *Ledger Integrity:* All underlying movements triggered by the Shortfall Resolution Prompt must be executed as a single ACID-compliant database transaction. The Old Asset's historical accumulated target remains immutable in the records (via a timestamp boundary freeze at the moment of retirement).
  * *Analytics:* System calculates a 4-Ticker Matrix ($\Delta\% = (V_{\text{new}} - V_{\text{old}}) / V_{\text{old}}$) comparing Old vs. New Tickers and reports the true cost of the lifestyle upgrade (the deficit) or the savings yielded from the cheaper replacement (the surplus).
* **Partial Batch Upgrade:**
  * *Behavior:* System isolates $X$ items from a Batch of $Y$, extracts their proportional saved funds, retires the $X$ items, and rolls their funds into new entities while smoothly recalculating the amortization schedule for the remaining $Y-X$ items.

##### UC-5: Dashboards & Reporting
* **Main Dashboard:** Displays Global Liquidity vs Target, Active Arrears Queue (actionable with checkboxes for Grouped Transactions), and End-of-Life (EOL) Radar.
* **Sidebar Nav:** Distinct panels for Assets, Batches, Accounts, Tags, Rules, and Transactions.
* **Asset Detail View:** Displays TCO breakdown pie chart, Target vs Saved historical line chart, upcoming localized arrears, and specific transaction ledger.

### 2.3 Usability Requirements

#### 2.3.1 Performance
UI must render complex data grids (Transactions, Asset Lists) without blocking the main thread. Filtering by Super Tags across large ledgers must be sub-second.

#### 2.3.2 Productivity
The system must support Presets and bulk-action capabilities (Grouped Transactions, manual capitalization splits) to minimize repetitive data entry.

#### 2.3.3 Accessibility
Standard contrast ratios and keyboard navigability across all data grids and input modals.

#### 2.3.4 User Friendliness
Complex calculations (e.g., Shortfall Recovery, Direct/Inverse Proportional distribution ratios) must be visually explained in the UI via tooltips or preview panes before the user commits the transaction.

### 2.4 Performance requirements

#### 2.4.1 Concurrent Users
System is primarily single-tenant or low-concurrency multi-tenant (household use).

#### 2.4.2 Response Time
Internal API endpoints for ledger updates must resolve in $< 200\text{ ms}$. External API polling is asynchronous to prevent UI blocking.

#### 2.4.3 Data Access
Read-heavy architecture. Aggregation queries (e.g., Super Tag roll-ups) must utilize database indexing on foreign keys and tags.

#### 2.4.4 Scalability
System must support $100,000+$ transaction rows without degradation in TCO or Arrears calculation speeds.

### 2.5 Logical database requirements
The RDBMS must strictly enforce relational integrity.
* **Accounts Table:** `id`, `name`, `ticker`, `type`.
* **Assets Table:** `id`, `batch_id` (nullable), `purchase_ticker`, `pegged_ticker`, `unit_cost`, `manual_buffer`, `state` (Enum), `salvage_value` (signed decimal).
* **Asset_Lineage Table:** Join table (`old_asset_id`, `new_asset_id`) to map rollovers and support $1:N$ and $N:1$ asset replacements.
* **Batches Table:** `id`, `name`, `capitalized_fixed_costs`.
* **Time_Periods Table:** `id`, `name`, `duration_magnitude`, `duration_unit`, `recurrence_magnitude`, `recurrence_unit`.
* **Transactions Table (Immutable):** `id`, `type`, `timestamp`, `account_id`, `asset_id` (nullable), `amount_account_ticker`, `amount_pegged_ticker`, `exchange_rate_snapshot`.
* **Funding_Rules Table:** `id`, `rule_type`, `value`, `target_entity_id` (polymorphic FK), `time_period_id`.
* **Tags Table:** `id`, `name`, `parent_tag_id` (self-referential FK).
* **Entity_Tag_Mapping Table:** Polymorphic join table containing `tag_id`, `target_entity_id`, and `target_entity_type` to support mapping across all taggable system entities.

### 2.6 Software system attributes

#### 2.6.1 Reliability
* **Debounced Backup Engine:** A mutation triggers a 15-minute countdown in a persistent message broker (e.g., Redis with AOF). Any subsequent mutations within 15 minutes are ignored by the trigger. Upon expiration, a background worker dumps the database.
* **Daily Backup:** A strict 24-hour cron job executes an overarching backup regardless of user activity.

#### 2.6.2 Availability
System must degrade gracefully if external market APIs fail, utilizing the most recent cached exchange rates and alerting the user that pegging targets are temporarily static.

#### 2.6.3 Security
Standard JWT authentication if deployed over a network. Database backups must be encrypted at rest.

#### 2.6.4 Maintainability
Polymorphic entity design (Rules applying to both Assets and Accounts) reduces code duplication. Decoupled accounts ensure transaction logic remains isolated from asset valuation logic.

#### 2.6.5 Portability
System must be containerized (e.g., Docker) to allow for simple self-hosting across varied OS environments (Linux, Windows, macOS).

### 2.7 Supporting Information
The Debounce logic relies on external task queues. If the user opts for a purely local desktop deployment (e.g., SQLite via local executable), the debounce logic must theoretically be handled via in-process background threading. **Architectural Warning:** In-process threading is highly volatile. If the application process is terminated prior to the 15-minute expiration, the backup will silently fail, compromising the evidence-based integrity of the ledger. A persistent, decoupled broker is heavily recommended even for local setups.

---

## 3. References
1. Generally Accepted Accounting Principles (GAAP) - Double-entry ledger constraints.
2. Standard API documentation for Market Data providers (e.g., Yahoo Finance, Frankfurter).

---

## 4. Appendices

### 4.1 Acronyms and abbreviations
* **ACID:** Atomicity, Consistency, Isolation, Durability
* **API:** Application Programming Interface
* **CRUD:** Create, Read, Update, Delete
* **EOL:** End of Life
* **FK:** Foreign Key
* **GUI:** Graphical User Interface
* **RDBMS:** Relational Database Management System
* **SDD:** Software Design Document
* **SRS:** Software Requirements Specification
* **TCO:** Total Cost of Ownership
---
name: te-model-selector
description: |
  Intelligent identification of user data analysis intent, recommending the most suitable TE (ThinkingEngine) analysis model, and generating precise configuration based on actual tracking metadata. Automatically handles missing fields, model capability boundaries, and provides SQL alternatives when necessary.

  【Intent Validation: Model Selection Assistance】Triggered only when a user describes a data analysis scenario and explicitly expresses "question, hesitation, or request for help" about "model selection".

  [Strong Trigger Words] (must contain at least one of these to trigger):
    - "which model to use", "don't know which model", "recommend model", "which one should I choose"
    - "event analysis or XX analysis", "how to configure this requirement best"
    - "which analysis to use", "don't know retention or funnel"
    - Explicitly mentioning specific model names and seeking advice (e.g., "can retention analysis do this?", "is funnel suitable for this requirement?")

  [Exclusion Conditions (Never Trigger)]:
    1. Simple data query requests (no model selection keywords): "check XX data", "help me check XX", "count XX", "look at XX metric" → Do not trigger
    2. Clear instructions to execute specific business investigation (e.g., "check game inflation", "check whale profile") → Delegate to business Skill
    3. Direct request to write code (e.g., "help me write SQL") → Delegate to SQL/code Skill
    4. Clear request to design tracking table → Delegate to tracking design Skill
    5. Configuration request with known model name (e.g., "help me configure retention analysis") → If no selection hesitation, process directly with user-specified model

  [Key Distinctions]:
    - "help me check payment data" → Do not trigger (pure data query, no model selection help)
    - "which model to analyze payment data" → Trigger (has model selection intent)
    - "last 7 days DAU" → Do not trigger (clear metric, no selection help)
    - "which model to analyze DAU and MAU" → Trigger (has selection question)

---

# TE Analysis Model Selection & Configuration Assistant

Based on user data analysis needs, intelligently recommend the most suitable TE (ThinkingEngine) analysis model, and combine with actual tracking data dictionary to provide detailed configuration guidance down to specific events and properties.

---

# 🔄 Core Workflow

## Phase 1: Intent Recognition & Metadata Preparation

### 1.1 Confirm projectId

- Default to current project `projectId`
- If user doesn't provide or hasn't selected a project, ask proactively: `Please provide TE project ID (projectId)`
- Don't guess or auto-explore

### 1.2 Read Tracking Metadata (Mandatory, cannot skip unless no metadata exists)

Before generating configuration suggestions, **must** call tools to query real tracking dictionary, clearly distinguish the following three data types and correctly map them. Never fabricate English event names or property names.

**Event**: User's specific behavior (e.g., register, login).

**Event Property & Common Event Property**: Describe specific environment or values of event occurrence (e.g., amount, channel). Note: Common event properties can be used for filtering and grouping of all events.

**User Property / Tag / Cluster**: Describe user's inherent static characteristics (e.g., VIP level, cumulative payment amount).

```
Call tools:
- `mcp__te-mcp-analysis__list_events` → Get all tracked events in project with Chinese and English names
- `mcp__te-mcp-analysis__list_properties` → Get all properties (with type, description, tableType)
- `mcp__te-mcp-analysis__list_metrics` or `mcp__te-mcp-analysis__project_listMetricInfos` → Get predefined metrics (optional), check existing predefined metrics/formulas in project, if user needs match existing metrics/tags/clusters, can reuse
- `mcp__te-mcp-analysis__list_tags` → Get user tag metadata in project
- `mcp__te-mcp-analysis__list_clusters` → Get user cluster metadata in project
```

### 1.3 Intent Parsing & Field Mapping

- Map user colloquial language ("payment amount", "login") to real fields (`pay_amount`, `login`)
- Distinguish event properties (tableType=0) and user properties (tableType=1)
- Confirm property type (string/number/bool/date/datetime/array) for correct filter and group configuration
- Exact match first, fuzzy inference fallback, ambiguity must be confirmed

### Metadata Mapping Principles

- **Exact Match First**: User says "payment amount", first search for properties with propDesc containing "payment amount"
- **Fuzzy Inference Fallback**: If no exact match, show most relevant candidate properties for user confirmation
- **Ambiguity Must Confirm**: When event property and user property have same name/description, **must ask user which one to use**, cannot auto-select
- **Not Found Then Explain**: If no matching event or property in project, clearly inform user "current project has not tracked this event/property", and suggest tracking supplement

---

## Phase 2: Model Determination & Configuration Generation

### 2.1 Determine Model Based on Decision Tree

Follow 【Model Decision Tree】 and 【TE Analysis Model System Introduction】 below to select the most suitable analysis model

### 2.2 Generate Configuration (Based on Real Fields)

After determination, output recommended model in specified format, output configuration suggestions according to 【Each Model Configuration Method】.
- Fill **mapped real fields** into configuration template
- Check if required fields are complete

#### 📋 Output Format

```
## 📊 Recommended Model: [Model English Name] ([Model Identifier])

### 💡 Recommendation Reason
Briefly explain why this model was chosen, pointing out which decision rule was hit.

### 🛠️ Configuration Suggestions in TE
**(Agent Internal Instruction: Please strictly follow the structure below, convert natural language needs to corresponding underlying fields and calculation logic. Field format output as "Display Name (Identifier)")**
Based on determined model, use configuration templates below according to different model configuration methods.

**Configuration Template**: (Take event model as example, dynamically adjust according to different model configuration methods)

| **Configuration Item** | **Selection** |
| :--- | :--- |
| **Select Event** | Login (login) |
| **Calculation Method** | Unique Users |
| **Group By** | Channel (channel) |
| **Time Range** | Last 7 days |
| **Time Granularity** | By day (optional, view daily trend) or Total |
| ... | ... |

### 🔄 Alternative Solution** (if any):
[Explain under what conditions should switch to alternative model]

### ⚠️ Notes:
[Remind common pitfalls in this scenario, or tracking data口径 issues to note (e.g., payment amount doesn't include bonus part etc.)]
```

---

### 2.3 Exception Handling Mechanism

#### Case A: Field Missing

**Detection**: Configuration required field doesn't exist in metadata
**Handling**:
1. Inform user: `Current project missing [field name], cannot complete configuration`
2. Provide alternatives:
   - Recommend similar field replacement
   - Suggest tracking supplement (give tracking design suggestion)
   - Suggest creating virtual property/user tag/cluster

**Example**:
```
❌ Missing field: User Level (user_level)
✅ Alternative Solution:
  1. Use existing field vip_level instead
  2. Or create user tag: tier based on cumulative payment amount
  3. Or supplement tracking: add user_level in user properties
```

#### Case B: Model Capability Boundary

**Detection**: User needs exceed chosen model capability range
**Handling**:
1. Explain model limitations
2. Provide SQL alternative (based on Trino syntax)

**Example**:
```
⚠️ Model Limitation: Retention analysis cannot count by hour granularity
✅ SQL Alternative:
SELECT
  date_trunc('hour', "$part_time") AS hour,
  COUNT(DISTINCT "#user_id") AS user_count
FROM ta.v_event_1
WHERE "$part_event" = 'login'
  AND "$part_date" >= '2024-01-01'
GROUP BY 1
ORDER BY 1
```
**Field Explanation**:
- `"$part_date"`: Partition date
- `"$part_event"`: Event name
- `"#user_id"`: User ID
- `"[Real Property Name]"`: [Property description]

#### Case C: Needs Pre-data Preparation

**Detection**: Configuration needs user tag/cluster/virtual property but not yet created
**Handling**:
1. Inform user to create pre-data first
2. Give creation steps and configuration suggestions

**Example**:
```
⚠️ Needs Pre-preparation: High Value User Cluster
✅ Creation Steps:
  1. Go to TE → User Clusters → Create New Cluster
  2. Filter condition: Cumulative Payment Amount >= 1000
  3. Save as "High Value Users"
  4. Use this cluster for comparison in event analysis
```

---

## TE Analysis Model System Introduction (11 Models Total)

### 1. Event Analysis (event) — Default/Most Common Model

**Core Positioning**: Perform metric statistics and trend observation on single behavioral events. Most basic and core analysis model. When no clear signal points to other models, default to event analysis.

**Supported Aggregation Metrics**: Total count, unique users (DAU/MAU), average per user, cumulative value (e.g., total revenue), max/min, distinct count, per user value, mean, formula metrics (custom formula combining multiple metrics)

**Core Features**:
- Support viewing multiple metrics simultaneously
- Support dimension grouping comparison (e.g., by channel, region, device type)
- Support multi-date comparison
- Support filter conditions (event property, user property filters)

**Strong Signal Keywords**: DAU, MAU, count, number, amount, revenue, quantity, total, sum, statistics, query, calculation, trend, report, dashboard, payment rate (when alone), first day payment rate, active, new user, register, login, payment, revenue, ARPU, ARPPU, per user

**Typical Questions**: DAU last 7 days / Query revenue last 30 days / Paying users by channel / ARPU analysis / Help generate dashboard for registration count, login count, payment amount last 7 days

---

### 2. Retention Analysis (retention) — Second Most Common Model

**Core Positioning**: Analyze revisit situation of users who completed certain initial event in subsequent time periods. Quantify user retention and churn, measure product health. Also carrier for LTV and ROI calculation.

**Core Concepts**:
- **Initial Event**: User's "starting point" behavior (e.g., register, first login)
- **Return Event**: User's "revisit" behavior (e.g., login again, pay again)
- **Retention Rate**: Proportion of users who return on Nth day/week/month relative to initial users

**Retention Types**: N-day retention, N-week/N-month retention, unbounded retention, custom retention

**Extended Features**: Simultaneously display LTV (Lifetime Value) and ROI (Return on Investment)

**Strong Signal Keywords**: retention, retention rate, LTV, lifetime value, ROI (N-day ROI/calculate ROI), churn, churn rate, churned users, churn count, next day retention, N-day retention, revisit, decay, 1-day retention, 3-day retention, 7-day retention

**⚠️ Core Rule: LTV is always retention, churn is always retention, ROI defaults to retention**

**Typical Questions**: Query this month's user retention / LTV last 7 days / 7-day ROI last 10 days / 7-day churned users / Paying users 30-day retention

---

### 3. Funnel Analysis (funnel)

**Core Positioning**: Analyze conversion and loss at each step in multi-step process, find optimization directions.

**Core Configuration**: Support up to 30 steps, window period 1 minute to 180 days, support ordered/unordered funnel.

**Strong Signal Keywords**: funnel, funnel analysis, conversion (multi-step context), first...then/later..., A→B→C, penetration rate, conversion rate (when multi-step)

**⚠️ Key Distinctions**:
- "Payment Rate" alone → **event** (single metric)
- "First Day Payment Rate" → **event** (single calculated metric)
- "Penetration Rate" → **funnel** (implies A to B conversion process)
- "First A then B" + no other model keywords → **funnel**
- "First A then B" + "Retention" → **retention** (explicit model keyword priority)

**Typical Questions**: Funnel conversion from login to card draw to payment / Generate registration→login→payment conversion by channel / New user first day payment penetration

---

### 4. Interval Analysis (interval)

**Core Positioning**: Analyze time interval distribution between two causally related events users complete. Answer "how long does it take users to complete two things".

**Core Metrics**: Median, average, percentile (P75/P90), duration distribution

**Strong Signal Keywords**: interval, interval analysis, interval time, interval distribution ("interval" is strongest signal, almost 100% hit)

**⚠️ Distinction from Funnel**: Funnel focuses on "how many people converted", interval analysis focuses on "how long conversion took".

**Typical Questions**: First payment interval distribution / Upgrade to payment interval / Login interval last month

---

### 5. Distribution Analysis (scatter/distribution)

**Core Positioning**: Divide metric values into ranges, get user count and proportion in each range. Analyze user engagement depth and stickiness for certain feature.

**Distribution Dimensions**: By count distribution, by day distribution, by numeric property distribution

**Strong Signal Keywords**: distribution, distribution situation, distribution analysis, user distribution ("distribution" is almost decisive signal)

**⚠️ Distinction from Event Analysis**:
- "Payment amount trend" → **event** (time series aggregation)
- "Payment amount distribution" → **scatter** (user level distribution)
- "Daily XX distribution" → **scatter** ("distribution" priority > "daily")

**Typical Questions**: User login count distribution / Payment amount distribution / Active days distribution / Payment count distribution trend

---

### 6. Path Analysis (path)

**Core Positioning**: Explore user behavior trajectory, generate Sankey Diagram, visually show behavior inflow and outflow. Suitable for discovering "unexpected behavior patterns".

**Analysis Directions**: Forward path (what did after certain event), reverse path (what did before certain event), full path

**Strong Signal Keywords**: path, path analysis, behavior trajectory, Sankey diagram, inflow outflow, user path, what did

**Typical Questions**: What users do after entering game / Behavior before churned users leave / Behavior path before payment

---

### 7. Composition Analysis (composition)

**Core Positioning**: Profile analysis based on user properties (not behavioral events), support two-dimension cross analysis and group comparison. Answer "who are users / what do users look like".

**Strong Signal Keywords**: composition analysis, user profile, property distribution (when analysis object is static user property not behavior metric)

**⚠️ Distinction from Distribution Analysis**:
- **Composition Analysis**: Analyzes static user properties (level, region, device type) distribution
- **Distribution Analysis**: Analyzes behavior metrics (payment count, login count, payment amount) distribution among users

**Typical Questions**: User region distribution / Paying users' age group and device brand distribution / VIP users and regular users' profile difference

---

### 8. Attribution Analysis (attribution)

**Core Positioning**: Evaluate contribution degree of multiple touchpoints/channels to conversion target. Support last touch, first touch, linear, position decay, time decay attribution models.

**Strong Signal Keywords**: attribution, attribution analysis, touchpoint contribution, channel contribution, contribution degree, which touchpoint, which ad slot contributes most

**⚠️ Distinction from Funnel Analysis**: Funnel looks at conversion rate of one fixed process, attribution analysis looks at contribution degree of each touchpoint in multiple paths.

**Typical Questions**: Which ad slot contributes most to payment conversion / Promotion channel contribution evaluation / Impact of multiple popups on conversion

---

### 9. Ranking (rank)

**Core Positioning**: Rank dimensions by metric value size, show Top N leaderboard, can track ranking changes.

**Strong Signal Keywords**: TOP, Top N, ranking, leaderboard, which is highest (+ analysis dimension)

**⚠️ Key Distinctions**:
- "Which **game** has highest payment" → **rank** (ranking object is game, can be grouping dimension)
- "Which **user** has highest payment" → **other** (finding specific user needs user cluster or SQL)

**Typical Questions**: Top 5 payment games last 7 days / Top 10 revenue channel ranking / Top 20 most active servers

---

### 10. Heatmap (heatmap)

**Core Positioning**: Flexibly show user heat distribution in each region on game map or application interface. Game industry specific needs.

**Strong Signal Keywords**: heatmap, heat map, hot region, coordinate distribution, location distribution, map heat

**Typical Questions**: Hot regions where players gather/battle/die in game map / Player distribution density in different map regions

---

### 11. SQL Query (sql)

**Core Positioning**: Directly query underlying data through SQL. "Universal backup" of analysis system, meet custom needs above models cannot cover.

**Applicable Scenarios**: Cross-model complex calculation, special statistical口径, temporary data exploration validation, need JOIN multiple tables analysis

**Typical Questions**: Help check users with balance > 1000 / Cross-model complex calculation / Metrics needing special statistical口径

---

## Model Decision Tree (Execute by priority from high to low)

When determining, strictly check in following order, **first hit rule is final result**:

```
Step 1: Did user explicitly mention model name?
  → User says "funnel analysis" → FUNNEL
  → User says "retention analysis" → RETENTION
  → User says "interval analysis" → INTERVAL
  → User says "distribution analysis" → DISTRIBUTION
  → User says "path analysis" → PATH
  → User says "composition analysis" → COMPOSITION
  → User says "attribution analysis" → ATTRIBUTION
  (Explicit model name highest priority, use directly)

Step 2: Contains "interval" (time interval) keyword?
  → YES → INTERVAL

Step 3: Contains "funnel" keyword, or clear multi-step conversion description?
  Judgment condition: appears "funnel", "first A then B then C conversion", "A→B→C"
  → YES → If also contains "retention" → RETENTION
          → Else → FUNNEL

Step 4: Contains "retention" / "LTV" / "churn"?
  → YES → RETENTION

Step 5: Contains "ROI"?
  → YES → RETENTION (ROI in TE is retention model's derived metric)

Step 6: Contains "distribution"?
  → YES → Analysis object is static user property (level, region, age etc. inherent property)?
          → YES → COMPOSITION (composition analysis)
          → NO → DISTRIBUTION (distribution analysis)

Step 7: Contains "TOP" / "ranking" / "which is highest"?
  → YES → Ranking object is analysis dimension (game/channel/product etc.)?
          → YES → RANK
          → Ranking object is finding specific user?
          → YES → OTHER

Step 8: Contains "path" / "behavior trajectory" / "Sankey diagram" / "what did"?
  → YES → PATH

Step 9: Contains "attribution" / "touchpoint contribution" / "channel contribution degree"?
  → YES → ATTRIBUTION

Step 10: Contains "heatmap" / "hot region" / "coordinate distribution"?
  → YES → HEATMAP

Step 11: User filter/export/list needs? Or prediction type? Or cannot categorize?
  → YES → OTHER / SQL

Step 12: None of above hit?
  → EVENT (event analysis is default model)
```

---

## Edge Cases and Ambiguity Resolution Rules (Must strictly follow)

Below are core disambiguation rules summarized from 500+ real cases:

| # | Scenario | Correct Determination | Reason |
|---|----------|----------------------|--------|
| 1 | "Payment Rate" alone | event | Single calculated metric, not multi-step conversion |
| 2 | "First Day Payment Rate" | event | Although "rate" implies conversion, it's single metric |
| 3 | "Penetration Rate" | funnel | Implies A to B conversion process |
| 4 | "Payment Amount Trend" | event | Time series aggregation |
| 5 | "Payment Amount Distribution" | distribution | User level distribution |
| 6 | "Daily XX Distribution" | distribution | "Distribution" priority > "Daily" |
| 7 | "User with highest payment" | other | Finding specific user, not leaderboard |
| 8 | "Game with highest payment" | rank | Game is analysis dimension |
| 9 | "First A then B" + no model keyword | funnel | Sequential action implies conversion |
| 10 | "First A then B" + "Retention" | retention | Explicit model keyword priority |
| 11 | LTV (any form) | retention | 100% rule, no exception |
| 12 | Churn (any form) | retention | Churn is retention's opposite |
| 13 | ROI (any form) | retention | TE calculates ROI through retention model |
| 14 | User list/filter/cluster needs | other | Not analysis model scope |
| 15 | Prediction type needs | other | TE doesn't support predictive analysis |
| 16 | "Count distribution" / "User count distribution" | distribution | "Distribution" keyword decides |
| 17 | "Login user count daily distribution" | distribution | "Distribution" > "Daily" |
| 18 | Simply ask user count/count for certain event | event | Most basic event analysis |

---

## Each Model Configuration Method

### 【Event Analysis (event) Configuration Method】

#### Pre-data Mapping Requirements (Agent Must Read):

Before configuring, must check underlying data dictionary, clearly distinguish following three data types and correctly map:
- **Event**: User's specific behavior (e.g., register, login).
- **Event Property & Common Event Property**: Describe specific environment or values of event occurrence (e.g., amount, channel). Note: Common event properties can be used for filtering and grouping of all events.
- **User Property / User Tag / User Cluster**: Describe user's inherent static characteristics (e.g., VIP level, cumulative payment amount).

```
## Analysis Metrics (Support multi-metric combination and formula)
(For each user data need, configure corresponding metric line. Support following three modes)
### Mode A: Basic Event Aggregation
Select Event: [Mapped real event, e.g., Register (register)]
Calculation Method: Select [Total Count / Unique Users / Average Per User]
### Mode B: Event Property Aggregation (For numeric property)
Select Event: [Mapped real event, e.g., Payment (payment)]
Select Property: [Mapped event property, e.g., Payment Amount (pay_amount)]
Calculation Method (Event property type different, provided calculation methods also different):
  - If event property is numeric type, can only choose one: [Sum / Mean / Per User Value / Median / Max / Min / Distinct Count / Variance / Standard Deviation / 99th Percentile / 95th Percentile / 90th Percentile / 80th Percentile / 75th Percentile / 70th Percentile / 60th Percentile / 40th Percentile / 30th Percentile / 25th Percentile / 20th Percentile / 10th Percentile / 5th Percentile]
  - If event property is text/string or time, can only choose [Distinct Count]
  - If event property is boolean, can only choose one: [True Count / False Count / Null Count / Not Null Count / Distinct Count]
  - If event property is object/object array, can only choose one: [Null Count / Not Null Count / Distinct Count]
### Mode C: Composite Metric (Custom formula, support addition/subtraction/multiplication/division)
Formula Logic: Use above A or B mode built basic metrics for four arithmetic operations.
Example Configuration: [Metric A: Payment (payment) Unique Users] Divided by (/) [Metric B: Login (login) Unique Users]
Event-level Filter: Events in formula support event-level filter, filter condition "Event Property/User Property/User Cluster/User Tag" "Logical Operator" "Specific Value"
Event-level Filter Example Configuration: [Metric A: Payment (payment) channel = official site Unique Users] Divided by (/) [Metric B: Login (login) Unique Users]

📌 Filter Condition Level Explanation:
|Filter Type|Scope|Use Case|
|Global Filter|All metrics|Unified filter condition applicable to all metrics|
|Metric-level Filter|Current single metric|Filter condition only for specific metric, other metrics not affected|
|Event-level Filter|Certain event in formula|Only available in composite metric (custom formula) events|
Example: Query "Shenzhen different device type users last 30 days per user gold consumption count, V1.0 version total card draw user count"
Global Filter: City = Shenzhen (applicable to all metrics)
Metric-level Filter: Version = V1.0 (only card draw metric applicable, per user gold consumption count not affected by this filter)

🔍 Metric-level Filter (Optional, can add multiple, logical relation AND/OR, only acts on current single metric):
If certain metric needs separate filter condition setting (e.g., "V1.0 version total card draw user count"), need to add filter under this metric separately: [Property Name] [Logical Operator, e.g., equals/not equals/includes] [Specific Value].
🔍 Event-level Filter (Optional, can add multiple, logical relation AND/OR, only can use in complex metric (custom formula) events

## Global Filter (Optional, can add multiple, logical relation AND/OR, acts on entire chart)
(Used to define overall analysis sample range)
Select Filter Condition: [Mapped Event Property / Common Event Property / User Property / User Tag / User Cluster]
Select Logical Operator:
  - If filter condition is text type, can choose [equals / not equals / includes / excludes / has value / no value / regex match / regex not match]
  - If filter condition is time type, can choose [in range / less than or equal / greater than or equal / relative to current date / relative to event occurrence time / has value / no value]
  - If filter condition is numeric type, can choose [equals / not equals / less than / less than or equal / greater than / greater than or equal / has value / no value / in range]
  - If filter condition is boolean type, can choose [is true / is false / has value / no value]
Set Specific Value: [Specific numeric or string]
Example: 「Channel」「equals」「Official Site」; 「level」「less than or equal」「100」; 「is_first_login」「is true」

## Group By (Optional, can add multiple group items, for breakdown comparison)
(Break down overall data by specific dimension, observe multiple trend lines)
Group by Dimension: Select [Mapped Event Property / Common Event Property / User Property / User Tag / User Cluster]
Example: Group by User Value Tier (user_value_level)

## Time Range & Granularity
Time Range: Select based on mapped time range [e.g., Past 7 days / This Month]
Time Granularity: Display by [Day / Hour / Week / Month / Minute (1min/5min/10min) / Total]
```
---

### 【Retention Analysis (retention) Configuration Method】

#### Pre-data Mapping Requirements (Agent Must Read):

Before configuring retention model, must check underlying data dictionary, and strictly follow below mapping relations:
- Initial Event and Return Event: Must map to real Event.
- Property Level Constraint: When grouping using event property, must and can only use 「Initial Event」's event property (or common event property/user property). Absolutely cannot use Return Event's property for outer layer grouping!

```
## Analysis Subject
Select calculation perspective: [Mapped real analysis subject, e.g., User (user_id) / Role (role_id) / Visitor (distinct_id), if not exist prompt user to create]

## Core Event Configuration
▶️ Initial Event (Start Point): Select [Mapped real event, e.g., Register (register)]
🔙 Return Event (Target): Select [Mapped real event, e.g., Login (login)]

## Use Simultaneous Display (Advanced optional, calculate retention users' additional behavior or LTV)
(For users who completed 「Return Event」, calculate their performance on another specified event)
- Participate Event: [Mapped event, e.g., Login (login) or Payment (payment)]
- Analysis Metric (Choose one of two):
  1.Basic predefined metric: [Total Count / Unique Users / Average Per User / Total Count Period Cumulative Sum / Total Count Period Cumulative Per User Value / Unique Users Period Cumulative Sum / Unique Users Period Cumulative Per User Value], e.g., Card Draw (draw_card)]'s Unique Users / Total Count
  2.Event property metric: [Mapped event property, if event property is numeric type, can select Sum/Per User Value/Period Cumulative Sum/Period Cumulative Per User Value; if event property is boolean, can select True Count/False Count/Null Count/Not Null Count], e.g., Payment (payment)'s is_first_pay true count, means return users' first payment count
🔍 Metric-level Filter (Only for this display metric): Set [Event Property, e.g., Channel (channel)] [Logic, e.g., equals/not equals/includes/excludes/has value/no value/regex match/regex not match] [Specific Value, e.g., Official Site]

## Use Associated Property (Advanced optional, for extremely precise same category return)
(Used to limit user not only need to return, but return's specific object must match initial action)
Match Logic: Require 【Initial Event】's [Mapped event property, e.g., Activity Type] equals 【Return Event】's [Mapped event property, e.g., Activity Type] (If enabled use simultaneous display, also need equal to simultaneous display's mapped event property)

## Global Filter (Optional, define analysis overall population)
Select Filter Condition: [Mapped Event Property / Common Event Property / User Property / User Tag / User Cluster]
Select Logic: [Has Value / No Value / Greater Than or Equal / Less Than or Equal / In Range / Equals / Not Equals / Includes / Excludes etc.]
Set Specific Value: [Specific numeric or string]
Example: Operating System (os) equals ios

## Group By (Optional, breakdown retention curve)
(⚠️ Warning: If group by event property, can only extract 「Initial Event」's event property)
Group by Dimension: Select [Mapped Initial Event Event Property / Common Event Property / User Property / User Tag / User Cluster]
Example: Group by app_version (initial event property)

## Time Range & Granularity
Retention Period: By [Day / Week / Month] - [Current Day / Next Day / 7 Day / 14 Day / N Day]
Observation Range: [e.g., Yesterday / Today / Last Week / This Week / Last Month / This Month / Past 7 Days / Last 7 Days etc.]

📝 Complex Real Example Reference (Agent Learning Library):
User Need: View different app versions' last 7 days daily ios system's "new user retention" situation, simultaneously show official site retention user count.
Analysis Subject: Account
Initial Event: register
Return Event: login
Simultaneous Display: login's Unique Users + Metric Filter (channel equals Official Site)
Global Filter: os (common/user property) equals ios
Group By: app_version (this is register's event property)
Time & Granularity: By Day-7 Day Retention, Time Range: Last 7 Days.
```
---

### 【Funnel Analysis (funnel) Configuration Method】

#### Pre-data Mapping Requirements (Agent Must Read):

Before configuring funnel model, must check underlying data dictionary, ensure event sequence rationality:
- Multi-step Event Mapping: Each funnel step must map to a real Event.
- Single Event Deep Funnel: If business need is "same behavior's progressive conversion" (e.g., Level 1->Level 2->Level 3), need use same event, and overlay independent event property filter in each step.

```
## Analysis Subject
Select conversion tracking perspective: [Mapped real analysis subject, e.g., User (user_id) / Role (role_id) / Visitor (distinct_id), if not exist prompt user to create]

## Funnel Steps (Core Flow Configuration)
(Add events that must trigger in sequence by business logic, support up to 30 steps)
Step 1 (Start Point): Select [Mapped real event, e.g., Register (register)]
- Step-level Filter (Optional): Set [Property Name] [Logic] [Specific Value]
Step 2: Select [Mapped real event, e.g., Login (login)]
- Step-level Filter (Optional): Set [Property Name] [Logic] [Specific Value]
Step N (End Point): Select [Mapped real event, e.g., Level Pass (level_pass)]
- Step-level Filter (Optional): Set [Mapped property, e.g., Level (vip_level)] [Logic, e.g., equals] [Specific Value, e.g., 5]

## Analysis Window Period (Conversion Time Effect)
Set time limit for user to complete entire funnel from triggering 「Step 1」.
Window Range: Recommend setting based on business common sense [Minimum 1 minute, maximum 180 days. e.g., 1 day / 2 hours / 15 minutes].

## Use Associated Property (Advanced optional, for product/activity/match precise series connection)
(Not only require trigger in sequence, but also force certain core property value in series connection steps to be completely consistent, property null then exclude)
Match Logic: Require each step's [Mapped core event property, e.g., Item ID (item_id) / Activity ID (activity_id)] completely equal.

## Global Filter (Optional, define funnel analysis population)
Set Filter Condition: [Mapped Event Property / Common Event Property / User Property / User Tag / User Cluster]
Select Logic: [Has Value / No Value / Greater Than or Equal / Less Than or Equal / In Range / Equals / Not Equals / Includes / Excludes etc.]
Set Specific Value: [Specific numeric or string]
Example: Operating System (os) equals ios

## Group By (Optional, breakdown conversion rate comparison)
(Break down funnel into multiple lines by specified dimension, compare different groups' conversion capability)
Group by Dimension: Select [Mapped Step1 Event Property / Common Event Property / User Property / User Tag / User Cluster]（⚠️Note: Choose from Step1's event property, user property, user cluster, user tag, at most one added as group item）
Example: Group by Channel (channel)

## Time Range
Observation funnel trigger start point's time range: [e.g., Last Month / Past 7 Days]

> Chart Perspective: [Suggest perspective based on user need: If view overall loss situation select "Conversion Chart" / If view daily conversion rate fluctuation select "Trend Chart"]

📝 Complex Real Example Reference (Agent Learning Library):
User Need: Want to see last month different channels' iOS users, within 1 day from registration, login, level up, activity attend, card draw to logout entire conversion funnel.
Analysis Subject: Account (user_id)
Funnel Steps:
register (registration)
login (login)
level_up (level up)
activity_attend (activity attend)
draw_card (card draw)
logout (logout)
Analysis Window Period: 1 Day
Global Filter: os equals ios
Group By: channel (channel)
Time Range: Last Month

```
---

### 【Interval Analysis (interval) Configuration Method】

#### Pre-data Mapping & Calculation Rules (Agent Must Read):

- Underlying Calculation Mechanism (Must deeply understand):
  - Different events (A -> B) use shortest interval principle: If sequence is A1->A2->B1->B2, only calculate A2->B1's one interval.
  - Same event (A -> A) use adjacent interval principle: If sequence is A1->A2->A3, will produce A1->A2 and A2->A3 two intervals.
- Property Level Constraint (Very easy to mistake, must follow): If group item chooses event property, must and can only use 「Start Point Event」's event property. Absolutely cannot use End Point Event's property for global breakdown!

```
## Analysis Subject
Select calculation perspective: [Mapped real analysis subject, e.g., User (user_id) / Role (role_id) / Visitor (distinct_id), if not exist prompt user to create]
 (Note: Start and end point events must be triggered by same subject to count as one complete interval)

## Start Point Event & End Point Event (Support independent metric-level filter)
🟢 Start Point Event: Select [Mapped real event, e.g., Payment (payment)]
Metric-level Filter (Optional): Set [Property Name, e.g., Payment Amount (pay_amount)] [Logic, e.g., equals] [Specific Value, e.g., 0]
🛑 End Point Event: Select [Mapped real event, e.g., Payment (payment) or Login (login)]
Metric-level Filter (Optional): Set [Property Name, e.g., Payment Amount (pay_amount)] [Logic, e.g., greater than or equal] [Specific Value, e.g., 100]

## Interval Upper Limit (Exclude invalid long tail data)
Set maximum valid calculation duration: [Set based on business common sense, minimum 1 minute, maximum 180 days. e.g., 1 Day / 2 Hours] (Note: Data exceeding this upper limit span will be directly excluded)

## Use Associated Property (Advanced strong constraint, for same category/continuous action match)
(Not only require trigger by time sequence, but also force certain property association between two. Property null then exclude)
Regular Match: Require 【Start Point Event】's [Mapped event property, e.g., Account ID (account_id)] equals 【End Point Event】's [Mapped event property], both property types must match.
Advanced Numeric Match (Difference Calculation): If numeric property, can set deviation. E.g., require 【End Point Event】's [Level ID (level_id)] is 1 greater than 【Start Point Event】's this property (Used to exclude repeated grind same level interference, only see real promotion time cost).

## Global Filter (Optional, define analysis population)
Set Filter Condition: [Mapped Event Property / Common Event Property / User Property / User Tag / User Cluster]
Select Logic: [Has Value / No Value / Greater Than or Equal / Less Than or Equal / In Range / Equals / Not Equals / Includes / Excludes etc.]
Set Specific Value: [Specific numeric or string]
Example: Operating System (os) equals ios

## Group By (Optional, breakdown different groups' time cost difference)
(⚠️ Warning: If group by event property, can only extract 「Start Point Event」's property value at that moment)
Group by Dimension: Select [Mapped Start Point Event Event Property / Common Event Property / User Property / User Tag / User Cluster]
Example: Group by Start Point Event's Channel (channel)

## Time Range & Granularity
Display Dimension: By [Day / Hour / Week / Month / Total]
Time Range: [e.g., This Month / Past 7 Days]

> Chart & Metric Perspective Suggestion:
  - [If user wants to see max/min, average, median etc. aggregation statistics, suggest select "Box Plot"]
  - [If user wants to see time cost range distribution, e.g., how many people took 0-5 minutes, how many took 5-10 minutes, suggest select "Histogram", can prompt custom range boundary]

📝 Complex Real Example Reference (Agent Learning Library):
User Need: Want to see this month daily different channels' iOS users, same account ID within 1 day from payment amount 0 to payment amount greater than or equal 100's time cost interval.
Analysis Subject: User
Start Point Event: payment + Filter pay_amount equals 0
End Point Event: payment + Filter pay_amount greater than or equal 100
Interval Upper Limit: 1 Day
Associated Property: Start Point Event's Account ID equals End Point Event's Account ID
Global Filter: os equals ios
Group By: channel (must be start point event's property)
Time Granularity & Range: Display by day, This Month
```

---

### 【Distribution Analysis (distribution/scatter) Configuration Method】

#### Pre-data Mapping & Calculation Rules (Agent Must Read):

Before configuring distribution model, must check underlying data dictionary, and deeply understand below nested calculation logic:
- Aggregation Property Data Type Constraint: If aggregate by event property, text type property only supports "Distinct Count"; numeric type property supports "Sum/Mean/Median/Max/Min/Percentile/Variance etc.". Absolutely cannot use sum or mean on text property!
- Multi-level Filter (Very easy to confuse, must strictly distinguish):
  - Event-level Filter: Mounted on specific one event (e.g., only calculate is_first_pay=true's payment count).
  - Metric-level Filter (Global): Mounted on entire complete metric or formula's outside (e.g., (A/B) result calculated, overall only look at channel=official site data).

```
## Analysis Subject
Select calculation underlying perspective: [Mapped real analysis subject, e.g., User (user_id) / Role (role_id)] (Note: All subsequent frequency and numeric values will be aggregated based on this subject)

## Participate Event (Tiering Basis / Core Metric Construction)
(Support below three modes to build tiering basis, and support independent "Event Filter" on each event)
### Mode A: Based on Behavior Frequency Tiering
Select [Mapped real event, e.g., Login (login)]
Metric-level Filter (Optional): [Mapped Event Property / Common Event Property / User Property / User Tag / User Cluster, e.g., is_first_login] [Logic] [Specific Value, e.g., is true]
Aggregation口径: [Count / Days / Hours]
### Mode B: Based on Event Property Calculation Tiering
Select [Mapped real event, e.g., Payment (payment)]
Select [Mapped event property].(If text type, only "Distinct Count"; if numeric type, can select "Sum/Mean/Median/Max/Min/Variance/Standard Deviation/Percentile etc.")
Metric-level Filter (Optional): Same above
Aggregation口径: [Count / Days / Hours]
### Mode C: Based on Formula Advanced Calculation Tiering
Use basic event combination formula, e.g., [Event A's Aggregation口径] / [Event B's Aggregation口径]. Each event can have independent event-level filter.
Event-level Filter (Optional): [Mapped property, e.g., is_first_login] [Logic] [Specific Value, e.g., is true]

## Range Division (Bucketing Rules)
Set data ranges based on calculated metric distribution situation, support default range, discrete number and custom range. [e.g., Range boundaries inferred by common sense, like 0-1, 1-5, 5-10, Greater than 10]

## Use Simultaneous Display (Advanced optional,透视 specific range group users' value)
(After range division complete, further look at these people falling into this range's other performance)
Participate Event: [Mapped value event, e.g., Payment (payment)]
Analysis Metric: [Total Count / Unique Users / Average Per User / Event Property Numeric Aggregation etc.]. Can also use basic event combination advanced formula, same above Mode C.
Event-level Filter (Optional): [Mapped property, e.g., is_first_login] [Logic] [Specific Value, e.g., is true]
Metric-level Filter (Only for this display metric): Set [Property Name] [Logic] [Specific Value]

## Group By (Optional, breakdown different dimensions' distribution structure)
Break down range structure by dimension: Select [Mapped Event Property / Common Event Property / User Property / User Tag / User Cluster]

## Time Range & Display Form
Time Display Granularity: By [Day / Week / Month / Total]
Time Observation Range: [e.g., This Month / Past 7 Days etc.]

> Chart & Perspective Suggestion:
  - [If need see most detailed data comparison (or has group item), suggest priority view "Table"]
  - [If need see overall population's static distribution, suggest select "Histogram / Bar Chart"]
  - [If need see distribution structure's change trend over time, suggest select "Percentage Distribution / Numeric Distribution"]

📝 Complex Real Example Reference (Agent Learning Library):
User Need: Want to see this month daily each channel users' first payment count distribution, simultaneously show all payment user count.
Analysis Subject: User
Participate Event (Tiering Basis): payment
Aggregation口径: Count
Event-level Filter: is_first_pay (first payment status) is true
Use Simultaneous Display: payment's Unique Users (i.e., payment user count)
Group By: channel (channel)
Range Division: Count range auto or manually set based on business (e.g., 1 count, 2 count etc.)
```

---

### 【Path Analysis (path) Configuration Method】

#### Pre-data Mapping & Strict Constraints (Agent Must Read):

Before configuring path model, must check underlying data dictionary, and firmly remember below two red line rules:
- Absolutely Prohibit Event-level Filter: Selected participate events cannot attach any filter conditions (e.g., cannot only look at pay_amount>0's payment, must pull entire payment in).
- Filter Dimension's Dimensionality Reduction Strike: This model's global filter is called "User Filter", only accepts user table fields (user property/cluster/tag). Absolutely cannot use any event property (including common event property) to filter!

```
## Events Participating Analysis (Building trajectory's basic nodes)
Pick candidate action set to build behavior trajectory (max 30 meta events).
Selected Range: [Mapped core event list, e.g., Login (login), Level Up (level_up), Card Draw (draw_card), Logout (logout) etc.]
(⚠️ Warning: Here only select event name, absolutely cannot attach any filter conditions!)

## Event Split (Advanced optional, for refining same type action's specific flow direction)
(Split same event, based on its certain property value,裂变into multiple different nodes on Sankey diagram)
Split Target: Select [Above selected list's certain event, e.g., Level Up (level_up)]
Split Basis: By [Mapped this event's event property, e.g., Level (level)] split.
(Display Effect: Original single "Level Up" node on chart, will裂变into "Level Up(level=1)", "Level Up(level=2)" etc. independent nodes.)

## Analysis Path Start/End Point (Determine exploration direction)
(Choose one to set based on business need)
Forward Exploration (Look at flow direction after certain action): Select [Mapped specific event, e.g., Login (login)] as "Initial Event". System will trace backward.
Reverse Trace (Look at behavior before churn or conversion): Select [Mapped specific event, e.g., Logout (logout)] as "End Event". System will trace forward.

## Session Interval Duration (Session Cutting Knife)
(Set maximum timeout time for adjacent two actions to be considered same continuous session)
Interval Upper Limit: Recommend [Infer based on business, minimum 1 second, maximum 24 hours. e.g., 30 Minutes / 2 Hours].
(Note: If no selected event occurs after this interval, consider this session interrupted, trajectory ends.)

## User Filter (Global filter population)
(⚠️ Warning: Only can choose from user property, user tag, user cluster! Not support single event property filter)
Set User Filter Condition: [Mapped User Property / User Tag] [Logic] [Specific Value]; or [Mapped User Cluster] [belongs to cluster/does not belong to cluster].
Example: Mid Spender belongs to cluster.

## Time Range
Observe user trajectory's time window: [e.g., Past 7 Days / Last Month]

> Chart Interpretation Suggestion:
  - Prompt user to view generated **Sankey Diagram**
  - Rectangle block's height represents flow size, connection line represents flow proportion. Click specific node can highlight view flow through that node's exclusive trajectory

📝 Complex Real Example Reference (Agent Learning Library):
User Need: Want to see past 7 days Mid Spender users' behavior path from login, and split level up event by level, adjacent two behaviors not exceed 30 minutes.
Events Participating Analysis: login (login), gold_get (get gold), draw_card (card draw), level_up (level up), logout (logout) etc.
Event Split: Split level_up event by level (level) property
Analysis Path Start Point: Use login as Initial Event (forward exploration)
Session Interval Duration: 30 Minutes
User Filter: Select user cluster Mid Spender, logic belongs to cluster
Time Range: Past 7 Days
```

---

### 【Composition Analysis (composition) Configuration Method】

#### Pre-data Mapping & Strict Red Lines (Agent Must Read):

Before configuring composition model, must check underlying data dictionary, and firmly remember below three red lines:

- Absolute Forbidden Zone: **Strictly prohibit using any "Event (Event)" or "Event Property / Common Event Property"**! This model can only query user dictionary data (user property, user tag, user cluster).
- Time Zone & Version Irrelevance: User property has no time zone concept (cannot select display time zone); user tag forced use system's latest version to calculate.
- Aggregation Type's Strong Dependency: When as metric analysis, calculation formula strictly depends on this user property's data type (see below configuration steps).

```
## Analysis Metric (Profile's evaluation baseline)
(According to need evaluate baseline, choose below configuration one, note analysis metric can only have one, cannot add multiple metrics)
Default Option: Select [User Count] (i.e., matching condition's #user_id distinct count).
Advanced Option: Based on user property/tag calculation:
Select [Mapped User Property / User Tag, e.g., Cumulative Payment Amount (total_pay_amount)], and based on its data type select legal calculation口径:
  - Numeric Type: [Sum / Mean / Per User Value / Median / Max / Min / Distinct Count / Variance / Standard Deviation]
  - Text/Time Type: [Distinct Count]
  - Boolean Type (Property only): [True Count / False Count / Null Count / Not Null Count / Distinct Count]
  - List Type (Property only): [List Distinct Count / Set Distinct Count / Element Distinct Count]
  - Object/Object Array Type (Property only): [Distinct Count / Null Count / Not Null Count]

## Global Filter (Define overall profile base pool)
(⚠️ Warning: Only use user property/user tag/user cluster fields, cannot use event property)
Set User Filter Condition:
- Select User Property/User Tag: [Mapped User Property/Tag] [Logic] [Specific Value / String]
- Select User Cluster: [Mapped User Cluster, e.g., Mid Spender] [Logic: belongs to cluster / does not belong to cluster]

## Group Statistics (Two-mode profile breakdown, choose one)
(Choose group mode based on business want "see internal composition" or "see external comparison")
【Mode A: Group Statistics by "Property"】(Look at大盘internal composition & cross)
Select 1 to 2 [Mapped User Property / User Tag / User Cluster].
(Agent Prompt: If select 1, intuitively show distribution proportion; if select 2, show as stacked bar chart or multi-dimensional cross table.)
Example: Group by Province (province).
【Mode B: Group Statistics by "Population"】(Look at different groups' characteristic comparison)
Custom define max 10 groups for horizontal comparison.
Group 1: [e.g., All Users (as comparison大盘)]
Group 2: Set [Mapped User Property/User Tag/User Cluster] condition defined population [e.g., High Value Churn Warning Cluster]
Group 3: Set ... condition defined population.

> Chart Interpretation Suggestion:
  - [If need see different dimensions' absolute value difference, suggest select "Bar Distribution"]
  - [If need see each part's proportion in大盘, suggest select "Pie Distribution"]
  - [If configured two group items, suggest directly view "Table" to get multi-dimensional cross data]
  - ⚠️ Note: Composition analysis targets user's static profile (user table), no time zone concept, and cannot analyze specific dynamic behavior events (e.g., "yesterday payment" please use event analysis).

📝 Complex Real Example Reference (Agent Learning Library):
User Need: Want to see each province's Mid Spender user count.
Analysis Metric: User Count (default #user_id distinct count)
Global Filter: Mid Spender (user cluster), logic belongs to cluster
Group Statistics Mode: Group Statistics by "Property"
Group By: province (user property)

User Need Comparison Example: Want to see "All Users" and "Core Active Cluster"'s cumulative payment amount per user value whether has difference?
Analysis Metric: total_pay_amount (user property, numeric type), calculation口径select Per User Value
Group Statistics Mode: Group Statistics by "Population"
Comparison Population Settings:
Population 1: All Users
Population 2: Filter condition belongs to Core Active Cluster
```

---

### 【Attribution Analysis (attribution) Configuration Method】

#### Pre-data Mapping & Strict Red Lines (Agent Must Read):

Before configuring attribution model, must check underlying data dictionary, and firmly remember below calculation red lines:
- Target Event's Aggregation Limit: Conversion value can only be two口径: Event's **"Total Count", or certain numeric property's "Sum"**. Absolutely cannot use distinct count, mean etc. logic!
- Group Object's Isolation Mechanism:
  - Target Event's Grouping: Splitting is大盘conversion total amount (e.g., group by region, see each region's payment amount respectively attributed by whom).
  - Attribution Event's Grouping: Splitting is touchpoint (e.g., group by channel, see certain activity slot specifically which channel contributed).

```
## Analysis Subject
Select unique identifier passing through touchpoint and conversion: [Mapped real analysis subject, e.g., User (user_id) / Visitor (distinct_id), if not exist prompt user to create]

## Attribution Method (Choose distribution logic based on business goal):
[First Attribution]: 100% credit to first touchpoint in window (suitable for evaluating new user acquisition/ice breaking).
[Last Attribution]: 100% credit to last touchpoint in window (suitable for evaluating last push促conversion).
[Linear Attribution]: All valid touchpoints in window equally share credit (suitable for evaluating long-term seeding).

## Window Period (Touchpoint's shelf life, set time range to trace touchpoints forward from target event):
[Today]: Trace back to target event occurrence day's 0:00.
[Custom]: [Set fixed duration based on business inference, e.g., 1 Hour / 3 Days].

## Target Event (Define total prize pool)
(What value we want to distribute?)
- Conversion Event: Select [Mapped final conversion event, e.g., Payment (payment)]
- Conversion Value (Strict constraint): [Total Count] or [Mapped numeric event property, e.g., Payment Amount (pay_amount)'s Sum]
- Event-level Filter (Optional): Set [Event Property] [Logic] [Specific Value/String]
- Target Event Grouping (Optional): Group by [Mapped property, e.g., Province (province)], to respectively view different groups' brought contribution total pool.
- Direct Conversion Participate Attribution Calculation
[Agent Strongly Suggest Check]: If target event cannot find any attribution event in window period, this conversion value will be counted into "Direct Conversion" (i.e., natural traffic/no clear touchpoint conversion), avoid blindly elevating existing touchpoints' contribution degree.

## Attribution Event (Define touchpoints competing for prize, can add multiple)
(Who shares credit?)
Touchpoint 1: Select [Mapped touchpoint event, e.g., Click Banner (click_banner)]
- Event-level Filter (Optional): Same above.
- Attribution Event Grouping (Optional): Group by [Mapped touchpoint event property, e.g., Registration Channel (channel)].
Touchpoint 2: Select [Mapped touchpoint event, e.g., Click Recharge Icon (click_recharge_icon)]...
- Associated Property (Advanced strong constraint, for precise touchpoint match)
(Ensure touched object and purchased object is same thing, property null then exclude)
  Match Logic: Require 【Attribution Event】's [Mapped core event property, e.g., Item ID (item_id)] equals 【Target Event】's corresponding property.

## Global Filter (Define analysis大盘)
Set Filter Condition: [Mapped Event Property / User Property / User Tag / User Cluster] [Logic] [Specific Value]
Example: User Region has value.

## Time Range
Target event occurrence's time window: [e.g., Past 7 Days / This Month]

> Core Metric Interpretation: Focus on each touchpoint's [Contribution Value to Target Event (absolute value)] and [Contribution Degree (percentage)], and [Valid Trigger Rate].

📝 Complex Real Example Reference (Agent Learning Library):
User Need: View last 7 days user region has value situation, different registration channels' banner click and different registration channels' recharge activity icon click, brought how much payment total amount for each region (assume use last attribution, 1 hour window).
Analysis Subject: User
Attribution Method & Window: Last Attribution, Custom 1 Hour
Target Event (Output): payment event's pay_amount's Sum.【Target Event Grouping】: Group by User Region (province/city).
Direct Conversion: Check (Direct Conversion Participate Attribution Calculation)
Attribution Event 1 (Touchpoint): Banner Click Event.【Attribution Event Grouping】: Group by Channel (channel).
Attribution Event 2 (Touchpoint): Recharge Activity Icon Click Event.【Attribution Event Grouping】: Group by Channel (channel).
Global Filter: User Region (province/city) logic is has value
Time Range: Last 7 Days
```

---

### 【Ranking (rank) Configuration Method】

#### Pre-data Mapping & Strict Red Lines (Agent Must Read):

Before configuring ranking model, must check underlying data dictionary, and firmly remember below red lines:
- Data Association Iron Law: Events as "Ranking Metric" and "Simultaneous Display Metric", must contain selected "Ranking Subject"'s corresponding property field! Otherwise cannot perform data aggregation.
- Rank Processing's SQL Mapping: Must accurately map underlying three sorting logic (rank, dense_rank, row_number) based on business tolerance for "same score".

```
## Ranking Subject (Who to rank)
(This is ranking's core dimension, default is Account ID)
Select Dimension: [Mapped User Property / Event Property, e.g., Account ID (#account_id) / Level ID (level_id) / Topic (topic)]

## Ranking Metric (Ranking's numeric basis)
(Build sorting core score based on event related to ranking subject)
Participate Event: [Mapped real event, e.g., Payment (payment)]
Aggregation口径:
  - Predefined口径: [Total Count / Unique Users / Average Per User]
  - Numeric Property口径: Select [Mapped numeric event property]'s [Sum / Mean / Max / Min etc.]
Sort Direction: [Descending (Desc, default) / Ascending (Asc)]

## Tie Rank Processing (Same score arbitration rule)
(Strictly choose one of three based on business scenario)
- [Tie and Skip (rank)]: Support tie, subsequent rank断层 (e.g., 1, 2, 2, 4).
- [Tie Not Skip (dense_rank)]: Support tie, subsequent rank continuous (e.g., 1, 2, 2, 3).
- [By Default Sort (row_number)]: Not support tie, force order (e.g., 1, 2, 3, 4. Note: Bottom layer use dictionary order auxiliary sort).

## Simultaneous Display Metric (Advanced optional, supplement ranking subject's other performance)
(Not participate ranking, but parallel display with ranking. Support formula and multi-level filter)
Participate Event &口径: [Mapped event and calculation method, same above]
Custom Formula: Support basic metric's four arithmetic operations.
Event-level Filter: Set [Property Name] [Logic] [Specific Value] (Only acts on events in formula).
Metric-level Filter: Set [Property Name] [Logic] [Specific Value] (Acts on entire display metric).

## Global Filter (Optional, define leaderboard generation's大盘range)
Set Filter Condition: [Mapped Event Property / User Property / User Tag / User Cluster] [Logic] [Specific Value]

## Ranking Time Period & Comparison (Must fill if view rise/fall)
Current Time Range: [e.g., Past 7 Days / This Week]
Comparison Period (Optional): If need calculate "rank floating (e.g., rise 2 ranks, fall 3 ranks)", need set comparison baseline. [e.g., Previous Period]

> Chart Interpretation Suggestion: Prompt user focus on enabled "Comparison Period"后的**Rank Change Arrow (↑/↓), to quickly locate fastest rising or falling business subject.

📝 Complex Real Example Reference (Agent Learning Library):
User Need: Want to see past 7 days official site each account's payment total amount ranking (not support tie), simultaneously show their official site payment total count.
Ranking Subject: Account ID (#account_id)
Ranking Metric: payment (payment event)'s pay_amount (payment amount)'s Sum, descending.
Tie Rank Processing: By Default Sort (row_number)
Simultaneous Display Metric: payment's Total Count.
Global Filter: channel (channel) equals Official Site
Time Range: Past 7 Days
```

---

### 【Heatmap (heatmap) Configuration Method】

#### Pre-data Mapping & Strict Red Lines (Agent Must Read):

Before configuring heatmap model, must check underlying data dictionary, and firmly remember below red lines:
- Coordinate Property's Rigid Dependency: Selected heat event (or user associated with this event), must have numeric properties representing X axis and Y axis coordinates, otherwise absolutely cannot generate map!
- Extreme Value/First/Last Filter's "Single User" Principle: First/Last filter is dimensionality reduction strike on "each #user_id", ensure each player in selected window period only contributes 1 data (first/last/max/min), prevent single point grind interfere with overall heat distribution.
- Data Type Decides Filter Logic: Event filter's operator strictly depends on property type (e.g., text type supports regex, object array supports set judgment).

```
## Heat Metric (Locate action and decide heat value)
Heat Event: Select [Mapped space interaction event, e.g., Player Dead (player_dead) or Open Chest (open_chest)].
Event Filter (Optional): Support multiple conditions and set [AND / OR] relation. Based on filter property type, operator must遵守 bottom layer type constraint:
  - Text Type: [equals / not equals / includes / excludes / has value / no value / regex match / regex not match]
  - Time Type: [in range]
  - Numeric Type: [equals / not equals / less than / greater than etc.]
  - Object Type (Object): [has value / no value]
  - Object Array Type (Object Array): [exists object satisfying / no object satisfying / all objects satisfying / has value / no value]
Calculation Method: (Decide color depth):
  - If look at frequency/user count: Select [Total Count / Unique Users / Average Per User].
  - If look at specific value: Select [Mapped numeric property, e.g., Resource Get Amount (resource_amount)]'s [Sum / Mean / Max / Min / Distinct Count etc.]
First/Last Filter (Advanced optional, for eliminate single player repeated grind or locate extreme value)
(After enable, each user in entire time window only retain 1 event participate heat calculation)
Set only retain each user in this time window's [First / Last / Certain Property Max / Certain Property Min]'s that one record participate rendering.
Filter Method (Choose one of four):
  - [First / Last]: Retain this user's first or last record in time window.
  - [Max / Min]: Select certain [Numeric or Time Type Property], retain that property reaches extreme value's that one record.

## Event Coordinate (Soul Binding)
X Axis Property: Must bind matched numeric coordinate property in dictionary [e.g., pos_x or screen_width].
Y Axis Property: Must bind matched numeric coordinate property in dictionary [e.g., pos_y or screen_height].
(Note: Can use event property or user property)

## Map File
Please select existing map or upload new map（⚠️Note: If upload new map, need remind user input this map's bottom left and top right corner's real limit coordinate points for base map calibration).

## Multi-group Comparison (If has comparison need, optional, 2-4 groups): For left-right tile comparison different groups' heat difference.
Group 1: [e.g., condition empty, represents all data]
Group 2: Set filter condition [e.g., User Cluster = High Tier Players].
(🔥 Hidden Advanced Skill: If want compare different time periods (e.g., Last Week vs This Week), can use #event_time (Event Time) in each group's filter condition for separate limit!)

## Time Range
Set global analysis window: [e.g., Past 7 Days / This Month]

## Chart Interaction Suggestion:
  - Prompt user can use bottom right corner control to adjust "Heat Radius" and "Transparency" to prevent heat points糊成一团.
  - If enabled multi-group comparison, strongly suggest remind user check "Sync Zoom", for easy align view specific local map (e.g., certain specific bush/BOSS room)'s difference.

📝 Complex Real Example Reference (Agent Learning Library):
User Need: As map planner, want to compare past 7 days, "Newbie Players" and "Old Players"'s "First Battle" location distribution in map, see their encounter battle聚集点 what difference; simultaneously exclude those using specific illegal AFK script (use regex match device brand contains "emulator")'s records.
Heat Event: battle_occur (battle occur)
Event Filter: brand (device brand) regex not match .*emulator.*
First/Last Filter: Enable, select [First] (each player only take first battle).
Calculation (Heat Metric): Unique Users (see where most people卷入 battle).
Event Coordinate: X Axis = pos_x, Y Axis = pos_y.
Multi-group Comparison:
Group 1 (Newbie): Filter user_level (user level) < 10
Group 2 (Veteran): Filter user_level (user level) >= 10
Time Range: Past 7 Days
```

---

### 【SQL Query (sql) Method】

- 🎯 **Query Target**: Briefly explain what complex logic this SQL solves that other models cannot (e.g., cross-project query / complex multi-table JOIN / special custom口径).
- 📝 **SQL Statement**:
(Agent Internal Instruction: Based on underlying data dictionary, output Trino SQL that can directly run in TE. Must strictly遵守: Field names containing special symbols like $ or # must use double quotes `""`, string values must use single quotes `''`. Default event table is `ta.v_event_1`, user table is `ta.v_user_1`, unless clear other project ID.)

```sql
-- Example Format:
SELECT
  "$part_date",
  COUNT(DISTINCT "#user_id") AS "user_count"
FROM ta.v_event_1
WHERE "$part_event" = '[Mapped event name]'
  AND "[Mapped property name]" = 'specific value'
GROUP BY "$part_date"
ORDER BY "$part_date" ASC
```

---

## Multi-model Combination Suggestion

In actual work, complex analysis scenarios often need multiple models combined. When encountering complex problems, can suggest model combination scheme:

**Example — Analyze "Next Day Retention Rate Drop"**:
1. **Retention Analysis** → First confirm next day retention rate drop amplitude and time point
2. **Event Analysis** → Compare retained and churned users' key behavior metric difference
3. **Funnel Analysis** → Check if新手 guide process conversion rate dropped
4. **Path Analysis** → Explore churned users' behavior path before leaving
5. **Distribution Analysis** → View churned users' online duration/usage count distribution

**Example — Analyze "Payment Conversion Optimization"**:
1. **Funnel Analysis** → Find payment process's biggest loss step
2. **Interval Analysis** → Understand user's decision duration from browse to payment
3. **Path Analysis** → Discover paying users' typical behavior path
4. **Attribution Analysis** → Evaluate different touchpoints' contribution to payment

---

## Language Response Requirement

**IMPORTANT**: This skill must respond in the same language that the user used in their question.

- If user asks in **English**, respond entirely in **English**
- If user asks in **Chinese**, respond entirely in **Chinese**
- If user asks in other languages (e.g., Japanese, Korean), respond in that language if capable, otherwise default to English

This ensures clear communication and avoids confusion caused by mixed language responses.

---

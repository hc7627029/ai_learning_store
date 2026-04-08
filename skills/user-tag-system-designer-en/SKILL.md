---
name: "user-tag-system-designer-en"
description: | 
  Based on real user event tracking data, automatically identify industry and sub-category characteristics, and provide users who don't know how to design tag systems with a hierarchical, content-rich user tag system design guidance and best practices.
  
  The key emphasis is to design industry-specific tags with genuine industry insights based on industry and category differences, clarifying the business definition of each tag, use cases, recommended tag value hierarchy levels, and calculation logic.

  ⚠️ Core boundary: This Skill is solely responsible for "planning, design, and strategic advisory" (producing ideas, definitions, and calculation logic). If users explicitly request "building", "creating", or "configuring" specific tags, this Skill should not be triggered and should instead invoke the `create_tag` tool.

  Interaction characteristics: Strictly follow the multi-round dialogue process. After identifying the industry, require user confirmation. Allow users to view content by hierarchy on demand, and proactively guide users to the next step at the end of each response.

  Trigger conditions:
  - Consulting and planning requirements (precise trigger): Users clearly express not knowing how to design and seek solution guidance or design ideas. Examples: "how/how to design a tag system", "don't know what to do...", "need help with tag design".
  - Business scenario exploration: Mention user operations, precise targeting, user profiling, RFM models, lifecycle management scenarios, and the core need is **seeking advice and methodology for establishing indicator/tag systems**.
  - Ambiguous intent (self-introduction and guidance): When user prompts are vague (e.g., only input "tag system", "user tags", or "help me set up some tags"), and intent cannot be determined whether it's "design scheme/brainstorm" or "system setup/implementation":
    - If user explicitly replies needing "design scheme/brainstorm", formally trigger and continue this Skill process.
    - If user replies needing "system configuration/setup", end this Skill process and invoke the `create_tag` tool to assist users.
  - ❌ Mutually exclusive conditions (forbidden trigger): When user instructions contain explicit action verbs (such as "help me build", "create", "establish", "configure" specific tags), never trigger this Skill...

metadata:
  version: "1.0.0"
  dependencies:
    - thinkingengine-mcp
    - te-mcp
---

# User Tag System Design Assistant

- Based on real customer event tracking data, automatically identify industries and sub-categories, design hierarchical, content-rich user tag systems.
- Core value: Not just generic active/paid tags, but truly industry-insightful specialized tags designed based on industry and category differences.
- Core interaction principle: **Do not output all content at once; must confirm step by step through multi-round dialogue and proactively guide users at the end of each response.**

---

## 🔄 I. Interaction and Execution Workflow (Agent Action Guide)

Must interact with users following the steps below in sequence. **Strictly prohibit skipping steps and directly outputting lengthy content.**

---

### Step 1: Metadata Acquisition and Pre-validation (Silent Execution)
1. Confirm `projectId` (use current project by default; if not provided by user, proactively ask, do not guess or auto-explore).
2. Call tools in parallel to acquire event tracking metadata (event table `mcp__te-mcp-analysis__list_events`, event and user properties `mcp__te-mcp-analysis__list_properties`).
3. ⚠️ **Exception interception**: If valid data cannot be acquired, immediately stop and inform user: "Failed to acquire valid event tracking data for this project. Please verify the project ID is correct or if the project contains data..."
4. Compare with [Appendix 3.2 Industry & Category Recognition Feature Dictionary], first identify **industry major categories**, then identify **sub-category** hypothesis results.

---

### Step 2: First Response: Industry Confirmation and Option Provision (Combined)

After acquiring metadata and completing analysis, provide users with identification results and directly offer viewing options.

**Output template**:

```
📊 **Project Metadata Analysis Completed**:

- 🔍 **Total Events**: X
- 📝 **Total Properties**: X
- 🏢 **Identified Industry**: [Industry Name] (Match Rate XX%)
- 🏷️ **Sub-category Hypothesis**: [Category Name]
- 💡 **Reasoning Basis**: Discovered characteristic data such as `[Event A]`, `[Property B]`, etc.

**The entire tag system is divided into 6 levels**:

- 1️⃣ **User Profile & Identity Tags**
- 2️⃣ **Active Behavior Tags** (Login/Launch Frequency & Stickiness)
- 3️⃣ **Usage Depth Tags** (Core Feature Engagement)
- 4️⃣ **Payment & Commercial Value Tags** (Amount, Frequency & Payment Capacity)
- 5️⃣ **[Industry Specific] Specialized Tags** ⭐ (Customized core differentiation tags based on your business)
- 6️⃣ **Lifecycle & Comprehensive Tags** (Combined classification and early warning from previous 5 levels)

👉 **【Next Steps】**:

If the above industry identification is accurate, how would you like to view the tag scheme?
- **A.** View by level (Recommended - we start from level 1, or you can specify which level you want to see first, e.g., "I want to see level 2 first")
- **B.** View core directly: Generate level 5 [Industry] specialized tags directly
- **C.** View complete scheme (output all levels at once)
*(If industry identification is inaccurate, please tell me the actual industry/category directly, and I will redesign the solution for you)*
```
---

### Step 3: Subsequent Responses: Dynamic Content Generation and Additional Suggestions
Based on user selection (single level or all), combine [Appendix 3.3 Tag Hierarchy Architecture & Standard Specification Dictionary], [Appendix 3.4 TE Tag Type Selection Guide] to output content.
After outputting each level's tag definitions, **must immediately follow** with 3 additional sections for that level.

**Single tag definition output format (⚠️ must strictly follow line breaks and formatting below; tables must have blank lines before and after)**:

```
### 3.x Tag Category Name (e.g., 3.2 Active Behavior Tags)

#### 🏷️ 【Tag Chinese Name】

* **Business Definition**: [Describe what user characteristics this represents and what analysis or operation scenarios it applies to]
* **Recommended TE Tag Type**: `[Condition Tag / Metric Value Tag / First/Last Event Tag / SQL Tag / ID Tag]`

**⚙️ Calculation Logic**:

* **Analysis Subject**: `#user_id`
* **Data Source**: `[Event Name]` or `[User Property]`
* **Time Range**: `[Recent 7 Days / Recent 30 Days / All Time, etc.]`
* **Filter Conditions**: `[None / Event property conditions, e.g., payment amount > 0]`
* **Specific Rules**: `[Sum / Deduplicated Count / Boolean Check / Max Value / Days from Today, etc.]`

**📊 Recommended Tag Value Hierarchy**:

| Tag Value | Definition Condition | Business Meaning |
| :--- | :--- | :--- |
| [Value 1] | [Condition] | [What users this represents] |
| [Value 2] | [Condition] | [What users this represents] |

*(Note: Use `---` separator line between different tags)*
```

**Two additional sections that must be appended at the end of this level**:

```

### 🚀 Business Application Scenarios for This Level's Tags
- **Scenario One [Scenario Name]**: [Explain how to use these tags for filtering, stratification, or audience targeting]
- **Scenario Two [Scenario Name]**: [Explain how to combine with reports for analytical insights]

### ⚠️ Missing Fields & Supplementary Suggestions
- [List key events or properties that are missing from current tracking when designing this level's tags, and provide tracking supplementation recommendations. If none are missing, write "Current tracking supports well, no supplementation needed..."]
```

#### Complete Scheme Document Structure Reference

When users select to view all, output according to the following structure:

```markdown
# [Industry/Category] User Tag System Design Scheme

## I. Project Overview
- Project ID / Industry Identification / Sub-category / Matching Basis / Event Count / Property Count

## II. Tag System Architecture Overview

| Level | Tag Count | Type | Core Value |
|------|--------|------|---------|
| Basic Properties | X | Static Profile | User Basic Characteristics |
| Active Behavior | X | Generic | User Stickiness |
| Usage Depth | X | Generic+Industry | Core Feature Usage |
| Payment & Commercial Value | X | Generic | User Commercial Value |
| [Industry] Specialized | X | Industry Differentiation | Industry Core Insights |
| Lifecycle & Comprehensive | X | Comprehensive | User Status Management |

## III. Tag Detail Definitions
### 3.1 Basic Property Tags
### 3.2 Active Behavior Tags
### 3.3 Usage Depth Tags
### 3.4 Payment & Commercial Value Tags
### 3.5 [Industry Sub-category] Specialized Tags  ← Key output, richest content
### 3.6 Lifecycle & Comprehensive Tags

## IV. Priority Implementation Recommendations

**P0 Immediate Implementation**:
- Basic active tags (Recent 7/30 days active days, active stratification)
- Payment stratification tags
- [Industry core tags TOP3]

**P1 Second Batch Implementation**:
- Usage depth tags
- Payment behavior refinement tags
- [Industry specialized tags TOP5]

**P2 Subsequent Refinement**:
- SQL tags (complex logic)
- Lifecycle comprehensive tags
- Promotional/preference tags

## V. Business Application Scenarios

### User Operations Scenarios
- Precise targeting: [Tag combination] → [Operations Action]
- Churn recovery: [Tag combination] → [Recovery Strategy]
- Activity targeting: [Tag combination] → [Activity Scheme]

### Data Analysis Scenarios
- User segmentation comparison / Retention analysis / Conversion funnel analysis

### Product Optimization Scenarios
- Feature iteration / Core user needs identification

## VI. Missing Fields & Supplementary Suggestions
[List key missing fields discovered during design, provide tracking supplementation recommendations]
```

---

### Step 4: Mandatory Closing Guidance (After completing Step 3 content)

After outputting Step 3 content, **must** add guiding language at the very end of the response asking about user's next operation.

- **Single level viewing**: 👉 【Next Step Guidance】: The above is the tag design for level [X]. Would you like to view level [X+1] next, or directly output the complete scheme document summary?
- **Finished specialized level**: 👉 【Next Step Guidance】: These are industry-specialized tags customized for you. Do they align with your business intuition? Need adjustments, or continue viewing level six (lifecycle tags)...
- **Completed full scheme**: 👉 【Next Step Guidance】: The above is the complete tag system scheme. Do you need me to extract several core tags and provide detailed SQL calculation pseudocode, or help you plan implementation priorities?

---


## 📋 II. Output & Interaction Checklist (Agent Self-Check)

Before outputting the scheme, confirm the following:

- [ ] Have MCP tools been called to acquire real metadata?
- [ ] After receiving data for the first time, has **industry category confirmation (3.1)** been performed?
- [ ] Is the interaction currently at the correct workflow step? Has any level been skipped in output?
- [ ] Has **level selection menu (3.2)** been provided to let users choose by demand?
- [ ] When outputting a level's tags, does it include **configuration recommendations, business scenarios, missing field supplementation (3.3)**?
- [ ] 🚨**Most Important**: Does the last paragraph of each response include clear **【Next Step Guidance】(3.4)**?
- [ ] Level five industry-specialized tags have been customized per sub-category, not copied from generic template
- [ ] All event and property names come from real metadata, no fabrication
- [ ] Each tag includes: business definition, TE tag type, calculation rules, tag value hierarchy, usage scenarios
- [ ] Tag value hierarchy has clear numerical ranges or conditions, not vague descriptions
- [ ] When fields are missing, alternative solutions provided
- [ ] Provided priority sequencing (P0/P1/P2)
- [ ] Provided business application scenario examples
- [ ] Has language consistency been maintained? Does response language match user's question language?

### 🎯 Core Principles

1. **Step by step, refuse lengthy content**: Tag systems are substantial; one-time output affects readability. Must interact with users through menu selection and hierarchical decomposition.
2. **Mandatory next-step guidance**: Agent takes on the role of process leader, absolutely cannot end with "statement complete", must use questions or multiple choice to guide users toward next steps or normal conclusion.
3. **Industry differentiation priority**: Level five industry-specialized tags are core value, must truly reflect category differences; different categories' tag content should not be identical
4. **Based on real data**: All event and property names for tags must come from project's real tracking; missing data can only be in "supplementary recommendations"
5. **Clear, implementable definitions**: Each tag's business meaning and calculation rules are clear so team members can understand and use directly
6. **Reasonable tag value hierarchy**: Recommended tag values should have clear ranges, mutually exclusive and covering all users
7. **Scenario-driven**: Each tag explains applicable business scenarios, helping customers understand "what is this tag used for"
8. **No configuration code generation**: This SKILL only handles tag definition and calculation rules; actual configuration is done by MCP tools or manually
9.**Language Consistency Priority**: Agent must respond in the same language as user's question to ensure comfortable communication. Detect input language → Respond in same language → Verify consistency before output.

--- 

## 🧠 III. Appendix: Core Knowledge Base & Rules (Agent Reference Dictionary)

### 3.1 Recognition Logic
1. Statistics industry major category match rate, select highest match (≥25% considered hit)
2. Within matched industry, perform secondary matching on sub-category keywords
3. Sub-category will directly determine level 5 "industry specialized tags" content
4. If cannot identify sub-category, still output industry major category's generic specialized tags
5. If cannot identify industry, mark as "generic industry", output basic version tag system

### 3.2 Industry & Category Recognition Feature Dictionary

#### Industry Major Category Recognition Keywords

| Industry | Event Keywords | Property Keywords |
|------|-----------|-----------|
| Gaming | login, level_up, battle, quest, gacha, recharge, role_create, dungeon, pvp | level, exp, gold, diamond, vip_level, server_id, combat_power |
| E-Commerce | view_product, add_cart, place_order, pay, refund, search, collect, review | product_id, category, price, sku, cart_value, order_amount |
| Finance | apply_loan, invest, withdraw, transfer, kyc, risk_assess, repay, portfolio | amount, balance, credit_score, risk_level, product_type, interest_rate |
| Education | course_view, lesson_complete, exam, homework, enroll, replay, note, assignment | course_id, grade, score, study_time, teacher_id, subject |
| Social/Content | post, comment, like, share, follow, message, live, publish, subscribe | content_type, follower_count, topic, feed_type, interaction_count |
| Travel/Local Services | search_poi, book_ride, arrive, order_food, reserve, scan_bike, check_in | poi_id, distance, city, eta, price, rating, category |
| Health/Medical | record_health, consult_doctor, book_appointment, view_report, track, remind | symptom, metric_type, doctor_id, department, health_score |
| Enterprise SaaS | create_project, invite_member, export, api_call, workflow, permission, billing | workspace_id, plan, member_count, feature, module, usage_quota |
| Media/Entertainment | play_video, pause, seek, finish, subscribe, recommend_click, search, download | content_id, duration, genre, resolution, vip_type, play_position |
| Tools/Productivity | create_file, edit, export, sync, share, template_use, ai_generate | file_type, feature, template_id, output_format, frequency |

#### Sub-category Recognition (After major industry confirmed, further differentiate)

```
## Gaming Sub-categories:
- MMO/RPG: server_id, guild, role, quest, world_boss, etc.
- Card/Idle: card_id, deck, gacha, auto_battle, etc.
- Casual/Puzzle: level_id, life, booster, daily_challenge, etc.
- Strategy/SLG: city_id, alliance, troop, resource_type, etc.
- Competitive/FPS: match_id, rank, kill, team, etc.

## E-Commerce Sub-categories:
- General Marketplace: multiple categories, search-driven, platform-based
- Vertical E-Commerce (Fashion/Beauty/Food/Home): single category depth, matching, ingredients, reviews
- Social/Live Commerce: live_id, kol_id, flash_sale, group_buy
- Cross-border: country, currency, customs, international_shipping
- B2B Procurement: company_id, bulk_order, quote, approval_flow

## Finance Sub-categories:
- Consumer/Cash Loans: apply_loan, credit_limit, repay, overdue
- Wealth/Fund: invest, portfolio, yield, risk_preference
- Insurance: policy, premium, claim, renewal, coverage
- Securities/Stocks: trade, position, watchlist, market_data
- Digital Wallet/Payment: transfer, top_up, scan_pay, bill

## Education Sub-categories:
- K12: grade, subject, homework, exam, parent_view
- Vocational/Certification: certificate, practice_test, study_plan, flashcard
- Language Learning: vocabulary, pronunciation, speaking_practice, streak
- Higher Education/MOOC: university, credit, peer_review, project
- Arts Education (Music/Art/Sports): skill_level, practice_duration, feedback
```

---

### 3.3 Tag Hierarchy Architecture & Standard Specification Dictionary
**When Agent generates specific tags, must strictly reference this dictionary's dimensions for expansion, and substitute output with user's real tracking fields.**

#### Hierarchy Architecture Overview

Tag system consists of **universal baseline layers** and **industry-specialized layers**:

```
Level 1: Basic Property Tags       ← Universal, from user property table
Level 2: Active Behavior Tags      ← Universal, login/launch events
Level 3: Usage Depth Tags          ← Universal+Industry adjustment, core feature usage depth
Level 4: Payment & Commercial Value Tags ← Universal, payment/monetization events
Level 5: Industry Specialized Tags ← Key differentiation, designed per industry and sub-category
Level 6: Lifecycle & Comprehensive Tags ← Universal, comprehensive multi-level user state
```

> ⚠️ Important principle: Level 5 industry-specialized tags are core differentiation value of this scheme, must be carefully designed per actually identified industry sub-category, not generic template.

#### Level 1: User Profile & Identity Tags

```
**Data Source**: User property table (tableType=1) and first/last events
**Recommended TE Tag Types**: Condition Tag, SQL Tag, ID Tag
**Design Principle**: Absolutely do not directly export raw detail fields (e.g., direct output of province, single device type), only design tags that require [rule calculation, state extraction, range division, or multi-field merge] to conclude
**Generic Tag References**:
  - **Account Lifespan Stratification (Condition Tag)**: Convert registration time to business phases. E.g.: Early-stage(<7d), Growth(7-30d), Stable(1-6m), Veteran(>6m).
  - **User Acquisition Quality Attribution (Condition Tag)**: Consolidate hundreds of raw_channel into macro business categories. E.g.: High-quality paid, Social virality, Natural search, Matrix routing.
  - **Core Info Completeness (Condition Tag/SQL Tag)**: Comprehensively judge multiple raw properties. E.g.: High completeness(has avatar+real name+card), Low completeness(registration only).
  - **Cross-device/Multi-device Ecosystem (Metric Value/Condition Tag)**: Statistics independent devices logged in historically. E.g.: Single-device loyal users, Multi-device/cross-platform active users.
  - **Risk/Privilege Identity Features (ID Tag/SQL Tag)**: Black market/wool party suspect devices, beta whitelist users, high-frequency rebinding anomalous accounts.
```

#### Level 2: Active Behavior Tags
```
**Data Source**: Login/Launch/Session related events (tableType=0)
**Recommended TE Tag Types**: Metric Value Tag, Condition Tag
**Design Principle**: Measure user contact frequency and stickiness with product
**Standard Tag Group**:
  - **Active Frequency**: Recent 7/30/90 days active days
  - **Active Continuity**: Current consecutive active days, longest consecutive active days (SQL Tag)
  - **Active Timeliness**: Last active time, days since last active
  - **Active Stratification (Condition Tag)**:
    - High active: Recent 7 days ≥ 5 days
    - Medium active: Recent 7 days 2-4 days
    - Low active: Recent 7 days = 1 day
    - Dormant: 7-30 days inactive
    - Churned: 30+ days inactive
```

#### Level 3: Usage Depth Tags
```
**Data Source**: Core feature operation events (tableType=0)
**Recommended TE Tag Types**: Metric Value Tag, Condition Tag
**Design Principle**: Measure user usage depth of product core value, must combine industry core function definition
**Generic Tags** (applicable to all industries):
  - Recent 7/30 days usage duration
  - Recent 7/30 days core feature operation count
  - Feature coverage breadth (number of functional modules used)
  - Usage time preference (early/afternoon/evening/deep night)

**⚠️ Industry Adjustment Notes**:
- Core feature event names based on real tracking
- Usage depth measurement dimensions vary per industry (gaming measures battle count, education measures study duration, social measures interaction count, etc.)
```

#### Level 4: Payment & Commercial Value Tags

```
**Data Source**: Payment/Recharge/Order related events (tableType=0)
**Recommended TE Tag Types**: Metric Value Tag, Condition Tag, First/Last Event Tag
**Design Principle**: Measure user commercial value and payment behavior patterns
**Payment Capacity Tags**:
  - Cumulative payment amount, recent 30/90 days payment amount
  - Average order value, highest single payment amount
  - **Payment Stratification (must adjust per industry amount range)**:
    - Whale/High-value: Monthly payment ≥ [Industry threshold]
    - Mid-tier/Medium-value: Monthly payment [Industry mid-range]
    - Casual/Low-value: Monthly payment < [Industry low-end]
    - Non-paying: Never showed payment behavior
**Payment Behavior Tags**:
  - Cumulative payment count, recent 30 days payment count
  - First payment time, first payment amount, days from registration to first payment
  - Last payment time, days since last payment
  - Average payment interval days
  - Recent 30 days payment (Yes/No)
**Payment Preference Tags**:
  - Most frequently purchased product/package type
  - Product price preference (high/medium/low)
  - Promotion sensitivity (promotional period payment ratio)
```

#### Level 5: Industry Specialized Tags ⭐ Core Differentiation Layer
> This level must be customized designed per identified **industry + sub-category**. Below lists recommended tags per industry and category; output specific tags combining with real tracking fields.

##### 🎮 Gaming Industry
```
- **Generic Gaming Tags**:
  - Player level / Combat power range (low/medium/high/top tier)
  - Server/Region (old server/new server/merged server)
  - Account server start days
  - Guild/Clan membership (Yes/No)

- **MMO/RPG Exclusive**:
  - Character class/race preference
  - Main quest progress (current chapter/dungeon unlocks)
  - World Boss participation frequency
  - PVP activity (arena wins/rank tier)
  - Guild activity (guild contribution/activity participation)
  - Equipment enhancement depth (highest equipment score)

- **Card/Idle Exclusive**:
  - Card draw count (recent 30 days/cumulative)
  - Strongest card rarity (SSR/SR/R)
  - Deck variety (number of different decks used)
  - Idle time preference (heavy idle/light idle)
  - Event stage participation rate

- **Casual/Puzzle Exclusive**:
  - Current max level
  - Level failure retry rate
  - Item (life/speed/undo) usage frequency
  - Daily task completion rate
  - Level clear speed stratification (quick clear/stuck type)

- **Strategy/SLG Exclusive**:
  - City level/Power range
  - Alliance activity (alliance war participation)
  - Resource collection efficiency (daily harvest amount)
  - Troop recruitment activity (power growth speed)
  - Diplomatic behavior (alliance/attack frequency)

- **Competitive/FPS Exclusive**:
  - Current tier/Rank range
  - Recent 30 days match count
  - Win rate range
  - Most used hero/weapon
  - Team preference (solo/duo/squad)

- **Card/Fishing/Slots (Probability Competitive)**:
  - Betting volatility (aggressive/conservative)
  - Coin level status (broke/lucky win)
  - Average bet multiplier preference
  - Win streak/loss streak status
  - Big Win trigger frequency
  - Coin consumption speed

- **Anime/Open-World/Nurture Type**:
  - Map exploration percentage
  - Side quest coverage rate
  - Hidden achievement completion count
  - Favorite character/main character preference
  - Character skin/outfit ownership count

```

##### 🛍️ E-Commerce Industry
```
- **Generic E-Commerce Tags**:
  - Category purchase breadth (categories purchased)
  - Search preference keyword type
  - Review activity (review count)
  - Refund rate stratification (high refund/low refund)
  - Collected item count (purchase intent)

- **General Marketplace Exclusive**:
  - Cross-category buyer (Yes/No)
  - Platform coupon usage rate
  - Discount sensitivity (Yes/No)
  - Repurchase category distribution

- **Fashion/Beauty Exclusive**:
  - Style preference (basic/trendy/luxury)
  - Size/Skin type (if available)
  - Brand loyalty (same brand repeat count)
  - Sample/trial purchase tendency
  - Seasonal purchase pattern (seasonal/regular)

- **Food/Fresh Exclusive**:
  - Taste preference (spicy/bland/sweet/salty)
  - Purchase cycle (daily/weekly/irregular)
  - Organic/health food preference (Yes/No)
  - Near-expiration discount sensitivity

- **Home/Appliances Exclusive**:
  - Large purchase interval (low-frequency high-value type)
  - Home style preference (modern/Nordic/Chinese)
  - Trade-in participation tendency

- **Live/Social Commerce Exclusive**:
  - Live purchase ratio (live room orders/total orders)
  - Preferred streamer/KOL
  - Flash sale participation frequency
  - Group buying participation rate
```

##### 💰 Finance Industry
```
- **Generic Finance Tags**:
  - KYC verification status (unverified/basic/advanced)
  - Risk assessment level (conservative/stable/aggressive/risk-seeking)
  - Account balance range
  - Financial product usage breadth (products used count)

- **Consumer/Cash Loan Exclusive**:
  - Historical loan count
  - Max loan amount range
  - On-time repayment rate (good/fair/overdue)
  - Loan purpose preference (consumption/turnover/emergency)
  - Current debt status (normal/overdue/cleared)
  - Credit line utilization rate (used/total)

- **Wealth/Fund Exclusive**:
  - Cumulative investment amount range
  - Held product count
  - Preferred product type (money market/bonds/stocks/mixed)
  - Holding period preference (short/medium/long)
  - Portfolio diversification level
  - Auto-investment participation (Yes/No, consecutive months)

- **Insurance Exclusive**:
  - Policy count
  - Insurance type coverage (life/health/property)
  - Renewal rate stratification
  - Claim history (Yes/No, claim count)
  - Annual premium range

- **Securities/Stocks Exclusive**:
  - Recent 30 days trading activity (trading days)
  - Held stock count
  - Preferred market (A-share/HK/US)
  - Preferred sector (tech/healthcare/consumer/new energy)
  - Average holding period (short/medium/long)
```

##### 📚 Education Industry
```
- **Generic Education Tags**:
  - Learning consecutive days (check-in continuity)
  - Course completion rate range
  - Study time preference (early/afternoon/evening/deep night)
  - Course review activity

- **K12 Exclusive**:
  - Grade level (elementary/middle/high)
  - Weak subject (math/language/English...)
  - Parent dashboard activity (parent supervision type/self-study type)
  - Error problem re-attempt rate
  - Exam prep status (Yes/No, days until exam)

- **Vocational/Certification Exclusive**:
  - Target certificate (CPA/PMP/Teaching credential...)
  - Exam timeline (preparation/sprint/post-exam)
  - Mock test correct rate range
  - Study plan completion rate
  - Check-in consecutive days

- **Language Learning Exclusive**:
  - Target language (English/Japanese/Korean/French...)
  - Current level/score tier
  - Speaking practice activity
  - Vocabulary growth speed
  - Consecutive learning days (Streak)
  - Weak skill (listening/speaking/reading/writing)

- **Arts Education Exclusive**:
  - Practice duration (per session/cumulative)
  - Skill level progression speed
  - Teacher feedback interaction rate
  - Work submission frequency
```

##### 📱 Social/Content Industry
```
- **Generic Social/Content Tags**:
  - Content consumption preference (video/text/audio)
  - Interaction activity (like/comment/share combined)
  - Follow/Follower ratio (content consumer/creator)
  - Content completion rate range

- **Short-Video Platform Exclusive**:
  - Content preference topic (comedy/knowledge/food/gaming...)
  - Average single session watch duration
  - Completion rate preference (like short/long videos)
  - Share activity
  - Live stream gift frequency

- **Text/News Platform Exclusive**:
  - Followed topic type
  - Collection frequency
  - Article read completion rate
  - Comment interaction style (frequent commenter/read-only)
  - Novel genre preference (fantasy/urban/CEO)
  - Reading speed stratification
  - Update tolerance

- **Creator Tags**:
  - Publication frequency (high/medium/low/silent)
  - Content quality stratification (average interaction volume range)
  - Follower growth speed
  - Monetization participation (Yes/No)
```

##### 🚗 Travel/Local Services Industry
```
- **Generic Tags**:
  - Frequently used city/region
  - Peak time usage preference
  - Platform service usage breadth (service types used count)

- **Ride-sharing/Travel Exclusive**:
  - Ride frequency (recent 30 days ride count)
  - Preferred vehicle type (economy/comfort/premium)
  - Regular route pattern (commute type/non-commute)
  - Carpool acceptance (Yes/No)
  - Peak hour ride proportion

- **Food Delivery/Dining Exclusive**:
  - Order frequency (recent 7/30 days)
  - Preferred cuisine type
  - Average order value range
  - Order time preference (early/noon/evening/late night)
  - Coupon usage rate
  - Review activity

- **Hotel/Tourism Exclusive**:
  - Travel frequency (recent 180 days)
  - Preferred trip type (business/leisure/family)
  - Booking advance time (last-minute/1 week/1 month advance)
  - Preferred accommodation level (budget/comfort/luxury)
  - Destination type (city/resort/attraction)
```

##### 🏥 Health/Medical Industry
```
- **Generic Tags**:
  - Health concern type (chronic disease management/fitness/nutrition/mental health)
  - Data recording frequency (regular/occasional)
  - Goal setting status (clear goals/none)

- **Health Management App Exclusive**:
  - Monitored metric type (blood sugar/pressure/weight/heart rate...)
  - Health data achievement rate (recent 30 days)
  - Exercise completion rate
  - Reminder feature usage (medication/hydration/exercise)

- **Internet Medical Exclusive**:
  - Consultation frequency
  - Preferred department/disease area
  - Prescription refill rate (chronic disease refill type)
  - Report review frequency
```

##### 🏢 Enterprise SaaS Industry
```
- **Generic Tags**:
  - Subscription plan/Plan (Free/Starter/Pro/Enterprise)
  - Team organization scale (member count range)
  - Product usage duration (days since opening)
  - Feature usage breadth (activated features/total features)

- **Collaboration/Project Management Exclusive**:
  - Project creation frequency
  - Member invitation activity
  - Task assignment/workflow activity
  - Comment/feedback interaction frequency
  - Export/integration usage

- **Marketing SaaS Exclusive**:
  - Campaign creation frequency
  - Distribution channel count
  - Data report view frequency
  - A/B test usage

- **Renewal/Expansion Risk Tags**:
  - Quota usage rate (used/limit, high-risk for expansion or churn)
  - Recent 30 days feature activity decline rate
  - Core feature suspension flag
  - Renewal countdown (>90d/30-90d/within 30d/expired)
```

#### Level 6: Lifecycle & Comprehensive Tags

```
- **Data Source**: Combined multi-level tag calculation
- **Recommended TE Tag Types**: Condition Tag, SQL Tag
- **Design Principle**: Combine user activity, payment, industry-specialized behavior to define lifecycle stage and overall value

- **Lifecycle Stages** (Condition Tag, must adjust conditions per industry):
  - **Onboarding**: <7 days registration, not completed core behavior
  - **Growth**: 7-30 days registration, completed core behavior but not paying
  - **Mature**: Paid and active
  - **High-value**: High payment + High activity + Industry core behavior high frequency
  - **Declining**: Previously paid, active significantly decreased
  - **Dormant**: 7-30 days inactive
  - **Churned**: 30+ days inactive
  - **Returning**: Resumed activity after churn

- **Comprehensive Value Tags**:
  - **RFM Composite Score**: R(last active/payment timeliness) + F(usage/payment frequency) + M(payment amount)
  - **User Value Tier**: High-value/Medium-value/Low-value/Negative-value (high support cost)
  - **Churn Risk Early Warning**: Recent 7/14 days activity decline >50% + Days since last payment >30
  - **Returning User**: Resumed activity within 7 days after churn
```

### 3.4 TE Tag Type Selection Guide

| Tag Type | Application Scenario | Typical Example |
|---------|---------|---------|
| **Condition Tag** | Divide users into multiple mutually exclusive categories, determine if conditions met | Active stratification, payment stratification, lifecycle stage, player rank tier |
| **Metric Value Tag** | Statistics behavior count/amount/days within time range | Recent 30 days payment amount, recent 7 days active days, cumulative course completions |
| **First/Last Event Tag** | Record first/last time event occurred or associated property value | First payment amount, last active time, last purchase category |
| **SQL Tag** | Complex calculation logic (consecutive days, ratios, rankings, etc.) | Consecutive active days, payment conversion rate, quota utilization rate |
| **ID Tag** | Import external lists, blacklist/whitelist management | VIP list, test accounts, risk control blacklist |

### 3.5 Field Validation Baseline Principle

- Must use real fields: All event and property names must come from metadata query results.
- Missing field handling: If key fields needed for design are missing, cannot fabricate; must point out in [Missing Fields & Supplementary Suggestions] section and provide alternative solutions using existing fields or suggest new tracking requirements.

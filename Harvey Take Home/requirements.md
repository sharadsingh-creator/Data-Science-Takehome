# Analytics Engineering Project Requirements

## 1. Data Modeling Requirements

The core objective is to create a robust and scalable analytical layer (`dbt` or SQL scripts) using the provided `users`, `events`, and `firms` tables. All final model column names must be **lowercase** and **snake_case**.

---

### Model 1: `user_engagement`

**Goal:** Provide a monthly snapshot of individual user behavior for stickiness and adoption analysis.

| Requirement | Description | Source/Calculation | Grain |
| :--- | :--- | :--- | :--- |
| **Primary Key** | Composite: `user_id` + `activity_month` | Derived | User-Month |
| **`activity_month`** | Calendar month of activity | `DATE_TRUNC('month', events.created)` | |
| **`monthly_queries`** | Total events logged | `COUNT(events.created)` | |
| **`first_active_date`** | First event date in user history | `MIN(events.created)` | |
| **`last_active_date`** | Last event date in the month | `MAX(events.created)` | |
| **`average_feedback_score`** | Mean feedback score | `AVG(events.feedback_score)` | |
| **Event Type Metrics** | Count of specific event types (e.g., `assistant_queries`) | Conditional Aggregation on `events.event_type` | |
| **`distinct_event_types`** | Count of unique event types used in the month. | `COUNT(DISTINCT events.event_type)` | |
| **`engagement_level`** | Categorical classification of usage (Required for Question 2.A answer). | Defined by custom logic (See Section 3) | |

---

### Model 2: `firm_usage_summary`

**Goal:** Provide a single-row aggregate view of firm health, metadata, and overall usage.

| Requirement | Description | Source/Calculation | Grain |
| :--- | :--- | :--- | :--- |
| **Primary Key** | `firm_id` | `firms.id` | Firm |
| **`total_queries`** | Total lifetime queries by the firm | `COUNT(events.created)` grouped by firm | |
| **`active_users`** | Count of distinct users who have logged $\ge 1$ event (ever). | `COUNT(DISTINCT events.user_id)` | |
| **`provisioned_users`** | Total users mapped to the firm (Crucial requirement). | **Assumption-Based:** Count of unique users whose *primary activity* (most events) is linked to this firm. | |
| **`active_to_provisioned_ratio`** | Ratio of active users to provisioned users. | `active_users` / `provisioned_users` | |
| **`average_firm_feedback_score`** | Mean feedback score across all firm events. | `AVG(events.feedback_score)` | |

---

### Model 3: `daily_event_summary`

**Goal:** Provide daily time-series statistics for operational monitoring and trend analysis.

| Requirement | Description | Source/Calculation | Grain |
| :--- | :--- | :--- | :--- |
| **Primary Key** | `event_date` | `DATE(events.created)` | Day |
| **`daily_total_queries`** | Total events on that day. | `COUNT(events.created)` | |
| **`daily_distinct_users`** | Count of unique users active on that day. | `COUNT(DISTINCT events.user_id)` | |
| **`daily_distinct_firms`** | Count of unique firms active on that day. | `COUNT(DISTINCT events.firm_id)` | |
| **`daily_avg_feedback_score`** | Mean feedback score on that day. | `AVG(events.feedback_score)` | |
| **Event Type Metrics** | Daily counts of specific event types (e.g., `daily_assistant_queries`). | Conditional Aggregation on `events.event_type` | |

---

## 2. Analytical Question Requirements

The following questions must be answered using the modeled data, with justifications based on percentile analysis and data quality checks.

### 2.A. Power User Definition

* **Requirement:** Define a **Power User** segment using criteria from the `user_engagement` model.
* **Justification (Derived from Input):** The definition must rely on **two criteria**:
    1.  **Volume:** `monthly_queries` $\ge 30$ (Chosen based on being well above the 99th percentile of 27).
    2.  **Breadth:** `distinct_event_types` $\ge 3$ (Chosen based on covering all possible event types or the 90th/95th percentile).
* **Final Definition:** A Power User is defined as a user-month where **`monthly_queries` $\ge 30$ AND `distinct_event_types` $\ge 3$**.

### 2.B. Data Quality Concerns

* **Requirement:** Identify and document at least two significant data quality or definition concerns.
* **Required Concerns (Derived from Input):**
    1.  **Feedback Score Reliability:** The non-null(none columns missing), perfect completion rate of `feedback_score`, requiring clarification on whether it is customer-provided or an internal metric.
    2.  **User-to-Firm Mapping Ambiguity:** Lack of a primary `firm_id` in the `users` source table, necessitating a clear, documented assumption for calculating `provisioned_users`.
    3.  **Low Engagement Business Risk:** **~69%** of users are in the **Low + Inactive** tiers, flagging this as a major business health concern ("shelfware" risk).

---

## 3. Documentation Requirements

A `README.md` file must be included, providing clarity on the modeling approach and assumptions.

1.  **Assumptions:** Clearly stated the definition used for **Power User** and the assumption used for the **User-to-Firm Mapping** (`provisioned_users`).
2.  **Model Interpretation:** Provided a brief explanation of the goal and key metrics (e.g., grain) for all three final models.

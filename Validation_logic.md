# **Validation Logic**

## **1\. Purpose**

Define the engine and step-by-step logic that compares extracted document data against the structured rules produced by the Rule Parser. This document covers:

- Overall architecture and data flow  
- Support for atomic (>, <, ==, !=) and composite (AND/OR/NOT) rule evaluation  
- Multi-field comparisons (e.g. date1 < date2) and temporal windows (`within_X_days`)  
- Error handling, fallbacks, and logging  
- Sample validation report format  

  ## **1.1 Tools & Technologies**

  This validation module is implemented using:

  - **Language:** Python 3.10+  
  - **Date/Time:** `datetime` (standard library)  
  - **Serialization:** `json` (standard library)  
  - **Logging:** `logging` or `structlog` for structured, context-rich logs  
  - **Optional:** lightweight rule-DSL parser (e.g. `lark-parser`) for more complex rule grammars  

---


## **2\. Architecture Overview**

```
┌─────────────────┐
│  Structured     │
│  Rules          │   ┌─────────────────┐
└────────┬────────┘───▶ Rule Dispatcher
         │            └────────┬────────┘
┌────────▼────────┐             │
│ Extracted Fields│             │
└────────┬────────┘             │
         │                      │
         │                      ▼
         │            ┌────────────────────┐
         │            │  Preprocessor      │
         │            │  (normalize types) │
         │            └────────┬───────────┘
         │                      │
         │                      ▼
         │            ┌────────────────────┐
         │            │  Atomic Evaluator  │
         │            │  (simple checks)   │
         │            └────────┬───────────┘
         │                      │
         │                      ▼
         │            ┌────────────────────┐
         │            │ Composite Evaluator│
         │            │ (AND/OR/NOT logic) │
         │            └────────┬───────────┘
         │                      │
         │                      ▼
         │            ┌────────────────────┐
         └───────────▶  Error Handler    
                      └────────┬───────────┘
                               │
                               ▼
                      ┌────────────────────┐
                      │ Report Assembler   │
                      └────────────────────┘
```

* **Rule Dispatcher:** Routes each rule (atomic or composite) to the correct evaluator.

* **Condition Guard:** Normalizes and types values (dates, numbers, strings).

* **Evaluator Module:** Checks single comparisons (`>`, `<`, `==`, etc.) and specialized operators (`within_X_days`).

* **Composite Evaluator:** Recursively applies logical connectors (`AND`, `OR`, `NOT`).

* **Error Handler:** Applies `on_error` strategy (`raise`, `warn_and_skip`) when rules can’t be evaluated.

* **Report Collector:** Aggregates pass/fail status, actual values, and error messages into the final report.

---

## **3\. Data Models**

### **3.1 Structured Rule (Input)**


    `{`  
      `"rules": [`  
        `{`  
          `"field": "invoice_total",`  
          `"target": null,                // or "payment_date" for field-to-field`  
          `"operator": ">",`  
          `"value": 1000,`  
          `"logic": null,`  
          `"sub_rules": null,`  
          `"on_error": "warn_and_skip"`  
        `},`  
        `{`  
          `"logic": "AND",`  
          `"sub_rules": [ /* nested rules */ ]`  
        `}`  
      `]`  
    `}`

### **3.2 Extracted Fields (Input)**
  
    `{`  
      `"invoice_total": 875,`  
      `"invoice_date": "2025-01-14",`  
      `"payment_date": "2025-02-10"`  
    `}`

### **3.3 Validation Report (Output)**
 
    `{`  
      `"results": [`  
        `{`  
          `"rule": "invoice_total > 1000",`  
          `"status": "FAIL",`  
          `"actual": 875,`  
          `"expected": "> 1000"`  
        `},`  
        `{`  
          `"rule": "(invoice_date within_30_days payment_date)",`  
          `"status": "PASS",`  
          `"actual": 27,`  
          `"expected": "≤ 30 days"`  
        `}`  
      `],`  
      `"summary": {`  
        `"total": 2,`  
        `"passed": 1,`  
        `"failed": 1`  
      `}`  
    `}`

---

## **4\. Evaluation Flow**

1. **Preprocess Inputs**

   * Normalize field names to snake\_case.

   * Parse date strings into `datetime.date`.

   * Convert numeric strings (strip `$`, `,`) to floats/ints.

2. **Dispatch Rules**

   * If `logic` is set, send to Composite Evaluator.

   * Else send to Atomic Evaluator.

3. **Atomic Evaluation**

   * Pull `lhs = fields[field]`.

   * Pull `rhs = value` or `fields[target]`.

   * Call `evaluate_atomic(lhs, operator, rhs)`.

4. **Composite Evaluation**

   * Recursively evaluate each sub-rule.

   * Combine results using `AND`, `OR`, `NOT`.

5. **Error Handling**

   * If a field is missing or type mismatch:

     * If `on_error == "raise"`, abort and bubble error.

     * If `on_error == "warn_and_skip"`, mark rule `"ERROR"` and continue.

6. **Report Assembly**

   * For each rule: record `rule`, `status`, `actual`, `expected`.

   * Compute `summary` counts.

---

## **5\. Atomic Evaluator Details**

### **5.1 Numeric Comparisons**

*Example pseudocode:*

    def eval_numeric(lhs: float, op: str, rhs: float) -> bool:
        ops = {
            ">": lhs > rhs,
            "<": lhs < rhs,
            ">=": lhs >= rhs,
            "<=": lhs <= rhs,
            "==": lhs == rhs,
            "!=": lhs != rhs
        }
        return ops[op]

    # Examples:
    assert eval_numeric(875, ">", 1000) is False
    assert eval_numeric(1001, ">=", 1000) is True


### **5.2 Date Comparisons**

* `>` / `<` / `>=` / `<=` on dates
#### 5.2.a Fixed Window (within_X_days)
      from datetime import date

      def eval_within_X_days(lhs: date, rhs: date, X: int) -> bool:
          delta = abs((lhs - rhs).days)
          return delta <= X

      # Example:
      d1 = date(2025, 2, 10)
      d2 = date(2025, 1, 14)
      assert eval_within_X_days(d1, d2, 30) is True

#### 5.2.b Fixed Window (within_X_days)
      from datetime import date

      def eval_within_last_days(lhs: date, rhs: date, days: int) -> bool:
          delta = (lhs - rhs).days
          return 0 <= delta <= days

      # Example:
      today = date.today()
      past = date(2025, 1, 14)
      assert eval_within_last_days(today, past, 30) is True



      

### **5.3 String Comparisons**

* Case-insensitive equality/inequality for categorical fields.

      def eval_string(lhs: str, op: str, rhs: str) -> bool:
          lhs, rhs = lhs.lower(), rhs.lower()
          if op == "==": return lhs == rhs
          if op == "!=": return lhs != rhs
          raise ValueError(f"Unsupported op: {op}")

### **5.4 Field-to-Field**

* When `target` is provided, fetch both fields and apply same logic.

---

## **6\. Composite Evaluator**

      def evaluate_composite(rule: dict, fields: dict) -> (str, list[bool]):
      logic = rule["logic"]          # "AND", "OR", "NOT"
      subs   = rule["sub_rules"]     # list of rule dicts
      results = []

      for r in subs:
          if r.get("sub_rules"):
              status, _ = evaluate_composite(r, fields)
          else:
              status = "PASS" if evaluate_atomic(
                  fields.get(r["field"]),
                  r["operator"],
                  r.get("value", fields.get(r.get("target")))
              ) else "FAIL"
          results.append(status == "PASS")

      if logic == "AND":
          overall = all(results)
      elif logic == "OR":
          overall = any(results)
      elif logic == "NOT":
          overall = not results[0]
      else:
          raise ValueError(f"Unknown logic: {logic}")

      return ("PASS" if overall else "FAIL"), results

      # **Full Example**:
      fields = {"a": 10, "b": 5}
      rule = {
          "logic": "AND",
          "sub_rules": [
              {"field": "a", "operator": ">", "value": 8},
              {"field": "b", "operator": "<", "value": 10}
          ]
      }
      status, detail_flags = evaluate_composite(rule, fields)
      # status == "PASS", detail_flags == [True, True]

* Handles arbitrarily nested structures.

* Short-circuits evaluation for efficiency.

---

## **7\. Error Handling & Fallbacks**

* **Missing Fieldor type mismatch:** `warn_and_skip` → record `actual: null`, `status: "ERROR"`

* **Date Parse Error:** fallback to natural‐language parse via LLM (optional).

All errors are logged with context (rule index, field name, raw value).

---

## **8\. Logging & Monitoring**

* **Per-rule logs** include timestamp, rule text, status, actual/expected, any exception.

* **Metrics**: average evaluation time, error rates per rule type.

* **Alerting** if \> X% rules error in a batch.

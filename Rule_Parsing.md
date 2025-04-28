# **Rule Parsing** 

## **1\. Purpose**

Define how an LLM should **convert plain-English business rules into a strict, machine-readable JSON schema**. This document captures:
- The **prompt templates**.
- The **expected JSON rule format**.
- **Illustrative examples** and how the LLM should respond.
- **Operational notes** regarding LLM cost and scalability considerations.
---

## **2\. JSON Rule Schema**

    {

    "rules": \[

    {

      "field": "\<field\_name\>",         // e.g., "total\_amount", "contract\_date"

      "target": "\<target\_field\>",     // optional: for field-to-field comparisons (e.g. dates)

      "op": "\<operator\>",            // one of: "\>", "\<", "\>=", "\<=", "==", "\!=", "within\_last\_days", "within\_X\_days"

      "value": \<number|string|integer\>,// literal value (number or ISO date) or integer for days

      "logic": "\<connector\>",        // optional: "AND", "OR", "NOT" (for composite/inversion)

      "sub\_rules": \[\<nested rules\>\],  // only for composite rules

      "on\_error": "\<error\_strategy\>" // optional: e.g. "raise", "warn\_and\_skip"

      }

     \]

    }

* **field**: primary document field to check  
* **target**: secondary field when comparing two fields  
* **op**: comparison or time‑based operator  
* **value**: threshold, date literal, or days count  
* **logic**: how multiple rules connect (omit for single rules)  
* **sub\_rules**: nested rules for AND/OR/NOT logic  
* **on\_error**: how to handle parsing or extraction failures

**Note:** Use either `value` (literal) or `target` (field-to-field) in each rule. If comparing two fields, omit `value` and set `target` accordingly.

---

# **3. LLM Prompting Strategy**

We instruct the LLM precisely to ensure consistent structured outputs without any extra commentary.

---

## **3.1 Atomic Rule Prompt Template**

**System Instruction:**
> "You are a rules‑parsing assistant for compliance automation. Convert each plain-English rule into a strict JSON object following the provided schema. Output only valid JSON."

**Prompt Template:**

    "You are a document-rule parser. Convert the following plain-English rule into a JSON object with exactly these keys:• field: the normalized document field name (e.g. "total\_amount", "contract\_date")

    • op: one of {"\>", "\<", "\>=", "\<=", "==", "\!=", "within\_last\_days", "within\_X\_days"}

    • value: a numeric literal, ISO-formatted date string, or integer for “X” in within\_X\_days.

    Do not wrap in code fences or explanatory text.

    English rule: \"{RULE_TEXT}\"

    JSON:"

## **3.2 Composite Rule Prompt Template**

**System Instruction:**
> "You are a compliance-focused rule parser. Convert the English rule into a nested JSON logic tree using atomic and composite rule schemas. Only output strict JSON."

**Prompt Template:**

    You are a compliance-focused rule parser. Convert this English rule into a nested JSON logic tree using this schema:

    • Atomic rule

    { "field": "\<field\_name\>", "op": "\<operator\>", "value": \<number\_or\_date\_or\_days\> }

    • Composite rule

    { "op": "AND" | "OR" | "NOT", "rules": \[ \<atomic\_or\_composite\_rule\>, … \] }

    NOT inverts its single sub-rule’s condition.

    AND / OR combine multiple sub-rules.

    Support temporal “within X days” as either "within\_last\_days" or "within\_X\_days" with X replaced by the integer value.

    Include \`target\` for field-to-field comparisons, or \`value\` for literals.

    If unable to parse, set "on\_error": "warn\_and\_skip".

    Output only the JSON—no commentary.

    English rule: "{RULE\_TEXT}"

    JSON: ———

    ---


## **4\. Examples**

### **4.1 Single Comparison**

**Input:**

The invoice total must be greater than $1000

**Output:**

    {  
      "rules": \[  
        {  
          "field": "invoice\_total",  
          "operator": "\>",  
          "value": 1000  
        }  
      \]  
    }

### **4.2 Inequality and Negation**

**Input:**

Status must not equal "approved"

**Output:**

    {  
      "rules": \[  
        {  
          "field": "status",  
          "operator": "\!=",  
          "value": "approved"  
        }  
      \]  
    }

### **4.3 Time‑Based Condition**

**Input:**

Payment should be made within 30 days of invoice date

**Output:**

    {  
      "rules": \[  
        {  
          "field": "payment\_date \- invoice\_date",  
          "operator": "within\_X\_days",  
          "value": 30  
        }  
      \]  
    }

### **4.4 Composite Rule with AND/OR/NOT**

**Input:**

Total \> 5000 AND (contract\_date \>= "2025-01-01" OR NOT status \== "cancelled")

**Output:**

    {  
      "rules": \[  
        {  
          "logic": "AND",  
          "sub\_rules": \[  
            { "field": "total", "operator": "\>", "value": 5000 },  
            {  
              "logic": "OR",  
              "sub\_rules": \[  
                { "field": "contract\_date", "operator": "\>=", "value": "2025-01-01" },  
                {  
                  "logic": "NOT",  
                  "sub\_rules": \[  
                    { "field": "status", "operator": "==", "value": "cancelled" }  
                  \]  
                }  
              \]  
            }  
          \]  
        }  
      \]  
    }

---
# **5\. Selected LLM Tool for Rule Parsing**

> **Chosen LLM API:** Llama 3.3 70B Instruct Turbo (via Hugging Face or Together AI Free Tier)

**Reasons for Selection:**
- **Free access** under generous quotas.
- **Model:** Llama 3.3 70B Instruct Turbo — optimized for following instructions and structured outputs.
- **Token limit:** Supports up to up to 32k tokens.
- **Request/minute:** ~20 requests per minute free on Together AI.
- **latency:** around 0.26 seconds.
- **Open-source model**: No vendor lock-in, easy customization.
- **Cost Efficiency:** Saves ~$100–$500 monthly compared to using GPT-4 at similar usage levels.
- **Instruction following:** Very strong when system prompts are carefully designed.

**Alternative Considered:**
- **OpenAI GPT-4**: Excellent JSON reliability and robustness. However, it requires paid API usage (~$0.003–$0.009 per 1k tokens), making it less suitable under the "free/public" constraint but could be used in a production environment later when budgets allow.

✅ Llama 3.3 70B Instruct Turbo balances **quality**, **free access**, and **deployment flexibility**, making it the best fit for this project at this stage.

---
# **6\. Operational Considerations**

- **LLM Selection** considers **rate limits**, **API latency**, and **operational costs** depending on deployment scale.
- Designed **prompting** to minimize token usage and ensure concise JSON outputs.
  
---

## **7\. Notes**

* Added `!=` for strict inequality checks.

* Clarified that `NOT` always inverts the immediately nested sub\_rule.

* Chose both `within_last_days` (generic) and `within_X_days` (parameterized) for flexibility.

* Emphasized pure JSON output to simplify downstream validation logic.  
* Default to ISO date format and strip currency symbols in preprocessing.  
* Added `on_error` to surface parsing failures or ambiguities explicitly.

---


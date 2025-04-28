# Solution outline

---

## High-Level Summary

- **Supports complex English rule parsing into strict JSON.**
- **Handles multi-format document ingestion with fallback mechanisms.**
- **Robust validation engine with composite logic evaluation.**
- **Modular architecture for scaling and auditing.**
- **Secure deployment with Azure CI/CD pipelines.**

---

## 1. Rule Parsing

- **Purpose:**
  - Convert user-written English rules into structured JSON.

- **Method:**
  - Use LLMs (OpenAI, Claude, etc.) with refined prompt templates.
  - Two prompt strategies:
    - **Atomic Rule Parsing:** Direct mapping of simple conditions.
    - **Composite Rule Parsing:** Handling of AND/OR/NOT nested logic.

- **Format:**
  - JSON Schema: `field`, `target`, `op`, `value`, `logic`, `sub_rules`, `on_error`.

- **Edge Cases:**
  - Missing fields, ambiguous conditions.
  - Fallback to "warn_and_skip" on parsing uncertainty.

- **Scalability Note:**
  - API rate limits, latency, and operational cost considerations were factored in.

---

## 2. Document Extraction

- **Purpose:**
  - Extract important fields (dates, totals, names) from PDFs, images.

- **Primary Engine:**
  - **RAGFlow's DeepDoc** module for OCR, layout analysis, field segmentation.

- **Fallback Tools:**
  - **PDFPlumber** (PDF parsing)
  - **Tesseract OCR** (scanned image parsing)

- **Extraction Workflow:**
  - DeepDoc primary OCR ➔ confidence check ➔ fallback OCR if <80% confidence.
  - Use regex-based field templates and LLM normalization prompts.

- **Auditability:**
  - Grounded citations: extracted fields trace back to document coordinates.

---

## 3. Validation Engine

- **Purpose:**
  - Compare extracted data against structured rules.

- **Architecture Components:**
  - Rule Dispatcher ➔ Preprocessor ➔ Atomic Evaluator ➔ Composite Evaluator ➔ Error Handler ➔ Report Assembler.

- **Supported Operations:**
  - Numeric comparisons (>, <, ==, !=)
  - Date comparisons and `within_X_days` checks
  - String comparisons (case-insensitive)
  - Field-to-field dynamic comparisons

- **Error Handling:**
  - `warn_and_skip` for missing/malformed fields.

- **Output:**
  - Detailed validation report with per-rule status and a summary.

---

## 4. Frontend Plan

- **Framework:**
  - **Streamlit**

- **UI Components:**
  - Rule input textbox with "Parse Rule" button.
  - File uploader with "Extract Fields" button.
  - "Validate" button to trigger validation.
  - Progress bars and inline warning messages.

- **State Management:**
  - Streamlit's built-in session state.

- **API Integration:**
  - Calls backend FastAPI endpoints for rule parsing, extraction, and validation.

---

## 5. REST API Design

- **Framework:**
  - **FastAPI** (async-ready, OpenAPI auto-docs)

- **Endpoints:**
  - `POST /parse-rule`: Convert rule text to JSON.
  - `POST /extract`: Upload documents and extract fields.
  - `POST /validate`: Validate fields against parsed rules.
  - `GET /health`: Health check endpoint.

---

## 6. Dockerization Plan

- **Setup:**
  - `docker-compose.yml` orchestrating 3 services:
    - RAGFlow OCR server
    - Rule Engine API server
    - Frontend Streamlit UI
---

## 7. CI/CD & Hosting

- **Repository:**
  - **Azure DevOps**

- **Pipeline:**
  - Push triggers lint/test/build.
  - Builds Docker images.
  - Pushes to **Azure Container Registry**.
  - Deploys to **Azure Container Instances**.

- **Secrets Management:**
  - All secrets handled via **Azure Key Vault**.

---

> **Final Note:** The system is designed for real-world scalability, error tolerance, security, and extensibility across industries like compliance, finance, and legal document checking.

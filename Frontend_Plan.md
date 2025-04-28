# **Frontend & Deployment Plan** 

---

## **1\. Frontend** 

**Why Streamlit?**

* Rapid UI in Python

* Easy integration with our FastAPI backend

* Built-in state management and messaging

### **1.1 UI Components & Validation**

1. **Rule Input Panel**

* **Text Box** for plain-English rules  
   * **“Parse Rule” Button**  
     ```python
     if st.button("Parse Rule"):
         if not rule_text.strip():
             st.warning("Please provide rule text before parsing.")
             st.error_log("Parse Rule pressed with empty input.")
         else:
             # call POST /parse-rule
     ```
   * **“Clear All” Button** to reset both rule input and any displayed results  
     ```python
     if st.button("Clear All"):
         st.session_state.rule_text = ""
         st.session_state.parsed_rules = None
         st.session_state.extracted_fields = None
     ```

2. **Document Upload**

   * **File Uploader** (accepts PDF, JPG, PNG, DOCX)

**“Extract Fields” Button**

    
    `if st.button("Extract Fields"):`  
        `if not uploaded_files:`  
            `st.warning("Please upload one or more documents first.")`  
            `st.error_log("Extract pressed with no files.")`  
        `else:`  
            `# call POST /extract`

3. **Validation Section**

**“Validate” Button**

    
    `if st.button("Validate"):`  
        `if not parsed_rules or not extracted_fields:`  
            `st.warning("Ensure rules are parsed and documents are extracted before validating.")`  
            `st.error_log("Validate pressed without prerequisites.")`  
        `else:`  
            `# call POST /validate`

4. **Logs & Feedback**  
   * Display warnings/errors inline, highlighting the specific rule or field that failed  
   * Show a progress bar for any long-running operations (OCR, parsing, embedding)  


## **2\. REST API (FastAPI)**

**Why FastAPI vs. Django?**

* **Performance:** Async by default, very low latency

* **Lightweight:** Focused on APIs without the overhead of Django’s ORM/views

* **Auto-docs:** Built-in OpenAPI/Swagger UI for quick testing

* **Developer DX:** Clear Pydantic models, type hints, automatic validation

### 

### 

### 

### 

### **2.1 Endpoints & Error Handling**

| Endpoint | Method | Request Schema | Success (200) | Error Responses |
| ----- | ----- | ----- | ----- | ----- |
| `/parse-rule` | POST | `{ rule_text: str }` | `{ rules: Rule[] }` | 422: missing/invalid JSON, 400: empty `rule_text` |
| `/extract` | POST | `multipart/form-data: file: UploadFile` | `{ fields: {...}, errors: [FieldError] }` | 415: unsupported file, 400: no file uploaded |
| `/validate` | POST | `{ rules: Rule[], fields: {...} }` | `{ results: [...], summary: {...} }` | 400: missing `rules` or `fields`, 422: schema error |
| `/health` | GET | — | `{ status: "OK", uptime: str }` | 500: service unavailable |

---

## **3\. Dockerization Plan`docker-compose.yml`**

        `version: "3.8"`

        `services:`  
          `# 1. RAGFlow server (OCR, chunking, embedding)`  
          `ragflow:`  
            `image: infiniflow/ragflow:v0.18.0-slim`  
            `environment:`  
              `MINIO_PASSWORD: "${MINIO_PASSWORD}"`  
              `ES_PASSWORD: "${ES_PASSWORD}"`  
            `ports:`  
              `- "8001:80"`  
            `ulimits:`  
              `memlock:`  
                `soft: -1`  
                `hard: -1`  
            `sysctls:`  
              `- vm.max_map_count=262144`

          `# 2. Rule Engine API (FastAPI)`  
          `rule-engine:`  
            `build:`  
              `context: ./rule_engine`  
              `dockerfile: Dockerfile`  
            `image: myrepo/rule-engine:latest`  
            `depends_on:`  
              `- ragflow`  
            `ports:`  
              `- "8000:8000"`  
            `environment:`  
              `LOG_LEVEL: info`

          `# 3. Frontend UI (Streamlit)`  
          `ui:`  
            `build:`  
              `context: ./ui`  
              `dockerfile: Dockerfile`  
            `image: myrepo/frontend:latest`  
            `depends_on:`  
              `- rule-engine`  
            `ports:`  
              `- "8501:8501"`  
            `environment:`  
              `STREAMLIT_SERVER_PORT: 8501`

### **`rule_engine/Dockerfile`**

    `FROM python:3.10-slim`  
    `WORKDIR /app`  
    `COPY requirements.txt .`  
    `RUN pip install --no-cache-dir -r requirements.txt`  
    `COPY . .`  
    `EXPOSE 8000`  
    `CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]`

### 

### 

### **`ui/Dockerfile`**

    `FROM python:3.10-slim`  
    `WORKDIR /app`  
    `COPY requirements.txt .`  
    `RUN pip install --no-cache-dir -r requirements.txt`  
    `COPY . .`  
    `EXPOSE 8501`  
    `CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]`

---

## **4. CI/CD & Hosting 

1. **Where We Keep Our Code**  
   - We use **Azure DevOps** as our code repository.

2. **How New Code Gets Deployed**  
   - Whenever we push changes, Azure DevOps automatically:
     - Runs tests and lints our code to catch errors early.
     - Builds Docker images for each component (RAGFlow, API, UI).
     - Pushes those images to the **Azure Container Registry**.
     - Deploys the containers to Azure using Azure Container Instances so the app is live.

3. **Secrets & Configuration**  
   - We store passwords and keys in **Azure Key Vault** (a secure vault).
   - The pipeline pulls them in at build time—no hard-coding in our code.



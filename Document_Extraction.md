**Document Extraction**

---

**1\. Purpose**  
 Define how our system ingests and preprocesses uploaded documents (PDFs, images) using RAGFlow as the primary engine for OCR, text extraction, chunking, and field segmentation. This specification also describes fallback tools and the rationale behind each step.

---

**2\. File extraction methods**

**tools and methods would used to extract data from PDFs or images:  

* **Primary:** RAGFlow’s DeepDoc module, which combines vision and layout analysis to perform OCR on text blocks, tables, captions and more.

* **Fallback:** PDFPlumber for reliable text extraction, Tesseract OCR (with tuned language/model settings) for scanned images if DeepDoc’s confidence falls below 80 %.

**Extracting fields such as dates, totals, and names:  

1. **DeepDoc Extraction (Primary Approach):**
   - Use **RAGFlow’s DeepDoc** module as the main extractor for **field segmentation**.
   - Leverages layout and vision analysis to automatically identify key fields like `invoice_total`, `contract_date`, and `signature_date`.
   - Supports detection of:
     - Text blocks (paragraphs, headings)
     - Tables and key-value pairs
     - Spatial relationships between fields and values

2. **Template-Based Enhancements (Optional Layer):**
   - Where DeepDoc's extraction is uncertain or layouts are highly variable, define regex or label-based templates to match fields like totals and dates.

3. **Prompt-Driven Normalization (Fine-Tuning):**
   - After extraction, optionally send field values to an LLM to:
     - Normalize numbers (e.g., strip currency symbols).
     - Convert date formats to ISO 8601 standard (`YYYY-MM-DD`).
     - Correct common OCR artifacts.

4. **Confidence Threshold & Human Review:**
   - If DeepDoc’s extraction confidence is below 80%:
     - Fallback to PDFPlumber or Tesseract OCR.
     - Flag the document for human validation in the RAGFlow UI.
---

**3\. RAGFlow as Primary Extraction Engine**

**Why RAGFlow?**

1. **End-to-end integration** of OCR, PDF parsing, chunking, embedding, and retrieval.

2. **Multi­format support** for PDF, DOCX, images, and scanned documents.

3. **Explainable chunking** via templates and a user interface for human intervention.

4. **Grounded citations** that trace every extracted value back to its original document location.

**Components Utilized:**

* **DeepDoc:** Vision and layout analysis for text, tables, and images.

* **Chunk Templates:** Format-aware slicing of pages into semantically coherent chunks.

* **Field Segmentation:** Automatic detection of key–value pairs using templates and DeepDoc.

---

**4\. Step-by-Step Workflow**

**4.1 Server & Knowledge Base Setup**

1. Deploy RAGFlow via Docker.

2. Configure the desired LLM and embedding models in `service_conf.yaml`.

3. Initialize a placeholder knowledge base (KB) to receive ingested documents.

**4.2 Document Ingestion**

1. Upload files (PDF, JPG, PNG, DOCX) into the KB.

2. Trigger parsing by DeepDoc, which performs OCR on scanned pages and analyzes layout into text blocks, tables, and captions.

3. Monitor chunking:

   * Template-based splitting by headings or tables.

   * Optional manual keyword/tag injection for chunk refinement.

**4.3 Field Extraction & Segmentation**

1. Define field templates (e.g., regex patterns) for invoice\_total, contract\_date, signature\_date, etc.

2. Use DeepDoc to extract candidate values and map them to these fields.

3. Apply prompt-driven normalization, for example:

    “Verify that the invoice total extracted as ‘1,234.56’ matches numeric format and convert it to a float.”

4. Enforce a confidence threshold: if DeepDoc’s confidence is under 80 %, fall back to PDFPlumber or Tesseract, and/or flag the extraction for manual review.

**4.3.1 Example of Normalized Extraction**

`{`  
  `"invoice_total": 1234.56,`  
  `"contract_date": "2025-01-14",`  
  `"signature_date": "2025-01-14"`  
`}`

## 4.4 Embedding & Indexing

1. **Embedding Engine via RAGFlow:**
   - After field segmentation, RAGFlow automatically embeds each document chunk into a dense semantic vector.

2. **Chosen Embedding Model:**
   - **Model:** `all-MiniLM-L6-v2` from Hugging Face's Sentence Transformers.
   - **Why this model:**
     - **Free** to use and open-source.
     - **Lightweight (~80MB)**, suitable for CPU-based inference.
     - **Embedding size:** 384 dimensions.
     - Optimized for **semantic similarity** tasks.

3. **Vector Store Selected:**
   - **FAISS (Facebook AI Similarity Search)**
   - **Why FAISS:**
     - Open-source and production-ready.
     - Fast approximate nearest neighbor search.
     - Supports both CPU and GPU acceleration.
     - Scales to millions of embedded chunks efficiently.

4. **How It Fits Together:**
   - RAGFlow’s configuration (`service_conf.yaml`) is set to:
     - Use `all-MiniLM-L6-v2` for embedding.
     - Index the resulting vectors in FAISS.
   - This enables high-speed semantic retrieval of document sections during validation or interactive querying.

---

**5\. Notes & Considerations**

* **Performance:** Log parsing time and memory usage; adjust chunk-template granularity to optimize throughput.

* **Quality:** Use RAGFlow’s retrieval test UI to spot-check extraction accuracy.

* **Scalability:** Docker ensures consistent environments; scale the knowledge base horizontally as needed.

* **Auditability:** Record every extraction event with page number, coordinates, and confidence score.

* **Error Handling:** Define strategies (e.g., raise an exception, warn and skip) for malformed or ambiguous cases.


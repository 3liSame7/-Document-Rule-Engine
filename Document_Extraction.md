**Document Extraction**

---

**1\. Purpose**  
 Define how our system ingests and preprocesses uploaded documents (PDFs, images) using RAGFlow as the primary engine for OCR, text extraction, chunking, and field segmentation. This specification also describes fallback tools and the rationale behind each step.

---

**2\. Q\&A Alignment**

**Question:** What tools and methods would you use to extract data from PDFs or images?  
 **Answer:**

* **Primary:** RAGFlow’s DeepDoc module, which combines vision and layout analysis to perform OCR on text blocks, tables, captions and more.

* **Fallback:** PDFPlumber or PyMuPDF, for reliable text extraction, Tesseract OCR (with tuned language/model settings) for scanned images if DeepDoc’s confidence falls below 80 %.

*Rationale:* DeepDoc handles complex, multi­format layouts end-to-end; PDFPlumber/Tesseract step in when confidence is low.

**Question:** How would you extract fields such as dates, totals, and names?  
 **Answer:**

1. **Template-Based Segmentation:** In RAGFlow, define regex or label templates for fields like invoice\_total, contract\_date, signature\_date, etc.

2. **DeepDoc Extraction:** Map the OCR/text outputs to these predefined fields.

3. **Prompt-Driven Normalization:** Use an LLM prompt to verify and normalize each extracted value (for example, stripping currency symbols and converting dates to ISO format).

4. **Confidence Threshold & Human Review:** Flag any extraction with confidence below 80 % for manual inspection in the RAGFlow UI.

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

1. Deploy RAGFlow via Docker (version 0.18.0-slim or full) on an x86 host (or build for ARM). Ensure `vm.max_map_count` and resource prerequisites are met.

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

**4.4 Embedding & Indexing**

1. Embed each chunk using the selected model (for example, all-MiniLM-L6-v2).

2. Automatically index those embeddings in FAISS within RAGFlow for efficient semantic retrieval.

**4.5 Integration with Rule Engine**  
 Pass the structured `extracted_fields` along with their source chunks to the validation module. RAGFlow’s grounded citations allow each validated value to trace back to its exact location in the original document.

---

**5\. Notes & Considerations**

* **Performance:** Log parsing time and memory usage; adjust chunk-template granularity to optimize throughput.

* **Quality:** Use RAGFlow’s retrieval test UI to spot-check extraction accuracy.

* **Scalability:** Docker ensures consistent environments; scale the knowledge base horizontally as needed.

* **Auditability:** Record every extraction event with page number, coordinates, and confidence score.

* **Error Handling:** Define strategies (e.g., raise an exception, warn and skip) for malformed or ambiguous cases.


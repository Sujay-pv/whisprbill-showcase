# PDF Generation and Storage Strategy

## Problem

Invoices need to be generated as consistent, GST-compliant PDFs that render correctly across devices and browsers.

The initial approach generated PDFs on the backend using Puppeteer and stored them as binary blobs in the database. While this ensured consistency, it introduced compute overhead for generation and storage costs.

As usage grows, repeatedly rendering PDFs on-demand can increase memory usage and impact performance.

---

## Options Considered

### 1. Backend Rendering with Puppeteer (Current Approach)

- Generate PDF from HTML on the backend  
- Store the generated PDF buffer (or regenerate when needed)  
- Serve the PDF to the client for download or viewing  

**Pros**
- Consistent rendering across all devices and browsers  
- Handles multi-page invoices reliably  
- Works well with table-heavy GST invoice layouts  
- Enables server-side workflows (e.g., WhatsApp invoice sharing)  

**Cons**
- Higher compute usage during PDF generation  
- Storage overhead if PDFs are persisted  
- Potential memory pressure under concurrent load  

---

### 2. Client-Side PDF Generation

- Send invoice JSON to the frontend  
- Generate PDF in the browser using libraries like html2pdf or jsPDF  

**Pros**
- No backend compute cost for PDF generation  
- No storage required for PDFs  
- Scales naturally with users  

**Cons**
- Inconsistent rendering across browsers  
- Poor support for multi-page documents  
- Not reliable for table-dense layouts  
- Limited control over layout precision  
- Cannot support server-side workflows like WhatsApp delivery  

---

### 3. Backend Rendering with Caching (Explored)

- Generate PDF once using Puppeteer  
- Cache the PDF buffer for reuse  
- Serve cached version for subsequent requests  

**Pros**
- Reduces repeated compute cost  
- Maintains consistent rendering  
- Faster subsequent access  

**Cons**
- Additional cache management complexity  
- Still requires storage  
- Cache invalidation needed when invoice data changes  

---

## Decision

Continue with backend PDF generation using Puppeteer, with the option to introduce caching for generated PDFs.

The priority is correctness and consistency over reducing compute cost.

---

## Why this approach

- GST invoices are table-heavy and require precise formatting  
- Multi-page support must be reliable  
- Output must be consistent across all devices and browsers  
- Backend control allows integration with future workflows such as WhatsApp invoice delivery  
- Client-side libraries do not provide sufficient reliability for these requirements  

---

## Tradeoffs

- Accept higher compute and storage costs in exchange for predictable output  
- PDF generation is compute-intensive and can lead to memory pressure under concurrent load  
- Additional infrastructure is required to handle scaling safely  

---

## Operational Considerations

To prevent excessive memory usage and potential OOM errors on the backend instance, PDF generation is currently handled with a soft concurrency limit.

Only a limited number of PDF generation requests are processed at a time, ensuring the system remains stable under load.

To improve this, a queue-based approach is being introduced:

- PDF generation will be offloaded to a job queue (Bull + Redis)  
- Jobs will be processed asynchronously with controlled concurrency  
- Retries and failure handling will be built into the pipeline  
- The frontend will receive status updates for long-running generation tasks  

This allows the system to scale PDF generation safely without overwhelming the application server.

---

## Notes

While client-side PDF generation reduces compute cost, it was not suitable due to rendering inconsistencies and lack of support for complex layouts.

The current approach prioritizes correctness and reliability, with a clear path to improve scalability through asynchronous processing.

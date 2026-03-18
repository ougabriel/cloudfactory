# Cloudfactory

---

## **1. Proposed Solution Overview**

![Architectural_design](design3.gif)

This architecture resolves critical data drift and metadata loss issues from "Transfer Bridge" without the need to refactor the existing codebase. I introduced a lightweight, non-intrusive enhancement layer built around a JSON sidecar mechanism that operates alongside the Bridge’s data flow. By also introducing a **Serverless Enrichment Layer**, I preserve original image metadata, automate PII compliance for face detection, and provide high-fidelity, enriched data to CloudFactory. 


---

## **2. Metadata Extraction & Preservation Strategy**
The Transfer Bridge operates on a **Fetch-to-Local** model. My solution intercepts the data at the most critical point: the initial landing.

### **The Three-Step Intercept:**
* **Step 1: Fetch (The Landing):** The Bridge downloads the raw file from iCloud to an **Azure Blob Storage "Staging" container**.
* **Step 2: Process (The Transformation):** The Bridge processes/compresses the image, inadvertently stripping EXIF data.
* **Step 3: Upload:** The Bridge pushes the processed image to CloudFactory.

### **Implementation Logic:**
To prevent metadata loss, I implemented a lightweight Python script (utilizing **Pillow**) triggered by **Azure Functions**. This is an event-driven, serverless approach that monitors the Staging Blob.
* **Action:** The moment a file lands in the Staging Blob, the Azure Function triggers *before* the Bridge can complete its Step 2 processing. It extracts the EXIF data (GPS/Timestamps) and generates a matching **JSON Sidecar File** (e.g., `image_name.jpg  to image_name_metadata.json`).
* **Result:** The sidecar file is uploaded as a companion to the image. Even if the Bridge strips the binary metadata, the spatial-temporal context remains permanently attached to the record.

> **Technical Guardrail:** To ensure the script extracts data before the Bridge processes it, I utilized **Event Grid triggers** on the *Raw* container. Since the Bridge writes to a "Raw" blob and reads from it to create a "Processed" blob, our function targets the "Raw" creation event, ensuring the solution always touch the original file first.

---

## **3. Face Detection & PII Strategy**
To enforce the 24-hour "Hard Delete" mandate for human faces, I utilized a dual-layer enforcement strategy.

### **The Detection Workflow:**
1.  **AI Scan:** Each image is scanned using **YOLOv8** (Face-Detection) during the ingestion phase.
2.  **Tagging:** If a face is detected, the image is assigned a metadata tag: `PII_Status: True`.
3.  **Automated Purge:** * **Primary:** An **Azure Blob Lifecycle Policy** is set to permanently delete objects with the `PII_Status: True` tag less than 24 hours after the `Capture_Timestamp`.
    > **Secondary (Fail-safe):** A scheduled Azure Function runs every 6 hours to audit the container and force-delete any sensitive images that bypassed the lifecycle rule due to system lag.

---

## **4. Image Quality & Contextual Labeling**

### **A. Quality Validation (The "Blur" Solution)**
To restore model performance, I implemented an **Automated Quality Gate**.
* **Method:** I used the **OpenCV Laplacian Variance** method to calculate the edge-density of each frame.
* **Logic:** Images falling below a pre-set "Sharpness Threshold" are automatically flagged as `Quality: Poor`.
* **Action:** These images are diverted to a **Quarantine Bucket** for manual review or digital enhancement (CLAHE), preventing "trash" data from degrading the model.

### **B. Contextual Intelligence (The "Data Drift" Solution)**
I solved the "Office vs. Warehouse" discrepancy by adding an **Environmental Intelligence Layer**.
* **Logic:** Using brightness metrics and pattern recognition, the system auto-labels images as `Environment: Warehouse` or `Environment: Office`.
* **Impact:** This allows for **Targeted Calibration**. I can now tune the model specifically for warehouse conditions, transforming an environmental failure into a structured data science optimization.

---

## **5. Trade-offs & Risk Mitigation**

### **Decisions & Trade-offs:**
* **Investment Preservation:** I retained the existing Bridge to avoid stakeholder conflict, choosing a "Sidecar Layer" over a total rebuild for speed-to-market.
* **Serverless Efficiency:** Azure Functions were chosen for cost-efficiency, ensuring ZeroCorp only pays for the milliseconds the code is running.

### **Risks & Mitigations:**
* **Missing Metadata:** If EXIF is missing at the source, the script applies a "System Default" tag and logs the event for audit.
* **PII False Positives:** We utilize a high confidence threshold (0.90) in YOLO to prevent the accidental deletion of non-sensitive warehouse data.
* **Redundancy:** The combination of Cloud Lifecycle Policies and Scheduled Functions ensures 100% compliance with the 24-hour deletion rule.

### **Monitoring and Logging:**
While this solution focuses on solving the metadata loss and the 24-hour privacy rule, in the near future I will architect the system to support a Centralized Observability Layer in Phase 2. The trade-off here was **Speed vs. Insight**  
To get the CEO’s project back on track immediately, I prioritized the data recovery itself. However, the roadmap includes integrating Azure Monitor to create an immutable audit trail. This will transform our 'Compliance Rule' into a 'Compliance Proof,' giving ZeroCorp a verifiable record that PII is being handled correctly while providing a dashboard to track 'Warehouse vs. Office' data drift in real-time.
---
# **EMAIL**
---
Subject: Update on Model Performance and Image Processing Improvements
---
To: ceo@zerocorps.com  
Cc: vp@zerocorps.com  
---
Dear CEO,

I wanted to provide a clear update on the recent concerns regarding model performance and the delays observed in the image processing workflow.

Following a detailed analysis, I’ve identified two primary factors contributing to the current situation.

First, there is a data quality mismatch between the images used to train the model and the images now being processed. The model was originally trained on well-lit, high-quality office images, whereas the current dataset consists largely of darker, lower-quality warehouse images. This shift in data characteristics is impacting the model’s ability to perform reliably.

Second, we discovered that during the transfer process, certain image metadata (such as timestamps and location data) is being stripped. This metadata plays an important role in how the model interprets and contextualises the images, and its absence has contributed to the degradation in performance.

The good news is that we have a clear and non-disruptive path forward.

I have implemented a lightweight enhancement layer around the existing transfer workflow that preserves the original metadata using a sidecar approach, ensuring no loss of critical information moving forward. In parallel, we have introduced automated quality validation to filter out low-quality images, as well as an intelligent tagging system to distinguish between warehouse and office environments. This allows us to better align the data with the model’s expectations and improve overall accuracy.

Additionally, we have strengthened our compliance controls by introducing automated face detection and enforcing the 24-hour deletion requirement through both lifecycle policies and scheduled verification checks.

These improvements are designed to stabilise model performance while preserving the current infrastructure and prior investment. As the updated pipeline continues to process incoming data, we expect to see a measurable improvement in model quality and consistency.

I will continue to monitor progress closely and keep you updated as these enhancements take full effect. If helpful, I would be happy to walk you through the changes and expected impact in more detail.

Best regards,  
Gabriel Okom  
AI Platform Implementation Engineer  

---

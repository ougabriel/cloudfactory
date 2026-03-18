# cloudfactory

---

## **1. Proposed Solution Overview**


This architecture resolves critical data drift and metadata loss issues the "Transfer Bridge" pipeline without the need to refactor the existing codebase. I introduced a lightweight, non-intrusive enhancement layer built around a JSON sidecar mechanism that operates alongside the Bridge’s data flow. By also introducing a **Serverless Enrichment Layer**, I preserve original image metadata, automate PII compliance for face detection, and provide high-fidelity, enriched data to CloudFactory. 


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

> **Technical Guardrail:** To ensure the script extracts data before the Bridge processes it, we utilize **Blob Lease** or **Event Grid triggers** on the *Raw* container. Since the Bridge writes to a "Raw" blob and reads from it to create a "Processed" blob, our function targets the "Raw" creation event, ensuring we always touch the original file first.

---

## **3. Face Detection & PII Strategy**
To enforce the 24-hour "Hard Delete" mandate for human faces, we utilize a dual-layer enforcement strategy.

### **The Detection Workflow:**
1.  **AI Scan:** Each image is scanned using **YOLOv8** (Face-Detection) during the ingestion phase.
2.  **Tagging:** If a face is detected, the image is assigned a metadata tag: `PII_Status: True`.
3.  **Automated Purge:** * **Primary:** An **Azure Blob Lifecycle Policy** is set to permanently delete objects with the `PII_Status: True` tag exactly 24 hours after the `Capture_Timestamp`.
    * **Secondary (Fail-safe):** A scheduled Azure Function runs every 6 hours to audit the container and force-delete any sensitive images that bypassed the lifecycle rule due to system lag.

---

## **4. Image Quality & Contextual Labeling**

### **A. Quality Validation (The "Blur" Solution)**
To restore model performance, I implemented an **Automated Quality Gate**.
* **Method:** We use the **OpenCV Laplacian Variance** method to calculate the edge-density of each frame.
* **Logic:** Images falling below a pre-set "Sharpness Threshold" are automatically flagged as `Quality: Poor`.
* **Action:** These images are diverted to a **Quarantine Bucket** for manual review or digital enhancement (CLAHE), preventing "trash" data from degrading the model.

### **B. Contextual Intelligence (The "Data Drift" Solution)**
I solved the "Office vs. Warehouse" discrepancy by adding an **Environmental Intelligence Layer**.
* **Logic:** Using brightness metrics and pattern recognition, the system auto-labels images as `Environment: Warehouse` or `Environment: Office`.
* **Impact:** This allows for **Targeted Calibration**. I can now tune the model specifically for warehouse conditions, transforming an environmental failure into a structured data science optimization.

---

## **5. Trade-offs & Risk Mitigation**

### **Decisions & Trade-offs:**
* **Investment Preservation:** I retained the existing Bridge to avoid stakeholder conflict, choosing a "Recovery Layer" over a total rebuild for speed-to-market.
* **Serverless Efficiency:** Azure Functions were chosen for cost-efficiency, ensuring ZeroCorp only pays for the milliseconds the code is running.

### **Risks & Mitigations:**
* **Missing Metadata:** If EXIF is missing at the source, the script applies a "System Default" tag and logs the event for audit.
* **PII False Positives:** We utilize a high confidence threshold (0.90) in YOLO to prevent the accidental deletion of non-sensitive warehouse data.
* **Redundancy:** The combination of Cloud Lifecycle Policies and Scheduled Functions ensures 100% compliance with the 24-hour deletion rule.

---


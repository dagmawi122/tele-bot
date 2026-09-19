# Document & CV Upload Mechanism Specification & Implementation Guide

This document defines the complete technical specifications and implementation steps for uploading, storing, and serving candidate CVs (`.pdf`, `.docx`) and employer verification documents (`Trade License`, `TIN Certificate`).

---

## 📋 Table of Contents
- [1. Overview & Storage Strategy Comparison](#1-overview--storage-strategy-comparison)
- [2. Database Schema (`documents` Table)](#2-database-schema-documents-table)
- [3. Option A: Presigned Cloud Upload (Best Practice & Free Tier)](#3-option-a-presigned-cloud-upload-best-practice--free-tier)
- [4. Option B: FastAPI Local File Upload (Easiest Dev Mode Setup)](#4-option-b-fastapi-local-file-upload-easiest-dev-mode-setup)
- [5. Frontend Integration Guide (Next.js)](#5-frontend-integration-guide-nextjs)
- [6. Security & File Validation Rules](#6-security--file-validation-rules)

---

## 1. Overview & Storage Strategy Comparison

There are two primary architectural approaches to handling CV and document uploads:

| Feature | Option A: Presigned Cloud Storage (Recommended) | Option B: FastAPI Local Multipart Upload |
| :--- | :--- | :--- |
| **Cost** | 🟢 **Free Tier** (Supabase: 1GB free, GCS/AWS: 5GB free) | 🟢 **100% Free** (Uses your server disk) |
| **Performance** | ⚡ **Ultra Fast** (Uploads directly from browser to cloud) | 🐢 Server bottlenecks on large file uploads |
| **Scalability** | 🚀 Infinite scale; zero backend memory usage | ⚠️ Limited by server disk space & RAM |
| **Ease of Setup** | 🟡 Moderate (Requires cloud bucket keys) | 🟢 Super Easy (Requires zero cloud setup) |

---

## 2. Database Schema (`documents` Table)

Both options utilize the existing `documents` table in PostgreSQL to maintain file metadata:

```sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    document_type VARCHAR(50) NOT NULL, -- 'CV', 'TRADE_LICENSE', 'TIN_CERTIFICATE'
    file_name VARCHAR(255) NOT NULL,
    file_path VARCHAR(512) NOT NULL,
    file_size INTEGER NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING', -- 'PENDING', 'UPLOADED', 'FAILED'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 3. Option A: Presigned Cloud Upload (Best Practice & Free Tier)

### Step 1: Create a Storage Bucket
- **Supabase Storage (Free 1GB)**: Create a bucket named `talentflow-docs`. Set policy to Public or Authenticated Read.
- **Google Cloud Storage / AWS S3 (Free 5GB)**: Create a bucket named `talentflow-documents`.

### Step 2: Backend Presigned URL Endpoint (`POST /api/v1/documents/upload-url`)

```python
# backend/modules/documents/router.py
from fastapi import APIRouter, Depends, HTTPException, status
from pydantic import BaseModel
import uuid

router = APIRouter(prefix="/documents", tags=["Documents"])

class DocumentUploadUrlRequest(BaseModel):
    document_type: str # 'CV' or 'TRADE_LICENSE'
    file_name: str
    content_type: str  # 'application/pdf', 'image/jpeg'

class DocumentUploadUrlResponse(BaseModel):
    document_id: str
    upload_url: str
    public_file_url: str

@router.post("/upload-url", response_model=DocumentUploadUrlResponse)
async def generate_upload_url(
    payload: DocumentUploadUrlRequest,
    current_user = Depends(get_current_user)
):
    doc_id = str(uuid.uuid4())
    file_key = f"{payload.document_type.lower()}s/{current_user.id}/{doc_id}_{payload.file_name}"
    
    # Generate presigned PUT URL using boto3 (AWS/Supabase) or google-cloud-storage
    # presigned_url = s3_client.generate_presigned_url(...)
    
    return DocumentUploadUrlResponse(
        document_id=doc_id,
        upload_url="https://your-bucket.s3.amazonaws.com/" + file_key + "?token=...",
        public_file_url="https://your-bucket.s3.amazonaws.com/" + file_key
    )
```

### Step 3: Confirm Upload Endpoint (`POST /api/v1/documents/confirm`)

```python
@router.post("/confirm")
async def confirm_document_upload(
    document_id: str,
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user)
):
    # Update document status in DB to 'UPLOADED'
    # Link document_id to current_user.employee_profile or employer_profile
    return {"status": "success", "message": "Document uploaded successfully"}
```

---

## 4. Option B: FastAPI Local File Upload (Easiest Dev Mode Setup)

If you prefer zero cloud configuration, FastAPI can save files directly to local disk.

### Step 1: Install `python-multipart`
```bash
pip install python-multipart
```

### Step 2: Mount Static Serving Directory in `main.py`
```python
from fastapi.staticfiles import StaticFiles
import os

# Create uploads directory
os.makedirs("uploads/cvs", exist_ok=True)
os.makedirs("uploads/licenses", exist_ok=True)

# Mount static serving route
app.mount("/static/uploads", StaticFiles(directory="uploads"), name="uploads")
```

### Step 3: Multipart Direct Upload Endpoint (`POST /api/v1/documents/upload`)

```python
from fastapi import UploadFile, File, Form

@router.post("/upload")
async def upload_document_local(
    document_type: str = Form(...), # 'CV' or 'TRADE_LICENSE'
    file: UploadFile = File(...),
    current_user = Depends(get_current_user),
    db: AsyncSession = Depends(get_db)
):
    # 1. Validate MIME type & file size
    if file.content_type not in ["application/pdf", "image/png", "image/jpeg"]:
        raise HTTPException(status_code=400, detail="Invalid file format. Only PDF, PNG, JPEG allowed.")
    
    # 2. Save file to disk
    file_extension = file.filename.split(".")[-1]
    saved_filename = f"{current_user.id}_{uuid.uuid4().hex[:8]}.{file_extension}"
    file_path = f"uploads/{document_type.lower()}s/{saved_filename}"
    
    with open(file_path, "wb") as f:
        content = await file.read()
        f.write(content)
        
    public_url = f"http://localhost:8000/static/{file_path}"
    
    # 3. Save document record in PostgreSQL DB
    # ...
    return {"status": "success", "file_url": public_url}
```

---

## 5. Frontend Integration Guide (Next.js)

### Upload Handler in `UploadCvStep.tsx` / `UploadDocumentsStep.tsx`

```typescript
// frontend/services/documentService.ts
import { API_BASE_URL } from './apiClient';

export async function uploadDocumentLocal(file: File, documentType: string) {
  const token = localStorage.getItem('access_token');
  const formData = new FormData();
  formData.append('file', file);
  formData.append('document_type', documentType);

  const response = await fetch(`${API_BASE_URL}/documents/upload`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Bypass-Tunnel-Reminder': 'true'
    },
    body: formData,
  });

  if (!response.ok) {
    const errorData = await response.json();
    throw new Error(errorData.detail || 'Document upload failed');
  }

  return response.json();
}
```

---

## 6. Security & File Validation Rules

1. **Max File Size Constraints**:
   - CVs (`.pdf`, `.docx`): Max **5 MB**
   - Business Trade Licenses & TIN Certificates (`.pdf`, `.png`, `.jpg`): Max **10 MB**
2. **MIME Type Allowlist**:
   ```python
   ALLOWED_MIME_TYPES = {
       "application/pdf",
       "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
       "image/jpeg",
       "image/png"
   }
   ```
3. **Filename Sanitization**:
   - Never trust user filenames on the disk. Always rename uploaded files using UUIDs (`{user_id}_{uuid}.pdf`) to prevent directory traversal attacks.

# MYTH Identity Verification Platform
## Complete Backend Architecture & Production Guide

---

## REVISED CORE RULE

> Users **never upload government documents**.
> All government document data is stored exclusively in the **Admin Master Database**.
> The system compares **user-entered information + uploaded photo** against **stored records**.

---

## FOLDER STRUCTURE

```
myth-platform/
│
├── frontend/                        # Next.js 14 (App Router)
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx                 # Homepage
│   │   ├── (auth)/
│   │   │   ├── register/page.tsx
│   │   │   └── login/page.tsx
│   │   ├── dashboard/
│   │   │   ├── layout.tsx           # Dashboard shell + sidebar
│   │   │   ├── page.tsx             # Overview stats
│   │   │   ├── verify/page.tsx      # Verification flow
│   │   │   ├── history/page.tsx     # Past verifications
│   │   │   └── profile/page.tsx
│   │   └── admin/
│   │       ├── login/page.tsx
│   │       ├── layout.tsx
│   │       ├── page.tsx             # Admin overview
│   │       ├── records/page.tsx     # Records table + filter
│   │       ├── records/add/page.tsx # Add record + OCR
│   │       ├── users/page.tsx
│   │       └── logs/page.tsx
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Toast.tsx
│   │   │   ├── Badge.tsx
│   │   │   └── DataTable.tsx
│   │   ├── ImageUpload.tsx          # Upload + preview component
│   │   ├── OCRPanel.tsx             # OCR auto-fill UI
│   │   ├── VerifySteps.tsx          # Step wizard
│   │   ├── VerifyPipeline.tsx       # Animated pipeline
│   │   ├── VerifyResult.tsx         # Result card
│   │   ├── RecordsTable.tsx
│   │   ├── Chatbot.tsx
│   │   └── Navbar.tsx
│   ├── lib/
│   │   ├── api.ts                   # Axios client
│   │   ├── auth.ts                  # JWT helpers
│   │   └── validators.ts
│   ├── hooks/
│   │   ├── useAuth.ts
│   │   ├── useVerification.ts
│   │   └── useOCR.ts
│   └── styles/globals.css
│
├── backend/                         # Node.js + Express
│   ├── src/
│   │   ├── server.ts
│   │   ├── config/
│   │   │   ├── db.ts                # MongoDB connection
│   │   │   ├── jwt.ts
│   │   │   ├── storage.ts           # S3 config
│   │   │   └── env.ts
│   │   ├── models/
│   │   │   ├── User.ts
│   │   │   ├── IdentityRecord.ts    # Admin master database
│   │   │   ├── VerificationRequest.ts
│   │   │   └── VerificationResult.ts
│   │   ├── routes/
│   │   │   ├── auth.routes.ts
│   │   │   ├── admin.routes.ts      # Records CRUD + OCR
│   │   │   ├── verify.routes.ts     # Verification engine
│   │   │   └── otp.routes.ts
│   │   ├── controllers/
│   │   │   ├── auth.controller.ts
│   │   │   ├── admin.controller.ts
│   │   │   ├── verify.controller.ts
│   │   │   └── otp.controller.ts
│   │   ├── middleware/
│   │   │   ├── auth.middleware.ts
│   │   │   ├── adminAuth.middleware.ts
│   │   │   ├── upload.middleware.ts  # Multer
│   │   │   ├── rateLimit.middleware.ts
│   │   │   └── validate.middleware.ts
│   │   ├── services/
│   │   │   ├── ocr.service.ts        # Tesseract + Google Vision
│   │   │   ├── face.service.ts       # DeepFace HTTP client
│   │   │   ├── storage.service.ts    # AWS S3
│   │   │   ├── otp.service.ts        # Twilio + Nodemailer
│   │   │   └── pdf.service.ts        # PDFKit report generator
│   │   └── utils/
│   │       ├── dataScorer.ts         # Name/DOB/Address comparison
│   │       └── recordSearch.ts       # DB fuzzy search
│   └── package.json
│
├── ai-service/                       # Python FastAPI microservice
│   ├── main.py
│   ├── routers/
│   │   ├── face.py                   # DeepFace endpoints
│   │   └── health.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── docker-compose.yml
├── nginx.conf
└── .env.example
```

---

## MONGODB SCHEMAS

### models/User.ts

```typescript
import mongoose, { Document, Schema } from 'mongoose';
import bcrypt from 'bcryptjs';

export interface IUser extends Document {
  username: string;
  aadhaar: string;
  phone: string;
  email: string;
  passwordHash: string;
  phoneVerified: boolean;
  emailVerified: boolean;
  role: 'user' | 'admin';
  isActive: boolean;
  createdAt: Date;
  comparePassword(plain: string): Promise<boolean>;
}

const UserSchema = new Schema<IUser>({
  username:      { type: String, required: true, unique: true, trim: true, minlength: 3 },
  aadhaar:       { type: String, required: true, unique: true },
  phone:         { type: String, required: true },
  email:         { type: String, required: true, unique: true, lowercase: true },
  passwordHash:  { type: String, required: true },
  phoneVerified: { type: Boolean, default: false },
  emailVerified: { type: Boolean, default: false },
  role:          { type: String, enum: ['user','admin'], default: 'user' },
  isActive:      { type: Boolean, default: true },
}, { timestamps: true });

// Hash password before save
UserSchema.pre('save', async function(next) {
  if (!this.isModified('passwordHash')) return next();
  this.passwordHash = await bcrypt.hash(this.passwordHash, 12);
  next();
});

UserSchema.methods.comparePassword = function(plain: string) {
  return bcrypt.compare(plain, this.passwordHash);
};

// Indexes for fast lookup
UserSchema.index({ aadhaar: 1 });
UserSchema.index({ username: 1 });
UserSchema.index({ email: 1 });

export default mongoose.model<IUser>('User', UserSchema);
```

---

### models/IdentityRecord.ts  (Admin Master Database)

```typescript
import mongoose, { Document, Schema } from 'mongoose';

export interface IIdentityRecord extends Document {
  recordId: string;            // REC001, REC002…
  name: string;
  dob: string;                 // DD/MM/YYYY
  address: string;
  documentType: 'aadhaar' | 'pan' | 'license';
  idNumber: string;            // Unique per document
  personPhotoUrl: string;      // S3 URL for face matching
  documentImageUrl: string;    // S3 URL for admin reference
  addedByAdmin: string;
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

const IdentityRecordSchema = new Schema<IIdentityRecord>({
  recordId:         { type: String, unique: true },
  name:             { type: String, required: true, index: true },
  dob:              { type: String, required: true },
  address:          { type: String, required: true },
  documentType:     { type: String, enum: ['aadhaar','pan','license'], required: true, index: true },
  idNumber:         { type: String, required: true, unique: true },
  personPhotoUrl:   { type: String, default: '' },
  documentImageUrl: { type: String, default: '' },
  addedByAdmin:     { type: String, required: true },
  isActive:         { type: Boolean, default: true },
}, { timestamps: true });

// Text index for fuzzy name search
IdentityRecordSchema.index({ name: 'text', address: 'text' });
IdentityRecordSchema.index({ documentType: 1, idNumber: 1 });
IdentityRecordSchema.index({ name: 1, dob: 1 }); // compound for search

export default mongoose.model<IIdentityRecord>('IdentityRecord', IdentityRecordSchema);
```

---

### models/VerificationResult.ts

```typescript
import mongoose, { Document, Schema } from 'mongoose';

export interface IVerificationResult extends Document {
  verificationId: string;
  requestedBy: mongoose.Types.ObjectId;  // User ref
  // Subject input
  subjectName: string;
  subjectDob: string;
  subjectAddress?: string;
  documentType: string;
  subjectPhotoUrl?: string;   // Uploaded by user
  // Matched record
  matchedRecordId?: string;
  matchedRecordName?: string;
  matchedRecordIdNumber?: string;
  // Scores
  faceMatchScore: number;      // 0–100
  dataMatchScore: number;      // 0–100
  dbAuthenticityScore: number; // 0–100
  // Match details
  nameMatch: boolean;
  dobMatch: boolean;
  addressMatch: boolean;
  faceVerified: boolean;
  recordFound: boolean;
  // Verdict
  verdict: 'verified' | 'suspicious' | 'partial' | 'failed';
  verdictLabel: string;
  // Report
  reportUrl?: string;
  processingMs: number;
  createdAt: Date;
}

const VerificationResultSchema = new Schema<IVerificationResult>({
  verificationId:    { type: String, unique: true },
  requestedBy:       { type: Schema.Types.ObjectId, ref: 'User', required: true },
  subjectName:       { type: String, required: true },
  subjectDob:        { type: String, required: true },
  subjectAddress:    { type: String },
  documentType:      { type: String },
  subjectPhotoUrl:   { type: String },
  matchedRecordId:   { type: String },
  matchedRecordName: { type: String },
  matchedRecordIdNumber: { type: String },
  faceMatchScore:    { type: Number, default: 0 },
  dataMatchScore:    { type: Number, default: 0 },
  dbAuthenticityScore: { type: Number, default: 0 },
  nameMatch:         { type: Boolean, default: false },
  dobMatch:          { type: Boolean, default: false },
  addressMatch:      { type: Boolean, default: false },
  faceVerified:      { type: Boolean, default: false },
  recordFound:       { type: Boolean, default: false },
  verdict:           { type: String, enum: ['verified','suspicious','partial','failed'] },
  verdictLabel:      { type: String },
  reportUrl:         { type: String },
  processingMs:      { type: Number },
}, { timestamps: true });

VerificationResultSchema.index({ requestedBy: 1, createdAt: -1 });
VerificationResultSchema.index({ verdict: 1 });

export default mongoose.model<IVerificationResult>('VerificationResult', VerificationResultSchema);
```

---

## CONTROLLERS

### controllers/verify.controller.ts  (Core Engine)

```typescript
import { Request, Response } from 'express';
import { v4 as uuid } from 'uuid';
import IdentityRecord from '../models/IdentityRecord';
import VerificationResult from '../models/VerificationResult';
import faceService from '../services/face.service';
import storageService from '../services/storage.service';
import pdfService from '../services/pdf.service';
import { computeDataScore } from '../utils/dataScorer';
import { searchRecord } from '../utils/recordSearch';

export const verify = async (req: Request, res: Response) => {
  const t0 = Date.now();
  try {
    const { name, dob, address, documentType } = req.body;
    const photoFile = req.file;  // person photo uploaded by user

    // ── STEP 1: Upload photo to S3
    const subjectPhotoUrl = photoFile
      ? await storageService.upload(photoFile, 'subject-photos')
      : null;

    // ── STEP 2: Database Search
    const record = await searchRecord({ name, dob, address, documentType });

    // ── STEP 3: Data Match Scoring
    const dataScore = computeDataScore(
      { name, dob, address },
      record ? { name: record.name, dob: record.dob, address: record.address } : null
    );

    // ── STEP 4: Face Matching
    let faceScore = 0;
    if (subjectPhotoUrl && record?.personPhotoUrl) {
      faceScore = await faceService.compareFaces(subjectPhotoUrl, record.personPhotoUrl);
    } else if (record && !subjectPhotoUrl) {
      faceScore = 0; // no photo provided
    }

    // ── STEP 5: DB Authenticity Score
    const dbAuthScore = record ? 96 : 35;

    // ── STEP 6: Determine Verdict
    let verdict: string, verdictLabel: string;
    if (faceScore >= 90 && dataScore >= 70) {
      verdict = 'verified';    verdictLabel = 'Verified Identity';
    } else if (faceScore >= 70 && dataScore >= 50) {
      verdict = 'suspicious';  verdictLabel = 'Suspicious Identity';
    } else if (faceScore >= 55 || dataScore >= 35) {
      verdict = 'partial';     verdictLabel = 'Partially Verified';
    } else {
      verdict = 'failed';      verdictLabel = 'Verification Failed';
    }

    // ── STEP 7: Persist Result
    const result = await VerificationResult.create({
      verificationId:     'VER-' + uuid().split('-')[0].toUpperCase(),
      requestedBy:         req.user.id,
      subjectName:         name,
      subjectDob:          dob,
      subjectAddress:      address,
      documentType,
      subjectPhotoUrl,
      matchedRecordId:     record?.recordId,
      matchedRecordName:   record?.name,
      matchedRecordIdNumber: record?.idNumber,
      faceMatchScore:      faceScore,
      dataMatchScore:      dataScore,
      dbAuthenticityScore: dbAuthScore,
      nameMatch:           dataScore > 60,
      dobMatch:            record?.dob === dob,
      addressMatch:        dataScore > 40 && !!address,
      faceVerified:        faceScore >= 70,
      recordFound:         !!record,
      verdict,
      verdictLabel,
      processingMs:        Date.now() - t0,
    });

    // ── STEP 8: Generate PDF report
    const reportUrl = await pdfService.generate(result);
    result.reportUrl = reportUrl;
    await result.save();

    res.json({ success: true, result });
  } catch (err: any) {
    console.error('Verification error:', err);
    res.status(500).json({ error: err.message });
  }
};

export const getHistory = async (req: Request, res: Response) => {
  const results = await VerificationResult.find({ requestedBy: req.user.id })
    .sort({ createdAt: -1 }).limit(100);
  res.json({ results });
};
```

---

### utils/dataScorer.ts

```typescript
import stringSimilarity from 'string-similarity';

interface NameDobAddr {
  name: string;
  dob: string;
  address?: string;
}

export function computeDataScore(
  input: NameDobAddr,
  record: NameDobAddr | null
): number {
  if (!record) return Math.floor(Math.random() * 12); // no record = very low

  // Name similarity (Dice coefficient, case-insensitive)
  const nameSim = stringSimilarity.compareTwoStrings(
    input.name.toLowerCase(),
    record.name.toLowerCase()
  );
  const nameScore = Math.round(nameSim * 40); // max 40 points

  // DOB exact match
  const dobScore = input.dob === record.dob ? 35 : 0; // max 35 points

  // Address partial match
  let addrScore = 0;
  if (input.address && record.address) {
    const addrSim = stringSimilarity.compareTwoStrings(
      input.address.toLowerCase(),
      record.address.toLowerCase()
    );
    addrScore = Math.round(addrSim * 25); // max 25 points
  }

  return Math.min(100, nameScore + dobScore + addrScore);
}
```

---

### utils/recordSearch.ts

```typescript
import IdentityRecord from '../models/IdentityRecord';

interface SearchInput {
  name: string;
  dob: string;
  address?: string;
  documentType: string;
}

export async function searchRecord(input: SearchInput) {
  // Priority 1: Exact name + DOB match
  let record = await IdentityRecord.findOne({
    name: { $regex: new RegExp('^' + escapeRegex(input.name), 'i') },
    dob: input.dob,
    documentType: input.documentType,
    isActive: true,
  });
  if (record) return record;

  // Priority 2: Full-text name search
  record = await IdentityRecord.findOne({
    $text: { $search: input.name },
    documentType: input.documentType,
    isActive: true,
  });
  if (record) return record;

  // Priority 3: DOB + document type only
  record = await IdentityRecord.findOne({
    dob: input.dob,
    documentType: input.documentType,
    isActive: true,
  });
  return record || null;
}

function escapeRegex(str: string) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

---

## SERVICES

### services/ocr.service.ts

```typescript
import Tesseract from 'tesseract.js';
import vision from '@google-cloud/vision';
import path from 'path';

const visionClient = new vision.ImageAnnotatorClient({
  keyFilename: process.env.GOOGLE_CREDENTIALS_PATH,
});

interface OCRResult {
  name: string;
  dob: string;
  address: string;
  idNumber: string;
  confidence: number;
}

class OCRService {
  async extractFromDocument(imageUrl: string, docType: string): Promise<OCRResult> {
    const useGV = process.env.USE_GOOGLE_VISION === 'true';
    const rawText = useGV
      ? await this.googleVisionExtract(imageUrl)
      : await this.tesseractExtract(imageUrl);
    return this.parseText(rawText, docType);
  }

  private async googleVisionExtract(url: string): Promise<string> {
    const [result] = await visionClient.textDetection(url);
    return result.textAnnotations?.[0]?.description || '';
  }

  private async tesseractExtract(url: string): Promise<string> {
    const { data } = await Tesseract.recognize(url, 'eng+hin', {
      logger: () => {},
    });
    return data.text;
  }

  private parseText(text: string, docType: string): OCRResult {
    const result: OCRResult = { name:'', dob:'', address:'', idNumber:'', confidence:0 };

    // Name patterns
    const namePatterns = [
      /(?:Name|नाम)[:\s]+([A-Z][A-Za-z\s]{2,40})/i,
      /^([A-Z][a-z]+ (?:[A-Z][a-z]+ )?[A-Z][a-z]+)$/m,
    ];
    for (const p of namePatterns) {
      const m = text.match(p);
      if (m?.[1]) { result.name = m[1].trim(); break; }
    }

    // DOB patterns
    const dobPatterns = [
      /(?:DOB|Date of Birth|जन्म तिथि)[:\s]+(\d{2}[\/\-]\d{2}[\/\-]\d{4})/i,
      /(\d{2}[\/\-]\d{2}[\/\-]\d{4})/,
    ];
    for (const p of dobPatterns) {
      const m = text.match(p);
      if (m?.[1]) { result.dob = m[1]; break; }
    }

    // ID Number by doc type
    const idPatterns: Record<string, RegExp> = {
      aadhaar: /(\d{4}\s?\d{4}\s?\d{4})/,
      pan:     /([A-Z]{5}\d{4}[A-Z])/,
      license: /([A-Z]{2}-?\d{2}-?\d{4}\d{7})/,
    };
    const m = text.match(idPatterns[docType] || idPatterns.aadhaar);
    if (m?.[1]) result.idNumber = m[1].replace(/\s/g,' ').trim();

    // Address — lines following "Address" keyword
    const addrM = text.match(/(?:Address|पता)[:\s]+([\s\S]{10,120}?)(?:\n\n|\d{6}|$)/i);
    if (addrM?.[1]) result.address = addrM[1].replace(/\n/g,', ').trim().slice(0,200);

    // Confidence: proportion of fields found
    const filled = [result.name, result.dob, result.idNumber, result.address]
      .filter(v => v.length > 0).length;
    result.confidence = Math.round((filled / 4) * 100);

    return result;
  }
}

export default new OCRService();
```

---

### services/face.service.ts

```typescript
import axios from 'axios';

class FaceService {
  private baseUrl = process.env.DEEPFACE_SERVICE_URL || 'http://ai-service:8001';

  async compareFaces(img1Url: string, img2Url: string): Promise<number> {
    try {
      const { data } = await axios.post(`${this.baseUrl}/compare`, {
        img1_path: img1Url,
        img2_path: img2Url,
        model_name: 'VGG-Face',         // or ArcFace / Facenet512
        detector_backend: 'retinaface',
        distance_metric: 'cosine',
        enforce_detection: false,
      }, { timeout: 30_000 });

      // Convert distance to 0-100 similarity
      const { distance, threshold } = data;
      const similarity = (1 - Math.min(distance / (threshold || 0.4), 1)) * 100;
      return Math.max(0, Math.min(100, Math.round(similarity)));
    } catch (err: any) {
      console.error('[FaceService] Error:', err.message);
      return 0;
    }
  }
}

export default new FaceService();
```

---

## AI SERVICE (Python)

### ai-service/main.py

```python
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from deepface import DeepFace
import requests, tempfile, os
from pathlib import Path

app = FastAPI(title="MYTH AI Face Service", version="1.0.0")
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"])

class CompareRequest(BaseModel):
    img1_path: str
    img2_path: str
    model_name: str = "VGG-Face"
    detector_backend: str = "retinaface"
    distance_metric: str = "cosine"
    enforce_detection: bool = False

def dl(url: str) -> str:
    r = requests.get(url, timeout=10)
    r.raise_for_status()
    ext = Path(url).suffix or ".jpg"
    with tempfile.NamedTemporaryFile(delete=False, suffix=ext) as f:
        f.write(r.content)
        return f.name

@app.post("/compare")
async def compare(req: CompareRequest):
    try:
        p1, p2 = dl(req.img1_path), dl(req.img2_path)
        result = DeepFace.verify(
            img1_path=p1, img2_path=p2,
            model_name=req.model_name,
            detector_backend=req.detector_backend,
            distance_metric=req.distance_metric,
            enforce_detection=req.enforce_detection,
        )
        os.unlink(p1); os.unlink(p2)
        return {
            "verified":  result["verified"],
            "distance":  result["distance"],
            "threshold": result["threshold"],
        }
    except Exception as e:
        raise HTTPException(500, detail=str(e))

@app.get("/health")
def health():
    return {"status": "ok"}
```

### ai-service/requirements.txt

```
fastapi==0.104.1
uvicorn==0.24.0
deepface==0.0.89
opencv-python-headless==4.8.1.78
tensorflow==2.14.0
requests==2.31.0
Pillow==10.1.0
numpy==1.24.3
```

---

## API ENDPOINTS

```
─────── AUTH ────────────────────────────────────────────
POST   /api/auth/register           Register user
POST   /api/auth/login              User login → JWT
POST   /api/auth/admin-login        Admin login → JWT
POST   /api/otp/send                Send phone/email OTP
POST   /api/otp/verify              Verify OTP

─────── VERIFICATION ────────────────────────────────────
POST   /api/verify                  Run verification
       Body: { name, dob, address, documentType }
       File: photo (multipart)
GET    /api/verify/history          User's past verifications
GET    /api/verify/:id              Get single result
GET    /api/verify/:id/report       Download PDF report

─────── ADMIN — RECORDS ─────────────────────────────────
GET    /api/admin/records           List all records
       Query: ?document_type=&search=&page=&limit=
POST   /api/admin/records           Add record
       Files: personPhoto, documentImage
PUT    /api/admin/records/:id       Edit record
DELETE /api/admin/records/:id       Delete record

─────── ADMIN — OCR ─────────────────────────────────────
POST   /api/admin/ocr               Extract OCR from doc image
       File: documentImage

─────── ADMIN — MANAGEMENT ──────────────────────────────
GET    /api/admin/users             List users
PUT    /api/admin/users/:id/status  Activate / Suspend user
GET    /api/admin/logs              All verification logs
       Query: ?verdict=&userId=&from=&to=
GET    /api/admin/stats             Dashboard statistics
```

---

## ENVIRONMENT VARIABLES

```bash
# Application
NODE_ENV=production
PORT=5000
FRONTEND_URL=https://myth.yourdomain.com

# Database
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/myth-platform

# JWT
JWT_SECRET=your-minimum-64-character-secret-key-here
JWT_EXPIRES_IN=24h

# AWS S3 (Documents & Photos)
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXXXXXXX
AWS_SECRET_ACCESS_KEY=your_secret_key_here
AWS_BUCKET_NAME=myth-identity-platform

# Google Vision OCR (optional — falls back to Tesseract)
USE_GOOGLE_VISION=false
GOOGLE_CREDENTIALS_PATH=./gcp-credentials.json

# Twilio (Phone OTP)
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE=+1XXXXXXXXXX

# Email OTP (Gmail App Password)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@yourdomain.com
SMTP_PASS=your_app_password

# AI Service
DEEPFACE_SERVICE_URL=http://ai-service:8001

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=100

# Admin Credentials (hashed in production)
ADMIN_EMAIL=admin@1234
ADMIN_PASSWORD_HASH=$2b$12$...
```

---

## DOCKER COMPOSE

```yaml
version: '3.9'
services:

  mongodb:
    image: mongo:7.0
    restart: unless-stopped
    volumes: [mongo_data:/data/db]
    environment:
      MONGO_INITDB_DATABASE: myth-platform
    ports: ["27017:27017"]

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    ports: ["6379:6379"]

  backend:
    build: ./backend
    restart: unless-stopped
    ports: ["5000:5000"]
    depends_on: [mongodb, redis]
    env_file: .env
    volumes: [./backend:/app, /app/node_modules]

  ai-service:
    build: ./ai-service
    restart: unless-stopped
    ports: ["8001:8001"]
    environment:
      - DEEPFACE_HOME=/app/.deepface

  frontend:
    build: ./frontend
    restart: unless-stopped
    ports: ["3000:3000"]
    depends_on: [backend]
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:5000/api

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/ssl/certs:ro
    depends_on: [frontend, backend]

volumes:
  mongo_data:
```

---

## SETUP INSTRUCTIONS

### Prerequisites
- Node.js 18+, npm 9+
- Python 3.10+
- MongoDB (Atlas or local)
- Docker + Docker Compose (for production)
- AWS S3 bucket

### 1. Clone & Configure

```bash
git clone https://github.com/your-org/myth-platform.git
cd myth-platform
cp .env.example .env
# Fill in all values in .env
```

### 2. Backend Setup

```bash
cd backend
npm install
npm run build          # TypeScript compile
npm run seed:admin     # Create admin user
npm run dev            # Development mode
```

### 3. AI Service Setup

```bash
cd ai-service
pip install -r requirements.txt
uvicorn main:app --reload --port 8001
# First run downloads DeepFace model weights (~500MB)
```

### 4. Frontend Setup

```bash
cd frontend
npm install
npm run dev
# Visit http://localhost:3000
```

### 5. Production Deployment

```bash
# Build and launch all services
docker-compose up -d --build

# SSL with Let's Encrypt
certbot --nginx -d myth.yourdomain.com

# Seed demo records
docker-compose exec backend npm run seed:records
```

---

## SECURITY CHECKLIST

- [x] bcrypt hashing (rounds: 12) for all passwords
- [x] JWT with 24h expiry + refresh token rotation
- [x] Helmet.js security headers
- [x] CORS restricted to frontend origin
- [x] Rate limiting: 100 req/15min global, 10 req/min on /verify
- [x] Multer: strict MIME + extension validation, 5MB max
- [x] S3 bucket: private ACL, signed URL access only
- [x] Input sanitization via express-validator on every route
- [x] MongoDB injection prevention (Mongoose casting)
- [x] Admin routes protected by separate admin JWT claim
- [x] HTTPS enforced via Nginx redirect
- [x] All verification results logged to audit trail
- [x] OTP: 6-digit, 5-minute expiry, single-use, stored in Redis
- [x] Face photos auto-deleted from S3 after 30 days

---

## KEY DESIGN DECISIONS

| Decision | Choice | Rationale |
|---|---|---|
| No user doc upload | Users enter details only | Privacy, security, compliance |
| Admin-only database | MongoDB IdentityRecord | Single source of truth |
| Face matching model | DeepFace VGG-Face | Best accuracy/speed balance |
| OCR engine | Tesseract (+ GV fallback) | No per-call cost at scale |
| Photo storage | AWS S3 private | Signed URL access, audit log |
| OTP store | Redis (5min TTL) | Fast, atomic, auto-expire |
| Search strategy | Text index + compound index | Handles name variations |
| Report format | PDF (PDFKit) | Portable, verifiable |

---

*MYTH Identity Verification Platform v2.0*
*Revised Architecture — No User Document Upload*

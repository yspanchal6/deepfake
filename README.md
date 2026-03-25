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

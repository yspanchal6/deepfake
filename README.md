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

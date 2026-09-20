# SocialSync

**AI-powered social media automation platform** — schedule posts, generate content with AI, manage connected accounts, and publish automatically across platforms.

---

## Overview

SocialSync is a full-stack TypeScript application that helps users:

- 🔐 Manage social media accounts
- ✍️ Generate AI-written content
- 🖼️ Generate or attach media
- 🗓️ Schedule posts for future delivery
- 🚀 Publish posts through a connected social publishing workflow
- 📊 Review account activity and recent generations

The system is intentionally built around a **minimal AWS footprint**:

| AWS Service | Purpose |
|---|---|
| **EC2** | Runs the backend application runtime |
| **RDS (PostgreSQL)** | Relational data storage |
| **S3** | Media and asset storage |
| **IAM** | Least-privilege access control |

> This project does not use any additional AWS managed services beyond these four.

---

## Business Goal

SocialSync is a social content automation dashboard combining:

- User authentication and authorization
- AI content generation
- Media handling
- Scheduling logic
- External social publishing integration

The goal: give a user **one place** to create content and publish it across multiple connected social accounts, without juggling separate tools.

---

## System Architecture

```
User Browser
    │  HTTPS / REST API
    ▼
React Frontend (client/src) ── Vite + TypeScript
    │
    ▼
Nginx Reverse Proxy (EC2)
    │
    ▼
Express API (server/server.ts) ── Routes + Middleware
    │
    ├── Auth Middleware ── JWT validation + user lookup
    │
    ├── Controllers
    │     ├── authController
    │     ├── socialAuthController
    │     ├── accountControllers
    │     ├── postController
    │     └── activityController
    │
    ▼
Prisma ORM (server/prisma/schema.prisma)
    │
    ▼
AWS RDS PostgreSQL ── User, Account, Post, Generation, ActivityLog
    │
    ▼
AWS S3 Bucket ── Upload storage + media assets
    │
    ▼
IAM Role / Instance Profile ── least-privilege access

Scheduler Service (node-cron, server/services/scheduleService.ts)
    ├── checks scheduled posts
    └── publishes ready posts

External integrations:
    ├── Zernio Social API ── social publishing
    ├── Google Gemini ── AI text generation
    ├── Pollinations AI ── AI image generation
    └── Gmail SMTP (Nodemailer) ── password reset email
```

**Architecture notes**

- The frontend is a React single-page app delivered to the browser.
- The backend runs on EC2 behind NGINX and is the central application server.
- Database data is persisted in PostgreSQL on RDS.
- Uploaded and generated media are stored in S3.
- The application uses IAM roles attached to EC2 for restricted AWS access.
- AI-powered content creation depends on external non-AWS services (Gemini, Pollinations).
- Social publishing depends on the Zernio API.
- Scheduled publishing is processed in the background by a cron-based scheduler.

---

## Full Component Inventory

### Frontend — `client/`

Responsibilities:
- User authentication pages
- Dashboard and account overview
- AI content composer
- Scheduling UI
- Platform account connection flow

Key files:
```
client/src/App.tsx
client/src/context/AuthContext.tsx
client/src/api/axios.ts
client/src/pages/*
client/src/components/*
```

### Backend API — `server/`

Responsibilities:
- Express server bootstrapping
- Auth and JWT validation
- Social account sync
- Content generation
- Scheduling management
- Post publishing and activity tracking

Key files:
```
server/server.ts
server/routes/authRoutes.ts
server/routes/socialAuthRoutes.ts
server/routes/accountRoutes.ts
server/routes/postRoutes.ts
server/routes/activityRoutes.ts
server/middlewares/authMiddleware.ts
```

### Controller Layer

```
server/controllers/authController.ts
server/controllers/socialAuthController.ts
server/controllers/accountControllers.ts
server/controllers/postController.ts
server/controllers/activityController.ts
```

Responsibilities: sign up / login, password reset, OAuth URL generation, account sync from social providers, AI content generation, post scheduling and retrieval, activity logging.

### Data Layer — `server/prisma/schema.prisma`

Core models: `User`, `Account`, `Post`, `Generation`, `ActivityLog`

Key relationships:
- One user has many accounts
- One user has many posts
- One user has many generations
- One user has many activity logs
- Posts can reference activity logs

### Scheduler — `server/services/scheduleService.ts`

Responsibilities:
1. Look up scheduled posts whose time has arrived
2. Mark them as processing to avoid duplicates
3. Find connected social accounts for that user and platform
4. Generate temporary S3 URLs where required
5. Publish to the social API
6. Update status to `published` or `failed`
7. Create activity logs

### Storage & AWS — `server/config/*`

```
server/config/prisma.ts    # Prisma database setup
server/config/s3.ts        # AWS S3 client initialization
server/config/s3Url.ts     # Signed URL generation
server/config/multer.ts    # File upload handling
```

### External Services

| Service | Role |
|---|---|
| **Zernio** | Social account auth and publishing workflow |
| **Google Gemini** | AI text generation |
| **Pollinations AI** | AI image generation |
| **Gmail SMTP (Nodemailer)** | Password reset emails |

---

## AWS-Only Infrastructure Scope

| AWS component | Purpose |
|---|---|
| EC2 | Runs the backend and serves the app runtime |
| RDS PostgreSQL | Stores users, posts, accounts, generations, logs |
| S3 | Stores uploads, generated media, and object assets |
| IAM | Gives EC2 the minimal runtime permissions |

This is the supported production infrastructure model for the application — no other managed AWS services are used.

---

## Runtime and Data Flow

### Authentication flow
1. User logs in from the frontend.
2. The API validates credentials with bcrypt.
3. A JWT is generated and returned to the client.
4. Protected routes use `protect` middleware to verify the header.
5. The user is read from PostgreSQL and attached to the request.

### AI generation flow
1. User submits a prompt and selected options in the UI.
2. API calls the Gemini model to generate content.
3. If image generation is enabled, the backend calls Pollinations AI.
4. Generated assets and content are stored to the database and optionally to S3.
5. The UI displays the generated result for review and scheduling.

### Scheduling flow
1. User creates or edits a post and specifies a future date/time.
2. The API stores the scheduled post in RDS.
3. The scheduler polls for ready posts.
4. When due, the app resolves connected accounts and publishes to the target platform through Zernio.
5. Post status is updated to `published` or `failed`.
6. Activity logs are created for auditability.

### Media flow
1. Media is uploaded via multipart form to the backend.
2. The upload is processed and stored in S3.
3. Temporary signed URLs are generated when publishing requires a downloadable or public link.
4. Media metadata is persisted in the database alongside the post record.

---

## Database Model Summary

**User** → owns Accounts, creates Posts, creates Generations, logs ActivityLog

| Model | Key Fields |
|---|---|
| **User** | `id`, `email`, `password`, `name`, `zernioProfileId`, `resetPasswordToken`, `resetPasswordExpires`, `createdAt`, `updatedAt` |
| **Account** | `id`, `userId`, `platform` (enum), `handle`, `zernioAccountId`, `accessToken`, `refreshToken`, `tokenExpiresAt`, `status` (enum), `avatarUrl`, `createdAt`, `updatedAt` |
| **Post** | `id`, `userId`, `content`, `mediaUrl`, `mediaType` (enum), `platforms` (enum[]), `scheduledFor`, `status` (enum), `createdAt`, `updatedAt` |
| **Generation** | `id`, `userId`, `prompt`, `content`, `mediaUrl`, `mediaType` (enum), `tone`, `createdAt`, `updatedAt` |
| **ActivityLog** | `id`, `userId`, `actionType` (enum), `description`, `relatedPostId`, `platform`, `aiGeneratedText`, `createdAt`, `updatedAt` |

---

## Security Model

- ✅ JWT authentication for protected routes
- ✅ Password hashing with bcrypt
- ✅ Reset tokens hashed before storage
- ✅ Rate limiting on authentication routes
- ✅ Sanitized API error responses in production
- ✅ EC2 IAM role restricts AWS access to required resources only
- ✅ S3 used for media instead of storing binary assets on the app server

---

## Local Development Guide

### Prerequisites

- Node.js 18+
- PostgreSQL database
- AWS account (for deployment only)
- Google Gemini API key
- Zernio API key
- SMTP Gmail credentials for email delivery

### Backend environment (`server/.env`)

```env
PORT=3000
DATABASE_URL=postgresql://username:password@host:5432/socialsync
JWT_SECRET=your_secure_random_secret
JWT_EXPIRES_IN=7d

GEMINI_API_KEY=your_google_ai_key
ZERNIO_API_KEY=your_zernio_key

AWS_REGION=us-east-1
S3_BUCKET_NAME=socialsync-media

EMAIL_USER=your_email@gmail.com
EMAIL_PASS=your_gmail_app_password

FRONTEND_URL=http://localhost:5173
NODE_ENV=development
```

### Frontend environment (`client/.env`)

```env
VITE_API_BASE_URL=http://localhost:3000
```

### Run locally

```bash
# Backend
cd server
npm install
npx prisma generate
npm run server

# Frontend (in a separate terminal)
cd client
npm install
npm run dev
```

---

## Project Structure

```
SocialSync/
├── client/                     # React frontend
│   ├── src/
│   ├── public/
│   ├── package.json
│   ├── vite.config.ts
│   └── tsconfig*.json
│
├── server/                     # Express + TypeScript backend
│   ├── config/
│   ├── controllers/
│   ├── generated/
│   ├── middlewares/
│   ├── prisma/
│   ├── routes/
│   ├── services/
│   ├── server.ts
│   ├── package.json
│   └── tsconfig.json
│
├── docs/
│   ├── architect.md
│   ├── deployment.md
│   └── flow.md
├── README.md
├── package.json
└── .gitignore
```

---

## Summary

SocialSync is a practical social media automation system built with a simple, intentional architecture:

- Browser-based React frontend
- Node.js + Express backend on EC2
- PostgreSQL on RDS for persistence
- S3 for media storage
- IAM for controlled access
- External AI and social publishing services for generation and delivery

This design keeps the system easy to deploy, easy to reason about, and aligned with the AWS-only target scope requested for the project.
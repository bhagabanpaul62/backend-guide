# EnrouteLogTech - Project Context & Progress Tracker

**Last Updated**: June 18, 2026 (Session 2)  
**Current Phase**: Phase 1.5 - Frontend Authentication UI (Next.js)  
**Status**: 🟡 In Progress

---

## 🚀 START HERE WHEN YOU REOPEN THE PROJECT

**Where we stopped:** Phase 1 (Authentication Backend) is COMPLETE and FROZEN. Now building Phase 1.5 - the frontend UI for authentication.

### ✅ Phase 1 Complete (Backend)

- `sendOtp`, `verifyOtp`, `register`, `submitRegistration`
- `login`, `logout`, `refreshToken`, `forgetPassword`
- `POST /documents/upload`
- `authMiddleware` + `roleMiddleware`
- All schemas with UUID
- Cloudflare R2 document upload
- Email service (Nodemailer)

### 🎨 Current Task: Phase 1.5 - Frontend UI

Build these pages in Next.js connecting to real backend APIs:

```
PUBLIC PAGES:
□ Landing page
□ Login (email + password)
□ OTP verification
□ Registration (multi-step: user + company + branch)
□ Document upload
□ Waiting for approval page
□ Forgot password
□ 404 page

DASHBOARD (shell only - no business modules):
□ Layout (sidebar + header)
□ Protected routes (redirect if not logged in)
□ Profile menu
□ Logout
```

### Backend API Reference (All Working):

```
POST /api/v1/auth/send-otp
POST /api/v1/auth/otp-verification
POST /api/v1/auth/register
POST /api/v1/auth/submit-registration
POST /api/v1/auth/login
POST /api/v1/auth/logout
POST /api/v1/auth/refresh-token
POST /api/v1/auth/forget-password
POST /api/v1/documents/upload
```

### After Phase 1.5 is done → Phase 2: Company Network

---

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Business Model](#business-model)
3. [Tech Stack](#tech-stack)
4. [Database Design](#database-design)
5. [Progress Tracker](#progress-tracker)
6. [Current Tasks](#current-tasks)
7. [Upcoming Features](#upcoming-features)
8. [Architecture Decisions](#architecture-decisions)
9. [Learning Notes](#learning-notes)
10. [Daily Updates](#daily-updates)

---

## 🎯 Project Overview

**EnrouteLogTech** is a B2B Logistics Procurement and Transportation Management SaaS platform.

**Target Users:**

- Manufacturing Companies
- Logistics Companies
- Transporters
- Brokers
- Fleet Owners

**Core Features:**

- RFQ Creation & Live Bidding
- Vendor Network Management (Connection-based)
- Trip Execution & Fleet Management
- Multi-company TMS with Subscription Model
- Document Management & Compliance

**Platform Goal:**
Digitize the entire logistics procurement lifecycle from RFQ creation to trip completion, enabling manufacturers, transporters, brokers, and fleet owners to collaborate efficiently through a controlled business network.

---

## 💰 Business Model

### Subscription-Based Revenue

- **Plans**: Starter, Pro, Enterprise, Custom
- **Pricing Model**: Per-user seat licensing
- **Revenue Driver**: Additional seat purchases
- **Payment Verification**: Manual approval via payment proof screenshots (Razorpay/Stripe integration planned)

### Company Types (Strict - One Type Per Company)

**BUSINESS RULE**: A company can belong to ONLY ONE category. No mixing allowed.

1. **Customer** - Manufacturing/Shipper companies
2. **Transporter** - Logistics service providers
3. **Broker** - Freight brokers
4. **Fleet Owner** - Vehicle owners

**Important**: If a transporter wants to become a broker, they must register a separate company account.

### User Roles

- `rootAdmin` - Platform Owner (EnrouteLogTech Admin)
- `admin` - Company Administrator
- `viewer` - Read-only access
- `bidManager` - RFQ & Bidding Management
- `tripManager` - Trip Execution & Fleet Management
- `accounts` - Finance & Billing Management

### Multi-User Management

**BUSINESS RULE**: No credential sharing allowed. Each employee needs their own login.

**Example**:

- Company: ABC Logistics
- Allowed Users: 5
- Employees: Aman, Rahul, Sourav, Rakesh, Neha
- Need 6th user? → Must purchase additional seat

### Registration Workflow

```
Email Entry
    ↓
OTP Verification
    ↓
Existing User? → Login
    ↓
New User? → Registration:
    - Select Company Type
    - Enter User Details
    - Enter Company Details
    - Upload Documents (GST, PAN, RC, Insurance)
    - Select Subscription Plan
    - Upload Payment Proof
    ↓
Submit → Status: Pending
    ↓
Admin Manual Review:
    - Documents verification
    - Payment proof verification
    - Company information check
    ↓
Outcome: Approved / Rejected / Blocked
    ↓
Only Approved companies can access platform
```

### Enroute Company ID System

Every company receives a unique identifier:

- `CUS-1001` - Customer
- `TRN-2001` - Transporter
- `BRK-3001` - Broker
- `FLT-4001` - Fleet Owner

Companies can discover and connect using this ID.

### Company Network System

**Works like LinkedIn/Facebook connections for B2B logistics**

**Connection Flow**:

```
Customer: Tata Steel
    ↓
Sends connection request to
    ↓
Transporter: ABC Logistics
    ↓
Transporter accepts
    ↓
Connection Active
    ↓
Now customer can invite transporter to RFQs
```

**Supported Relationships**:

- Customer → Transporter
- Transporter → Broker
- Transporter → Fleet Owner

**BUSINESS RULE**: Connection required before RFQ invitation.

---

## 🛠️ Tech Stack

| Layer             | Technology     | Purpose                                              |
| ----------------- | -------------- | ---------------------------------------------------- |
| **Frontend**      | Next.js        | Server-side rendering, dynamic routing, modern UI/UX |
| **Backend**       | Express.js     | RESTful API development, middleware, business logic  |
| **Database**      | PostgreSQL     | Relational data management, ACID compliance          |
| **ORM**           | Drizzle ORM    | Type-safe queries, schema migrations                 |
| **Storage**       | Cloudflare R2  | Scalable object storage for documents                |
| **Auth**          | JWT            | Token-based authentication                           |
| **Container**     | Docker         | Consistent dev/prod environments                     |
| **Orchestration** | Docker Compose | Multi-container management                           |
| **Proxy**         | Nginx          | Request routing, SSL termination, load balancing     |
| **VCS**           | GitHub         | Source code management, collaboration                |
| **Hosting**       | Contabo VPS    | Cost-effective dedicated server                      |

### Why This Stack?

1. **Scalability**: PostgreSQL and Docker enable horizontal scaling
2. **Performance**: Next.js SSR and Nginx optimize load times
3. **Developer Experience**: Drizzle ORM provides type safety
4. **Cost Efficiency**: Cloudflare R2 and Contabo VPS reduce costs
5. **Security**: JWT + PostgreSQL row-level security
6. **Maintainability**: Modular architecture, clean separation

### Planned Integrations

- **Payment Gateway**: Razorpay / Stripe
- **Real-time**: WebSockets (for live bidding)
- **Email**: SendGrid / AWS SES
- **Monitoring**: PM2 / Grafana
- **Notifications**: WhatsApp Business API

---

## 🗄️ Database Design

### Completed Schemas (DBML)

#### 1. **users**

**Purpose**: Employee accounts with role-based access

```
Fields:
- id (PK)
- companyId (FK → companies)
- branchId (FK → branches)
- name
- email (unique)
- passwordHash
- phone
- profilePhoto
- status (active/inactive/blocked)
- userRole (rootAdmin/admin/viewer/bidManager/tripManager/accounts)
- lastLogin
- createdAt
```

#### 2. **companies**

**Purpose**: Company profiles and status management

```
Fields:
- id (PK)
- companyName
- enrouteCompanyId (unique) - e.g., CUS-1001
- companyType (customer/transporter/broker/fleetOwner)
- gstNumber
- panNumber
- companyEmail
- companyPhone
- logo
- status (pending/rejected/approved/blocked)
- createdAt
```

#### 3. **branches**

**Purpose**: Company branch locations

```
Fields:
- id (PK)
- companyId (FK → companies)
- branchName
- branchCode
- city
- state
- status (active/inactive/blocked)
- branchAddress
```

#### 4. **subscriptionPlans**

**Purpose**: Available subscription tiers

```
Fields:
- id (PK)
- planeName (Starter/Pro/Enterprise/Custom)
- maxUsers
- monthlyPrice
```

#### 5. **companySubscriptions**

**Purpose**: Active company subscriptions

```
Fields:
- companyId (FK → companies)
- planId (FK → subscriptionPlans)
- allowedUsers
- startDate
- expiryDate
- paymentStatus (pending/paid/failed)
```

#### 6. **documents**

**Purpose**: Company document storage (compliance)

```
Fields:
- id (PK)
- companyId (FK → companies)
- documentType (GST/PAN/Logo/RC/Insurance/PaymentProof)
- fileUrl (Cloudflare R2 path)
```

#### 7. **companyConnections**

**Purpose**: B2B network relationships

```
Fields:
- id (PK)
- sourceCompanyId (FK → companies)
- targetCompanyId (FK → companies)
- connectionType (company type enum)
- status (pending/accepted/rejected/blocked)
- createdAt
- acceptedAt
```

### Schema Organization Structure (Recommended)

```
backend/src/db/
├── index.js                    # Main DB connection
└── schemas/
    ├── core/                   # companies.js, branches.js
    ├── auth/                   # users.js, sessions.js, tokens.js
    ├── subscriptions/          # plans.js, company-subscriptions.js
    ├── documents/              # documents.js
    ├── connections/            # company-connections.js
    ├── rfq/                    # (future) RFQ management
    ├── trips/                  # (future) Trip management
    ├── vehicles/               # (future) Fleet management
    └── tracking/               # (future) GPS tracking
```

**Pattern**: Domain-driven organization with barrel exports (index.js in each folder)

---

## ✅ Progress Tracker

### Phase 1: Foundation ⏳ In Progress

- [x] Business model finalized
- [x] Database design completed (DBML)
- [x] Tech stack decided
- [x] Project structure planned
- [x] Project context documentation created
- [ ] Docker PostgreSQL setup
- [ ] Drizzle ORM configuration
- [ ] Schema files created (Drizzle format)
- [ ] First migration generated
- [ ] Express-PostgreSQL connection established
- [ ] Database connection test

### Phase 2: Authentication & Onboarding 📋 Planned

- [ ] Registration API
- [ ] OTP verification (email)
- [ ] Login/Logout APIs
- [ ] JWT middleware
- [ ] Role-based access control middleware
- [ ] Document upload (Cloudflare R2 integration)
- [ ] Admin approval workflow API
- [ ] Password reset flow

### Phase 3: Company Management 📋 Planned

- [ ] Company profile CRUD
- [ ] Branch management CRUD
- [ ] User management (seat enforcement logic)
- [ ] Subscription management APIs
- [ ] Connection request system
- [ ] Connection approval/rejection workflow
- [ ] Company search (by Enroute ID)

### Phase 4: RFQ System 📋 Planned

- [ ] RFQ creation
- [ ] RFQ invitations (connection-based filtering)
- [ ] Live bidding system (WebSocket)
- [ ] Bid submission
- [ ] Contract awarding
- [ ] RFQ status tracking

### Phase 5: Trip Management 📋 Planned

- [ ] Indent management
- [ ] Trip creation
- [ ] Vehicle assignment
- [ ] Driver assignment
- [ ] GPS tracking integration
- [ ] POD (Proof of Delivery) upload
- [ ] Trip completion workflow

### Phase 6: Analytics & Reporting 📋 Planned

- [ ] Admin dashboard
- [ ] Company dashboard
- [ ] Reports generation
- [ ] Usage analytics
- [ ] Financial reports

---

## 🎯 Current Tasks

**Week**: Week 1  
**Focus**: Database Setup & Schema Migration

### Today's Goals (June 13, 2026):

1. [x] Create PROJECT_CONTEXT.md documentation
2. [ ] Set up PostgreSQL in Docker
3. [ ] Configure Drizzle ORM
4. [ ] Create first schema files

### This Week's Goals:

- [ ] Complete database setup
- [ ] Generate first migration
- [ ] Test database connection
- [ ] Create user and company schemas in Drizzle format

### Blockers:

- None currently

### Questions/Decisions Needed:

- None currently

---

## 🔮 Upcoming Features

### Next Sprint (After Auth):

1. RFQ Management Module
2. Live Bidding System (WebSocket)
3. Connection-based invitation system

### Q3 2026 Roadmap:

- Payment gateway integration (Razorpay/Stripe)
- WhatsApp notifications
- Mobile-responsive frontend
- Email notification system

### Q4 2026 Roadmap:

- Mobile app (React Native)
- Advanced analytics dashboard
- GPS tracking integration
- Multi-language support

### Future Considerations:

- AI-based route optimization
- Predictive pricing
- Automated load matching
- Blockchain for POD verification

---

## 🏗️ Architecture Decisions

### 1. Why Drizzle ORM over Prisma?

**Decision**: Use Drizzle ORM  
**Reasoning**:

- Lighter weight and faster
- Better TypeScript support
- More control over raw queries
- Faster migration generation
- Less abstraction = easier debugging

### 2. Why PostgreSQL over MongoDB?

**Decision**: Use PostgreSQL  
**Reasoning**:

- Complex relational data (companies → users → branches)
- ACID compliance required for financial/subscription data
- Better support for JOINs (connections, RFQs, trips)
- Row-level security for multi-tenancy
- Industry standard for SaaS platforms

### 3. Why Multi-Tenant Architecture?

**Decision**: Single database with row-level isolation  
**Reasoning**:

- Each company's data isolated by companyId
- Cost-effective (no separate DB per tenant)
- Easier to maintain and backup
- Scalable for B2B SaaS model
- Simpler analytics across tenants

### 4. Why Connection-Based Network?

**Decision**: Require connection before RFQ invitation  
**Reasoning**:

- Controlled access (security)
- Business relationships tracked
- Prevents spam RFQs
- Can add connection-specific pricing later
- Builds trust between parties

### 5. Why Manual Approval for Companies?

**Decision**: Admin reviews all new companies  
**Reasoning**:

- Quality control (prevent fake accounts)
- Regulatory compliance (verify GST/PAN)
- Payment verification (manual screenshot review initially)
- Prevents fraud and misuse
- Can automate later with AI verification

### 6. Why Strict "One Company = One Type" Rule?

**Decision**: No mixing of company types  
**Reasoning**:

- Clear business logic
- Prevents conflicts of interest
- Simplified permission system
- Easier to track metrics
- Industry best practice

---

## 📚 Learning Notes

### Key Concepts Learned:

#### 1. **Barrel Export Pattern (index.js)**

- `index.js` in each folder re-exports all schemas
- Enables clean imports: `import { users, sessions } from './schemas/auth'`
- Hides internal file structure
- Makes refactoring easier

#### 2. **Schema Organization**

- Organize by business domain (auth, core, subscriptions)
- Not by technical layer (models, controllers)
- Each domain is independent module
- Easier to scale and maintain

#### 3. **Naming Conventions**

- **File names**: Plural, kebab-case (`users.js`, `company-connections.js`)
- **Export names**: Plural, camelCase (`users`, `companyConnections`)
- **Database tables**: Plural, snake_case (`users`, `company_connections`)

#### 4. **Multi-Tenancy Pattern**

- Always filter queries by `companyId`
- Use middleware to inject companyId automatically
- Prevent cross-tenant data leaks
- Implement at database and application level

#### 5. **Subscription Enforcement**

- Check `allowedUsers` vs current user count before adding users
- Throw error if limit exceeded
- Suggest plan upgrade in error message
- Track active users, not total created users

### Useful Resources:

- Drizzle Docs: https://orm.drizzle.team/
- PostgreSQL Best Practices: https://wiki.postgresql.org/wiki/Don%27t_Do_This
- Express.js Security: https://expressjs.com/en/advanced/best-practice-security.html
- Multi-Tenancy Patterns: https://docs.microsoft.com/en-us/azure/architecture/guide/multitenant/overview

---

## 🔄 Daily Updates

### June 13, 2026 (Saturday)

**Status**: Learning & Planning

**What We Discussed**:

- Schema folder structure and organization
- Barrel export pattern (index.js usage)
- File naming conventions (plural vs singular)
- Complete project context review
- Created PROJECT_CONTEXT.md for future reference

**Decisions Made**:

- Use plural naming for schema files
- Organize schemas by business domain
- Use index.js for clean imports
- Document everything in PROJECT_CONTEXT.md

**Next Steps**:

- Set up Docker PostgreSQL
- Configure Drizzle ORM
- Create schema files in Drizzle format

**Blockers**: None

---

### [Add new date entries below as you progress]

---

### June 18, 2026 (Thursday)

**Status**: Backend Foundation & Auth Design

**What We Did**:

- Pushed all schemas to PostgreSQL (companies, branches, users, subscription plans, company subscriptions)
- Fixed multiple schema issues (gel-core → pg-core typos, enum name conflicts, broken references)
- Set up Node.js subpath imports (`#db/*`, `#schemas/*`, etc.) for clean path aliases
- Installed backend packages: bcrypt, jsonwebtoken, cors, helmet, express-validator
- Designed complete authentication & session architecture (JWT access + refresh tokens, user_sessions table, role middleware)
- Reviewed auth architecture (scored 8.8/10) - added suggestions: rate limiting, account lockout, audit logging, refresh token rotation
- Created README.md with full project summary
- Set up GitHub repository and pushed code

**Decisions Made**:

- Generate enrouteCompanyId in service layer (not schema) - needs companyType
- Use application-level security (not RLS) - database is not exposed, only APIs are
- No Passport.js - custom email/OTP/JWT is sufficient for now
- All URL fields use length 1000

**Next Steps**:

- Build utility helpers (response.js, generators.js)
- Build error handler middleware
- Create first API: Company Registration (service → controller → route)
- Implement auth middleware chain (authMiddleware → sessionMiddleware → roleMiddleware)

**Blockers**: None

---

### June 18, 2026 (Session 2 - Evening)

**Status**: Understanding asyncHandler & error handling

**What We Did**:

- Deep-dived into how `asyncHandler` works (higher-order function that wraps controllers in try/catch)
- Understood why `next(error)` is better than sending response directly (one central error handler, different status codes, works with Express design)
- Learned the relationship: `AppError` (throws, no res needed) vs `errorResponse` (sends, needs res) - they are partners, not alternatives
- Decided: NO service layer for now → logic in controllers → use response utilities directly, skip AppError
- Searched full codebase and identified what's missing (see "START HERE" section at top)

**Key Findings (what's missing)**:

- 🔴 No global error handler (errorHandler.js) - asyncHandler's next(error) has nowhere to go
- 🔴 Routes not wired in app.js
- 🟡 Bug in errorResponse: `{ errors }` undefined, should be `{ errors: error }`
- 🟡 CORS typo: `Credential` → `credentials`
- 🟡 controller/, routes/, middleware/ folders are empty

**Next Steps**: See the "START HERE WHEN YOU REOPEN THE PROJECT" section at the top of this file.

**Blockers**: None

---

### June 30, 2026 — Phase 1 Complete 🎉

**Status**: Authentication Backend COMPLETE → Moving to Phase 1.5 (Frontend)

**What Was Completed (Full Auth System):**

- ✅ All auth APIs: sendOtp, verifyOtp, register, submitRegistration, login, logout, refreshToken, forgetPassword
- ✅ Document upload: Multer + Cloudflare R2
- ✅ authMiddleware (JWT verification + session check)
- ✅ roleMiddleware (role-based access control)
- ✅ Email service (Nodemailer + Hostinger SMTP)
- ✅ Email templates (OTP, Welcome, Approval, Rejection)
- ✅ Session management (one user one session)
- ✅ JWT hybrid system (stateless speed + stateful control)
- ✅ All schemas with UUID IDs
- ✅ Clean folder structure (controllers/routes by module)

**Key Architecture Decisions:**

- Hybrid auth: JWT (speed) + session DB (revocation)
- HttpOnly cookies (XSS protection)
- One session per user (prevent credential sharing)
- Forget password kills all sessions

**Phase 1 is FROZEN. No more auth changes.**

**Next:** Phase 1.5 — Build authentication UI in Next.js

**Blockers**: None

---

## 📝 Notes for AI Assistant (Kiro)

### When Starting a New Chat:

1. ✅ Read this `PROJECT_CONTEXT.md` file FIRST
2. ✅ Check "Current Tasks" section for today's focus
3. ✅ Review "Progress Tracker" to see what's completed
4. ✅ Check "Daily Updates" for latest progress
5. ✅ Continue from where we left off

### Project Philosophy:

**Business First → Database → Backend → Frontend**

### Development Approach:

- Build only what's needed for current requirements
- Avoid premature optimization
- Focus on business logic first
- Scalability through clean architecture, not over-engineering

### Communication Style:

- I'm here to **guide and teach**, not write code for you
- Ask questions to understand concepts
- Learn by doing, not by copy-pasting
- Document your learnings in this file

---

## 🎓 Learning Goals

### Technical Skills to Master:

- [ ] PostgreSQL query optimization
- [ ] Drizzle ORM advanced patterns
- [ ] JWT authentication & authorization
- [ ] Multi-tenancy implementation
- [ ] Docker containerization
- [ ] API design best practices
- [ ] File upload handling (Cloudflare R2)
- [ ] WebSocket for real-time features

### Business Domain Knowledge:

- [ ] Logistics industry workflows
- [ ] RFQ/bidding processes
- [ ] Transportation management concepts
- [ ] Fleet management operations
- [ ] B2B SaaS business models

---

**END OF CONTEXT**

---

**This file will be updated daily as the project progresses. Always refer to this file when starting a new conversation to maintain context continuity.**

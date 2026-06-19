# Backend Essentials
### A Personal Reference Guide - Written from Real Experience Building EnrouteLogTech

> Read this before starting any new backend project. Every concept here was learned by actually building something real.

---

## Table of Contents

1. [Chapter 1: Setting Up the Backend](#chapter-1-setting-up-the-backend)
2. [Chapter 2: Utility Functions & Error Handling System](#chapter-2-utility-functions--error-handling-system)
3. [Chapter 3: Setting Up Email Notification System](#chapter-3-setting-up-email-notification-system)
2. [Chapter 2: Utility Functions & Error Handling System](#chapter-2-utility-functions--error-handling-system)

---

---

# Chapter 1: Setting Up the Backend

## 1.1 Project Structure

Before writing a single line of code, create this folder structure:

```
backend/
└── src/
    ├── controllers/     # Handle HTTP requests/responses
    ├── services/        # Business logic (optional, add later)
    ├── routes/          # Define API endpoints
    ├── middleware/      # Auth, error handling, validation
    │   └── error/
    │       └── errorHandler.js
    ├── utils/           # Helper functions
    │   ├── asyncHandler.js
    │   ├── response.js
    │   ├── AppError.js
    │   └── generators.js
    ├── db/              # Database connection and schemas
    │   ├── Schemas/
    │   └── index.js
    ├── app.js           # Express app setup
    └── index.js         # Server entry point (start server)
```

**Rule:** Every folder has a clear, single responsibility. Never mix concerns.

---

## 1.2 Required Packages

Install these before you start:

```bash
# Core packages
pnpm add express dotenv

# Security
pnpm add helmet cors bcrypt jsonwebtoken

# Validation
pnpm add express-validator

# Database (for this project)
pnpm add drizzle-orm pg

# Dev tools
pnpm add -D nodemon drizzle-kit
```

**What each package does:**

| Package | Purpose |
|---------|---------|
| `express` | Web framework |
| `dotenv` | Load environment variables from .env |
| `helmet` | Adds security headers automatically |
| `cors` | Allows frontend to call your API |
| `bcrypt` | Hash passwords (never store plain passwords) |
| `jsonwebtoken` | Create and verify JWT tokens |
| `express-validator` | Validate request body inputs |
| `drizzle-orm` | Type-safe ORM for database |
| `pg` | PostgreSQL driver |
| `nodemon` | Auto-restart server on file changes |

---

## 1.3 Environment Variables (.env)

Always create a `.env` file in your backend folder. **Never push this to GitHub.**

```env
# Server
PORT=8000
NODE_ENV=development

# Database
DATABASE_URL="postgres://user:password@localhost:5432/dbname"

# Frontend URL (for CORS)
FRONTEND_URL=http://localhost:3000

# JWT
JWT_SECRET=your-super-secret-key-change-in-production
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=30d
```

**Rules:**
- Add `.env` to `.gitignore` immediately
- Create a `.env.example` with empty values for teammates
- Never hardcode secrets in your code

---

## 1.4 Path Aliases (Clean Imports)

Instead of `../../../utils/response.js`, use `#utils/response.js`.

Add to `package.json`:

```json
{
  "type": "module",
  "imports": {
    "#db/*": "./src/db/*",
    "#schemas/*": "./src/db/Schemas/*",
    "#utils/*": "./src/utils/*",
    "#middleware/*": "./src/middleware/*",
    "#controllers/*": "./src/controllers/*",
    "#routes/*": "./src/routes/*"
  }
}
```

**Benefit:** Move files around without breaking imports.

---

## 1.5 The `app.js` File (Express Setup)

This is where you configure Express. Think of it as the "settings" file for your server.

**What goes in `app.js`:**
1. Security middleware (helmet, cors)
2. Body parsers (json, urlencoded)
3. All API routes
4. 404 handler
5. Global error handler (LAST)

**Template:**
```javascript
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import { errorHandler } from '#middleware/error/errorHandler.js';
import { errorResponse } from '#utils/response.js';

export const app = express();

// 1. Security
app.use(helmet());
app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true   // lowercase 'c', with 's' at end
}));

// 2. Body parsers
app.use(express.json({ limit: '16kb' }));
app.use(express.urlencoded({ extended: true, limit: '16kb' }));

// 3. Routes
// app.use('/api/v1/companies', companyRoutes);
// app.use('/api/v1/auth', authRoutes);

// 4. 404 handler (before error handler)
app.use((req, res) => {
  return errorResponse(res, `Route ${req.path} not found`, 404);
});

// 5. Global error handler (ALWAYS LAST)
app.use(errorHandler);
```

---

## 1.6 The `index.js` File (Server Start)

This is the entry point. Its only job is to:
1. Check database connection
2. Start the server

```javascript
import 'dotenv/config';
import { app } from './app.js';
import { db } from '#db/index.js';

const startServer = async () => {
  try {
    await db.execute('SELECT 1');   // test DB connection
    console.log('Database connected');

    app.listen(process.env.PORT, () => {
      console.log(`Server running on port ${process.env.PORT}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);   // exit with error code (important for Docker)
  }
};

startServer();
```

**Why check DB before listening?**
If the database is down, there's no point starting the server. This prevents accepting requests that will all fail.

---

## 1.7 Naming Conventions

| Location | Convention | Example |
|----------|-----------|---------|
| File names | kebab-case | `company-routes.js` |
| Schema property | camelCase | `companyName` |
| Database column | snake_case | `company_name` |
| JavaScript variables | camelCase | `const companyData` |
| Constants | UPPER_SNAKE | `const MAX_USERS = 5` |
| Classes | PascalCase | `class AppError` |
| URL endpoints | kebab-case | `/api/v1/company-types` |

**One rule for all imports:**
- Use `#alias` paths everywhere (never relative paths like `../../`)
- Always include `.js` extension in imports

---

---

# Chapter 2: Utility Functions & Error Handling System

## 2.1 The Big Picture

Before writing any controller, understand how these 4 pieces work together:

```
Controller
    ↓ (success)           ↓ (error thrown)
successResponse()     asyncHandler catches it
    ↓                     ↓
Client gets           next(error)
success JSON              ↓
                      errorHandler middleware
                          ↓
                      errorResponse()
                          ↓
                      Client gets
                      error JSON
```

**The 4 pieces:**
1. `asyncHandler` - wraps controller, auto-catches errors
2. `AppError` - creates errors with status codes
3. `response.js` - formats success and error JSON
4. `errorHandler` - middleware that receives all errors

---

## 2.2 Response Utilities (`utils/response.js`)

**Purpose:** Ensure every API response has the same structure.

**Three functions:**

### `successResponse` - for successful operations
```javascript
export function successResponse(res, data, message = 'Success', statusCode = 200) {
  return res.status(statusCode).json({
    success: true,
    message,
    data
  });
}
```

Usage in controller:
```javascript
return successResponse(res, company, 'Company created', 201);
```

Client receives:
```json
{
  "success": true,
  "message": "Company created",
  "data": { "id": 1, "companyName": "ABC Logistics" }
}
```

---

### `errorResponse` - for known business errors
```javascript
export function errorResponse(res, message = 'Error', statusCode = 500, error = null) {
  return res.status(statusCode).json({
    success: false,
    message,
    ...(error && { error })
  });
}
```

Usage:
```javascript
return errorResponse(res, 'Company not found', 404);
return errorResponse(res, 'Internal Server Error', 500);
```

---

### `validationErrorResponse` - for input validation failures
```javascript
export function validationErrorResponse(res, message = 'Validation failed', error) {
  return res.status(400).json({
    success: false,
    message,
    error
  });
}
```

Usage:
```javascript
return validationErrorResponse(res, 'Validation failed', {
  email: 'Email is required',
  phone: 'Invalid phone number'
});
```

**Key rule:** Always use these 3 functions for ALL responses. Never write `res.status().json()` directly in controllers. Consistency is the goal.

---

## 2.3 AppError Class (`utils/AppError.js`)

**Purpose:** Create errors that carry a status code, so the error handler knows what HTTP status to send.

```javascript
export class AppError extends Error {
  constructor(message, statusCode) {
    super(message);          // sets error.message
    this.statusCode = statusCode;  // custom field
  }
}
```

**Why extend Error?**
- `throw` only works with Error objects (or subclasses)
- You need the standard `error.message` field
- You add your own `statusCode` field

**When to use:**
```javascript
// When something is wrong but it's a known business rule
throw new AppError('Email already exists', 409);
throw new AppError('Company not found', 404);
throw new AppError('Not authorized', 401);
```

**Common HTTP status codes:**

| Code | Meaning | Example |
|------|---------|---------|
| 200 | OK | Data retrieved |
| 201 | Created | Resource created |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Not logged in |
| 403 | Forbidden | Not allowed |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Email already exists |
| 500 | Server Error | Unexpected crash |

---

## 2.4 AsyncHandler (`utils/asyncHandler.js`)

**Purpose:** Wrap async controllers to automatically catch errors, so you never write try/catch in every controller.

```javascript
export const asyncHandler = (fn) => {
  return async (req, res, next) => {
    try {
      await fn(req, res, next);
    } catch (error) {
      next(error);
    }
  };
};
```

**Breaking it down:**

```javascript
// asyncHandler is a function that TAKES a function (your controller)
const asyncHandler = (fn) => {

  // It RETURNS a new function (this is what Express actually calls)
  return async (req, res, next) => {

    try {
      await fn(req, res, next);  // runs YOUR controller
    } catch (error) {
      next(error);  // forwards error to errorHandler middleware
    }

  };
};
```

**Why does it return a function?**
Express needs a function to call on each request. You can't call your controller immediately - you just hand it to Express wrapped in a safety net.

**Usage:**
```javascript
// Wrap your controller when defining routes
router.post('/register', asyncHandler(registerCompany));
//                        ↑ returns a wrapped function, not calls it
```

**The trade:**
- ❌ Without asyncHandler: Write try/catch in every controller
- ✅ With asyncHandler: Write zero try/catch, errors auto-forwarded

---

## 2.5 ErrorHandler Middleware (`middleware/error/errorHandler.js`)

**Purpose:** The "receiver" of all errors forwarded by asyncHandler. Sends clean JSON to client.

```javascript
import { errorResponse } from '#utils/response.js';

export const errorHandler = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const message = err.message || 'Internal Server Error';
  return errorResponse(res, message, statusCode);
};
```

**The 4-parameter rule:**
Express identifies an error handler by its 4 parameters `(err, req, res, next)`. If you write 3 params, Express treats it as normal middleware and errors won't reach it.

**Register it LAST in app.js:**
```javascript
app.use(errorHandler);  // Must be the very last app.use()
```

**Why last?**
Express processes middleware top to bottom. The error handler must be at the bottom so it can catch errors from everything above.

---

## 2.6 How All 4 Pieces Work Together

### Scenario 1: Everything works fine
```
POST /api/companies/register
        ↓
asyncHandler(registerCompany) called by Express
        ↓
registerCompany runs successfully
        ↓
successResponse(res, data, 'Created', 201)
        ↓
Client receives: { success: true, data: {...} }
```

---

### Scenario 2: Known business error (duplicate email)
```
POST /api/companies/register
        ↓
asyncHandler(registerCompany) called
        ↓
registerCompany detects duplicate email
        ↓
throw new AppError('Email already exists', 409)
        ↓
asyncHandler's catch block runs
        ↓
next(error) called
        ↓
Express jumps to errorHandler (4-param middleware)
        ↓
err.statusCode = 409, err.message = 'Email already exists'
        ↓
errorResponse(res, 'Email already exists', 409)
        ↓
Client receives: { success: false, message: 'Email already exists' }
```

---

### Scenario 3: Unexpected crash (DB down)
```
POST /api/companies/register
        ↓
asyncHandler(registerCompany) called
        ↓
db.insert() throws (database connection lost)
        ↓
asyncHandler's catch block runs
        ↓
next(error) called
        ↓
errorHandler receives error
        ↓
err.statusCode = undefined → uses default 500
err.message = 'connection refused'
        ↓
errorResponse(res, 'connection refused', 500)
        ↓
Client receives: { success: false, message: 'connection refused' }
```

---

## 2.7 Writing a Controller (Putting It All Together)

```javascript
import { asyncHandler } from '#utils/asyncHandler.js';
import { AppError } from '#utils/AppError.js';
import { successResponse, validationErrorResponse } from '#utils/response.js';
import { db } from '#db/index.js';
import { companies } from '#schemas/companies/index.js';
import { eq } from 'drizzle-orm';
import { generateEnrouteCompanyId } from '#utils/generators.js';

export const registerCompany = asyncHandler(async (req, res) => {
  const { companyName, companyEmail, companyType } = req.body;

  // Step 1: Validate input
  if (!companyName || !companyEmail || !companyType) {
    return validationErrorResponse(res, 'Validation failed', {
      companyName: !companyName ? 'Required' : null,
      companyEmail: !companyEmail ? 'Required' : null,
      companyType: !companyType ? 'Required' : null,
    });
  }

  // Step 2: Check for duplicates
  const existing = await db.select()
    .from(companies)
    .where(eq(companies.companyEmail, companyEmail));

  if (existing.length > 0) {
    throw new AppError('Company email already exists', 409);
    // asyncHandler catches this → next(error) → errorHandler handles it
  }

  // Step 3: Generate Enroute ID
  const enrouteCompanyId = generateEnrouteCompanyId(companyType);

  // Step 4: Insert into database
  const [newCompany] = await db.insert(companies).values({
    companyName,
    companyEmail,
    companyType,
    enrouteCompanyId,
    status: 'pending'
  }).returning();

  // Step 5: Send success response
  return successResponse(res, newCompany, 'Company registered successfully', 201);
});
```

**Notice:**
- ✅ No try/catch (asyncHandler handles it)
- ✅ Uses validationErrorResponse for bad input
- ✅ Uses AppError for business rule violations
- ✅ Uses successResponse for success
- ✅ Clean, readable, focused on business logic only

---

## 2.8 Writing Routes

```javascript
import { Router } from 'express';
import { registerCompany } from '#controllers/company.controller.js';

const router = Router();

// POST /api/v1/companies/register
router.post('/register', registerCompany);
//           ↑ path    ↑ controller (asyncHandler is inside the controller)

export default router;
```

Connect in `app.js`:
```javascript
import companyRoutes from '#routes/company.routes.js';

app.use('/api/v1/companies', companyRoutes);
// Full path becomes: POST /api/v1/companies/register
```

---

## 2.9 Quick Reference - When to Use What

| Situation | What to Use |
|-----------|------------|
| Request succeeded | `successResponse(res, data, message, statusCode)` |
| Input validation failed | `validationErrorResponse(res, message, errors)` |
| Known business error (email exists, not found) | `throw new AppError(message, statusCode)` |
| Route not found | `errorResponse(res, message, 404)` in 404 handler |
| Wrapping a controller | `asyncHandler(yourController)` |
| Receiving all thrown errors | `errorHandler` middleware (auto, last in app.js) |

---

## 2.10 Common Mistakes to Avoid

### ❌ Mistake 1: Forgetting `return`
```javascript
// Wrong - sends 2 responses, crashes
if (!email) {
  validationErrorResponse(res, 'Email required', {});
}
// continues execution and sends another response!

// Correct
if (!email) {
  return validationErrorResponse(res, 'Email required', {});
}
```

### ❌ Mistake 2: errorHandler not last
```javascript
// Wrong - routes added after errorHandler won't have error handling
app.use(errorHandler);
app.use('/api/v1/companies', companyRoutes);  // errors here won't be caught

// Correct
app.use('/api/v1/companies', companyRoutes);
app.use(errorHandler);  // LAST
```

### ❌ Mistake 3: Only 3 params in errorHandler
```javascript
// Wrong - Express won't treat this as error handler
app.use((req, res, next) => { ... });

// Correct - must have 4 params
app.use((err, req, res, next) => { ... });
```

### ❌ Mistake 4: Storing JWT in localStorage
```javascript
// Wrong - vulnerable to XSS attacks
localStorage.setItem('token', token);

// Correct - use HttpOnly cookies
res.cookie('accessToken', token, { httpOnly: true, secure: true });
```

### ❌ Mistake 5: Storing plain passwords
```javascript
// Wrong - never ever
await db.insert(users).values({ password: req.body.password });

// Correct - always hash
const hash = await bcrypt.hash(req.body.password, 10);
await db.insert(users).values({ passwordHash: hash });
```

---

*This document is updated as new backend concepts are learned and implemented in the EnrouteLogTech project.*


---

---

# Chapter 3: Setting Up Email Notification System

## 3.1 Why a Global Email Service?

In a real application, you send many different types of emails:
- OTP verification
- Welcome after registration
- Company approved/rejected by admin
- Password reset
- Trip notifications
- Invoice reminders

If you write email logic separately in every controller, it becomes messy and hard to maintain. Instead, build **one global email service** that every part of your app uses.

**The pattern:**
```
Any Controller
      ↓
calls sendOtpEmail() / sendWelcomeEmail() etc.
      ↓
These call the core sendEmail() function
      ↓
sendEmail() uses nodemailer transporter
      ↓
Email goes to user's inbox
```

---

## 3.2 Package Required

```bash
pnpm add nodemailer
```

`nodemailer` is the standard Node.js package for sending emails via SMTP.

---

## 3.3 Folder Structure

```
src/
└── services/
    └── email/
        ├── email.service.js      ← Core sender + specific email functions
        └── emailTemplates.js     ← HTML templates for each email type
```

**Why separate templates from service?**
- Templates are just HTML strings (UI concern)
- Service is the sending logic (backend concern)
- Easy to update a template without touching sending logic
- Easy to add new templates without touching service

---

## 3.4 Environment Variables

Add to your `.env` file:

```env
# Email Server Configuration
EMAIL_HOST=smtp.hostinger.com
EMAIL_PORT=465
EMAIL_SECURE=true
EMAIL_USER=hello@yourdomain.com
EMAIL_PASSWORD=your-email-password
EMAIL_FROM=noreply@yourdomain.com
```

**Important `.env` rules:**
- Comments use `#` NOT `//`
- Port `465` → `EMAIL_SECURE=true` (SSL)
- Port `587` → `EMAIL_SECURE=false` (TLS/STARTTLS)
- Never push `.env` to GitHub (make sure it's in `.gitignore`)

**For development/testing use Mailtrap:**
```env
EMAIL_HOST=smtp.mailtrap.io
EMAIL_PORT=2525
EMAIL_SECURE=false
EMAIL_USER=your-mailtrap-user
EMAIL_PASSWORD=your-mailtrap-password
EMAIL_FROM=noreply@enroutelogtech.com
```

[Mailtrap](https://mailtrap.io) captures all emails in a fake inbox - emails never reach real users during development.

---

## 3.5 The Transporter (Connection to Email Server)

A transporter is created ONCE when the server starts and reused for every email:

```javascript
import nodemailer from 'nodemailer';

const transporter = nodemailer.createTransport({
  host: process.env.EMAIL_HOST,
  port: Number(process.env.EMAIL_PORT),   // ← Must be a number, not string
  secure: process.env.EMAIL_SECURE === 'true',  // ← Convert string to boolean
  auth: {
    user: process.env.EMAIL_USER,
    pass: process.env.EMAIL_PASSWORD,
  },
});
```

**Common mistakes:**
- `port` must be `Number(process.env.EMAIL_PORT)` - env vars are strings, nodemailer needs a number
- `secure` must be a boolean - use `=== 'true'` to convert from string
- Create transporter at module level (once), not inside the function (every call)

---

## 3.6 Core `sendEmail()` Function

This is the base function that all specific emails use:

```javascript
export const sendEmail = async ({ to, subject, html, text }) => {
  await transporter.sendMail({
    from: process.env.EMAIL_FROM,
    to,
    subject,
    html,    // HTML version (shown in modern email clients)
    text,    // Plain text fallback (shown if HTML fails)
  });
};
```

**Parameters:**
- `to` - recipient email address
- `subject` - email subject line
- `html` - HTML content of the email
- `text` - plain text version (always provide this as fallback)

---

## 3.7 Specific Email Functions

Each specific function wraps `sendEmail` with the right subject and template:

```javascript
// OTP verification
export async function sendOtpEmail(email, otp) {
  await sendEmail({
    to: email,
    subject: 'EnrouteLogTech - Email Verification OTP',
    html: otpEmailTemplate(otp),
    text: `Your OTP is: ${otp}. Valid for 10 minutes. Do not share.`
  });
}

// After registration submitted
export async function sendWelcomeEmail(email, name) {
  await sendEmail({
    to: email,
    subject: 'Welcome to EnrouteLogTech - Application Received',
    html: welcomeEmailTemplate(name),
    text: `Welcome ${name}! Your application is under review.`
  });
}

// When admin approves company
export async function sendApprovalEmail(email, companyName) {
  await sendEmail({
    to: email,
    subject: 'EnrouteLogTech - Your Account is Approved!',
    html: approvalEmailTemplate(companyName),
    text: `Congratulations! ${companyName} has been approved.`
  });
}

// When admin rejects company
export async function sendRejectionEmail(email, companyName, reason) {
  await sendEmail({
    to: email,
    subject: 'EnrouteLogTech - Application Status Update',
    html: rejectionEmailTemplate(companyName, reason),
    text: `Application for ${companyName} was not approved. Reason: ${reason}`
  });
}
```

---

## 3.8 HTML Email Templates

Templates are functions that accept data and return HTML strings:

```javascript
// emailTemplates.js

export function otpEmailTemplate(otp) {
  return `
    <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
      <div style="background-color: #1a56db; padding: 20px; text-align: center;">
        <h1 style="color: white; margin: 0;">EnrouteLogTech</h1>
      </div>
      <div style="padding: 30px; background-color: #f9fafb;">
        <h2>Your Verification Code</h2>
        <div style="background-color: #1a56db; color: white;
                    font-size: 32px; font-weight: bold;
                    text-align: center; padding: 20px;
                    border-radius: 8px; letter-spacing: 8px;">
          ${otp}
        </div>
        <p style="color: #6b7280; margin-top: 20px;">
          Expires in <strong>10 minutes</strong>. Do not share.
        </p>
      </div>
    </div>
  `;
}
```

**Template rules:**
- Always use inline CSS (email clients don't support stylesheets)
- Keep it simple - complex layouts break in some email clients
- Always have a plain text fallback in `sendEmail()`
- Dynamic data goes in using template literals: `${variable}`

---

## 3.9 How to Use in Controllers

```javascript
import { sendOtpEmail, sendWelcomeEmail } from '#services/email/email.service.js';

// In your auth controller
export const sendOtp = asyncHandler(async (req, res) => {
  const { email } = req.body;

  // Generate OTP
  const otp = Math.floor(100000 + Math.random() * 900000).toString();

  // Save to DB
  await db.insert(otpVerification).values({
    email,
    otp,
    expiresAt: new Date(Date.now() + 10 * 60 * 1000)  // 10 min
  });

  // Send email - wrap in try/catch so email failure doesn't crash the API
  try {
    await sendOtpEmail(email, otp);
  } catch (emailError) {
    console.error('Email sending failed:', emailError.message);
    // Don't throw - still respond to user, just log the error
  }

  return successResponse(res, null, 'OTP sent to your email', 200);
});
```

**Important:** Always wrap `sendEmail()` calls in try/catch inside controllers. If the email server is down, you don't want the entire API to crash.

---

## 3.10 OTP Security Best Practices

### Generate OTP securely:
```javascript
import crypto from 'crypto';

// More secure than Math.random()
const otp = crypto.randomInt(100000, 999999).toString();
```

### Hash the OTP before storing (optional but more secure):
```javascript
import bcrypt from 'bcrypt';

// Store hashed OTP in DB
const hashedOtp = await bcrypt.hash(otp, 10);

// Send plain OTP to user's email
await sendOtpEmail(email, otp);

// Store hashed version
await db.insert(otpVerification).values({
  email,
  otp: hashedOtp,    // store hash
  expiresAt: ...
});

// When verifying, compare with bcrypt
const isValid = await bcrypt.compare(userEnteredOtp, storedHashedOtp);
```

### OTP Expiry Check in DB:
```javascript
const otpRecord = await db.select()
  .from(otpVerification)
  .where(
    and(
      eq(otpVerification.email, email),
      eq(otpVerification.isUsed, false),
      gt(otpVerification.expiresAt, new Date())  // not expired
    )
  )
  .limit(1);

if (!otpRecord.length) {
  throw new AppError('OTP expired or invalid', 400);
}
```

---

## 3.11 Email Notification Flow for EnrouteLogTech

```
Registration Flow:
User enters email
      ↓
sendOtpEmail(email, otp)
      ↓
User verifies OTP
      ↓
User fills registration form
      ↓
Form submitted → company status: pending
      ↓
sendWelcomeEmail(email, name)

Admin Approval Flow:
Admin approves company
      ↓
sendApprovalEmail(companyEmail, companyName)
      ↓
Company can now login

Admin Rejection Flow:
Admin rejects company
      ↓
sendRejectionEmail(companyEmail, companyName, reason)
      ↓
Company reapplies after fixing issues
```

---

## 3.12 Path Alias for Services

Make sure `package.json` has this in `imports`:

```json
"imports": {
  "#services/*": "./src/services/*"
}
```

Then import like:
```javascript
import { sendOtpEmail } from '#services/email/email.service.js';
```

---

## 3.13 Quick Reference - Common Mistakes

| Mistake | Why it breaks | Fix |
|---------|--------------|-----|
| `port: process.env.EMAIL_PORT` | String, not number | `Number(process.env.EMAIL_PORT)` |
| `secure: process.env.EMAIL_SECURE` | String, not boolean | `=== 'true'` |
| Port 465 + `secure: false` | SSL mismatch | Port 465 needs `secure: true` |
| Port 587 + `secure: true` | TLS mismatch | Port 587 needs `secure: false` |
| Comments with `//` in `.env` | Invalid syntax | Use `#` for comments |
| No plain text fallback | Breaks plain email clients | Always provide `text:` field |
| Not wrapping in try/catch in controller | Email crash crashes API | Always wrap email calls in try/catch |
| Pushing `.env` to GitHub | Exposes credentials | Always check `.gitignore` |

---

*Chapter 3 complete. Next: Chapter 4 - Authentication (JWT, Sessions, Middleware)*

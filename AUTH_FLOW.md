# EnrouteLogTech - Authentication Flow Diagrams

---

## 1. Landing Page Decision


```
┌─────────────────────────────────────┐
│         EnrouteLogTech              │
│                                     │
│  ┌─────────────┐ ┌───────────────┐  │
│  │   SIGN UP   │ │   SIGN IN     │  │
│  │  (New User) │ │(Existing User)│  │
│  └──────┬──────┘ └───────┬───────┘  │
└─────────┼────────────────┼──────────┘
          ↓                ↓
    SIGN UP FLOW      SIGN IN FLOW
```

---

## 2. SIGN UP Flow (New User Registration)


```
┌──────────────────────────────────────────────────────┐
│                    SIGN UP FLOW                      │
└──────────────────────────────────────────────────────┘

STEP 1: Enter Email
┌─────────────────┐
│  User enters    │
│     email       │
└────────┬────────┘
         ↓
POST /auth/send-otp { email }
         ↓
┌─────────────────────────────┐
│  Backend:                   │
│  1. Validate email format   │
│  2. Generate 6-digit OTP    │
│  3. Save to otpVerification │
│     (expires in 10 min)     │
│  4. Send OTP email          │
└─────────────────────────────┘
         ↓
RESPONSE: { success: true, message: "OTP sent" }

─────────────────────────────────────────────────────

STEP 2: Verify OTP
┌─────────────────┐
│  User enters    │
│  OTP from email │
└────────┬────────┘
         ↓
POST /auth/verify-otp { email, otp }
         ↓
┌─────────────────────────────────────┐
│  Backend:                           │
│  1. Find OTP in DB for this email   │
│  2. Check OTP not expired           │
│  3. Check OTP not already used      │
│  4. Compare OTP                     │
│  5. Mark OTP as isUsed = true       │
│  6. Check: does user exist?         │
└───────────┬─────────────────────────┘
            ↓
    ┌───────┴────────┐
    ↓                ↓
userExists=FALSE  userExists=TRUE
    ↓                ↓
Show Register    "Account exists"
Form             → Go to Sign In

─────────────────────────────────────────────────────

STEP 3: Registration Form
┌─────────────────────────────────────┐
│  User fills registration form:      │
│                                     │
│  USER INFO:                         │
│  - Full Name                        │
│  - Email (pre-filled)               │
│  - Phone Number                     │
│  - Password                         │
│  - Confirm Password                 │
│                                     │
│  COMPANY INFO:                      │
│  - Company Name                     │
│  - Company Type (dropdown)          │
│    [Customer/Transporter/Broker/    │
│     Fleet Owner]                    │
│  - Company Email                    │
│  - Company Phone                    │
│  - GST Number                       │
│  - PAN Number                       │
└────────────┬────────────────────────┘
             ↓
POST /auth/register { all fields }
             ↓
┌──────────────────────────────────────────┐
│  Backend:                                │
│  1. Validate all required fields         │
│  2. Check user email not registered      │
│  3. Check company email not registered   │
│  4. Check GST not already registered     │
│  5. Hash password (bcrypt, 10 rounds)    │
│  6. Generate enrouteCompanyId            │
│     CUS-1001 / TRN-2001 etc.            │
│  7. INSERT company                       │
│     status = 'pending'                   │
│  8. INSERT user                          │
│     role = 'admin' (first user)          │
│     companyId = new company id           │
│  9. Send welcome email                   │
└──────────────────┬───────────────────────┘
                   ↓
RESPONSE: 201 { message: "Application submitted" }
                   ↓
┌──────────────────────────────────────────┐
│  Show: "Your application is submitted.  │
│  Admin will review and approve within   │
│  24-48 hours. You'll receive an email." │
└──────────────────────────────────────────┘
                   ↓
            WAIT FOR ADMIN APPROVAL
```

---

## 3. Admin Approval Flow


```
┌──────────────────────────────────────────────────────┐
│                  ADMIN APPROVAL FLOW                 │
└──────────────────────────────────────────────────────┘

Admin logs in to platform
         ↓
GET /admin/companies?status=pending
         ↓
┌──────────────────────────┐
│  Admin reviews:          │
│  - Company details       │
│  - Documents (GST, PAN)  │
│  - Payment proof         │
└──────────┬───────────────┘
           ↓
    ┌──────┴───────┐
    ↓              ↓
 APPROVE         REJECT/BLOCK
    ↓              ↓
PATCH            PATCH
/admin/          /admin/
companies/       companies/
:id/approve      :id/reject
    ↓              ↓
company          company
status=          status=
'approved'       'rejected'
    ↓              ↓
Send             Send
Approval         Rejection
Email            Email
    ↓
User Can Now Login
```

---

## 4. SIGN IN Flow (Login)


```
┌──────────────────────────────────────────────────────┐
│                    SIGN IN FLOW                      │
└──────────────────────────────────────────────────────┘

┌──────────────────┐
│  User enters:    │
│  - Email         │
│  - Password      │
└────────┬─────────┘
         ↓
POST /auth/login { email, password }
         ↓
┌──────────────────────────────────────────────────────┐
│  Backend Validation Chain:                           │
│                                                      │
│  CHECK 1: Validate input                             │
│  → email exists? password exists?                   │
│  → If no → 400 Validation Error                     │
│                                                      │
│  CHECK 2: Find user                                  │
│  → SELECT user WHERE email = email                  │
│  → If not found → 404 "User not found"              │
│                                                      │
│  CHECK 3: Verify password                            │
│  → bcrypt.compare(password, user.passwordHash)      │
│  → If wrong → 401 "Invalid credentials"             │
│                                                      │
│  CHECK 4: Company status                             │
│  → SELECT company WHERE id = user.companyId         │
│  → If pending  → 403 "Account pending approval"     │
│  → If rejected → 403 "Account rejected"             │
│  → If blocked  → 403 "Account blocked"              │
│                                                      │
│  CHECK 5: User status                                │
│  → If user.status = inactive → 403 "Account inactive│
│  → If user.status = blocked  → 403 "Account blocked"│
│                                                      │
│  CHECK 6: ONE USER ONE SESSION                       │
│  → SELECT session WHERE userId = user.id            │
│    AND isActive = true                               │
│    AND expiresAt > now                               │
│  → If session exists →                              │
│    409 "Account active on another device"           │
└──────────────────────────┬───────────────────────────┘
                           ↓
                    ALL CHECKS PASSED
                           ↓
┌──────────────────────────────────────────────────────┐
│  CREATE SESSION:                                     │
│  INSERT into userSessions:                           │
│  {                                                   │
│    userId: user.id,                                  │
│    deviceId: req.headers['user-agent'],              │
│    ipAddress: req.ip,                                │
│    isActive: true,                                   │
│    expiresAt: 30 days from now                       │
│  }                                                   │
└──────────────────────────┬───────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────┐
│  GENERATE TOKENS:                                    │
│                                                      │
│  accessToken = jwt.sign({                            │
│    userId, companyId, role, sessionId                │
│  }, JWT_SECRET, { expiresIn: '15m' })                │
│                                                      │
│  refreshToken = jwt.sign({                           │
│    userId, sessionId                                 │
│  }, JWT_SECRET, { expiresIn: '30d' })                │
│                                                      │
│  UPDATE userSessions SET                             │
│  refreshToken = refreshToken                         │
│  WHERE id = session.id                               │
└──────────────────────────┬───────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────┐
│  SET COOKIES:                                        │
│                                                      │
│  accessToken cookie:                                 │
│  { httpOnly: true, maxAge: 15min }                   │
│                                                      │
│  refreshToken cookie:                                │
│  { httpOnly: true, maxAge: 30days }                  │
└──────────────────────────┬───────────────────────────┘
                           ↓
RESPONSE: 200 {
  success: true,
  message: "Login successful",
  data: { id, name, email, role, companyId }
  // NO password hash in response!
}
                           ↓
                  USER ENTERS PLATFORM
```

---

## 5. Protected API Flow (After Login)


```
┌──────────────────────────────────────────────────────┐
│              PROTECTED API REQUEST FLOW              │
└──────────────────────────────────────────────────────┘

Frontend makes any API request
(cookies sent automatically)
         ↓
┌──────────────────────────────┐
│  authMiddleware:             │
│  1. Read accessToken cookie  │
│  2. jwt.verify(token)        │
│  3. If invalid → 401         │
│  4. Extract payload          │
│     { userId, companyId,     │
│       role, sessionId }      │
│  5. Check session in DB      │
│  6. If not active → 401      │
│  7. req.user = payload       │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│  roleMiddleware (optional):  │
│  Check if user role is       │
│  allowed for this endpoint   │
│  If not → 403 Forbidden      │
└──────────────┬───────────────┘
               ↓
         Controller runs
         req.user available
```

---

## 6. LOGOUT Flow


```
┌──────────────────────────────────────────────────────┐
│                     LOGOUT FLOW                      │
└──────────────────────────────────────────────────────┘

User clicks Logout
         ↓
POST /auth/logout (with cookies)
         ↓
┌──────────────────────────────────────────────────────┐
│  Backend:                                            │
│  1. Read accessToken from cookie                     │
│  2. Extract sessionId from token                     │
│  3. UPDATE userSessions                              │
│     SET isActive = false                             │
│     WHERE id = sessionId                             │
│  4. res.clearCookie('accessToken')                   │
│  5. res.clearCookie('refreshToken')                  │
└──────────────────────────┬───────────────────────────┘
                           ↓
RESPONSE: 200 { message: "Logged out successfully" }
                           ↓
                 Redirect to Sign In page
```

---

## 7. REFRESH TOKEN Flow


```
┌──────────────────────────────────────────────────────┐
│                  REFRESH TOKEN FLOW                  │
└──────────────────────────────────────────────────────┘

Frontend gets 401 (access token expired)
         ↓
Frontend auto-calls:
POST /auth/refresh-token (with cookies)
         ↓
┌──────────────────────────────────────────────────────┐
│  Backend:                                            │
│  1. Read refreshToken from cookie                    │
│  2. jwt.verify(refreshToken)                         │
│  3. Extract { userId, sessionId }                    │
│  4. Find session in DB                               │
│     WHERE id = sessionId                             │
│     AND isActive = true                              │
│     AND expiresAt > now                              │
│  5. If not found → 401 "Session expired, login again"│
│  6. Generate NEW accessToken                         │
│  7. Set new accessToken cookie                       │
└──────────────────────────┬───────────────────────────┘
                           ↓
RESPONSE: 200 { message: "Token refreshed" }
                           ↓
Frontend retries original request
```

---

## 8. FORGOT PASSWORD Flow


```
┌──────────────────────────────────────────────────────┐
│                FORGOT PASSWORD FLOW                  │
└──────────────────────────────────────────────────────┘

STEP 1: Request Reset
┌─────────────────┐
│  User enters    │
│  email address  │
└────────┬────────┘
         ↓
POST /auth/forgot-password { email }
         ↓
┌──────────────────────────────────────────────────────┐
│  Backend:                                            │
│  1. Find user by email                               │
│  2. Whether user exists or NOT →                    │
│     ALWAYS respond same message                      │
│     (security: don't reveal if email registered)     │
│  3. If user exists:                                  │
│     a. Generate reset token (crypto.randomBytes(32)) │
│     b. Hash the token (bcrypt)                       │
│     c. Save to passwordResets table:                 │
│        { email, hashedToken, expiresAt: 1 hour }     │
│     d. Send email with reset link:                   │
│        https://frontend.com/reset?token=xxxxx        │
└──────────────────────────────────────────────────────┘
         ↓
RESPONSE: 200 { message: "If email exists, reset link sent" }

─────────────────────────────────────────────────────

STEP 2: Reset Password
┌──────────────────────────┐
│  User clicks email link  │
│  Enters new password     │
└────────────┬─────────────┘
             ↓
POST /auth/reset-password { token, newPassword, confirmPassword }
             ↓
┌──────────────────────────────────────────────────────┐
│  Backend:                                            │
│  1. Find token in passwordResets table               │
│  2. Check token not expired                          │
│  3. Check token not already used                     │
│  4. Hash new password                                │
│  5. UPDATE user.passwordHash                         │
│  6. Mark token as used                               │
│  7. DEACTIVATE ALL sessions for this user            │
│     (force logout everywhere - security)             │
└──────────────────────────┬───────────────────────────┘
                           ↓
RESPONSE: 200 { message: "Password reset successful" }
                           ↓
Redirect to Sign In page
```

---

## 9. ONE USER ONE SESSION Rule


```
┌──────────────────────────────────────────────────────┐
│             ONE USER ONE SESSION RULE                │
└──────────────────────────────────────────────────────┘

Aman logs in → Session created (isActive=true)
         ↓
Aman shares credentials with Rahul
         ↓
Rahul tries to login with Aman's credentials
         ↓
┌──────────────────────────────────────────────────────┐
│  Backend checks:                                     │
│  SELECT * FROM userSessions                          │
│  WHERE userId = aman.id                              │
│  AND isActive = true                                 │
│  AND expiresAt > now                                 │
└──────────────────────────────────────────────────────┘
         ↓
    Session found!
         ↓
RESPONSE: 409 {
  success: false,
  message: "This account is currently active on another
            device. Please logout first."
}
         ↓
Rahul CANNOT login ✅
Credential sharing PREVENTED ✅
```

---

## 10. Summary - All Auth Endpoints

```
┌──────────────────────────────────────────────────────┐
│                ALL AUTH ENDPOINTS                    │
├──────────────────────┬───────────────────────────────┤
│ ENDPOINT             │ PURPOSE                       │
├──────────────────────┼───────────────────────────────┤
│ POST /send-otp       │ Send OTP to email             │
│ POST /verify-otp     │ Verify OTP + check user exists│
│ POST /register       │ Create user + company         │
│ POST /login          │ Login → get tokens            │
│ POST /logout         │ Logout → clear session+cookies│
│ POST /refresh-token  │ Get new access token          │
│ POST /forgot-password│ Send reset email              │
│ POST /reset-password │ Reset password with token     │
└──────────────────────┴───────────────────────────────┘

Base URL: /api/v1/auth
```

---

## 11. Database Tables Used in Auth

```
otpVerification     ← Stores OTPs temporarily
users               ← User accounts
companies           ← Company profiles
userSessions        ← Active login sessions
passwordResets      ← Password reset tokens (create this schema!)
```

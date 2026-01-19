### Authentication / Login Implementation Plan

This backend uses a *JWT + Database Session based authentication system*
to ensure secure login, real logout, and session control.

---

## 🔑 Login Flow Design API

### 1️⃣ Login Request
User sends:
- Email OR Mobile
- Password

Validation rules:
- Password is mandatory
- Either email or mobile must be present

---

### 2️⃣ User Verification
- User is searched using email OR mobile
- Password is verified using bcrypt.compare
- Blocked users are denied access

---

### 3️⃣ Device Detection
Device is detected from the User-Agent header and normalized into:
- Android
- iPhone
- iPad
- Windows
- Mac
- Linux
- Postman
- Unknown

This value is stored with the session for security tracking.

---

### 4️⃣ Session Creation (Database)

On every successful login, a new session is created.

Each login = *one unique session*

Session contains:
- userId
- sessionId (UUID)
- tokenId (JWT jti)
- device
- isActive: true
- auto-expiry after 7 days

---

### 5️⃣ JWT Refresh Token Generation

A refresh token is generated with payload:
{ userId, role, type: "AUTH", jti: tokenId }
- Signed using JWT_SECRET
- Expires in 7 days
- Token is valid only if DB session exists

---

### 6️⃣ Secure Cookie Storage

Two HTTP-only cookies are set:
- refreshToken (JWT)
- sessionId (Session identifier)

Cookie settings:
- httpOnly: true
- sameSite: strict
- secure: enabled in production
- maxAge: 7 days

---

## 🛡 Authentication Middleware (Token.js) File

All protected routes use a central authentication middleware.

### Middleware Responsibilities:
1. Read refreshToken and sessionId from cookies
2. Reject request if any value is missing
3. Verify JWT signature using JWT_SECRET
4. Validate token type (AUTH)
5. Find session in database using:
   - userId
   - sessionId
   - tokenId (jti)
   - isActive = true
6. If session not found → unauthorized
7. Attach authenticated user info to request:
req.user = { id, role }


This ensures:
- Token alone is not trusted
- Every request is validated against DB state

---


## 🧾 Session Design (SessionSchema / ORM Model)

The session storage is used to track *active login sessions*  
independent of the database or ORM being used (MongoDB / MySQL / PostgreSQL).

This can be implemented as:
- MongoDB collection (Mongoose)
- SQL table (MySQL / PostgreSQL using ORM)

---

### Fields

- userId
  → Reference to User (foreign key or relation)

- sessionId
  → Unique session identifier (UUID)

- tokenId
  → JWT jti value (used to bind token with session)

- device
  → Detected device (Android / iPhone / Windows / etc.)

- isActive
  → Boolean flag to indicate active session

- createdAt
  → Timestamp of session creation

- expiresAt
  → Session expiry time (e.g. 7 days)

---

### Purpose

- Enable *real logout*
- Allow *token revocation*
- Track *multiple device logins*
- Prevent reuse of *stolen or old tokens*

---

### Notes

- In SQL databases, this is stored as a *sessions table*
- In NoSQL databases, this is stored as a *sessions collection*
- Authentication logic remains the same regardless of ORM or database

----



## 🚪 Logout Flow (Based on Actual Implementation)
### Logout API

---
### On Logout

When a user logs out, the backend performs *server-side invalidation*:

1. *Session Invalidation*
   - sessionId is read from HTTP-only cookies
   - The corresponding session record is updated:
     
     isActive = false
     
   - This ensures the session cannot be reused again

2. *Cookie Cleanup*
   - refreshToken cookie is cleared
   - sessionId cookie is cleared

---

### After Logout

- The JWT refresh token becomes *useless*
- The session is *no longer valid*
- Any further request using old cookies is rejected by auth middleware

---

### Why This Logout Design?

- Logout is *real*, not client-side only
- Prevents reuse of stolen tokens
- Works correctly across multiple devices
- Session-based control gives full security ownership to backend

---

### Result

After logout:
- Token cannot be reused
- Session cannot be reactivated
- User must login again to access protected routes
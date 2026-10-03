# 🏗️ Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    BROWSER EXTENSION                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   popup.html │  │  content.js  │  │background.js │      │
│  │   popup.js   │  │              │  │              │      │
│  │   popup.css  │  │ (Auto-fill)  │  │(Service Wkr) │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                 │                  │               │
│         └─────────────────┴──────────────────┘               │
│                           │                                  │
│                           │ HTTP Requests                    │
│                           │ (CORS enabled)                   │
└───────────────────────────┼──────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      BACKEND API                             │
│                      (api.py)                                │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              REST API Endpoints                      │    │
│  │  /register  /login  /logout  /check-session         │    │
│  │  /add  /get-all-passwords  /edit  /delete           │    │
│  └─────────────────┬───────────────────────────────────┘    │
│                    │                                         │
│  ┌─────────────────▼───────────────────────────────────┐    │
│  │         Authentication & Authorization              │    │
│  │       @login_required decorator                      │    │
│  │       Session management                             │    │
│  └─────────────────┬───────────────────────────────────┘    │
│                    │                                         │
│  ┌─────────────────▼───────────────────────────────────┐    │
│  │              Business Logic                          │    │
│  │    User.set_password() / check_password()           │    │
│  │    Password.encrypt_password() / decrypt_password() │    │
│  └─────────────────┬───────────────────────────────────┘    │
│                    │                                         │
│  ┌─────────────────▼───────────────────────────────────┐    │
│  │              Database Layer                          │    │
│  │         SQLAlchemy Models                            │    │
│  │         User & Password tables                       │    │
│  └─────────────────┬───────────────────────────────────┘    │
└────────────────────┼─────────────────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   DATABASE            │
         │   SQLite/PostgreSQL   │
         │                       │
         │  ┌────────────────┐   │
         │  │  users         │   │
         │  │  - id          │   │
         │  │  - username    │   │
         │  │  - password_hash│  │
         │  └────────────────┘   │
         │                       │
         │  ┌────────────────┐   │
         │  │  passwords     │   │
         │  │  - id          │   │
         │  │  - user_id     │   │
         │  │  - service     │   │
         │  │  - username    │   │
         │  │  - password_enc│   │
         │  └────────────────┘   │
         └───────────────────────┘
```

## Component Interactions

### 1. User Registration Flow

```
User clicks "Sign Up" in extension popup
    │
    ▼
popup.js: register() function
    │
    ▼
API POST /register with {username, password}
    │
    ▼
api.py: register() endpoint
    │
    ├─► Validate input (length, required fields)
    │
    ├─► Check if username exists
    │
    ├─► User.set_password() → hash with Werkzeug
    │
    ├─► Save to database
    │
    └─► Create session
    │
    ▼
Return success → User logged in automatically
```

### 2. Adding Password Flow

```
User clicks "Add Password" in popup
    │
    ▼
popup.js: Shows modal
    │
    ▼
User enters: service, username, password
    │
    ▼
popup.js: addPassword() → API POST /add
    │
    ▼
api.py: add_password() endpoint
    │
    ├─► @login_required checks session
    │
    ├─► Validate input
    │
    ├─► get_current_user() → User from session
    │
    ├─► Password.encrypt_password() → Fernet encryption
    │
    ├─► Save to database
    │
    └─► Return success
    │
    ▼
popup.js: Shows notification, refreshes list
```

### 3. Auto-Fill Flow

```
User visits login page
    │
    ▼
content.js: detectLoginForms()
    │
    ├─► Find <form> elements
    │
    ├─► Find password fields
    │
    ├─► Find username fields
    │
    └─► Add "Auto-fill" button
    │
    ▼
User clicks "Auto-fill" button
    │
    ▼
content.js: autofillForm()
    │
    ├─► Extract current domain
    │
    ├─► API GET /get-all-passwords
    │
    ├─► Filter by domain match
    │
    ├─► Show credential selector (if multiple)
    │
    └─► Fill username & password fields
    │
    ▼
Credentials filled! User can login
```

## Security Flow

### Password Storage

```
Plain Text Password
    │
    ▼
User enters: "MySecurePass123"
    │
    ▼
[Frontend] → Send via HTTPS
    │
    ▼
[Backend] Password.encrypt_password()
    │
    ▼
Fernet.encrypt() using SECRET_KEY
    │
    ▼
Encrypted: "gAAAAABk1x2y..."
    │
    ▼
Store in database (encrypted)
```

### Password Retrieval

```
User requests password
    │
    ▼
[Backend] Query encrypted password
    │
    ▼
Password.decrypt_password()
    │
    ▼
Fernet.decrypt() using SECRET_KEY
    │
    ▼
Return plain text (only to authenticated user)
    │
    ▼
[Frontend] Display or auto-fill
```

### User Authentication

```
User Login Attempt
    │
    ▼
User enters: username + password
    │
    ▼
[Backend] User.query.filter_by(username)
    │
    ▼
User.check_password(password)
    │
    ▼
Werkzeug: check_password_hash(stored_hash, password)
    │
    ├─► Match: Create session
    │
    └─► No match: Return error 401
```

## Data Flow Example

### Complete "Add & Retrieve Password" Journey

```
┌──────────────────────────────────────────────────┐
│ USER ACTION: Add Facebook password               │
└──────────────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────┐
│ EXTENSION: popup.js                              │
│ {                                                 │
│   service: "Facebook",                           │
│   username: "user@email.com",                    │
│   password: "SecurePass123"                      │
│ }                                                 │
└──────────────────────┬───────────────────────────┘
                       │ POST /add
                       ▼
┌──────────────────────────────────────────────────┐
│ API: @login_required                             │
│ ✓ Session valid                                  │
│ ✓ User authenticated                             │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ API: Validation                                  │
│ ✓ All fields present                             │
│ ✓ Service: "Facebook" ✓                         │
│ ✓ Username: "user@email.com" ✓                  │
│ ✓ Password: "SecurePass123" ✓                   │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ API: Encryption                                  │
│ Fernet.encrypt("SecurePass123")                  │
│ → "gAAAAABk1x2yRandom..."                        │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ DATABASE: Insert                                 │
│ INSERT INTO passwords VALUES (                   │
│   user_id: 1,                                    │
│   service: "Facebook",                           │
│   username: "user@email.com",                    │
│   password_encrypted: "gAAAAABk1x2yRandom..."    │
│ )                                                 │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ API: Response                                    │
│ {success: "Password added successfully"}         │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ EXTENSION: UI Update                             │
│ • Show success notification                      │
│ • Refresh password list                          │
│ • Close modal                                    │
└──────────────────────────────────────────────────┘

...Later...

┌──────────────────────────────────────────────────┐
│ USER: Visit facebook.com                         │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ CONTENT SCRIPT: Detect login form                │
│ • Found: <input type="email">                    │
│ • Found: <input type="password">                 │
│ • Add auto-fill button                           │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ USER: Click auto-fill button                     │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ CONTENT SCRIPT: Request credentials              │
│ GET /get-all-passwords                           │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ API: Retrieve & Decrypt                          │
│ SELECT * FROM passwords WHERE user_id = 1        │
│ → Decrypt each password                          │
│ → Return JSON array                              │
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ CONTENT SCRIPT: Filter & Fill                    │
│ • Filter: domain matches "facebook"              │
│ • Found: user@email.com                          │
│ • Fill username field: "user@email.com"          │
│ • Fill password field: "SecurePass123"           │
│ • Show notification: "Credentials filled!"       │
└──────────────────────────────────────────────────┘
```

## Technology Stack

### Frontend (Browser Extension)

- **HTML5** - Popup UI structure
- **CSS3** - Modern styling with gradients and animations
- **JavaScript (ES6+)** - Async/await, modules
- **Web Extensions API** - Chrome/Firefox compatibility

### Backend (API)

- **Flask 3.1** - Web framework
- **Flask-SQLAlchemy** - ORM
- **Flask-CORS** - Cross-origin support
- **Werkzeug** - Password hashing (PBKDF2)
- **Cryptography (Fernet)** - Symmetric encryption

### Database

- **SQLite** - Development (default)
- **PostgreSQL** - Production (optional)

### Security

- **PBKDF2 SHA256** - User password hashing
- **Fernet (AES-128)** - Password encryption
- **Session-based auth** - Secure cookies
- **CORS** - Origin validation
- **HTTPS** - Production requirement

---

This architecture ensures:
✅ **Separation of Concerns** - Clean layers  
✅ **Security** - Multiple layers of protection  
✅ **Scalability** - Easy to extend  
✅ **Maintainability** - Clear structure  
✅ **User Experience** - Fast and intuitive
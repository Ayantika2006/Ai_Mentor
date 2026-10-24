# AI Mentor — Security & Engineering Update Plan

**Review snapshot:** `main` branch, commit `3f6003a` (23 Oct 2026)  
**Review type:** Static code review of payments, authentication, AI access, uploads, CI and production setup. Live deployment, production secrets and gateway configuration were not tested.

## Executive summary

Project mein CI, tests, linting, formatting, environment examples, Gitleaks secret scanning, security headers aur database TLS jaise achhe foundations pehle se hain. Isliye basic test/CI setup scratch se banane ki zaroorat nahi hai.

**Live paid launch se pehle sabse pehle fix karein:**

1. Payment amount client se trust na ho.
2. Stripe webhook app mein mount ho aur raw request body use kare.
3. AI video/transcript ke har read par course entitlement check ho.
4. Avatar uploads par strict type/size validation ho.
5. Validation failures par passwords ya request bodies logs mein na jaayein.

---

## P0 / P1 — Launch blockers

### 1. Stripe aur Razorpay price client request se le rahe hain

**Problem:** Checkout/order amount `req.body.course.priceValue` se calculate hota hai. Client request modify karke course ki asli database price se kam payment kar sakta hai. Razorpay signature check isko nahi rokta, kyunki order backend khud client ke bheje hue kam amount par bana raha hai.

**Files update karein:**

- `backend/routes/payment.js`
- `backend/routes/razorpay.js`
- `backend/models/Payment.js` — agar idempotency key scope/index badalna ho
- `backend/tests/paymentRoutes.test.js` — existing tests ko extend karein

**Changes:**

- Request se sirf `courseId` accept karein; `Course` ko database se load karein.
- Course missing, unpublished/disabled ya deleted ho to checkout reject karein.
- Title, currency aur amount server-side course record se lein; amount ko sahi minor units (paise) mein calculate karein.
- Razorpay verification mein gateway se payment/order ka captured/paid status, amount aur currency database record ke saath verify karein.
- Stripe idempotency lookup mein `userId` bhi check karein; ek user ka cached checkout response doosre user ko return na ho.

**Done jab:** client `priceValue`, `title` ya currency badal kar bhi payment amount ko change nahi kar sakta; altered-price, unpublished-course aur duplicate-key tests pass hon.

### 2. Stripe webhook production app mein mount nahi hai

**Problem:** `backend/routes/webhook.js` handler hai, lekin `backend/server.js` mein import/mount nahi dikh raha. Current app par Stripe webhook 404 ho sakta hai. Saath hi `express.json()` webhook routes se pehle chalta hai; Stripe signature verification ko raw body chahiye.

**Files update karein:**

- `backend/server.js`
- `backend/routes/webhook.js`
- `backend/tests/webhook.test.js`

**Changes:**

- Webhook router ko `express.json()` se **pehle** mount karein, jaise `/api/stripe/webhook`.
- Webhook ko matching `Payment` record ke bina enrollment nahi deni chahiye.
- Session ka paid status, amount, currency, user aur course ko stored payment record se match karein; event processing idempotent rakhein.
- Existing webhook test ko actual exported Express app ke against chalayein. Abhi isolated Express app mein router mount karke test karne se production route wiring test nahi hoti.

**Done jab:** valid signed test webhook payment ko ek baar success mark kare aur enrollment de; invalid signature, missing payment record ya wrong amount par enrollment na ho.

### 3. AI transcripts/status mein course entitlement enforce nahi hota

**Problem:** `/api/ai/transcript/:filename` route par auth middleware nahi hai. `/api/ai/status/:jobId` login maangta hai, lekin user ke course purchase ko check kiye bina completed video URL return karta hai. Cloudinary URL public delivery mode mein ho sakta hai. `generate-video` mein bhi verify karein ki requested lesson usi course se belong karta hai.

**Files update karein:**

- `backend/routes/aiRoutes.js`
- `backend/controllers/courseController.js`
- `ai_service/backend/api.py` — agar AI service ko non-local network par deploy karna hai
- `backend/queues/videoQueue.js` — internal service auth header add karne ke liye, agar AI service ko expose kiya jaaye
- `backend/tests/videoRoute.test.js` aur naya `backend/tests/aiAuthorization.test.js`

**Changes:**

- Transcript, status aur video read ke liye ek shared course-access check banayein; user enrolled/purchased ho, ya explicitly allowed admin ho.
- Verify karein `lessonId` requested `courseId` ke andar hi hai.
- Paid content ke liye Cloudinary signed/short-lived URLs ya authenticated proxy use karein; permanent public URL directly na dein.
- Python AI service ka internal API auth/private-network boundary define karein. Local runner `127.0.0.1` par bind karta hai—production mein is isolation ko maintain karein; wildcard CORS ko restrict karein.

**Done jab:** non-enrolled user transcript/status/video access nahi kar sake; same-course enrolled user access kar sake; cross-course lesson IDs reject hon.

### 4. `₹1` course ko free access mil sakta hai

**Problem:** `backend/controllers/courseController.js` mein `priceValue <= 1` ko free treat kiya gaya hai, jabki purchase logic `priceValue > 0` ko paid maanta hai. Agar ₹1 course paid hai, learning endpoint usay bina purchase khol sakta hai.

**Files update karein:**

- `backend/controllers/courseController.js`
- `backend/tests/courseController.test.js`

**Changes:** Free-course rule ko `priceValue === 0` (ya documented free flag) par consistent banayein; course access aur checkout mein ek hi rule use ho.

**Done jab:** har positive-price course—₹1 samet—payment/enrollment ke bina protected ho.

### 5. Avatar upload unrestricted hai

**Problem:** Multer setup mein file-size limit ya file-type allowlist nahi hai. Authenticated user oversized file upload karke disk bhar sakta hai; arbitrary `.html`/`.svg` public static path se serve ho sakti hai.

**Files update karein:**

- `backend/routes/userRoutes.js`
- `backend/server.js` — zaroorat ho to public upload serving/header policy
- Naya `backend/tests/avatarUpload.test.js`

**Changes:**

- `limits.fileSize` set karein; sirf supported image types allow karein.
- MIME type ke saath file signature/magic bytes validate karein; sirf extension par bharosa na karein.
- Random server-generated filename use karein; active HTML/SVG ko public serving se block karein.
- Rejected ya failed profile update par partially uploaded file remove karein.

**Done jab:** oversized, fake-MIME aur non-image upload reject ho; valid image upload ka expected flow pass ho.

### 6. Validation failure logs mein password aa sakta hai

**Problem:** `backend/middleware/validate.js` invalid request par poora `req.body` log karta hai. Login/register/password inputs validation fail hon to plaintext password logs mein aa sakta hai. `backend/server.js` ka global error handler bhi internal `err.message` client ko bhej sakta hai.

**Files update karein:**

- `backend/middleware/validate.js`
- `backend/server.js`
- Naya `backend/tests/validationLogging.test.js` (ya existing auth tests mein coverage)

**Changes:**

- Full request body log karna hata dein; sirf request ID, route aur validation error paths log karein.
- Password, token, reset token, API key aur payment identifiers redact karein.
- Unknown 500 errors par generic response dein; details sirf redacted server logs mein rakhein.

**Done jab:** invalid login/register request ke baad captured logs mein password/request body na aaye aur client ko internal stack/database detail na mile.

---

## P2 — Agle remediation sprint mein

### 7. Auth rate-limit bypass aur admin login hardening

**Problem:** `/api/auth/login` aur `/api/auth/register` par rate limit hai, lekin duplicate `/api/users/login` aur `/api/users/register` routes par nahi. Admin login route par bhi rate-limit middleware nahi dikhta. AI generation/explain endpoints par per-user cost quota bhi add karein.

**Files update karein:**

- `backend/routes/auth.js`
- `backend/routes/userRoutes.js`
- `backend/middleware/rateLimiter.js`
- `backendAdmin/routes/adminRoutes.js`
- `backendAdmin/package.json` — agar application-level rate-limit dependency add ho
- Relevant auth/rate-limit tests

**Changes:** Duplicate public auth endpoints remove karein ya same limiter use karwayein. Admin login aur costly AI endpoints par shared-store limiter (multi-instance production mein Redis/API gateway) aur per-user quota lagayein.

### 8. Password change/logout ke baad active token revoke nahi hota

**Problem:** User `changePassword` flow `passwordChangedAt` update nahi karta, jabki auth middleware purane tokens ko isi timestamp se revoke karta hai. Admin token 30 din valid hai aur logout endpoint client ko token delete karne ko kehta hai; server-side revoke nahi karta. Frontend bearer token `localStorage` mein rakhta hai.

**Files update karein:**

- `backend/controllers/userController.js`
- `backend/middleware/authMiddleware.js`
- `backendAdmin/controllers/authController.js`
- `backendAdmin/middleware/adminAuthMiddleware.js`
- `frontend/src/context/AuthContext.jsx`
- `frontendAdmin/src/pages/LoginPage.jsx` / admin token handling

**Changes:** Password change/reset par purane sessions revoke karein. Admin ke liye token version/JTI/session table ya equivalent revocation mechanism add karein. Short-lived access token aur refresh rotation consider karein; `HttpOnly; Secure; SameSite` cookie approach use ho to CSRF protection bhi add karein. Admin/superadmin ke liye MFA flow implement karein.

**Note:** Settings screen par “Two-Factor Authentication” toggle dikh raha hai, lekin actual second-factor login flow implement na ho to ise “coming soon” mark ya remove karein—sirf setting save hona 2FA nahi hai.

### 9. Frontend Vitest tests CI mein run nahi hote

**Problem:** `frontend/package.json` ka `npm test` sirf `test/*.test.js` chalata hai. `src/__tests__` ke component tests `test:vitest` script se hain, lekin CI `npm test` chalata hai.

**Files update karein:**

- `frontend/package.json`
- Root `package.json`
- `.github/workflows/ci.yml`

**Changes:** CI mein `npm run test:vitest` bhi chalayein; ideally root `npm test` sab intended suites run kare. Coverage report/threshold add karein, khaaskar payments, course access aur admin authorization ke liye.

### 10. Dependency scanning aur Python dependency pinning

**Problem:** `ai_service/backend/requirements.txt` mein dependency versions pin nahi hain. Repo workflows mein automated dependency vulnerability scan/Dependabot config nahi dikha. GitHub Actions major tags par hain, immutable SHAs par nahi.

**Files update/create karein:**

- `ai_service/backend/requirements.txt` ko pin karein; preferred tool ke hisaab se naya `uv.lock` ya `requirements.lock` add karein.
- Nayi `.github/dependabot.yml`
- Naya `.github/workflows/security.yml` (ya existing CI mein SCA jobs)
- `.github/workflows/ci.yml` aur `.github/workflows/code-review.yml`

**Changes:** `pip-audit`/OSV aur npm dependency audit add karein; high-severity finding par CI fail ho. GitHub Actions ko reviewed commit SHA par pin karein. Gitleaks scan ko retain karein.

### 11. Production schema migrations runtime startup se alag hon

**Problem:** Backend startup par `sequelize.sync()` chalta hai; migration files hain, lekin package/CI mein migration-run command visible nahi. Runtime app ko schema DDL permissions dena production blast radius badha sakta hai.

**Files update/create karein:**

- `backend/server.js`
- `backend/package.json`
- `backend/migrations/<timestamp>-<change>.js`
- `.github/workflows/ci.yml` ya deployment workflow

**Changes:** Production deploy step mein versioned migrations run karein; app startup par schema sync na karein. Development-only auto sync ko alag rakhein. Database migration aur app runtime credentials alag rakhna consider karein.

### 12. AI local fallback video URL path mismatch

**Problem:** `backend/routes/aiRoutes.js` local URL ko `/api/ai/video/{courseId}/{jobId}/{filename}` banata hai, jabki route `/api/ai/video/:courseId/:filename` expect karta hai. Fallback URL route se match nahi hota.

**Files update karein:**

- `backend/routes/aiRoutes.js`
- `backend/tests/videoRoute.test.js` ya naya status/fallback test

**Done jab:** Cloudinary unavailable hone par generated local video route se successfully stream ho aur sirf enrolled user access kar sake.

---

## Suggested new files

Har item zaroori nahi—existing tests ko extend karna pehli preference hai.

- `.github/dependabot.yml` — npm, Python aur GitHub Actions dependency updates.
- `.github/workflows/security.yml` — dependency audit/SCA jobs, agar CI se alag rakhna ho.
- `SECURITY.md` — vulnerability report karne ka formal process; `CONTRIBUTING.md` ko link karein.
- `backend/tests/aiAuthorization.test.js` — transcript/status/video authorization aur cross-course tests.
- `backend/tests/avatarUpload.test.js` — file-size/type validation tests.
- `backend/tests/webhookAppIntegration.test.js` — actual app par Stripe webhook route/raw-body test; existing webhook tests ko yahan move/extend bhi kar sakte hain.
- `backend/migrations/<timestamp>-<change>.js` — schema updates; change ke mutabiq.
- `uv.lock` ya `requirements.lock` — Python dependency versions/hash lock.
- `backend/services/courseAccess.js` — agar repeated course-entitlement logic ko shared helper banana ho.

## Acceptance checklist

- [x] Client price badal kar underpayment nahi kar sakta.
- [x] Stripe webhook actual app par signed raw-body ke saath process hota hai; repeated event duplicate enrollment nahi karta.
- [x] Non-enrolled users AI transcript/status/video nahi dekh sakte.
- [x] `lessonId` hamesha requested course ka hi hota hai.
- [x] Avatar upload size/type restrictions aur safe serving active hain.
- [x] Invalid requests ke logs mein password/token nahi aate.
- [x] User/admin login aur AI endpoints par effective rate limits/quotas hain.
- [x] Frontend component tests CI mein run hote hain.
- [x] Python dependencies pinned/audited hain; production DB changes migrations se apply hote hain.

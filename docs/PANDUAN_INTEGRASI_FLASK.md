# Panduan Integrasi Frontend → Flask
**VeloraSec — Persiapan Phase 11 (Integrasi Frontend-Backend)**

---

## File Baru yang Dibuat

| File | Deskripsi |
|---|---|
| [`assets/js/config.js`](../assets/js/config.js) | Konfigurasi terpusat (API URL, keys, mode) |
| [`assets/js/token.js`](../assets/js/token.js) | JWT token & session manager |
| [`assets/js/api.js`](../assets/js/api.js) | Service layer placeholder (semua endpoint Flask) |

---

## Load Order di HTML

Ketiga file ini **harus dimuat** sebelum `script.js` dan `auth.js`.
Tambahkan ke setiap halaman yang membutuhkan API:

```html
<!-- Di dalam <head> atau sebelum </body> -->
<script src="../../assets/js/config.js"></script>
<script src="../../assets/js/token.js"></script>
<script src="../../assets/js/api.js"></script>

<!-- Setelah ketiga file di atas: -->
<script src="../../assets/js/script.js"></script>
```

Untuk halaman auth (login/register), path-nya berbeda:
```html
<script src="../../assets/js/config.js"></script>
<script src="../../assets/js/token.js"></script>
<script src="../../assets/js/api.js"></script>
<script src="auth.js"></script>
```

---

## Cara Menggunakan

### Memanggil API

Semua fungsi API tersedia via `window.VeloraSec.API`:

```javascript
// Login
const result = await VeloraSec.API.Auth.login(email, password);

// Ambil progress user
const progress = await VeloraSec.API.Progress.getAll();

// Simpan hasil quiz
await VeloraSec.API.Quiz.saveResult('Network', 8, 10);

// Update planner task
await VeloraSec.API.Planner.updateTask('w0d0', true);
```

### Mengecek Status Login

```javascript
if (VeloraSec.Token.isLoggedIn()) {
  // User sudah login
  const user = VeloraSec.Session.get();
  console.log('Logged in as:', user.username);
}

if (VeloraSec.Token.isExpired()) {
  // Token expired — perlu refresh atau re-login
  await VeloraSec.API.Auth.refreshToken();
}
```

### Error Handling

```javascript
try {
  await VeloraSec.API.Auth.login(email, password);
} catch (err) {
  if (err.name === 'ApiError') {
    // Error dari server Flask
    if (err.status === 401) { /* credentials salah */ }
    if (err.status === 429) { /* rate limited */ }
    if (err.status === 408) { /* timeout */ }
    console.error(err.message);
  } else {
    // Network error (offline, CORS, dst.)
    console.error('Network error:', err);
  }
}
```

---

## Cara Migrasi ke Live Backend (Phase 11)

### Step 1: Update `config.js`

```javascript
// config.js — ubah dua baris ini:
API_BASE_URL: 'http://localhost:5000',  // ← URL Flask server kamu
DEMO_MODE: false,                        // ← matikan demo mode
```

### Step 2: Implementasikan fungsi di `api.js`

Cari setiap fungsi yang berisi `_notImplemented(...)` dan ganti dengan implementasi nyata.

Contoh untuk `AuthAPI.login()`:
```javascript
// SEBELUM (stub):
async function login(email, password) {
  return _notImplemented('AuthAPI.login');
}

// SESUDAH (implementasi):
async function login(email, password) {
  const data = await _request('/api/auth/login', {
    method: 'POST',
    body: JSON.stringify({ email, password }),
  });
  TokenManager.save(data.access_token, data.refresh_token);
  SessionManager.save(data.user);
  return data;
}
```

### Step 3: Update `auth.js`

Ganti `handleLogin()` dan `handleRegister()` untuk memanggil `VeloraSec.API.Auth`:

```javascript
// auth.js — handleLogin() yang diupdate:
async function handleLogin() {
  const email = document.getElementById('login-email')?.value.trim();
  const pw    = document.getElementById('login-pw')?.value;

  if (!email || !pw) { /* ... validasi ... */ }

  showAuthMsg('login-msg', 'info', 'Authenticating...');
  try {
    await VeloraSec.API.Auth.login(email, pw);
    showAuthMsg('login-msg', 'success', 'Login successful! Redirecting...');
    setTimeout(() => { window.location.href = '../index.html#dashboard'; }, 1200);
  } catch (err) {
    const msg = err.status === 401
      ? 'Email atau password salah.'
      : err.message || 'Login gagal. Coba lagi.';
    showAuthMsg('login-msg', 'error', msg);
  }
}
```

### Step 4: Update `script.js` — `buildDashboard()`

Ganti data hardcoded dengan API call:

```javascript
// Sebelum: data hardcoded
// Sesudah:
async function buildDashboard() {
  const el = document.getElementById('sec-dashboard');
  el.innerHTML = '<p>Loading...</p>';
  try {
    const data = await VeloraSec.API.Dashboard.getSummary();
    // render dengan data nyata dari API
    el.innerHTML = buildDashboardHTML(data);
  } catch {
    // fallback ke data hardcoded jika API gagal
    el.innerHTML = buildDashboardHTMLFallback();
  }
}
```

---

## Endpoint Flask yang Harus Dibuat (Checklist Phase 7-9)

### Auth Routes (`backend/routes/auth.py`)

```
POST   /api/auth/register     → daftarkan user baru
POST   /api/auth/login        → login, return JWT token
POST   /api/auth/refresh      → refresh access token
POST   /api/auth/logout       → (opsional) blacklist token
```

### User Routes (`backend/routes/users.py`)

```
GET    /api/users/me          → profil user saat ini
PUT    /api/users/me          → update profil
```

### Progress Routes (`backend/routes/progress.py`)

```
GET    /api/progress          → semua progress user
POST   /api/progress          → update progress modul
DELETE /api/progress          → reset semua progress
```

### Quiz Routes (`backend/routes/quiz.py`)

```
GET    /api/quiz/results      → riwayat hasil quiz
POST   /api/quiz/results      → simpan hasil quiz baru
```

### Planner Routes (`backend/routes/planner.py`)

```
GET    /api/planner           → semua task planner user
POST   /api/planner/:taskKey  → toggle task selesai/belum
DELETE /api/planner           → reset semua planner
```

### Dashboard Routes (`backend/routes/dashboard.py`)

```
GET    /api/dashboard/summary → aggregated dashboard data
```

---

## Format Response Flask yang Diharapkan

### Login Success (`POST /api/auth/login`)

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "username": "operator_x",
    "email": "operator@velorasec.io",
    "created_at": "2026-07-21T00:00:00Z"
  }
}
```

### Error Response (semua endpoint)

```json
{
  "error": "Unauthorized",
  "message": "Invalid email or password.",
  "status": 401
}
```

### Progress Response (`GET /api/progress`)

```json
{
  "progress": [
    { "module_id": "net-1", "is_completed": true,  "completed_at": "2026-07-20T10:00:00Z" },
    { "module_id": "net-2", "is_completed": false, "completed_at": null }
  ]
}
```

---

*Dokumen ini adalah panduan teknis untuk Phase 11 — Integrasi Frontend-Backend VeloraSec.*
*Lihat [AUDIT_VELORASEC.md](./AUDIT_VELORASEC.md) untuk roadmap lengkap.*

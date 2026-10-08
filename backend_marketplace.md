# Merchant API Documentation — Ovisito

Dokumentasi lengkap Merchant API untuk integrasi multi-platform (Web, Android, iOS).

**Versi:** 2.6 · **Terakhir diupdate:** 2026-10-08

**Base URL API:** https://api.ovisito.com/api/v2

**Web Dashboard:** https://merchant.ovisito.com

## Daftar Isi

1. Overview
2. Authentication
3. Common Conventions
4. Error Handling
5. Endpoints — Auth
6. Endpoints — Profile
7. Endpoints — Dashboard
8. Endpoints — Store
9. Endpoints — Categories (Read-only)
10. Endpoints — Products
11. Endpoints — Orders
12. Endpoints — Transactions & Withdraw
13. Order Lifecycle & Business Rules
14. Constants Reference
15. Integration Guide per Platform
16. Sample Flows (End-to-End)
17. Changelog

## 1. Overview

### 1.1 Role & Akses

| Role | Header Tambahan | Token |
|---|---|---|
| Public (browsing) | X-Client-ID, X-Client-Secret | Tidak perlu |
| Merchant | X-Client-ID, X-Client-Secret + Authorization: Bearer <token> | Sanctum token |

Dokumen ini fokus ke Merchant API.

### 1.2 Base URL

| Environment | URL |
|---|---|
| Production | https://api.ovisito.com/api/v2 |
| Staging | https://staging-api.ovisito.com/api/v2 |
| Local | http://localhost:8000/api/v2 |

### 1.3 Client Credentials

Setiap request (kecuali login admin) wajib sertakan:

```text
X-Client-ID: client_web
X-Client-Secret: <secret_dari_backend_team>
Accept: application/json
```

**Catatan untuk mobile:**

- JANGAN hardcode `client_secret` di source code mobile (bisa di-reverse engineer)
- Gunakan environment variable atau secure config (Android: `local.properties`; iOS: `xcconfig`)
- Idealnya: mobile dapat token via backend proxy atau OAuth flow terpisah (roadmap v2.7)

### 1.4 Content-Type

| Endpoint | Content-Type |
|---|---|
| Login, register, forgot, reset, logout | application/json |
| Profile update | application/json |
| Store create/update | multipart/form-data |
| Product create/update | multipart/form-data |
| Product stock/toggle | application/json |
| Order status/ship | application/json atau application/x-www-form-urlencoded |
| Withdraw | application/json |

## 2. Authentication

### 2.1 Flow Overview

```text
┌─────────────────────────────────────────────────────────────┐
│  1. POST /merchant/register                                 │
│     → Merchant dibuat (status=pending, email belum verify)  │
│     → Email verifikasi terkirim                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  2. Klik link verifikasi di email                           │
│     GET /merchant/verify-email/{uuid}?hash=<sha1_email>     │
│     → email_verified_at ter-set                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  3. POST /merchant/login                                    │
│     → Dapat Sanctum token                                   │
│     → Simpan di secure storage                              │
│       (Web: session, Android: EncryptedSharedPreferences,   │
│        iOS: Keychain)                                       │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  4. Akses semua endpoint merchant                           │
│     Authorization: Bearer <token>                           │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  5. POST /merchant/logout                                   │
│     → Token di-revoke                                       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Token Management

| Aspek | Nilai |
|---|---|
| Tipe | Laravel Sanctum Personal Access Token |
| Format | `{id}\|{token}` — contoh: `123\|abcdefg...` |
| Berlaku | Selamanya (tidak ada expiry otomatis) |
| Revoke | Via `POST /merchant/logout` |
| Storage (Web) | Server-side session (bukan localStorage) |
| Storage (Android) | EncryptedSharedPreferences |
| Storage (iOS) | Keychain |
| Header | `Authorization: Bearer {token}` |

> ⚠️ **Mobile security:**
>
> - Jangan simpan token di plain SharedPreferences (Android) atau UserDefaults (iOS)
> - Jangan log token ke console / crash report
> - Rotate token jika user ganti password — sudah otomatis di backend (`changePassword` revoke semua token lain)

## 3. Common Conventions

### 3.1 Response Format — Sukses

Single object:

```json
{
  "status": true,
  "message": "Optional message",
  "data": {
    "id": "...",
    "name": "..."
  }
}
```

List / paginated:

```json
{
  "status": true,
  "data": [ ],
  "meta": {
    "current_page": 1,
    "last_page": 5,
    "per_page": 15,
    "total": 60
  }
}
```

### 3.2 Response Format — Error

```json
{
  "status": false,
  "message": "Pesan error yang jelas",
  "errors": {
    "field_name": ["Pesan error field"]
  }
}
```

`errors` hanya muncul untuk 422 Validation Error. Endpoint lain hanya punya `status` + `message`.

### 3.3 HTTP Status Codes

| Code | Arti | Kapan |
|---|---|---|
| 200 | OK | Request sukses |
| 201 | Created | Resource baru dibuat (register, store, product) |
| 400 | Bad Request | Request tidak valid (mis. merchant sudah punya toko) |
| 401 | Unauthorized | Token tidak ada / invalid / expired |
| 403 | Forbidden | Token valid tapi tidak punya akses (mis. email belum verify) |
| 404 | Not Found | Resource tidak ditemukan |
| 422 | Validation Error | Body request tidak lulus validasi |
| 429 | Too Many Requests | Rate limit terlampaui |
| 500 | Internal Server Error | Bug di backend |
| 502 | Bad Gateway | Upstream error (mis. KiriminAja, Flip timeout) |

### 3.4 Pagination

Query parameters:

| Param | Tipe | Default | Max |
|---|---|---|---|
| page | int | 1 | — |
| per_page | int | 15 | 50 (order/product), 60 (store) |

Response:

```json
{
  "status": true,
  "data": [],
  "meta": {
    "current_page": 1,
    "last_page": 5,
    "per_page": 15,
    "total": 60
  }
}
```

Cara build URL halaman berikutnya:

```text
GET /merchant/souvenir/orders?page=2&per_page=15
```

### 3.5 Format Data

| Data | Format | Contoh |
|---|---|---|
| Tanggal-waktu | ISO 8601 UTC | 2026-10-08T10:30:00.000000Z |
| Tanggal | YYYY-MM-DD | 2026-10-08 |
| Jam | HH:MM | 08:30 |
| Uang | String decimal | "50000.00" |
| Uang (formatted) | String Rupiah | "Rp 50.000" |
| Boolean | true / false | true |
| ID | UUID v4 (36 char) | 01a0e882-b4fe-737c-... |
| Slug | kebab-case | kopi-aceh-gayo |

### 3.6 Rate Limiting

| Endpoint | Limit |
|---|---|
| Semua endpoint | 120 req/menit (default) |
| POST /merchant/login | 5 req/menit |
| POST /merchant/register | 10 req/menit |
| POST /merchant/forgot-password | 3 req/menit |
| POST /merchant/reset-password | 5 req/menit |
| POST /merchant/withdraw | 10 req/menit |
| PUT /merchant/souvenir/orders/{id}/status | 60 req/menit |
| POST /merchant/souvenir/orders/{id}/ship | 60 req/menit |

Response saat rate limit terlampaui:

```json
HTTP 429
{
  "status": false,
  "message": "Too Many Attempts."
}
```

## 4. Error Handling

### 4.1 Pola Error yang Konsisten

Semua error dari backend menggunakan format:

```json
{
  "status": false,
  "message": "Pesan error"
}
```

### 4.2 Error Spesifik per Status Code

**401 — Token invalid / expired:**

```json
{
  "status": false,
  "message": "Unauthenticated. Silakan login terlebih dahulu."
}
```

Aksi di client: hapus token yang tersimpan, redirect ke halaman login.

**403 — Email belum verify:**

```json
{
  "status": false,
  "message": "Silakan verifikasi email Anda terlebih dahulu.",
  "data": { "code": "EMAIL_NOT_VERIFIED" }
}
```

Aksi di client: redirect ke halaman verifikasi email / tampilkan banner.

**404 — Order / product / store tidak ditemukan:**

```json
{
  "status": false,
  "message": "Pesanan tidak ditemukan."
}
```

**422 — Validation error:**

```json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "email": ["Email sudah terdaftar."],
    "phone": ["Nomor telepon sudah terdaftar."]
  }
}
```

Aksi di client: tampilkan error di field masing-masing.

**500 — Server error:**

```json
{
  "status": false,
  "message": "Terjadi kesalahan pada server."
}
```

Aksi di client: tampilkan pesan generik + tombol retry.

**502 — Upstream error:**

```json
{
  "status": false,
  "message": "Gagal membuat pengiriman via KiriminAja."
}
```

### 4.3 Handling per Platform

**Web (Laravel Blade):**

```php
$response = Http::withToken($token)->get($url);

if ($response->status() === 401) {
    Session::forget('merchant_token');
    return redirect()->route('merchant.login');
}
```

**Android (Kotlin + Retrofit):**

```kotlin
val response = api.getOrders()
if (response.code() == 401) {
    authRepository.clearToken()
    navigateToLogin()
}
```

**iOS (Swift + URLSession):**

```swift
if response.statusCode == 401 {
    KeychainService.deleteToken()
    navigateToLogin()
}
```

## 5. Endpoints — Auth

### 5.1 POST /merchant/register

**Deskripsi:** Registrasi merchant baru.

**Auth:** tidak perlu token · **Rate limit:** 10 req/menit · **Content-Type:** application/json

Request body:

```json
{
  "name": "Budi Santoso",
  "email": "budi@merchant.com",
  "password": "password123",
  "password_confirmation": "password123",
  "phone": "081234567890",
  "business_name": "Toko Kopi Aceh",
  "business_type": "souvenir",
  "address": "Jl. Cut Nyak Dhien No. 10",
  "city": "Banda Aceh",
  "province": "Aceh",
  "postal_code": "23116",
  "description": "Toko oleh-oleh khas Aceh",
  "website": "https://tokokopi-aceh.com",
  "category_id": null
}
```

Validation rules:

| Field | Tipe | Wajib | Deskripsi |
|---|---|---|---|
| name | string | ✅ | Max 255 |
| email | email | ✅ | Max 255, unik |
| password | string | ✅ | Min 8, harus ada password_confirmation |
| phone | string | ✅ | Max 20, unik |
| business_name | string | ✅ | Max 255 |
| business_type | enum | ✅ | hotel, kuliner, rental, tour, destinasi, souvenir |
| address | string | ✅ | — |
| city | string | ✅ | — |
| province | string | ✅ | — |
| postal_code | string | ❌ | Max 10 |
| description | string | ❌ | Max 500 |
| website | url | ❌ | Max 255 |
| category_id | uuid | ❌ | Harus ada di souvenir_categories |

Response 201:

```json
{
  "status": true,
  "message": "Registrasi berhasil. Silakan cek email untuk verifikasi.",
  "data": {
    "uuid": "01a0e882-b4fe-737c-8936-68152828b651",
    "email": "budi@merchant.com"
  }
}
```

Response 422 — Email sudah terdaftar:

```json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "email": ["Email sudah terdaftar."]
  }
}
```

**Catatan untuk mobile:**

- Setelah register sukses, arahkan user ke halaman "Cek email untuk verifikasi"
- Sediakan tombol "Kirim ulang email" — panggil `POST /merchant/resend-verification`

### 5.2 GET /merchant/verify-email/{uuid}

**Deskripsi:** Verifikasi email via link dari email.

**Auth:** tidak perlu token (link dari email)

Path parameter:

| Param | Tipe | Deskripsi |
|---|---|---|
| uuid | uuid | UUID merchant |

Query parameter:

| Param | Tipe | Deskripsi |
|---|---|---|
| hash | string | SHA-1 dari email merchant |

Contoh link di email:

```text
https://merchant.ovisito.com/verify-email/01a0e882-b4fe-737c-8936-68152828b651?hash=abc123...
```

Response 200:

```json
{
  "status": true,
  "message": "Email berhasil diverifikasi."
}
```

Response 403 — Hash tidak valid:

```json
{
  "status": false,
  "message": "Hash verifikasi tidak valid."
}
```

Response 404 — Merchant tidak ditemukan:

```json
{
  "status": false,
  "message": "Link verifikasi tidak valid."
}
```

**Catatan untuk mobile:**

- Deep link — registrasikan URL scheme: `ovisito-merchant://verify-email/{uuid}?hash=...`
- Backend bisa kirim link universal atau app-link
- Alternatif: register dengan email, lalu buka email di device, klik link web → web cek → redirect ke app via deep link

### 5.3 POST /merchant/resend-verification

**Deskripsi:** Kirim ulang email verifikasi.

**Rate limit:** 6 req/menit · **Content-Type:** application/json

Request body:

```json
{
  "email": "budi@merchant.com"
}
```

Response 200:

```json
{
  "status": true,
  "message": "Jika email terdaftar dan belum diverifikasi, link baru telah dikirim."
}
```

> ⚠️ Response selalu sukses (anti user-enumeration), bahkan jika email tidak terdaftar.

### 5.4 POST /merchant/login

**Deskripsi:** Login merchant, dapat token Sanctum.

**Rate limit:** 5 req/menit · **Content-Type:** application/json

Request body:

```json
{
  "email": "budi@merchant.com",
  "password": "password123",
  "device_name": "web-merchant"
}
```

`device_name` untuk mobile:

- Android: `"android-merchant"` atau `"android-{Build.MODEL}"`
- iOS: `"ios-merchant"` atau `"ios-{UIDevice.current.name}"`
- Web: `"web-merchant"`

Response 200:

```json
{
  "status": true,
  "message": "Login berhasil.",
  "data": {
    "token": "1|abcdefghijklmnopqrstuvwxyz1234567890",
    "merchant": {
      "uuid": "01a0e882-b4fe-737c-8936-68152828b651",
      "name": "Budi Santoso",
      "email": "budi@merchant.com",
      "phone": "081234567890",
      "business_name": "Toko Kopi Aceh",
      "business_type": "souvenir",
      "address": "Jl. Cut Nyak Dhien No. 10",
      "city": "Banda Aceh",
      "province": "Aceh",
      "category_id": null,
      "logo": null,
      "balance": 0,
      "status": "pending",
      "verified_status": "unverified",
      "partnership_type": "regular",
      "email_verified_at": "2026-10-08T10:00:00.000000Z"
    }
  }
}
```

Response 403 — Akun tidak aktif:

```json
{
  "status": false,
  "message": "Akun merchant tidak aktif."
}
```

Response 403 — Email belum verify:

```json
{
  "status": false,
  "message": "Silakan verifikasi email Anda terlebih dahulu.",
  "data": { "code": "EMAIL_NOT_VERIFIED" }
}
```

Response 422 — Password salah:

```json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "email": ["Email atau password salah."]
  }
}
```

**Catatan untuk mobile:**

- Simpan token di secure storage: Android `EncryptedSharedPreferences` atau Jetpack Security; iOS Keychain
- Setelah login, simpan juga merchant object untuk cache local (nama, business_name, dll)
- Handle `EMAIL_NOT_VERIFIED` → redirect ke layar verifikasi

### 5.5 POST /merchant/logout

**Deskripsi:** Revoke token aktif.

**Auth:** ✅ Bearer token · **Content-Type:** application/json

**Request:** kosong (token di header)

Response 200:

```json
{
  "status": true,
  "message": "Logout berhasil."
}
```

**Catatan:** setelah logout, hapus token + merchant data dari storage client.

### 5.6 POST /merchant/forgot-password

**Deskripsi:** Request link reset password via email.

**Rate limit:** 3 req/menit · **Content-Type:** application/json

Request body:

```json
{
  "email": "budi@merchant.com"
}
```

Response 200:

```json
{
  "status": true,
  "message": "Jika email terdaftar, link reset password telah dikirim."
}
```

> ⚠️ Response selalu sukses (anti user-enumeration).

### 5.7 POST /merchant/reset-password

**Deskripsi:** Reset password dengan token dari email.

**Rate limit:** 5 req/menit · **Content-Type:** application/json

Request body:

```json
{
  "token": "abcdefghij...",
  "email": "budi@merchant.com",
  "password": "newpassword456",
  "password_confirmation": "newpassword456"
}
```

Response 200:

```json
{
  "status": true,
  "message": "Password berhasil direset. Silakan login."
}
```

Response 422 — Token invalid/expired:

```json
{
  "status": false,
  "message": "Token reset tidak valid atau sudah kadaluarsa."
}
```

> ⚠️ **Setelah reset:**
>
> - Semua token Sanctum lama di-revoke
> - User harus login ulang di semua device

**Catatan untuk mobile:**

- Link reset di email → web page → user isi password → submit ke API
- Alternatif: deep link `ovisito-merchant://reset-password?token=...&email=...`

## 6. Endpoints — Profile

### 6.1 GET /merchant/profile

**Deskripsi:** Ambil data profil merchant.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": {
    "uuid": "01a0e882-b4fe-737c-8936-68152828b651",
    "name": "Budi Santoso",
    "email": "budi@merchant.com",
    "phone": "081234567890",
    "business_name": "Toko Kopi Aceh",
    "business_type": "souvenir",
    "address": "Jl. Cut Nyak Dhien No. 10",
    "city": "Banda Aceh",
    "province": "Aceh",
    "postal_code": "23116",
    "description": "Toko oleh-oleh khas Aceh",
    "website": "https://tokokopi-aceh.com",
    "logo": "merchants/01a0e882/logo.png",
    "status": "active",
    "verified_status": "verified",
    "partnership_type": "regular",
    "balance": 150000,
    "category_id": null,
    "email_verified_at": "2026-10-08T10:00:00.000000Z",
    "created_at": "2026-10-01T08:00:00.000000Z",
    "updated_at": "2026-10-08T10:30:00.000000Z"
  }
}
```

> ⚠️ Field `password` dan `remember_token` TIDAK di-expose.

### 6.2 PUT /merchant/profile

**Deskripsi:** Update profil merchant.

**Auth:** ✅ Bearer token · **Content-Type:** multipart/form-data (untuk upload logo) atau application/json

Request body:

| Field | Tipe | Wajib | Deskripsi |
|---|---|---|---|
| name | string | ❌ | Max 255 |
| phone | string | ❌ | Max 20 |
| business_name | string | ❌ | Max 255 |
| address | string | ❌ | — |
| city | string | ❌ | Max 100 |
| province | string | ❌ | Max 100 |
| postal_code | string | ❌ | Max 10 |
| description | string | ❌ | Max 1000 |
| website | url | ❌ | Max 255 |
| logo | file | ❌ | jpeg/png/jpg/webp, max 2 MB |

> ⚠️ Semua field `sometimes` — kirim hanya field yang mau diubah.

Response 200:

```json
{
  "status": true,
  "message": "Profil berhasil diperbarui.",
  "data": { }
}
```

`data` berisi object merchant yang sudah diperbarui.

**Catatan untuk mobile:**

- Untuk multipart, gunakan `MultipartBody.Part` (Retrofit) atau `multipartFormData` (URLSession)
- Logo lama otomatis dihapus kalau upload logo baru

### 6.3 PUT /merchant/change-password

**Deskripsi:** Ganti password.

**Auth:** ✅ Bearer token · **Content-Type:** application/json

Request body:

```json
{
  "current_password": "password123",
  "new_password": "newpassword456",
  "new_password_confirmation": "newpassword456"
}
```

Rules:

- `new_password` minimal 8 karakter
- Harus ada `new_password_confirmation` yang sama
- Password baru ≠ password lama

Response 200:

```json
{
  "status": true,
  "message": "Password berhasil diubah."
}
```

Response 422 — Password lama salah:

```json
{
  "status": false,
  "message": "Password saat ini salah."
}
```

> ⚠️ **Setelah ganti password:**
>
> - Semua token LAIN di-revoke (kecuali token yang dipakai request ini)
> - Device lain otomatis logout

**Catatan untuk mobile:** setelah sukses, tetap gunakan token yang sama. Kalau ingin revoke semua termasuk device ini, panggil logout setelahnya.

## 7. Endpoints — Dashboard

### 7.1 GET /merchant/dashboard

**Deskripsi:** Dashboard summary — stats, chart, recent orders.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": {
    "summary": {
      "total_orders": 42,
      "total_revenue": 5250000,
      "formatted_revenue": "Rp 5.250.000",
      "total_products": 15,
      "active_products": 12,
      "out_of_stock": 2,
      "low_stock": 3,
      "balance": 150000,
      "formatted_balance": "Rp 150.000"
    },
    "order_status": {
      "pending": 3,
      "paid": 1,
      "processing": 2,
      "shipped": 1,
      "completed": 34,
      "cancelled": 1
    },
    "recent_orders": [
      {
        "order_number": "SO-20261008-ABC123XY",
        "total_amount": 135000,
        "formatted_total": "Rp 135.000",
        "status": "paid",
        "status_label": "Dibayar",
        "status_badge": "bg-blue-100 text-blue-800",
        "customer_name": "Budi",
        "ordered_at": "2026-10-08T10:30:00.000000Z",
        "items_count": 2
      }
    ],
    "sales_chart": [
      {
        "date": "2026-10-02",
        "label": "Wed",
        "total_orders": 3,
        "total_revenue": 350000
      }
    ]
  }
}
```

`sales_chart` berisi 7 entri (satu per hari, 7 hari terakhir); contoh di atas dipersingkat.

**Catatan untuk mobile:**

- Chart data (7 hari terakhir) dalam format array
- `label` adalah nama hari singkat (Mon, Tue, ...)
- FE mobile render bar chart / line chart dari `sales_chart`

### 7.2 GET /merchant/dashboard/summary

**Deskripsi:** Ringkasan cepat untuk widget dashboard.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": {
    "today_orders": 5,
    "today_revenue": 675000,
    "formatted_today_revenue": "Rp 675.000",
    "week_orders": 28,
    "pending_orders": 3,
    "balance": 150000,
    "formatted_balance": "Rp 150.000"
  }
}
```

## 8. Endpoints — Store

### 8.1 GET /merchant/store

**Deskripsi:** Ambil toko default merchant.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": {
    "id": "01a0e882-b4fe-737c-8936-68152828b651",
    "merchant_uuid": "44cdb641-7787-4c2c-90e8-8cb561b8bb61",
    "name": "Toko Kopi Aceh",
    "slug": "toko-kopi-aceh",
    "description": "Toko oleh-oleh khas Aceh",
    "address": "Jl. Cut Nyak Dhien No. 10",
    "phone": "081234567890",
    "email": "tokokopi@example.com",
    "website": "https://tokokopi-aceh.com",
    "logo": "stores/01a0e882/logo.png",
    "is_physical": true,
    "is_active": true,
    "is_default": true,
    "kabupaten_kota_id": 1101,
    "kecamatan_id": null,
    "desa_id": null,
    "latitude": 5.5483,
    "longitude": 95.3238,
    "jam_buka": "08:00",
    "jam_tutup": "17:00",
    "category_ids": ["uuid1", "uuid2"],
    "categories": [
      { "id": "uuid1", "name": "Makanan" },
      { "id": "uuid2", "name": "Minuman" }
    ],
    "created_at": "2026-10-01T08:00:00.000000Z",
    "updated_at": "2026-10-08T10:30:00.000000Z"
  }
}
```

Response 404 — Belum punya toko:

```json
{
  "status": false,
  "message": "Toko tidak ditemukan.",
  "data": null
}
```

### 8.2 POST /merchant/store

**Deskripsi:** Buat toko baru. Satu merchant hanya boleh punya 1 toko.

**Auth:** ✅ Bearer token · **Content-Type:** multipart/form-data

Request body:

| Field | Tipe | Wajib | Deskripsi |
|---|---|---|---|
| name | string | ✅ | Max 255 |
| address | string | ✅ | — |
| phone | string | ✅ | Max 20 |
| category_ids[] | array uuid | ✅ | 1–5 UUID kategori |
| description | string | ❌ | — |
| email | email | ❌ | Max 255 |
| website | url | ❌ | Max 255 |
| is_physical | bool | ❌ | Default true |
| logo | file | ❌ | jpeg/png/jpg/webp, max 2 MB |
| kabupaten_kota_id | int | ❌ | — |
| kecamatan_id | int | ❌ | — |
| desa_id | int | ❌ | — |
| latitude | numeric | ❌ | -90 s/d 90 |
| longitude | numeric | ❌ | -180 s/d 180 |
| jam_buka | string | ❌ | Format HH:MM |
| jam_tutup | string | ❌ | Format HH:MM |

Response 201:

```json
{
  "status": true,
  "message": "Toko berhasil dibuat.",
  "data": { }
}
```

`data` berisi object store lengkap.

Response 422 — Sudah punya toko:

```json
{
  "status": false,
  "message": "Merchant sudah memiliki toko."
}
```

**Catatan untuk mobile:**

- Multipart upload untuk logo
- Array `category_ids[]` — di Android Retrofit pakai `@Part("category_ids[]")` atau FormData
- Di iOS URLSession, kirim sebagai multipart form fields dengan key `category_ids[]`

### 8.3 PUT /merchant/store

**Deskripsi:** Update toko default.

**Auth:** ✅ Bearer token · **Content-Type:** multipart/form-data

**Request body:** sama seperti `POST /merchant/store`, semua field `sometimes`.

Response 200:

```json
{
  "status": true,
  "message": "Toko berhasil diperbarui.",
  "data": { }
}
```

**Catatan:**

- Kalau `name` diubah → slug otomatis regenerate (unique)
- Field yang di-set `null` benar-benar di-null (mis. `{"website": null}`)

### 8.4 GET /merchant/store/{uuid}

**Deskripsi:** Detail toko by UUID.

**Auth:** ✅ Bearer token

Response 200: sama struktur dengan `GET /merchant/store`.

## 9. Endpoints — Categories (Read-only)

### 9.1 GET /merchant/souvenir/categories

**Deskripsi:** Daftar kategori aktif berbentuk tree (parent + children). Untuk form tambah/edit produk.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": [
    {
      "id": "uuid-makanan",
      "name": "Makanan",
      "slug": "makanan",
      "description": "Makanan khas Aceh",
      "parent_id": null,
      "is_active": true,
      "children": [
        {
          "id": "uuid-kue-kering",
          "name": "Kue Kering",
          "slug": "kue-kering",
          "description": null,
          "parent_id": "uuid-makanan",
          "is_active": true,
          "children": []
        }
      ]
    }
  ],
  "message": "Daftar kategori berhasil diambil."
}
```

**Catatan untuk mobile:**

- Kategori read-only — tidak bisa CRUD dari merchant
- Cache di local storage (mis. Room/SQLite/UserDefaults) — kategori jarang berubah
- Refresh cache 1x per hari atau on-demand

## 10. Endpoints — Products

### 10.1 GET /merchant/souvenir/products

**Deskripsi:** List produk merchant.

**Auth:** ✅ Bearer token

Query parameters:

| Param | Tipe | Default | Deskripsi |
|---|---|---|---|
| search | string | — | Cari nama atau SKU |
| category_id | uuid | — | Filter kategori |
| is_active | bool | — | Filter status aktif |
| featured | bool | — | Filter produk unggulan |
| per_page | int | 15 | Max 60 |
| page | int | 1 | — |

Response 200:

```json
{
  "status": true,
  "data": [
    {
      "id": "uuid",
      "name": "Kopi Aceh Gayo",
      "slug": "kopi-aceh-gayo",
      "sku": "SKU-ABCD1234",
      "description": "...",
      "price": "50000.00",
      "discount_price": "40000.00",
      "final_price": 40000,
      "stock": 100,
      "weight": "250.00",
      "images": ["souvenir/products/xyz.jpg"],
      "is_active": true,
      "featured": false,
      "views": 42,
      "category": { "id": "uuid", "name": "Minuman", "slug": "minuman" },
      "store": { "id": "uuid", "name": "Toko Kopi Aceh", "slug": "toko-kopi-aceh" },
      "created_at": "2026-10-01T08:00:00.000000Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 3,
    "per_page": 15,
    "total": 42
  }
}
```

### 10.2 POST /merchant/souvenir/products

**Deskripsi:** Buat produk baru.

**Auth:** ✅ Bearer token · **Content-Type:** multipart/form-data

Request body:

| Field | Tipe | Wajib | Deskripsi |
|---|---|---|---|
| name | string | ✅ | Max 255 |
| price | numeric | ✅ | Min 0 |
| stock | int | ✅ | Min 0 |
| sku | string | ❌ | Max 100, unik. Kalau kosong, auto-generate |
| category_id | uuid | ❌ | — |
| store_id | uuid | ❌ | Default store merchant |
| description | string | ❌ | — |
| discount_price | numeric | ❌ | Harus < price |
| weight | numeric | ❌ | Gram |
| images[] | file array | ❌ | Max 5 file, jpeg/png/jpg/gif/webp, max 2 MB each |
| is_active | bool | ❌ | Default true |
| featured | bool | ❌ | Default false |

Response 201:

```json
{
  "status": true,
  "message": "Produk berhasil ditambahkan.",
  "data": { }
}
```

Response 422 — SKU duplikat:

```json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "sku": ["Kode produk (SKU) ini sudah digunakan, gunakan kode lain."]
  }
}
```

### 10.3 GET /merchant/souvenir/products/{id}

**Deskripsi:** Detail produk.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": { }
}
```

Response 404:

```json
{
  "status": false,
  "message": "Produk tidak ditemukan."
}
```

### 10.4 PUT /merchant/souvenir/products/{id}

**Deskripsi:** Update produk.

**Auth:** ✅ Bearer token · **Content-Type:** multipart/form-data

**Request body:** semua field `sometimes`, sama seperti POST.

Response 200:

```json
{
  "status": true,
  "message": "Produk berhasil diperbarui.",
  "data": { }
}
```

### 10.5 DELETE /merchant/souvenir/products/{id}

**Deskripsi:** Soft delete produk. Semua file gambar dihapus dari disk.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "message": "Produk berhasil dihapus."
}
```

### 10.6 PUT /merchant/souvenir/products/{id}/stock

**Deskripsi:** Update stok produk.

**Auth:** ✅ Bearer token · **Content-Type:** application/json

Request body:

```json
{
  "stock": 25
}
```

Response 200:

```json
{
  "status": true,
  "message": "Stok produk berhasil diperbarui.",
  "data": {
    "id": "uuid",
    "name": "Kopi Aceh Gayo",
    "stock": 25
  }
}
```

### 10.7 PUT /merchant/souvenir/products/{id}/toggle-active

**Deskripsi:** Toggle status aktif produk.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "message": "Produk diaktifkan.",
  "data": {
    "id": "uuid",
    "name": "Kopi Aceh Gayo",
    "is_active": true
  }
}
```

### 10.8 POST /merchant/souvenir/products/{id}/images

**Deskripsi:** Upload gambar produk (tambahan, opsional).

**Auth:** ✅ Bearer token · **Content-Type:** multipart/form-data

Request body:

```text
images[]: <file>
images[]: <file>
```

Max 5 file, total max 5 gambar per produk.

Response 200:

```json
{
  "status": true,
  "message": "Gambar berhasil diupload.",
  "data": ["souvenir/products/xyz.jpg", "..."]
}
```

### 10.9 DELETE /merchant/souvenir/products/{id}/images

**Deskripsi:** Hapus gambar produk.

**Auth:** ✅ Bearer token · **Content-Type:** application/json

Request body:

```json
{
  "image": "souvenir/products/xyz.jpg"
}
```

Response 200:

```json
{
  "status": true,
  "message": "Gambar berhasil dihapus."
}
```

## 11. Endpoints — Orders

### 11.1 GET /merchant/souvenir/orders

**Deskripsi:** List pesanan merchant.

**Auth:** ✅ Bearer token

Query parameters:

| Param | Tipe | Default | Deskripsi |
|---|---|---|---|
| status | enum | — | pending, paid, processing, shipped, completed, cancelled |
| search | string | — | Cari order number |
| date_from | date | — | Format YYYY-MM-DD |
| date_to | date | — | Format YYYY-MM-DD |
| per_page | int | 15 | Max 50 |
| page | int | 1 | — |

Response 200:

```json
{
  "status": true,
  "data": [
    {
      "id": "uuid",
      "order_number": "SO-20261008-ABC123XY",
      "status": "paid",
      "status_label": "Dibayar",
      "status_badge_class": "bg-blue-100 text-blue-800",
      "payment_status": "paid",
      "payment_status_label": "Dibayar",
      "total_amount": "135000.00",
      "formatted_total": "Rp 135.000",
      "shipping_cost": "15000.00",
      "discount_total": "0.00",
      "user": {
        "uuid": "uuid",
        "name": "Budi Santoso",
        "phone": "628123456789",
        "email": "budi@example.com"
      },
      "shipping_address": {
        "name": "Budi Santoso",
        "phone": "628123456789",
        "address": "Jl. ...",
        "city": "Banda Aceh",
        "postal_code": "23116"
      },
      "items": [
        {
          "id": "uuid",
          "product_name": "Kopi Aceh Gayo",
          "price": "50000.00",
          "quantity": 2,
          "subtotal": "100000.00"
        }
      ],
      "shipping_trackings": [],
      "ordered_at": "2026-10-08T10:30:00.000000Z",
      "paid_at": "2026-10-08T10:35:00.000000Z"
    }
  ],
  "meta": { }
}
```

> ⚠️ **PENTING — Filter data customer:**

| Status Pembayaran | shipping_address | user.phone | user.email | user.name |
|---|---|---|---|---|
| paid | Full | Full | Full | Full |
| belum paid | null | null | null | Di-mask ("Budi S.") |

Ini aturan bisnis: merchant tidak boleh hubungi customer sebelum pesanan dibayar. Kontak lengkap muncul di dashboard setelah pelunasan.

### 11.2 GET /merchant/souvenir/orders/{uuid}

**Deskripsi:** Detail pesanan.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": {
    "id": "uuid",
    "order_number": "SO-20261008-ABC123XY",
    "status": "paid",
    "payment_status": "paid",
    "total_amount": "135000.00",
    "shipping_address": { },
    "user": { },
    "items": [ ],
    "shipping_trackings": [ ],
    "courier": null,
    "tracking_number": null,
    "shipped_at": null,
    "completed_at": null,
    "notes": null,
    "ordered_at": "2026-10-08T10:30:00.000000Z",
    "paid_at": "2026-10-08T10:35:00.000000Z"
  }
}
```

`shipping_address` bernilai `null` untuk order yang belum paid (lihat aturan filter di 11.1).

### 11.3 PUT /merchant/souvenir/orders/{uuid}/status

**Deskripsi:** Update status pesanan.

**Auth:** ✅ Bearer token · **Rate limit:** 60 req/menit · **Content-Type:** application/json

Request body:

```json
{
  "status": "processing",
  "reason": null
}
```

Field `reason` wajib diisi kalau `status = cancelled`.

Transisi valid:

| Status Sekarang | Status Tujuan Valid |
|---|---|
| paid | processing, cancelled |
| processing | cancelled |
| shipped | completed |

> ⚠️ `shipped` TIDAK BOLEH lewat endpoint ini. Untuk kirim pesanan, gunakan `POST /merchant/souvenir/orders/{uuid}/ship`.

Response 200:

```json
{
  "status": true,
  "message": "Status pesanan berhasil diperbarui.",
  "data": { }
}
```

Response 422 — Transisi tidak valid:

```json
{
  "status": false,
  "message": "Transisi status dari 'pending' ke 'processing' tidak diizinkan."
}
```

Response 422 — Status tidak diizinkan:

```json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "status": ["Status hanya boleh: processing, completed, atau cancelled. Untuk mengirim pesanan gunakan endpoint /ship."]
  }
}
```

### 11.4 POST /merchant/souvenir/orders/{uuid}/ship

**Deskripsi:** Kirim pesanan manual (input resi).

**Auth:** ✅ Bearer token · **Rate limit:** 60 req/menit · **Content-Type:** application/json atau form-data

Request body:

```json
{
  "tracking_number": "JNE1234567890",
  "courier": "jne",
  "delivery_note": "Dikirim via JNE Reguler"
}
```

Rules:

- Order harus berstatus `paid` atau `processing`
- Belum punya `tracking_number` — cegah overwrite

Response 200:

```json
{
  "status": true,
  "message": "Pesanan berhasil dikirim.",
  "data": {
    "id": "uuid",
    "status": "shipped",
    "status_label": "Dikirim",
    "courier": "jne",
    "tracking_number": "JNE1234567890",
    "shipped_at": "2026-10-08T11:00:00.000000Z",
    "estimated_delivery_at": "2026-10-11T11:00:00.000000Z"
  }
}
```

Response 422:

```json
{
  "status": false,
  "message": "Pesanan sudah punya nomor resi: JNE1234567890."
}
```

### 11.5 POST /merchant/souvenir/orders/{uuid}/ship-kiriminaja

**Deskripsi:** Kirim via KiriminAja (auto AWB).

**Auth:** ✅ Bearer token · **Content-Type:** application/json

Request body:

```json
{
  "sender_name": "Toko Kopi Aceh",
  "sender_phone": "081234567890",
  "sender_address": "Jl. Cut Nyak Dhien No. 10, Banda Aceh",
  "sender_district_id": 1101010,
  "recipient_name": "Budi Santoso",
  "recipient_phone": "628123456789",
  "recipient_address": "Jl. Sudirman No. 5, Banda Aceh",
  "recipient_district_id": 1101020,
  "courier_code": "jne",
  "service_type": "REG",
  "schedule": null,
  "weight": 500,
  "width": 20,
  "height": 10,
  "length": 15,
  "delivery_note": "Handle with care"
}
```

`courier_code` enum: `jne`, `jnt`, `sicepat`, `pos`, `anteraja`

Response 200:

```json
{
  "status": true,
  "message": "Pesanan berhasil dikirim via KiriminAja.",
  "data": {
    "order": { },
    "shipping": {
      "success": true,
      "awb": "JNE1234567890",
      "courier": "jne",
      "order_id": "KRM-20261008-123",
      "message": "Berhasil request pickup."
    }
  }
}
```

`order` berisi object order yang sudah diperbarui.

Response 502 — KiriminAja error:

```json
{
  "status": false,
  "message": "Gagal membuat pengiriman via KiriminAja."
}
```

### 11.6 GET /merchant/souvenir/orders/{uuid}/track

**Deskripsi:** Lacak pesanan.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": {
    "tracking_number": "JNE1234567890",
    "courier": "jne",
    "tracking": {
      "summary": { },
      "detail": [ ]
    },
    "history": [
      {
        "id": "uuid",
        "status": "shipped",
        "description": "Paket telah dikirim melalui jne dengan nomor resi JNE1234567890",
        "location": "Banda Aceh",
        "tracked_at": "2026-10-08T11:00:00.000000Z"
      },
      {
        "id": "uuid",
        "status": "in_transit",
        "description": "Paket sedang dalam perjalanan",
        "location": "Medan",
        "tracked_at": "2026-10-08T15:30:00.000000Z"
      }
    ],
    "remote_error": null
  }
}
```

`tracking.summary` dan `tracking.detail` berisi data dari KiriminAja.

Kalau KiriminAja timeout:

```json
{
  "status": true,
  "data": {
    "tracking_number": "JNE1234567890",
    "courier": "jne",
    "tracking": null,
    "history": [ ],
    "remote_error": "Connection timeout"
  }
}
```

`history` pada kasus ini berisi history lokal.

**Catatan untuk mobile:**

- Kalau `remote_error` ada → tampilkan warning kecil + tampil history lokal saja
- Kalau `tracking` ada → tampilkan data live dari kurir

## 12. Endpoints — Transactions & Withdraw

### 12.1 GET /merchant/transactions

**Deskripsi:** Riwayat transaksi (gabungan order + withdrawal).

**Auth:** ✅ Bearer token

Query parameters:

| Param | Tipe | Default |
|---|---|---|
| per_page | int | 15 |
| page | int | 1 |

Response 200:

```json
{
  "status": true,
  "data": [
    {
      "id": "uuid",
      "type": "order",
      "amount": 135000,
      "formatted_amount": "Rp 135.000",
      "is_credit": true,
      "reference": "SO-20261008-ABC123XY",
      "description": "Pembayaran order",
      "status": "completed",
      "created_at": "2026-10-08T10:30:00.000000Z"
    },
    {
      "id": "uuid",
      "type": "withdrawal",
      "amount": -50000,
      "formatted_amount": "Rp 50.000",
      "is_credit": false,
      "reference": "WD-20261007-XYZ789",
      "description": "Penarikan saldo",
      "status": "pending",
      "created_at": "2026-10-07T15:00:00.000000Z"
    }
  ],
  "meta": { }
}
```

Field `type`:

- `"order"` — pemasukan dari order completed (amount positif, `is_credit = true`)
- `"withdrawal"` — penarikan saldo (amount negatif, `is_credit = false`)

### 12.2 POST /merchant/withdraw

**Deskripsi:** Ajukan penarikan saldo.

**Auth:** ✅ Bearer token · **Rate limit:** 10 req/menit · **Content-Type:** application/json

Request body:

```json
{
  "amount": 50000,
  "bank_name": "BCA",
  "bank_account": "1234567890",
  "account_name": "Budi Santoso"
}
```

Rules:

- Minimum Rp 10.000
- Maximum Rp 100.000.000 per transaksi
- Saldo harus mencukupi

Response 200:

```json
{
  "status": true,
  "message": "Permintaan penarikan berhasil. Proses 1-3 hari kerja.",
  "data": {
    "withdrawal_id": "uuid",
    "reference": "WD-20261008-ABC123",
    "amount": 50000,
    "formatted_amount": "Rp 50.000",
    "balance_after": 100000,
    "formatted_balance_after": "Rp 100.000"
  }
}
```

Response 422 — Saldo tidak cukup:

```json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "amount": ["Saldo tidak mencukupi. Saldo Anda: Rp 30.000"]
  }
}
```

### 12.3 GET /merchant/withdrawals

**Deskripsi:** Riwayat penarikan.

**Auth:** ✅ Bearer token

Response 200:

```json
{
  "status": true,
  "data": [
    {
      "id": "uuid",
      "reference": "WD-20261008-ABC123",
      "amount": 50000,
      "formatted_amount": "Rp 50.000",
      "bank_name": "BCA",
      "bank_account": "1234567890",
      "account_name": "Budi Santoso",
      "status": "pending",
      "status_label": "Menunggu",
      "notes": null,
      "processed_at": null,
      "completed_at": null,
      "created_at": "2026-10-08T15:00:00.000000Z"
    }
  ],
  "meta": { }
}
```

Status withdrawal:

| Status | Label | Deskripsi |
|---|---|---|
| pending | Menunggu | Baru diajukan |
| processed | Diproses | Sedang diproses admin |
| completed | Selesai | Dana sudah ditransfer |
| failed | Gagal | Transfer gagal |

## 13. Order Lifecycle & Business Rules

### 13.1 Status Flow

```text
pending  →  paid  →  processing  →  shipped  →  completed
   ↓         ↓           ↓
cancelled cancelled  cancelled
```

| Status | Deskripsi | Transisi Boleh |
|---|---|---|
| pending | Menunggu pembayaran customer | paid, cancelled |
| paid | Sudah dibayar, siap diproses | processing, cancelled |
| processing | Sedang disiapkan merchant | shipped, cancelled |
| shipped | Sudah dikirim | completed |
| completed | Selesai | — |
| cancelled | Dibatalkan | — |

### 13.2 Aksi per Status

| Status | Aksi yang Tersedia |
|---|---|
| pending | Tidak ada — tunggu customer bayar |
| paid | Update ke processing, atau cancelled |
| processing | Kirim (`/ship` atau `/ship-kiriminaja`), atau cancelled |
| shipped | Lacak (`/track`), atau completed |
| completed | Lihat detail, tidak ada aksi |
| cancelled | Lihat detail, tidak ada aksi |

### 13.3 Aturan Kontak Customer (PENTING)

Defense in depth — 2 layer:

**Layer 1 — API (backend):**

- Order `payment_status !== 'paid'`: `shipping_address = null`, `user.phone = null`, `user.email = null`, `user.name` di-mask ("Budi S.")
- Order `payment_status === 'paid'`: full data

**Layer 2 — Client (web/mobile):**

- Cek `payment_status === 'paid'` sebelum render alamat
- Kalau `!== 'paid'` → tampilkan pesan "Detail pembeli akan tersedia setelah pesanan dibayar"

**Alasan:** cegah merchant hubungi customer di luar sistem (lewat WA/telepon langsung).

### 13.4 Notifikasi Otomatis

Backend otomatis kirim notifikasi:

| Event | Email Customer | Email Merchant | WA Customer | WA Merchant |
|---|---|---|---|---|
| Order created | ✅ + tombol bayar | ✅ (tanpa kontak) | ✅ + link bayar | ✅ (info dasar) |
| Order paid | ✅ | ✅ (tanpa kontak) | ✅ | ✅ (info dasar) |
| Order shipped | ✅ | — | ✅ | — |
| Order completed | ✅ | — | ✅ | — |
| Order cancelled | ✅ | ✅ (info dasar) | ✅ | — |

Merchant TIDAK perlu implement notifikasi manual — backend sudah handle via observer.

### 13.5 Rate Limit Ops

| Endpoint | Limit | Alasan |
|---|---|---|
| PUT /orders/{uuid}/status | 60/menit | Cegah spam transisi |
| POST /orders/{uuid}/ship | 60/menit | Cegah spam ship |
| POST /withdraw | 10/menit | Cegah spam withdrawal |

## 14. Constants Reference

### 14.1 Order Status

```php
SouvenirOrder::STATUS_PENDING    = 'pending'
SouvenirOrder::STATUS_PAID       = 'paid'
SouvenirOrder::STATUS_PROCESSING = 'processing'
SouvenirOrder::STATUS_SHIPPED    = 'shipped'
SouvenirOrder::STATUS_COMPLETED  = 'completed'
SouvenirOrder::STATUS_CANCELLED  = 'cancelled'
```

### 14.2 Payment Status

```php
SouvenirOrder::PAYMENT_UNPAID   = 'unpaid'
SouvenirOrder::PAYMENT_PAID     = 'paid'
SouvenirOrder::PAYMENT_FAILED   = 'failed'
SouvenirOrder::PAYMENT_REFUNDED = 'refunded'
```

### 14.3 Merchant Status

```php
Merchant::STATUS_PENDING   = 'pending'    // Menunggu persetujuan admin
Merchant::STATUS_ACTIVE    = 'active'     // Aktif
Merchant::STATUS_SUSPENDED = 'suspended'  // Ditangguhkan
Merchant::STATUS_INACTIVE  = 'inactive'   // Tidak aktif
```

### 14.4 Business Types

```php
'hotel'     => 'Hotel & Penginapan'
'kuliner'   => 'Kuliner & Restoran'
'rental'    => 'Rental Kendaraan & Perlengkapan'
'tour'      => 'Paket Wisata'
'destinasi' => 'Destinasi Wisata'
'souvenir'  => 'Souvenir & Oleh-oleh'   // ← Merchant marketplace
```

### 14.5 Business Modules (per business_type)

```php
'hotel'     => ['hotel', 'wisata']
'kuliner'   => ['kuliner']
'rental'    => ['rental']
'tour'      => ['tour', 'wisata']
'destinasi' => ['destinasi', 'wisata']
'souvenir'  => ['marketplace', 'souvenir']   // ← Fokus merchant.ovisito.com
```

### 14.6 Withdrawal Status

```php
MerchantWithdrawal::STATUS_PENDING   = 'pending'
MerchantWithdrawal::STATUS_PROCESSED = 'processed'
MerchantWithdrawal::STATUS_COMPLETED = 'completed'
MerchantWithdrawal::STATUS_FAILED    = 'failed'
```

### 14.7 Courier Codes (KiriminAja)

```text
jne, jnt, sicepat, pos, anteraja
```

## 15. Integration Guide per Platform

### 15.1 Web (Laravel Blade)

Pattern:

```text
Browser → Laravel FE (SSR) → API → Response
         ↓
       Session (server-side)
```

Contoh request dari FE:

```php
$response = Http::withHeaders([
    'Authorization'   => 'Bearer ' . Session::get('merchant_token'),
    'X-Client-ID'     => config('app.client_id'),
    'X-Client-Secret' => config('app.client_secret'),
    'Accept'          => 'application/json',
])->get(config('app.api_base_url') . '/merchant/dashboard');
```

**Token storage:** server-side session (bukan localStorage).

**Keuntungan:** tidak ada CORS, tidak expose token ke JS.

### 15.2 Android (Kotlin + Retrofit)

Setup Retrofit:

```kotlin
interface MerchantApi {
    @POST("merchant/login")
    suspend fun login(@Body req: LoginRequest): ApiResponse<LoginData>

    @GET("merchant/souvenir/orders")
    suspend fun getOrders(
        @Query("status") status: String? = null,
        @Query("page") page: Int = 1,
        @Query("per_page") perPage: Int = 15,
    ): ApiResponse<List<Order>>

    @PUT("merchant/souvenir/orders/{uuid}/status")
    suspend fun updateStatus(
        @Path("uuid") uuid: String,
        @Body req: UpdateStatusRequest,
    ): ApiResponse<Order>
}
```

Interceptor untuk token + headers:

```kotlin
class AuthInterceptor(private val tokenStorage: TokenStorage) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val builder = chain.request().newBuilder()
            .addHeader("X-Client-ID", BuildConfig.CLIENT_ID)
            .addHeader("X-Client-Secret", BuildConfig.CLIENT_SECRET)
            .addHeader("Accept", "application/json")

        tokenStorage.getToken()?.let {
            builder.addHeader("Authorization", "Bearer $it")
        }

        return chain.proceed(builder.build())
    }
}
```

Token storage — EncryptedSharedPreferences:

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val sharedPrefs = EncryptedSharedPreferences.create(
    context,
    "merchant_secure_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM,
)
```

> ⚠️ **Android security:**
>
> - Jangan hardcode `CLIENT_SECRET` di source code — pakai `local.properties` (exclude dari git)
> - Atau lebih aman: backend proxy endpoint (roadmap v2.7)
> - Aktifkan certificate pinning untuk production

### 15.3 iOS (Swift + URLSession)

Setup URLSession:

```swift
class APIClient {
    static let shared = APIClient()
    private let baseURL = "https://api.ovisito.com/api/v2"

    func request<T: Decodable>(
        _ endpoint: String,
        method: String = "GET",
        body: [String: Any]? = nil
    ) async throws -> T {
        var request = URLRequest(url: URL(string: "\(baseURL)/\(endpoint)")!)
        request.httpMethod = method
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        request.setValue(Config.clientID, forHTTPHeaderField: "X-Client-ID")
        request.setValue(Config.clientSecret, forHTTPHeaderField: "X-Client-Secret")

        if let token = KeychainService.getToken() {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        if let body = body {
            request.httpBody = try JSONSerialization.data(withJSONObject: body)
        }

        let (data, response) = try await URLSession.shared.data(for: request)
        // Handle status, decode, dll
    }
}
```

Token storage — Keychain:

```swift
import Security

enum KeychainService {
    static func save(token: String) {
        let data = token.data(using: .utf8)!
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "merchant_token",
            kSecValueData as String: data,
        ]
        SecItemDelete(query as CFDictionary)
        SecItemAdd(query as CFDictionary, nil)
    }

    static func getToken() -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "merchant_token",
            kSecReturnData as String: true,
        ]
        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)
        guard let data = result as? Data else { return nil }
        return String(data: data, encoding: .utf8)
    }

    static func deleteToken() {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "merchant_token",
        ]
        SecItemDelete(query as CFDictionary)
    }
}
```

> ⚠️ **iOS security:**
>
> - Jangan hardcode `clientSecret` — pakai `.xcconfig` (exclude dari git)
> - Aktifkan App Transport Security + certificate pinning untuk production

## 16. Sample Flows (End-to-End)

### 16.1 Flow: Register → Verify → Login

```text
1. Client → POST /merchant/register
   Body: { name, email, password, phone, business_name, business_type, ... }
   ← Response 201: { status: true, data: { uuid, email } }

2. Client tampilkan layar "Cek email"

3. User buka email → klik link verifikasi
   GET /merchant/verify-email/{uuid}?hash=<sha1_email>
   ← Response 200: { status: true, message: "Email berhasil diverifikasi." }

4. Client → POST /merchant/login
   Body: { email, password, device_name }
   ← Response 200: { status: true, data: { token, merchant: {...} } }

5. Client simpan token + merchant data
6. Client → halaman Dashboard
```

### 16.2 Flow: Setup Store → Product

```text
1. Client → GET /merchant/store
   ← Response 404: belum punya toko

2. Client → GET /merchant/souvenir/categories
   ← Response: kategori tree (untuk form)

3. User isi form toko
4. Client → POST /merchant/store (multipart)
   Body: { name, address, phone, category_ids[], logo }
   ← Response 201: { status: true, data: {...} }

5. Client → POST /merchant/souvenir/products (multipart)
   Body: { name, price, stock, category_id, images[] }
   ← Response 201: { status: true, data: {...} }

6. Client → GET /merchant/souvenir/products
   ← List produk
```

### 16.3 Flow: Handle Order

```text
1. (Customer bayar pesanan → webhook Flip)
2. Backend update order → status=paid
3. Notifikasi otomatis terkirim

4. Client → GET /merchant/souvenir/orders?status=paid
   ← List order baru
   ← Karena sudah paid: shipping_address + kontak customer tampil lengkap
     (untuk order belum paid, field tersebut null — lihat §11.1)

5. Merchant klik detail → GET /merchant/souvenir/orders/{uuid}
   ← Data lengkap + kontak customer

6. Merchant proses → PUT /merchant/souvenir/orders/{uuid}/status
   Body: { status: "processing" }
   ← Response 200

7. Merchant siapkan barang, kirim manual
   POST /merchant/souvenir/orders/{uuid}/ship
   Body: { tracking_number, courier }
   ← Response 200: { status: "shipped" }

   ATAU via KiriminAja:
   POST /merchant/souvenir/orders/{uuid}/ship-kiriminaja
   Body: { sender_*, recipient_*, courier_code, ... }
   ← Response 200: { data: { order, shipping } }

8. Webhook shipping (dari kurir) → status delivered
   → Backend update otomatis → status=completed

9. Client → GET /merchant/souvenir/orders/{uuid}/track
   ← Riwayat tracking lengkap
```

### 16.4 Flow: Withdraw

```text
1. Client → GET /merchant/dashboard
   ← summary.balance: 150000

2. Client → GET /merchant/transactions
   ← Riwayat transaksi (order + withdrawal)

3. User isi form withdraw
4. Client → POST /merchant/withdraw
   Body: { amount, bank_name, bank_account, account_name }
   ← Response 200: {
     data: {
       withdrawal_id: "uuid",
       reference: "WD-20261008-ABC123",
       balance_after: 100000,
     }
   }

5. Client refresh balance
   GET /merchant/dashboard
   ← summary.balance: 100000
```

## 17. Changelog

### v2.6 — 2026-10-08

**Breaking Changes:**

- Response `status` field sekarang boolean (`true`/`false`), bukan `'success'`/`'error'`
- Pagination format: `data[]` + `meta{}`
- `PUT /orders/{uuid}/status` tidak menerima `shipped` — wajib via `/ship`
- Filter customer data: `shipping_address`, `phone`, `email` di-null-kan untuk order belum paid

**Additions:**

- Endpoint `POST/DELETE /products/{id}/images`
- Response `user.name` di-mask ("Budi S.") untuk order belum paid
- Accessor `payment_method_label`, `final_price`
- Notifikasi merchant email di setiap event

**Fixes:**

- Auth guard: `auth:merchant_api` (bukan `merchant`)
- `forgotPassword` — benar-benar kirim email
- `changePassword` — fix double hash bug
- Store slug race condition
- Category `orWhere` grouping
- Transaction UNION cross-DB

### v2.5 — 2026-10-08

- Auto trigger payment + email/WA tombol bayar
- WA Fonnte aktif

### v2.2 — 2026-10-08

- Multi-database refactor

### v2.1 — 2026-10-07

- Dokumentasi awal

## Kontak & Support

- **Backend Team:** backend@ovisito.com
- **API Status:** https://api.ovisito.com/api/v2/ping
- **Health Check:** https://api.ovisito.com/up

---

*Maintained by: Backend Team Ovisito · Last updated: 2026-10-08 · Version: 2.6*

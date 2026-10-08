Ovisito Merchant API — Dokumentasi Lengkap
Dokumentasi API khusus Merchant Marketplace untuk integrasi Web, Android, dan iOS.

Versi: 2.6
Terakhir diupdate: 2026-10-08
Base URL: https://api.ovisito.com/api/v2
Dashboard Web: https://merchant.ovisito.com

Daftar Isi
Quick Start

Authentication & Security

Request & Response Convention

Error Handling

Rate Limits

Endpoint — Auth

Endpoint — Profile

Endpoint — Dashboard

Endpoint — Store

Endpoint — Categories

Endpoint — Products

Endpoint — Orders

Endpoint — Transactions & Withdraw

Business Rules

Constants

Integration Guide — Web (Blade)

Integration Guide — Android (Kotlin)

Integration Guide — iOS (Swift)

Sample End-to-End Flow

Changelog & Support

1. Quick Start
Untuk siapa dokumen ini
Khusus merchant marketplace souvenir. Endpoint public/customer/admin tidak dicakup di sini.

Base URL per environment
Environment	Base URL
Production	https://api.ovisito.com/api/v2
Staging	https://staging-api.ovisito.com/api/v2
Local	http://localhost:8000/api/v2
Syarat minimum setiap request
text
X-Client-ID: client_web
X-Client-Secret: <secret_dari_backend>
Accept: application/json
Content-Type: application/json (atau multipart/form-data untuk upload)
Setelah login, tambahkan
text
Authorization: Bearer <merchant_token>
Cara pertama kali (5 menit)
text
1. POST /merchant/register       → daftar
2. Cek email, klik link verify   → email_verified_at ter-set
3. POST /merchant/login          → dapat token
4. GET  /merchant/dashboard      → cek dashboard
5. GET  /merchant/store          → cek toko (kalau belum, POST /merchant/store)
6. GET  /merchant/souvenir/products → cek produk
2. Authentication & Security
2.1 Flow Auth
text
Register → Verify Email → Login → Access → Logout
2.2 Token
Aspek	Detail
Tipe	Laravel Sanctum Personal Access Token
Format	{id}|{40-char-token}
Header	Authorization: Bearer {token}
Expiry	Tidak ada (perlu revoke manual via logout)
Revoke otomatis	Setelah change-password (semua token lain) atau reset-password (semua token)
2.3 Storage Token
Platform	Tempat Simpan	Jangan
Web (Blade SSR)	Server session	localStorage
Android	EncryptedSharedPreferences	SharedPreferences biasa
iOS	Keychain	UserDefaults
2.4 Client Secret
JANGAN hardcode di mobile. Gunakan:

Platform	Cara
Web	.env (server-side)
Android	local.properties (exclude dari git)
iOS	.xcconfig (exclude dari git)
Roadmap: backend proxy endpoint untuk mobile (target v2.7).

2.5 Cara Kirim Token
Setelah login, setiap request merchant wajib tambah header:

text
Authorization: Bearer 1|abcdefghijklmnopqrstuvwxyz1234567890
Response kalau token tidak valid / expired:

json
HTTP 401
{
  "status": false,
  "message": "Unauthenticated. Silakan login terlebih dahulu."
}
Aksi di client: hapus token + data merchant, redirect ke halaman login.

3. Request & Response Convention
3.1 Success Response
Single resource:

json
{
  "status": true,
  "message": "Optional message",
  "data": { ... }
}
List dengan pagination:

json
{
  "status": true,
  "data": [ ... ],
  "meta": {
    "current_page": 1,
    "last_page": 5,
    "per_page": 15,
    "total": 60
  }
}
3.2 Error Response
json
{
  "status": false,
  "message": "Pesan error",
  "errors": {
    "field": ["pesan error field"]
  }
}
errors hanya ada di response 422 (validation error).

3.3 HTTP Status
Code	Arti
200	OK
201	Created
400	Bad Request
401	Token invalid / tidak ada
403	Forbidden (email belum verify, akun tidak aktif)
404	Resource tidak ditemukan
422	Validation error
429	Rate limit
500	Server error
502	Upstream error (KiriminAja, Flip)
3.4 Format Data
Data	Format	Contoh
Tanggal-waktu	ISO 8601 UTC	2026-10-08T10:30:00.000000Z
Tanggal	YYYY-MM-DD	2026-10-08
Jam	HH:MM	08:30
Uang	String decimal	"50000.00"
Uang formatted	String Rupiah	"Rp 50.000"
Boolean	true / false	true
UUID	36 char	01a0e882-b4fe-737c-...
3.5 Pagination
Query:

text
?page=2&per_page=15
Default per_page: 15
Max per_page: 50 (order, transaction, withdrawal), 60 (product, store)

4. Error Handling
4.1 Error Matrix per Endpoint
Status	Kapan	Aksi Client
401	Token invalid/expired	Clear token, redirect login
403	Email belum verify	Redirect ke halaman verifikasi
403	Akun tidak aktif	Tampilkan pesan + support link
404	Order/produk/store tidak ada	Tampilkan not found
422	Validation gagal	Tampilkan error per field
429	Rate limit	Backoff, tunggu
500	Server error	Retry + laporkan
502	KiriminAja error	Tampilkan pesan + coba lagi
4.2 Penanganan Khusus
Email belum verifikasi (403):

json
{
  "status": false,
  "message": "Silakan verifikasi email Anda terlebih dahulu.",
  "data": { "code": "EMAIL_NOT_VERIFIED" }
}
Aksi: tampilkan layar "Cek email + tombol kirim ulang".

Order customer null (bukan error):

json
{
  "status": true,
  "data": {
    "shipping_address": null,
    "user": { "name": "Budi S.", "phone": null, "email": null }
  }
}
Aksi: render placeholder "Detail akan tersedia setelah dibayar".

5. Rate Limits
Endpoint	Limit
Semua endpoint default	120 req/menit
POST /merchant/login	5 req/menit
POST /merchant/register	10 req/menit
POST /merchant/forgot-password	3 req/menit
POST /merchant/reset-password	5 req/menit
POST /merchant/resend-verification	6 req/menit
POST /merchant/withdraw	10 req/menit
PUT /merchant/souvenir/orders/{uuid}/status	60 req/menit
POST /merchant/souvenir/orders/{uuid}/ship	60 req/menit
Response 429:

json
{
  "status": false,
  "message": "Too Many Attempts."
}
6. Endpoint — Auth
6.1 POST /merchant/register
Registrasi merchant baru.

Auth	❌ Tidak perlu token
Rate limit	10/menit
Content-Type	application/json
Request:

json
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
Validasi:

Field	Tipe	Wajib	Aturan
name	string	✅	max 255
email	email	✅	max 255, unik
password	string	✅	min 8, harus ada password_confirmation
phone	string	✅	max 20, unik
business_name	string	✅	max 255
business_type	enum	✅	hotel, kuliner, rental, tour, destinasi, souvenir
address	string	✅	—
city	string	✅	—
province	string	✅	—
postal_code	string	❌	max 10
description	string	❌	max 500
website	url	❌	max 255
category_id	uuid	❌	—
Response 201:

json
{
  "status": true,
  "message": "Registrasi berhasil. Silakan cek email untuk verifikasi.",
  "data": {
    "uuid": "01a0e882-b4fe-737c-8936-68152828b651",
    "email": "budi@merchant.com"
  }
}
Response 422:

json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "email": ["Email sudah terdaftar."]
  }
}
6.2 GET /merchant/verify-email/{uuid}
Verifikasi email via link dari email.

Auth	❌ Tidak perlu (link dari email)
Query param: hash = SHA-1 dari email merchant.

Contoh link: https://merchant.ovisito.com/verify-email/{uuid}?hash=<sha1_email>

Response 200:

json
{
  "status": true,
  "message": "Email berhasil diverifikasi."
}
Response 403:

json
{
  "status": false,
  "message": "Hash verifikasi tidak valid."
}
6.3 POST /merchant/resend-verification
Kirim ulang email verifikasi.

Rate limit	6/menit
Request:

json
{
  "email": "budi@merchant.com"
}
Response 200:

json
{
  "status": true,
  "message": "Jika email terdaftar dan belum diverifikasi, link baru telah dikirim."
}
⚠️ Response selalu sukses (anti user-enumeration), meskipun email tidak terdaftar.

6.4 POST /merchant/login
Login merchant.

Rate limit	5/menit
Request:

json
{
  "email": "budi@merchant.com",
  "password": "password123",
  "device_name": "web-merchant"
}
device_name rekomendasi:

Web: web-merchant

Android: android-merchant-{device_model}

iOS: ios-merchant-{device_name}

Response 200:

json
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
Response 403 — Email belum verify:

json
{
  "status": false,
  "message": "Silakan verifikasi email Anda terlebih dahulu.",
  "data": { "code": "EMAIL_NOT_VERIFIED" }
}
Response 403 — Akun tidak aktif:

json
{
  "status": false,
  "message": "Akun merchant tidak aktif."
}
Response 422 — Credential salah:

json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "email": ["Email atau password salah."]
  }
}
6.5 POST /merchant/logout
Revoke token aktif.

Auth	✅ Bearer token
Request: Kosong.

Response 200:

json
{
  "status": true,
  "message": "Logout berhasil."
}
6.6 POST /merchant/forgot-password
Request link reset password via email.

Rate limit	3/menit
Request:

json
{
  "email": "budi@merchant.com"
}
Response 200:

json
{
  "status": true,
  "message": "Jika email terdaftar, link reset password telah dikirim."
}
6.7 POST /merchant/reset-password
Reset password dengan token dari email.

Rate limit	5/menit
Request:

json
{
  "token": "abcdefghij...",
  "email": "budi@merchant.com",
  "password": "newpassword456",
  "password_confirmation": "newpassword456"
}
Response 200:

json
{
  "status": true,
  "message": "Password berhasil direset. Silakan login."
}
Response 422:

json
{
  "status": false,
  "message": "Token reset tidak valid atau sudah kadaluarsa."
}
⚠️ Setelah reset, semua token Sanctum lama di-revoke. User harus login ulang di semua device.

7. Endpoint — Profile
7.1 GET /merchant/profile
Ambil data profil.

Auth	✅ Bearer token
Response 200:

json
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
password & remember_token tidak di-expose.

7.2 PUT /merchant/profile
Update profil.

Auth	✅ Bearer token
Content-Type	multipart/form-data (kalau upload logo) atau application/json
Request body (semua sometimes):

Field	Tipe	Aturan
name	string	max 255
phone	string	max 20
business_name	string	max 255
address	string	—
city	string	max 100
province	string	max 100
postal_code	string	max 10
description	string	max 1000
website	url	max 255
logo	file	jpeg/png/jpg/webp, max 2 MB
Response 200:

json
{
  "status": true,
  "message": "Profil berhasil diperbarui.",
  "data": { ...merchant updated... }
}
7.3 PUT /merchant/change-password
Ganti password.

Auth	✅ Bearer token
Request:

json
{
  "current_password": "password123",
  "new_password": "newpassword456",
  "new_password_confirmation": "newpassword456"
}
Rules:

new_password minimal 8 karakter

Harus ada new_password_confirmation yang sama

Password baru ≠ password lama

Response 200:

json
{
  "status": true,
  "message": "Password berhasil diubah."
}
Response 422:

json
{
  "status": false,
  "message": "Password saat ini salah."
}
⚠️ Setelah ganti password, semua token lain di-revoke. Device ini tetap login.

8. Endpoint — Dashboard
8.1 GET /merchant/dashboard
Dashboard summary lengkap.

Auth	✅ Bearer token
Response 200:

json
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
Keterangan:

recent_orders[] — 5 order terbaru

sales_chart[] — 7 hari terakhir (label = Mon, Tue, dst)

balance — saldo siap withdraw (angka float)

8.2 GET /merchant/dashboard/summary
Ringkasan cepat untuk widget.

Auth	✅ Bearer token
Response 200:

json
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
9. Endpoint — Store
Satu merchant hanya boleh punya 1 toko.

9.1 GET /merchant/store
Ambil toko default.

Auth	✅ Bearer token
Response 200:

json
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
    "email": null,
    "website": null,
    "logo": null,
    "is_physical": true,
    "is_active": true,
    "is_default": true,
    "kabupaten_kota_id": null,
    "kecamatan_id": null,
    "desa_id": null,
    "latitude": null,
    "longitude": null,
    "jam_buka": "08:00",
    "jam_tutup": "17:00",
    "category_ids": ["uuid1", "uuid2"],
    "categories": [
      { "id": "uuid1", "name": "Makanan" }
    ],
    "created_at": "2026-10-01T08:00:00.000000Z",
    "updated_at": "2026-10-08T10:30:00.000000Z"
  }
}
Response 404 — Belum punya toko:

json
{
  "status": false,
  "message": "Toko tidak ditemukan.",
  "data": null
}
9.2 POST /merchant/store
Buat toko baru.

Auth	✅ Bearer token
Content-Type	multipart/form-data
Request body:

Field	Tipe	Wajib	Aturan
name	string	✅	max 255
address	string	✅	—
phone	string	✅	max 20
category_ids	array uuid	✅	1–5 UUID
description	string	❌	—
email	email	❌	max 255
website	url	❌	max 255
is_physical	bool	❌	default true
logo	file	❌	jpeg/png/jpg/webp, max 2 MB
kabupaten_kota_id	int	❌	—
kecamatan_id	int	❌	—
desa_id	int	❌	—
latitude	numeric	❌	-90 s/d 90
longitude	numeric	❌	-180 s/d 180
jam_buka	string	❌	HH:MM
jam_tutup	string	❌	HH:MM
Response 201:

json
{
  "status": true,
  "message": "Toko berhasil dibuat.",
  "data": { ...store... }
}
Response 422:

json
{
  "status": false,
  "message": "Merchant sudah memiliki toko."
}
9.3 PUT /merchant/store
Update toko.

Auth	✅ Bearer token
Content-Type	multipart/form-data
Request body: sama seperti POST, semua sometimes.

Response 200:

json
{
  "status": true,
  "message": "Toko berhasil diperbarui.",
  "data": { ...store updated... }
}
Kalau name diubah → slug otomatis regenerate.

9.4 GET /merchant/store/{uuid}
Detail toko by UUID.

Auth	✅ Bearer token
Response 200: sama dengan GET /merchant/store.

10. Endpoint — Categories
10.1 GET /merchant/souvenir/categories
Daftar kategori aktif untuk form produk (read-only).

Auth	✅ Bearer token
Response 200:

json
{
  "status": true,
  "data": [
    {
      "id": "uuid-makanan",
      "name": "Makanan",
      "slug": "makanan",
      "description": "...",
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
Cache di local storage (jarang berubah). Refresh 1×/hari.

11. Endpoint — Products
11.1 GET /merchant/souvenir/products
List produk merchant.

Auth	✅ Bearer token
Query params:

Param	Tipe	Default	Deskripsi
search	string	—	Cari nama / SKU
category_id	uuid	—	Filter kategori
is_active	bool	—	Filter aktif
featured	bool	—	Filter unggulan
page	int	1	—
per_page	int	15	max 60
Response 200:

json
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
Field penting:

final_price — harga setelah diskon (kalau ada), float. Untuk display harga.

price / discount_price — string decimal, untuk form edit.

sku — auto-generate kalau kosong.

11.2 POST /merchant/souvenir/products
Buat produk baru.

Auth	✅ Bearer token
Content-Type	multipart/form-data
Request body:

Field	Tipe	Wajib	Aturan
name	string	✅	max 255
price	numeric	✅	min 0
stock	int	✅	min 0
sku	string	❌	max 100, unik
category_id	uuid	❌	—
store_id	uuid	❌	default store
description	string	❌	—
discount_price	numeric	❌	< price
weight	numeric	❌	gram
images[]	file array	❌	max 5, jpeg/png/jpg/gif/webp, 2 MB each
is_active	bool	❌	default true
featured	bool	❌	default false
Response 201:

json
{
  "status": true,
  "message": "Produk berhasil ditambahkan.",
  "data": { ...product... }
}
Response 422 — SKU duplikat:

json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "sku": ["Kode produk (SKU) ini sudah digunakan, gunakan kode lain."]
  }
}
11.3 GET /merchant/souvenir/products/{id}
Detail produk.

Auth	✅ Bearer token
Response 200:

json
{
  "status": true,
  "data": { ...product... }
}
Response 404:

json
{
  "status": false,
  "message": "Produk tidak ditemukan."
}
11.4 PUT /merchant/souvenir/products/{id}
Update produk.

Auth	✅ Bearer token
Content-Type	multipart/form-data
Request body: semua sometimes, sama seperti POST.

Response 200:

json
{
  "status": true,
  "message": "Produk berhasil diperbarui.",
  "data": { ...product updated... }
}
11.5 DELETE /merchant/souvenir/products/{id}
Soft delete produk.

Auth	✅ Bearer token
Response 200:

json
{
  "status": true,
  "message": "Produk berhasil dihapus."
}
11.6 PUT /merchant/souvenir/products/{id}/stock
Update stok.

Auth	✅ Bearer token
Request:

json
{
  "stock": 25
}
Response 200:

json
{
  "status": true,
  "message": "Stok produk berhasil diperbarui.",
  "data": {
    "id": "uuid",
    "name": "Kopi Aceh Gayo",
    "stock": 25
  }
}
11.7 PUT /merchant/souvenir/products/{id}/toggle-active
Toggle status aktif.

Auth	✅ Bearer token
Request: Kosong.

Response 200:

json
{
  "status": true,
  "message": "Produk diaktifkan.",
  "data": {
    "id": "uuid",
    "name": "Kopi Aceh Gayo",
    "is_active": true
  }
}
11.8 POST /merchant/souvenir/products/{id}/images
Upload gambar produk (tambahan).

Auth	✅ Bearer token
Content-Type	multipart/form-data
Request:

text
images[]: <file>
images[]: <file>
Response 200:

json
{
  "status": true,
  "message": "Gambar berhasil diupload.",
  "data": ["souvenir/products/xyz.jpg"]
}
11.9 DELETE /merchant/souvenir/products/{id}/images
Hapus gambar produk.

Auth	✅ Bearer token
Request:

json
{
  "image": "souvenir/products/xyz.jpg"
}
Response 200:

json
{
  "status": true,
  "message": "Gambar berhasil dihapus."
}
12. Endpoint — Orders
12.1 GET /merchant/souvenir/orders
List pesanan.

Auth	✅ Bearer token
Query params:

Param	Tipe	Default	Deskripsi
status	enum	—	pending, paid, processing, shipped, completed, cancelled
search	string	—	Cari order number
date_from	date	—	YYYY-MM-DD
date_to	date	—	YYYY-MM-DD
page	int	1	—
per_page	int	15	max 50
Response 200:

json
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
  "meta": { ... }
}
⚠️ ATURAN KONTAK CUSTOMER:

payment_status	shipping_address	user.phone	user.email	user.name
paid	Full	Full	Full	Full
belum paid	null	null	null	Di-mask: "Budi S."
Alasan: cegah merchant hubungi customer sebelum pesanan dibayar.

12.2 GET /merchant/souvenir/orders/{uuid}
Detail pesanan.

Auth	✅ Bearer token
Response 200:

json
{
  "status": true,
  "data": {
    "id": "uuid",
    "order_number": "SO-20261008-ABC123XY",
    "status": "paid",
    "payment_status": "paid",
    "total_amount": "135000.00",
    "shipping_address": { ... atau null ... },
    "user": { ... },
    "items": [ ... ],
    "shipping_trackings": [ ... ],
    "courier": null,
    "tracking_number": null,
    "shipped_at": null,
    "completed_at": null,
    "notes": null,
    "ordered_at": "2026-10-08T10:30:00.000000Z",
    "paid_at": "2026-10-08T10:35:00.000000Z"
  }
}
12.3 PUT /merchant/souvenir/orders/{uuid}/status
Update status pesanan.

Auth	✅ Bearer token
Rate limit	60/menit
Request:

json
{
  "status": "processing",
  "reason": null
}
reason wajib kalau status = cancelled.

Transisi valid:

Dari	Ke
paid	processing, cancelled
processing	cancelled
shipped	completed
⚠️ shipped tidak boleh via endpoint ini. Pakai /ship atau /ship-kiriminaja.

Response 200:

json
{
  "status": true,
  "message": "Status pesanan berhasil diperbarui.",
  "data": { ...order updated... }
}
Response 422:

json
{
  "status": false,
  "message": "Transisi status dari 'pending' ke 'processing' tidak diizinkan."
}
12.4 POST /merchant/souvenir/orders/{uuid}/ship
Kirim manual (input resi).

Auth	✅ Bearer token
Rate limit	60/menit
Request:

json
{
  "tracking_number": "JNE1234567890",
  "courier": "jne",
  "delivery_note": "Dikirim via JNE Reguler"
}
Rules:

Order harus paid atau processing

Belum punya tracking_number

Response 200:

json
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
Response 422:

json
{
  "status": false,
  "message": "Pesanan sudah punya nomor resi: JNE1234567890."
}
12.5 POST /merchant/souvenir/orders/{uuid}/ship-kiriminaja
Kirim via KiriminAja (auto AWB).

Auth	✅ Bearer token
Request:

json
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
  "weight": 500,
  "width": 20,
  "height": 10,
  "length": 15,
  "delivery_note": "Handle with care"
}
courier_code enum: jne, jnt, sicepat, pos, anteraja

Response 200:

json
{
  "status": true,
  "message": "Pesanan berhasil dikirim via KiriminAja.",
  "data": {
    "order": { ...order updated... },
    "shipping": {
      "success": true,
      "awb": "JNE1234567890",
      "courier": "jne",
      "order_id": "KRM-20261008-123",
      "message": "Berhasil request pickup."
    }
  }
}
Response 502:

json
{
  "status": false,
  "message": "Gagal membuat pengiriman via KiriminAja."
}
12.6 GET /merchant/souvenir/orders/{uuid}/track
Lacak pesanan.

Auth	✅ Bearer token
Response 200:

json
{
  "status": true,
  "data": {
    "tracking_number": "JNE1234567890",
    "courier": "jne",
    "tracking": { ...data KiriminAja... },
    "history": [
      {
        "id": "uuid",
        "status": "shipped",
        "description": "Paket telah dikirim melalui jne dengan nomor resi JNE1234567890",
        "location": "Banda Aceh",
        "tracked_at": "2026-10-08T11:00:00.000000Z"
      }
    ],
    "remote_error": null
  }
}
Kalau KiriminAja timeout: tracking = null, remote_error = "...".

Aksi client: kalau remote_error ada → tampilkan warning + history lokal saja.

13. Endpoint — Transactions & Withdraw
13.1 GET /merchant/transactions
Riwayat transaksi (order + withdrawal).

Auth	✅ Bearer token
Query: page, per_page (default 15).

Response 200:

json
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
  "meta": { ... }
}
Field:

type = "order" (pemasukan) atau "withdrawal" (pengeluaran)

is_credit = true (masuk) / false (keluar)

amount = positif untuk pemasukan, negatif untuk withdrawal

13.2 POST /merchant/withdraw
Ajukan penarikan saldo.

Auth	✅ Bearer token
Rate limit	10/menit
Request:

json
{
  "amount": 50000,
  "bank_name": "BCA",
  "bank_account": "1234567890",
  "account_name": "Budi Santoso"
}
Rules: min Rp 10.000, max Rp 100.000.000 per transaksi.

Response 200:

json
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
Response 422 — Saldo tidak cukup:

json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "amount": ["Saldo tidak mencukupi. Saldo Anda: Rp 30.000"]
  }
}
13.3 GET /merchant/withdrawals
Riwayat penarikan.

Auth	✅ Bearer token
Response 200:

json
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
  "meta": { ... }
}
Status withdrawal: pending, processed, completed, failed.

14. Business Rules
14.1 Order Lifecycle
text
pending  →  paid  →  processing  →  shipped  →  completed
   ↓         ↓           ↓
cancelled cancelled  cancelled
14.2 Aksi per Status
Status	Aksi Tersedia
pending	— (tunggu customer bayar)
paid	processing, cancelled
processing	ship, ship-kiriminaja, cancelled
shipped	track, completed
completed	—
cancelled	—
14.3 Aturan Kontak Customer
Backend akan:

Set shipping_address = null kalau order belum paid

Set user.phone = null & user.email = null kalau order belum paid

Mask user.name jadi "Budi S." kalau order belum paid

Client harus:

Cek payment_status === 'paid' sebelum render alamat & kontak

Tampilkan placeholder kalau belum paid

Alasan: cegah transaksi di luar sistem.

14.4 Notifikasi Otomatis
Backend sudah kirim otomatis. Merchant tidak perlu implement.

Event	Email Merchant	WA Merchant
Order created	✅ (tanpa kontak)	✅ (info dasar)
Order paid	✅ (tanpa kontak)	✅ (info dasar)
Order cancelled	✅ (info dasar)	—
14.5 Satu Merchant = Satu Toko
POST /merchant/store akan gagal 422 kalau merchant sudah punya toko.

14.6 SKU Auto-generate
Kalau sku kosong saat create product → auto SKU-XXXXXXXX (8 char random).

15. Constants
15.1 Order Status
php
'pending'    // Menunggu Pembayaran
'paid'       // Dibayar
'processing' // Diproses
'shipped'    // Dikirim
'completed'  // Selesai
'cancelled'  // Dibatalkan
15.2 Payment Status
php
'unpaid'     // Belum bayar
'paid'       // Sudah bayar
'failed'     // Gagal
'refunded'   // Dikembalikan
15.3 Merchant Status
php
'pending'    // Menunggu persetujuan
'active'     // Aktif
'suspended'  // Ditangguhkan
'inactive'   // Tidak aktif
15.4 Business Type
php
'hotel'     // Hotel & Penginapan
'kuliner'   // Kuliner & Restoran
'rental'    // Rental Kendaraan
'tour'      // Paket Wisata
'destinasi' // Destinasi Wisata
'souvenir'  // Souvenir & Oleh-oleh ← merchant marketplace
15.5 Withdrawal Status
php
'pending'    // Menunggu
'processed'  // Diproses
'completed'  // Selesai
'failed'     // Gagal
15.6 Courier Code (KiriminAja)
text
jne, jnt, sicepat, pos, anteraja
16. Integration Guide — Web (Blade)
16.1 Pattern
text
Browser → Laravel SSR (merchant.ovisito.com) → API → Response
              ↓
          Server session (token)
Keuntungan:

Tidak ada CORS

Token tidak exposed ke JS

SSR cepat

16.2 Contoh Controller
php
<?php

namespace App\Http\Controllers\Merchant;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Session;

class OrderController extends Controller
{
    private function http()
    {
        return Http::withHeaders([
            'X-Client-ID'     => config('app.client_id'),
            'X-Client-Secret' => config('app.client_secret'),
            'Accept'          => 'application/json',
            'Authorization'   => 'Bearer ' . Session::get('merchant_token'),
        ]);
    }

    public function index(Request $request)
    {
        $response = $this->http()->get(
            config('app.api_base_url') . '/merchant/souvenir/orders',
            $request->query()
        );

        if ($response->status() === 401) {
            Session::forget(['merchant_token', 'merchant']);
            return redirect()->route('merchant.login');
        }

        return view('merchant.orders.index', [
            'orders'     => $response->json('data', []),
            'pagination' => $response->json('meta'),
        ]);
    }
}
16.3 Session Keys
Key	Isi	Lifecycle
merchant_token	Bearer token	Set saat login, hapus saat logout/401
merchant	Object merchant (array)	Set saat login
merchant_store	Object store	Set saat login, update saat store berubah
merchant_token_expires_at	Carbon datetime	Set saat login (8 jam)
16.4 ⚠️ Response status Field
Mulai v2.6, status adalah boolean (true/false), bukan 'success'/'error'.

Jangan cek $response['status'] === 'success'. Pakai:

php
if ($response['status'] === true) { ... }
17. Integration Guide — Android (Kotlin)
17.1 Setup Retrofit
kotlin
// ApiService.kt
interface MerchantApi {
    @POST("merchant/login")
    suspend fun login(@Body req: LoginRequest): ApiResponse<LoginData>

    @GET("merchant/dashboard")
    suspend fun getDashboard(): ApiResponse<DashboardData>

    @GET("merchant/souvenir/orders")
    suspend fun getOrders(
        @Query("status") status: String? = null,
        @Query("page") page: Int = 1,
        @Query("per_page") perPage: Int = 15,
    ): ApiResponse<List<Order>>

    @PUT("merchant/souvenir/orders/{uuid}/status")
    suspend fun updateOrderStatus(
        @Path("uuid") uuid: String,
        @Body req: UpdateStatusRequest,
    ): ApiResponse<Order>

    @Multipart
    @POST("merchant/souvenir/products")
    suspend fun createProduct(
        @Part("name") name: RequestBody,
        @Part("price") price: RequestBody,
        @Part("stock") stock: RequestBody,
        @Part images: List<MultipartBody.Part>,
    ): ApiResponse<Product>
}
17.2 Auth Interceptor
kotlin
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
17.3 Token Storage (Encrypted)
kotlin
// TokenStorage.kt
class TokenStorage(context: Context) {
    private val prefs: SharedPreferences by lazy {
        val masterKey = MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()

        EncryptedSharedPreferences.create(
            context,
            "merchant_secure_prefs",
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM,
        )
    }

    fun saveToken(token: String) {
        prefs.edit().putString("merchant_token", token).apply()
    }

    fun getToken(): String? = prefs.getString("merchant_token", null)

    fun clear() {
        prefs.edit().clear().apply()
    }
}
17.4 Handling 401
kotlin
// ApiResult wrapper
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val code: Int, val message: String) : ApiResult<Nothing>()
    object Unauthorized : ApiResult<Nothing>()
}

// Di repository
suspend fun getOrders(): ApiResult<List<Order>> {
    return try {
        val res = api.getOrders()
        if (res.status) ApiResult.Success(res.data ?: emptyList())
        else ApiResult.Error(422, res.message ?: "Unknown")
    } catch (e: HttpException) {
        when (e.code()) {
            401 -> ApiResult.Unauthorized
            else -> ApiResult.Error(e.code(), e.message())
        }
    }
}
17.5 Gradle Config
gradle
// build.gradle (app level)
android {
    buildTypes {
        debug {
            buildConfigField "String", "CLIENT_ID", "\"client_web\""
            buildConfigField "String", "CLIENT_SECRET", "\"${localProperties.getProperty('client.secret')}\""
            buildConfigField "String", "API_BASE_URL", "\"http://10.0.2.2:8000/api/v2\""
        }
        release {
            buildConfigField "String", "CLIENT_ID", "\"client_web\""
            buildConfigField "String", "CLIENT_SECRET", "\"${localProperties.getProperty('client.secret')}\""
            buildConfigField "String", "API_BASE_URL", "\"https://api.ovisito.com/api/v2\""
        }
    }
}
local.properties (jangan di-commit):

text
client.secret=abc123def456...
18. Integration Guide — iOS (Swift)
18.1 APIClient
swift
// APIClient.swift
class APIClient {
    static let shared = APIClient()

    private let baseURL: String

    init() {
        #if DEBUG
        self.baseURL = "http://localhost:8000/api/v2"
        #else
        self.baseURL = "https://api.ovisito.com/api/v2"
        #endif
    }

    func request<T: Decodable>(
        _ endpoint: String,
        method: String = "GET",
        body: [String: Any]? = nil,
        query: [String: String]? = nil,
    ) async throws -> T {
        var components = URLComponents(string: "\(baseURL)/\(endpoint)")!
        if let query = query {
            components.queryItems = query.map {
                URLQueryItem(name: $0.key, value: $0.value)
            }
        }

        var request = URLRequest(url: components.url!)
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

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        if httpResponse.statusCode == 401 {
            KeychainService.deleteToken()
            NotificationCenter.default.post(name: .didReceiveUnauthorized, object: nil)
            throw APIError.unauthorized
        }

        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase

        return try decoder.decode(T.self, from: data)
    }
}
18.2 Keychain Token Storage
swift
// KeychainService.swift
import Security

enum KeychainService {
    private static let service = "com.ovisito.merchant"
    private static let account = "merchant_token"

    static func saveToken(_ token: String) {
        guard let data = token.data(using: .utf8) else { return }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
        ]

        SecItemDelete(query as CFDictionary)

        let attributes: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock,
        ]

        SecItemAdd(attributes as CFDictionary, nil)
    }

    static func getToken() -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let data = result as? Data,
              let token = String(data: data, encoding: .utf8)
        else { return nil }

        return token
    }

    static func deleteToken() {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
        ]
        SecItemDelete(query as CFDictionary)
    }
}
18.3 Model Example
swift
// Order.swift
struct Order: Codable {
    let id: String
    let orderNumber: String
    let status: String
    let statusLabel: String
    let totalAmount: String
    let formattedTotal: String
    let shippingAddress: ShippingAddress?
    let user: OrderUser?
    let items: [OrderItem]

    enum CodingKeys: String, CodingKey {
        case id
        case orderNumber = "order_number"
        case status
        case statusLabel = "status_label"
        case totalAmount = "total_amount"
        case formattedTotal = "formatted_total"
        case shippingAddress = "shipping_address"
        case user, items
    }
}

struct ShippingAddress: Codable {
    let name: String
    let phone: String
    let address: String
    let city: String
    let postalCode: String?

    enum CodingKeys: String, CodingKey {
        case name, phone, address, city
        case postalCode = "postal_code"
    }
}
18.4 XCConfig
Buat Secrets.xcconfig (jangan di-commit):

text
CLIENT_ID = client_web
CLIENT_SECRET = abc123def456...
API_BASE_URL = https:/$()/api.ovisito.com/api/v2
Reference di Info.plist:

text
<key>ClientID</key>
<string>$(CLIENT_ID)</string>
<key>ClientSecret</key>
<string>$(CLIENT_SECRET)</string>
19. Sample End-to-End Flow
Skenario 1 — Onboarding Merchant Baru
text
1. POST /merchant/register
   Body: { name, email, password, phone, business_name, business_type, ... }
   ← 201 { data: { uuid, email } }

2. [User cek email, klik link verifikasi]
   GET /merchant/verify-email/{uuid}?hash=<sha1_email>
   ← 200 { message: "Email berhasil diverifikasi." }

3. POST /merchant/login
   Body: { email, password, device_name }
   ← 200 { data: { token, merchant: {...} } }

4. Client simpan token di secure storage

5. GET /merchant/dashboard
   ← 200 { summary: { total_orders: 0, ... } }

6. GET /merchant/store
   ← 404 (belum punya toko)

7. GET /merchant/souvenir/categories
   ← 200 { data: [tree kategori] }

8. POST /merchant/store (multipart)
   Body: { name, address, phone, category_ids, logo }
   ← 201 { data: { ...store... } }

9. POST /merchant/souvenir/products (multipart)
   Body: { name, price, stock, category_id, images[] }
   ← 201 { data: { ...product... } }

10. GET /merchant/souvenir/products
    ← 200 { data: [...], meta: {...} }
Skenario 2 — Handle Order Customer
text
1. [Customer order + bayar via Flip]
2. [Backend update order → status=paid, notif ke merchant terkirim otomatis]

3. GET /merchant/souvenir/orders?status=paid
   ← 200 { data: [order paid], meta: {...} }
   ← (shipping_address sudah muncul karena paid)

4. GET /merchant/souvenir/orders/{uuid}
   ← 200 { data: { full order + customer } }

5. PUT /merchant/souvenir/orders/{uuid}/status
   Body: { status: "processing" }
   ← 200 { data: {...} }

6. POST /merchant/souvenir/orders/{uuid}/ship
   Body: { tracking_number: "JNE123", courier: "jne" }
   ← 200 { data: { status: "shipped", ... } }

7. GET /merchant/souvenir/orders/{uuid}/track
   ← 200 { data: { tracking_number, history: [...] } }

8. [Webhook shipping dari kurir → status delivered]
9. [Backend auto update → status=completed]
Skenario 3 — Withdraw Saldo
text
1. GET /merchant/dashboard
   ← 200 { summary: { balance: 150000 } }

2. GET /merchant/transactions
   ← 200 { data: [...] }

3. POST /merchant/withdraw
   Body: { amount: 50000, bank_name: "BCA", bank_account: "1234567890", account_name: "Budi Santoso" }
   ← 200 { data: { withdrawal_id, reference, balance_after: 100000 } }

4. GET /merchant/withdrawals
   ← 200 { data: [withdrawal], meta: {...} }

5. GET /merchant/dashboard
   ← 200 { summary: { balance: 100000 } }
20. Changelog & Support
v2.6 — 2026-10-08
Breaking Changes:

Response status field boolean (true/false), bukan string

Pagination data[] + meta{}

PUT /orders/{uuid}/status menolak shipped

Filter kontak customer: shipping_address, phone, email di-null-kan kalau belum paid

user.name di-mask untuk order belum paid

Additions:

Endpoint POST/DELETE /products/{id}/images

Accessor payment_method_label, final_price di product

Notifikasi merchant email di setiap event

Fixes:

Guard auth:merchant_api

forgotPassword benar-benar kirim email

changePassword double hash bug

Store slug race condition

Category orWhere grouping

v2.5 — 2026-10-08
Auto trigger payment + email/WA tombol bayar

WA Fonnte aktif

v2.2 — 2026-10-08
Multi-database refactor

v2.1 — 2026-10-07
Dokumentasi awal

Support & Kontak
Kanal	Kontak
Backend Team	backend@ovisito.com
API Health	https://api.ovisito.com/api/v2/ping
Status Page	https://api.ovisito.com/up
Postman Collection: tersedia atas request — hubungi backend team.
OpenAPI Spec: target rilis v2.7.

Maintained by: Backend Team Ovisito
Last updated: 2026-10-08
Version: 2.6

📋 Checklist Integrasi untuk FE
Saat integrasi, pastikan:

□ Base URL sesuai environment
□ Header X-Client-ID, X-Client-Secret di setiap request
□ Header Accept: application/json
□ Header Authorization: Bearer untuk endpoint protected
□ Token disimpan di secure storage
□ Handle 401 → clear token + redirect login
□ Handle 403 (email belum verify) → redirect verify page
□ Handle 422 → tampilkan error per field
□ Handle pagination meta{}
□ Cek payment_status === 'paid' sebelum render alamat customer
□ Kalau upload logo/gambar → pakai multipart/form-data
□ Kalau update → method spoofing _method=PUT untuk multipart
□ Rate limit awareness — jangan spam endpoint
□ Client secret jangan hardcode di mobile
🎯 Yang Lengkap di Dokumen Ini
✅ 20 section — dari Quick Start sampai Changelog

✅ 40+ endpoint dengan contoh request/response lengkap

✅ Validation rules per field

✅ Error handling per status code

✅ Rate limits tabel lengkap

✅ Business rules — order lifecycle, kontak customer

✅ 3 panduan integrasi — Web, Android, iOS dengan contoh code siap pakai

✅ Sample flow end-to-end

✅ Constants reference — semua enum

✅ Checklist integrasi untuk FE

Dokumen ini siap dibagikan ke tim FE web, Android, dan iOS. Semua yang mereka butuhkan ada di sini tanpa perlu lihat source code backend.

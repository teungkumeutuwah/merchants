Public & Customer Endpoints (Final v2.6)
Siap replace di marketplacev2.5.md → rename jadi marketplacev2.6.md

Perubahan utama vs draft sebelumnya:

✅ Tambah final_price di semua response produk

✅ Response status boolean (true/false)

✅ Cancel rules: pending, paid, processing (sesuai model)

✅ Tambah catatan integrasi mobile (Android, iOS)

✅ Tambah error handling lengkap

6. Public Endpoints
Base: /api/v2/public/souvenir
Auth: client.auth (header X-Client-ID + X-Client-Secret), throttle:120,1
Login User: ❌ Tidak perlu

Semua endpoint di section ini tidak butuh login. FE customer (browsing) bisa akses langsung.

6.1 Products
GET /products
List produk aktif dengan filter lengkap.

Query Parameters:

Param	Tipe	Default	Deskripsi
search	string	—	Cari nama & deskripsi produk
category	string	—	Slug atau UUID kategori (auto-include sub-kategori 2 level)
merchant_uuid	uuid	—	Filter per merchant
featured	boolean	false	Hanya produk unggulan
min_price	numeric	—	Harga minimum (Rp)
max_price	numeric	—	Harga maksimum (Rp)
in_stock	boolean	false	Hanya yang ada stok
sort	enum	latest	latest | price_asc | price_desc | popular
per_page	integer	12	Max 60
page	integer	1	—
Contoh Request:

http
GET /api/v2/public/souvenir/products?search=kaos&category=pakaian&min_price=50000&sort=price_asc&per_page=12
Response 200:

json
{
  "status": true,
  "data": [
    {
      "id": "f49f9e1e-04f9-4a40-9395-29ea0dda0bf3",
      "merchant_uuid": "9d77266e-a0ac-45de-9320-c12323c6add9",
      "store_id": "01a08c17-ee6e-70f7-bb22-96f4fac75ecd",
      "category_id": "ec0b0021-a8eb-11f1-bf0a-78ac4432fe0b",
      "name": "KAOS EXCLUSIVE – ULANG TAHUN FAKULTAS TEKNIK USK KE-63",
      "slug": "kaos-exclusive-ulang-tahun-fakultas-teknik-usk-ke-63",
      "sku": "SKU-RNBTNWFJ",
      "description": "Kaos eksklusif edisi terbatas...",
      "price": "150000.00",
      "discount_price": "120000.00",
      "final_price": 120000,
      "stock": 92,
      "weight": "0.10",
      "images": [
        "souvenir/products/xxx.png",
        "souvenir/products/yyy.png"
      ],
      "is_active": true,
      "featured": false,
      "views": 16,
      "category": {
        "id": "ec0b0021-a8eb-11f1-bf0a-78ac4432fe0b",
        "name": "Kaos & Jaket Komunitas",
        "slug": "kaos-jaket-komunitas",
        "parent_id": "ec0b0020-a8eb-11f1-bf0a-78ac4432fe0b"
      },
      "store": {
        "id": "01a08c17-ee6e-70f7-bb22-96f4fac75ecd",
        "name": "Merchandise Store",
        "slug": "merchandise-store",
        "logo": "stores/.../logo.png"
      },
      "merchant": {
        "uuid": "9d77266e-a0ac-45de-9320-c12323c6add9",
        "name": "Muhammad Aiyub",
        "business_name": "Ade Kak Nah"
      },
      "created_at": "2026-10-04T18:55:42.000000Z"
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 2,
    "per_page": 12,
    "total": 15
  }
}
Field penting untuk FE:

Field	Tipe	Kegunaan
price	string	Harga normal ("150000.00")
discount_price	string | null	Harga diskon ("120000.00")
final_price	float	Harga final untuk display — sudah dihitung backend (discount jika ada, else price)
images	array string	Path relatif — gabung dengan base storage URL
stock	int	Jumlah stok tersedia
⭐ Rekomendasi FE: langsung pakai final_price untuk display harga. Jangan hitung manual di client.

Fitur spesial category filter:

Filter pakai slug atau UUID — keduanya jalan

Otomatis include sub-kategori (1 level) + sub-sub-kategori (2 level)

Contoh: category=pakaian → tampilkan produk dari pakaian, kaos, kemeja, dst

Contoh Filter Kategori Lengkap:

http
# Filter dengan slug
GET /products?category=pakaian

# Filter dengan UUID
GET /products?category=ec0b0021-a8eb-11f1-bf0a-78ac4432fe0b

# Cari kaos dengan stok ada, urut harga termurah
GET /products?search=kaos&in_stock=true&sort=price_asc&per_page=20
GET /products/{slug}
Detail produk + review + produk terkait.

Path Parameter:

Param	Deskripsi
slug	Slug produk (contoh: kaos-exclusive-ulang-tahun-fakultas-teknik-usk-ke-63)
Response 200:

json
{
  "status": true,
  "data": {
    "product": {
      "id": "f49f9e1e-...",
      "name": "KAOS EXCLUSIVE...",
      "slug": "kaos-exclusive...",
      "sku": "SKU-RNBTNWFJ",
      "description": "Kaos eksklusif edisi terbatas...",
      "price": "150000.00",
      "discount_price": "120000.00",
      "final_price": 120000,
      "stock": 92,
      "weight": "0.10",
      "images": [
        "souvenir/products/xxx.png",
        "souvenir/products/yyy.png"
      ],
      "views": 17,
      "is_active": true,
      "featured": false,
      "category": {
        "id": "ec0b0021-...",
        "name": "Kaos & Jaket Komunitas",
        "slug": "kaos-jaket-komunitas",
        "parent_id": "ec0b0020-..."
      },
      "store": {
        "id": "01a08c17-...",
        "name": "Merchandise Store",
        "slug": "merchandise-store",
        "phone": "08778878889",
        "address": "Aceh, Banda Aceh",
        "categories": [
          {
            "id": "...",
            "name": "Kaos & Pakaian Souvenir",
            "slug": "kaos-pakaian-souvenir",
            "pivot": {
              "store_id": "...",
              "category_id": "..."
            }
          }
        ]
      },
      "merchant": {
        "uuid": "9d77266e-...",
        "name": "Muhammad Aiyub",
        "business_name": "Ade Kak Nah"
      },
      "reviews": [
        {
          "id": "uuid",
          "rating": 5,
          "review": "Kualitas bagus, jahitan rapi!",
          "is_verified_purchase": true,
          "images": [],
          "user": {
            "uuid": "...",
            "name": "Budi"
          },
          "created_at": "2026-10-05T10:00:00.000000Z"
        }
      ]
    },
    "related": [
      {
        "id": "...",
        "name": "Produk Serupa",
        "slug": "produk-serupa",
        "price": "140000.00",
        "discount_price": null,
        "final_price": 140000,
        "images": ["souvenir/products/zzz.png"]
      }
    ]
  }
}
Fitur:

views auto-increment setiap kali diakses

reviews — max 10 review terbaru

related — max 8 produk kategori sama (kecuali produk ini sendiri)

Response 404:

json
{
  "status": false,
  "message": "Produk tidak ditemukan."
}
GET /featured
Produk unggulan (featured = true).

Query Parameters:

Param	Tipe	Default	Max
limit	integer	8	24
Response 200:

json
{
  "status": true,
  "data": [
    {
      "id": "...",
      "name": "Produk Unggulan 1",
      "slug": "...",
      "price": "150000.00",
      "discount_price": "120000.00",
      "final_price": 120000,
      "images": ["souvenir/products/..."]
    },
    {
      "id": "...",
      "name": "Produk Unggulan 2",
      "slug": "...",
      "price": "200000.00",
      "discount_price": null,
      "final_price": 200000,
      "images": ["souvenir/products/..."]
    }
  ]
}
6.2 Categories
GET /categories
List kategori dengan hierarki 2 level (parent → children).

Query Parameters:

Param	Tipe	Default	Deskripsi
with_children	boolean	true	Include sub-kategori
only_parents	boolean	true	Hanya kategori root (parent_id = null)
Contoh Request:

http
# Default: hanya root + children
GET /categories

# Flat (semua kategori, tidak ada nesting)
GET /categories?with_children=false&only_parents=false

# Hanya root, tanpa children
GET /categories?with_children=false
Response 200 (default):

json
{
  "status": true,
  "data": [
    {
      "id": "eb0a0001-a8eb-11f1-bf0a-78ac4432fe0b",
      "name": "Kaos & Pakaian Souvenir",
      "slug": "kaos-pakaian-souvenir",
      "description": "Kaos, polo, kemeja, dan pakaian souvenir khas Aceh.",
      "parent_id": null,
      "is_active": true,
      "children": [
        {
          "id": "eb0a0002-a8eb-11f1-bf0a-78ac4432fe0b",
          "name": "Kaos Oleh-Oleh Khas Aceh",
          "slug": "kaos-oleh-oleh-khas-aceh",
          "children": []
        },
        {
          "id": "eb0a0003-a8eb-11f1-bf0a-78ac4432fe0b",
          "name": "Polo Shirt Custom Aceh",
          "slug": "polo-shirt-custom-aceh",
          "children": []
        }
      ]
    }
  ]
}
Struktur hierarki maksimal 2 level:

text
Root Category (parent_id: null)
    ├─ Child (parent_id: root_id)
    │   └─ Grandchild (parent_id: child_id)   ← ditampilkan juga
    └─ Child
Rekomendasi FE: cache response kategori di local storage (AsyncStorage Android, UserDefaults iOS, IndexedDB web). Kategori jarang berubah — refresh 1×/hari atau saat user buka halaman kategori.

GET /categories/{slug}
Detail kategori + produk dalamnya (auto-include sub-kategori).

Path Parameter:

Param	Deskripsi
slug	Slug kategori (contoh: kaos-pakaian-souvenir)
Query Parameters:

Param	Tipe	Default	Deskripsi
sort	enum	latest	latest | price_asc | price_desc | popular
per_page	integer	12	Max 60
page	integer	1	—
Response 200:

json
{
  "status": true,
  "data": {
    "category": {
      "id": "eb0a0001-...",
      "name": "Kaos & Pakaian Souvenir",
      "slug": "kaos-pakaian-souvenir",
      "description": "...",
      "parent_id": null,
      "is_active": true,
      "children": [
        {
          "id": "...",
          "name": "Kaos Oleh-Oleh",
          "slug": "kaos-oleh-oleh-khas-aceh"
        }
      ]
    },
    "products": [
      {
        "id": "...",
        "name": "Produk 1",
        "slug": "...",
        "price": "150000.00",
        "discount_price": "120000.00",
        "final_price": 120000,
        "images": ["souvenir/products/..."]
      }
    ],
    "meta": {
      "current_page": 1,
      "last_page": 1,
      "per_page": 12,
      "total": 0
    }
  }
}
⚠️ Catatan: Produk yang ditampilkan termasuk produk dari sub-kategori. Kalau filter kategori pakaian punya sub-kategori kaos + kemeja, produk dari ketiganya akan tampil.

6.3 Stores
GET /stores
List toko aktif.

Query Parameters:

Param	Tipe	Default	Deskripsi
search	string	—	Cari nama & deskripsi toko
merchant_uuid	uuid	—	Filter per merchant
kabupaten_id	integer	—	Filter per kabupaten/kota
is_physical	boolean	—	true = toko fisik, false = online
per_page	integer	12	Max 60
page	integer	1	—
Response 200:

json
{
  "status": true,
  "data": [
    {
      "id": "01a08c17-ee6e-70f7-bb22-96f4fac75ecd",
      "merchant_uuid": "9d77266e-a0ac-45de-9320-c12323c6add9",
      "name": "Merchandise Store",
      "slug": "merchandise-store",
      "description": "Toko Souvenir, merchandise dan oleh oleh",
      "address": "Aceh, Banda Aceh, Aceh",
      "phone": "08778878889",
      "email": "ceo@ovisito.com",
      "logo": "stores/.../logo.png",
      "jam_buka": "08:00:00",
      "jam_tutup": "22:00:00",
      "is_physical": true,
      "is_active": true,
      "is_default": true,
      "category_ids": [
        "eb0a0001-a8eb-11f1-bf0a-78ac4432fe0b",
        "ec0b0040-a8eb-11f1-bf0a-78ac4432fe0b"
      ],
      "merchant": {
        "uuid": "9d77266e-...",
        "name": "Muhammad Aiyub",
        "business_name": "Ade Kak Nah"
      }
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 1,
    "per_page": 12,
    "total": 2
  }
}
GET /stores/{slug}
Detail toko + 12 produk terbaru.

Path Parameter:

Param	Deskripsi
slug	Slug toko (contoh: merchandise-store)
Response 200:

json
{
  "status": true,
  "data": {
    "store": {
      "id": "...",
      "name": "Merchandise Store",
      "slug": "merchandise-store",
      "description": "Toko Souvenir, merchandise dan oleh oleh",
      "address": "Aceh, Banda Aceh, Aceh",
      "phone": "08778878889",
      "email": "ceo@ovisito.com",
      "logo": "stores/.../logo.png",
      "jam_buka": "08:00:00",
      "jam_tutup": "22:00:00",
      "is_physical": true,
      "is_active": true,
      "categories": [
        {
          "id": "...",
          "name": "Kaos & Pakaian Souvenir",
          "slug": "kaos-pakaian-souvenir",
          "pivot": {
            "store_id": "...",
            "category_id": "..."
          }
        }
      ],
      "merchant": {
        "uuid": "9d77266e-...",
        "name": "Muhammad Aiyub",
        "business_name": "Ade Kak Nah"
      }
    },
    "products": [
      {
        "id": "...",
        "name": "Produk 1",
        "slug": "...",
        "price": "120000.00",
        "discount_price": null,
        "final_price": 120000,
        "images": ["souvenir/products/..."]
      },
      {
        "id": "...",
        "name": "Produk 2",
        "slug": "...",
        "price": "150000.00",
        "discount_price": "140000.00",
        "final_price": 140000,
        "images": ["souvenir/products/..."]
      }
    ]
  }
}
6.4 Shipping Methods
GET /shipping-methods
List metode pengiriman aktif.

Response 200:

json
{
  "status": true,
  "data": [
    {
      "id": "uuid",
      "name": "JNE Reguler",
      "courier_code": "jne",
      "description": "Estimasi 2-3 hari",
      "base_cost": "15000.00",
      "is_active": true
    },
    {
      "id": "uuid",
      "name": "J&T Express",
      "courier_code": "jnt",
      "base_cost": "18000.00",
      "is_active": true
    }
  ]
}
⚠️ Kalau array data kosong ([]), artinya admin belum setup metode pengiriman. FE wajib handle empty state ini — tampilkan pesan "Metode pengiriman belum tersedia" dan disable tombol checkout.

POST /shipping-methods/calculate
Hitung biaya pengiriman.

Request Body:

json
{
  "weight": 500,
  "method_id": "uuid"
}
Field	Tipe	Wajib	Deskripsi
weight	numeric	✅	Berat dalam gram
method_id	string	✅	UUID dari /shipping-methods
Response 200:

json
{
  "status": true,
  "data": {
    "method": "JNE Reguler",
    "courier_code": "jne",
    "weight": 500,
    "cost": 15500,
    "cost_formatted": "Rp 15.500"
  }
}
Response 404:

json
{
  "status": false,
  "message": "Metode pengiriman tidak ditemukan atau tidak aktif."
}
Response 422:

json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "method_id": ["The selected method id is invalid."],
    "weight": ["The weight field is required."]
  }
}
7. Customer Endpoints
Base: /api/v2/customer
Auth: client.auth (header) + auth:sanctum (Bearer user token)
Login User: ✅ Wajib

Semua endpoint di section ini butuh user login (Sanctum token).

7.0 Cara Dapat User Token
Endpoint: POST /api/v2/user/login

Request Body:

json
{
  "email": "user@example.com",
  "password": "password123",
  "device_name": "web-customer"
}
device_name rekomendasi per platform:

Web: "web-customer"

Android: "android-customer-{device_model}"

iOS: "ios-customer-{device_name}"

Response 200:

json
{
  "status": true,
  "message": "Login berhasil",
  "data": {
    "token": "172|YPq0vQ2HnuEaCbd0lrRC9cmZX9g8Z7eWmzj02Jys88132c9a",
    "user": {
      "uuid": "bbb6859f-7de1-4d35-899c-eef4cf6b6732",
      "name": "Muhammad Aiyub",
      "email": "aiyub@ovisito.com",
      "phone": "+6281370313188",
      "status": "active"
    }
  }
}
Pakai token di header:

http
Authorization: Bearer 172|YPq0vQ2HnuEaCbd0lrRC9cmZX9g8Z7eWmzj02Jys88132c9a
⚠️ Token storage per platform:

Platform	Tempat Simpan
Web (Blade SSR)	Server session
Web (SPA)	httpOnly cookie atau sessionStorage (bukan localStorage)
Android	EncryptedSharedPreferences
iOS	Keychain
7.1 Orders
POST /orders ⭐
Buat order baru. Auto-trigger payment ke Flip.

Headers:

http
X-Client-ID: client_web
X-Client-Secret: <secret>
Authorization: Bearer <user_token>
Content-Type: application/json
Accept: application/json
Request Body:

json
{
  "merchant_uuid": "9d77266e-a0ac-45de-9320-c12323c6add9",
  "items": [
    {
      "product_uuid": "f49f9e1e-04f9-4a40-9395-29ea0dda0bf3",
      "quantity": 1
    }
  ],
  "shipping_address": {
    "name": "Muhammad Aiyub",
    "phone": "6281360313113",
    "address": "Jl. Test No. 1",
    "city": "Banda Aceh",
    "postal_code": "23116"
  },
  "shipping_cost": 15000,
  "courier": "jne",
  "notes": "Opsional"
}
Field Wajib:

Field	Tipe	Validasi
merchant_uuid	uuid	Wajib, max 36
items	array	Wajib, min 1 item
items.*.product_uuid	uuid	Wajib
items.*.quantity	integer	Min 1, max 999
shipping_address.name	string	Wajib, max 255
shipping_address.phone	string	Wajib, format 628xxx ⚠️
shipping_address.address	string	Wajib, max 500
shipping_address.city	string	Opsional
shipping_address.postal_code	string	Opsional, max 10
shipping_cost	numeric	Opsional, min 0
courier	string	Opsional, max 100
notes	string	Opsional, max 1000
⚠️ Catatan shipping_address.phone:

Harus format 628xxx (bukan 08xxx)

Dipakai sebagai target WA notifikasi customer

Kalau format 08xxx, service akan auto-normalize ke 628xxx

Business Rules:

Satu order = satu merchant (semua item harus dari merchant_uuid yang sama)

Stock di-decrement atomic (lockForUpdate) — cegah oversell

Harga pakai final_price — discount_price jika ada, else price

Total = sum(subtotal) + shipping_cost - discount_total

Status awal: pending, payment_status: unpaid

Auto-generate order_number: format SO-YYYYMMDD-XXXXXXXX

⭐ Auto-trigger Flip → customer dapat payment_url langsung

⭐ Auto-kirim email + WA dengan tombol bayar

Response 201 (Sukses):

json
{
  "status": true,
  "message": "Pesanan berhasil dibuat. Silakan lanjut ke pembayaran.",
  "data": {
    "order": {
      "id": "01a11b5c-a4c3-708a-9383-875a484f8753",
      "order_number": "SO-20261008-XIMAMZL3",
      "user_uuid": "c4ec7a92-610b-4f17-82ea-c00d11d52d69",
      "merchant_uuid": "9d77266e-a0ac-45de-9320-c12323c6add9",
      "total_amount": "135000.00",
      "shipping_cost": "15000.00",
      "discount_total": "0.00",
      "status": "pending",
      "payment_status": "unpaid",
      "flip_bill_id": "362xxx",
      "shipping_address": {
        "name": "Muhammad Aiyub",
        "phone": "6281360313113",
        "address": "Jl. Test No. 1",
        "city": "Banda Aceh",
        "postal_code": "23116"
      },
      "courier": "jne",
      "items": [
        {
          "id": "uuid",
          "product_uuid": "f49f9e1e-...",
          "product_name": "KAOS EXCLUSIVE...",
          "price": "120000.00",
          "quantity": 1,
          "subtotal": "120000.00",
          "product": {
            "id": "f49f9e1e-...",
            "name": "KAOS EXCLUSIVE...",
            "slug": "kaos-exclusive...",
            "images": ["souvenir/products/..."]
          }
        }
      ],
      "merchant": {
        "uuid": "9d77266e-...",
        "name": "Muhammad Aiyub",
        "business_name": "Ade Kak Nah"
      },
      "ordered_at": "2026-10-08T11:53:38.000000Z",
      "created_at": "2026-10-08T11:53:38.000000Z"
    },
    "payment": {
      "next_step": "POST /api/v2/customer/payment/process",
      "booking_type": "marketplace",
      "booking_id": "SO-20261008-XIMAMZL3",
      "payment_url": "https://flip.id/pwf-sandbox/$muhammadaiyub/#pembayaran..."
    }
  }
}
Error 422 (Validasi Gagal):

json
{
  "status": false,
  "message": "Validasi gagal.",
  "errors": {
    "merchant_uuid": ["The merchant uuid field is required."],
    "items": ["The items field is required."],
    "shipping_address.phone": ["The shipping address.phone field is required."]
  }
}
Error 422 (Business Logic — Stok Tidak Cukup):

json
{
  "status": false,
  "message": "Stok 'KAOS EXCLUSIVE' tidak cukup (tersedia 50)."
}
Error 422 (Business Logic — Produk Bukan Milik Merchant):

json
{
  "status": false,
  "message": "Produk 'Kopi Aceh' bukan milik merchant yang dipilih."
}
Error 401 (Unauthorized):

json
{
  "status": false,
  "message": "Unauthenticated. Silakan login terlebih dahulu."
}
Flow Notifikasi Otomatis:

text
1. Order dibuat (status: pending, flip_bill_id: null)
    ↓
2. Observer::created() → skip notif (tunggu flip_bill_id)
    ↓
3. AUTO panggil PaymentService::processPayment()
    ↓
4. Flip API return bill_id + payment_url
    ↓
5. Set flip_bill_id di order → trigger Observer::updated()
    ↓
6. 📧 Email "Pesanan Diterima + 💳 Bayar Sekarang →" ke customer
7. 📧 Email "Pesanan Diterima" ke merchant
8. 📱 WA "Pesanan Diterima + Link Bayar" ke customer
9. 📱 WA ke merchant (skip kalau nomor invalid)
Fallback: Kalau auto-payment gagal (mis. Flip down), order tetap dibuat, notif tetap terkirim tanpa payment_url. Customer bisa retry via POST /payment/process.

GET /orders
List order milik customer yang login.

Query Parameters:

Param	Tipe	Default	Deskripsi
status	enum	—	Filter: pending | paid | processing | shipped | completed | cancelled
per_page	integer	15	Max 50
page	integer	1	—
Response 200:

json
{
  "status": true,
  "data": [
    {
      "id": "uuid",
      "order_number": "SO-20261008-HYNTEBQ5",
      "status": "cancelled",
      "status_label": "Dibatalkan",
      "status_badge_class": "bg-red-100 text-red-800",
      "payment_status": "unpaid",
      "payment_status_label": "Menunggu Pembayaran",
      "total_amount": "135000.00",
      "formatted_total": "Rp 135.000",
      "shipping_cost": "15000.00",
      "discount_total": "0.00",
      "payment_method": null,
      "flip_bill_id": null,
      "shipping_address": {
        "name": "Muhammad Aiyub",
        "phone": "6281360313113",
        "address": "Jl. Test No. 1",
        "city": "Banda Aceh",
        "postal_code": "23116"
      },
      "courier": "jne",
      "tracking_number": null,
      "shipping_order_id": null,
      "shipped_at": null,
      "estimated_delivery_at": null,
      "delivery_note": null,
      "notes": "Fix Flip config",
      "ordered_at": "2026-10-07T18:57:21.000000Z",
      "paid_at": null,
      "completed_at": null,
      "cancelled_at": "2026-10-07T18:59:08.000000Z",
      "items": [
        {
          "id": "uuid",
          "product_uuid": "...",
          "product_name": "KAOS EXCLUSIVE...",
          "quantity": 1,
          "price": "120000.00",
          "subtotal": "120000.00",
          "product": {
            "id": "...",
            "name": "KAOS EXCLUSIVE...",
            "slug": "...",
            "images": ["souvenir/products/..."]
          }
        }
      ],
      "merchant": {
        "uuid": "...",
        "name": "Muhammad Aiyub",
        "business_name": "Ade Kak Nah"
      },
      "shipping_trackings": []
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 1,
    "per_page": 15,
    "total": 3
  }
}
GET /orders/{order_number}
Detail order + relasi lengkap.

Path Parameter:

Param	Deskripsi
order_number	Format SO-YYYYMMDD-XXXXXXXX
Response 200:

json
{
  "status": true,
  "data": {
    "id": "uuid",
    "order_number": "SO-20261008-HYNTEBQ5",
    "status": "pending",
    "status_label": "Menunggu Pembayaran",
    "payment_status": "unpaid",
    "payment_status_label": "Menunggu Pembayaran",
    "total_amount": "135000.00",
    "formatted_total": "Rp 135.000",
    "items": [
      {
        "id": "uuid",
        "product_uuid": "...",
        "product_name": "KAOS EXCLUSIVE...",
        "quantity": 1,
        "price": "120000.00",
        "formatted_price": "Rp 120.000",
        "subtotal": "120000.00",
        "formatted_subtotal": "Rp 120.000",
        "product": {
          "id": "...",
          "name": "KAOS EXCLUSIVE...",
          "slug": "...",
          "images": ["souvenir/products/..."],
          "category": {
            "id": "...",
            "name": "Kaos & Jaket Komunitas",
            "slug": "kaos-jaket-komunitas"
          }
        }
      }
    ],
    "merchant": {
      "uuid": "...",
      "name": "Muhammad Aiyub",
      "business_name": "Ade Kak Nah",
      "verified_status": "verified"
    },
    "shipping_trackings": []
  }
}
Error 404:

json
{
  "status": false,
  "message": "Pesanan tidak ditemukan."
}
POST /orders/{order_number}/cancel
Batalkan order.

Path Parameter:

Param	Deskripsi
order_number	Format SO-YYYYMMDD-XXXXXXXX
Request Body (opsional):

json
{
  "reason": "Berubah pikiran"
}
Rules:

Bisa cancel kalau status = pending, paid, atau processing

Stock otomatis dikembalikan ke produk (atomic)

Set cancelled_at

Alasan di-append ke field notes

Response 200:

json
{
  "status": true,
  "message": "Pesanan berhasil dibatalkan.",
  "data": {
    "order_number": "SO-20261008-HYNTEBQ5",
    "status": "cancelled",
    "cancelled_at": "2026-10-07T18:59:08.000000Z",
    "notes": "Fix Flip config"
  }
}
Error 422 — Tidak bisa dibatalkan:

json
{
  "status": false,
  "message": "Pesanan tidak dapat dibatalkan (sudah dikirim atau selesai)."
}
Error 404:

json
{
  "status": false,
  "message": "Pesanan tidak ditemukan."
}
GET /orders/{order_number}/track
Lacak status order + history tracking.

Path Parameter:

Param	Deskripsi
order_number	Format SO-YYYYMMDD-XXXXXXXX
Response 200 (sudah dikirim):

json
{
  "status": true,
  "data": {
    "order_number": "SO-20261008-HYNTEBQ5",
    "status": "shipped",
    "status_label": "Dikirim",
    "courier": "jne",
    "tracking_number": "JNE1234567890",
    "history": [
      {
        "id": "uuid",
        "status": "shipped",
        "description": "Paket telah dikirim melalui jne dengan nomor resi JNE1234567890",
        "location": "Banda Aceh",
        "tracked_at": "2026-10-08T11:00:00.000000Z"
      }
    ]
  }
}
Response 200 (belum dikirim):

json
{
  "status": true,
  "data": {
    "order_number": "SO-20261008-HYNTEBQ5",
    "status": "pending",
    "status_label": "Menunggu Pembayaran",
    "courier": "jne",
    "tracking_number": null,
    "history": []
  }
}
7.2 Payment
POST /payment/process
Panggil Flip untuk buat bill + dapat payment_url.

⚠️ Catatan: Endpoint ini jarang dipakai manual karena POST /orders sudah auto-trigger. Tapi berguna untuk:

Retry kalau auto-payment gagal

Customer ganti metode pembayaran

Re-generate payment URL yang expired

Request Body:

json
{
  "booking_type": "marketplace",
  "booking_id": "SO-20261008-HYNTEBQ5",
  "payment_method": "qris",
  "payment_channel": null
}
Field	Tipe	Wajib	Deskripsi
booking_type	string	✅	Nilai: marketplace
booking_id	string	✅	Format SO-YYYYMMDD-XXXXXXXX
payment_method	string	✅	qris | bank_transfer | ewallet
payment_channel	string	❌	Spesifik: bca, bni, gopay, dll
Response 200:

json
{
  "status": true,
  "message": "Pembayaran berhasil diproses. Silakan selesaikan pembayaran.",
  "data": {
    "booking_code": "SO-20261008-HYNTEBQ5",
    "booking_type": "marketplace",
    "amount": "135000.00",
    "status": "pending",
    "payment_url": "https://flip.id/pwf-sandbox/$muhammadaiyub/#pembayaran...",
    "qr_code": "data:image/png;base64,iVBOR...",
    "flip_bill_id": "362438"
  }
}
Fitur idempotent: Kalau bill aktif sudah ada (belum expired), akan reuse bill lama — tidak create baru.

Error 404:

json
{
  "status": false,
  "message": "Booking tidak ditemukan."
}
Error 422 — Sudah Lunas:

json
{
  "status": false,
  "message": "Booking ini sudah lunas."
}
GET /payment/status/{bookingCode}
Cek status pembayaran terbaru.

Path Parameter:

Param	Deskripsi
bookingCode	Format SO-YYYYMMDD-XXXXXXXX
Response 200 (sudah bayar):

json
{
  "status": true,
  "data": {
    "booking_code": "SO-20261008-HYNTEBQ5",
    "is_paid": true,
    "amount": 135000,
    "status": "paid",
    "booking_type": "marketplace"
  }
}
Response 200 (belum bayar):

json
{
  "status": true,
  "data": {
    "booking_code": "SO-20261008-HYNTEBQ5",
    "is_paid": false,
    "amount": 135000,
    "status": "pending",
    "booking_type": "marketplace"
  }
}
Rekomendasi FE: polling endpoint ini setiap 30 detik saat customer di halaman pembayaran. Stop polling kalau is_paid = true.

GET /payment/qr/{bookingCode}
Generate QR code lokal (bukan QRIS Flip).

Path Parameter:

Param	Deskripsi
bookingCode	Format SO-YYYYMMDD-XXXXXXXX
Response 200:

json
{
  "status": true,
  "data": {
    "booking_code": "SO-20261008-HYNTEBQ5",
    "qr_code": "data:image/png;base64,...",
    "expires_in": 3600
  }
}
⚠️ Catatan: QR ini adalah QR lokal (payload booking), bukan QRIS Flip. Untuk QRIS, customer harus buka payment_url dari Flip.

📊 Flow End-to-End (Customer)
text
1. [PUBLIC] Browse produk
   GET /public/souvenir/products
   GET /public/souvenir/products/{slug}

2. [AUTH] Login user
   POST /user/login → dapat token
   Simpan token di secure storage

3. [CUSTOMER] Buat order (auto-payment)
   POST /customer/orders
   → Response: order_number + payment_url
   → 📧 Email + WA "Pesanan Diterima + Tombol Bayar"

4. [CUSTOMER] Klik tombol dari email
   → Redirect ke Flip → bayar

5. [WEBHOOK] Flip callback
   POST /api/webhook/flip
   → Order jadi paid
   → 📧 + 📱 "Pembayaran Berhasil"

6. [CUSTOMER] Cek status (polling)
   GET /customer/payment/status/{code}
   GET /customer/orders/{order_number}

7. [MERCHANT] Proses & kirim
   PUT /merchant/souvenir/orders/{uuid}/status → processing
   POST /merchant/souvenir/orders/{uuid}/ship → shipped

8. [WEBHOOK] Shipping callback
   POST /api/webhook/shipping → delivered
   → Backend auto-update ke completed

9. [CUSTOMER] Cek tracking
   GET /customer/orders/{order_number}/track
🎯 Checklist Integrasi FE (Public & Customer)
Public (browsing):

□ Pakai final_price untuk display harga produk
□ Handle empty data untuk /shipping-methods
□ Cache response kategori (jarang berubah)
□ Handle related produk kosong (bisa [])
Customer (order + payment):

□ Token user disimpan di secure storage
□ Handle 401 → clear token, redirect login
□ shipping_address.phone wajib format 628xxx
□ Handle auto-payment gagal — order tetap dibuat tanpa payment_url
□ Polling /payment/status/{code} setiap 30 detik
□ Handle 422 business logic (stok habis, produk bukan milik merchant)
□ Handle cancel: enable tombol cancel hanya untuk status pending, paid, processing
□ Setelah cancel, refresh list order
Mobile-specific:

□ Android: pakai EncryptedSharedPreferences untuk token
□ iOS: pakai Keychain untuk token
□ Client secret jangan hardcode — pakai local.properties (Android) / .xcconfig (iOS)
□ Deep link untuk verify-email dan reset-password
📌 Catatan Penting
1. Response status field:

Sejak v2.6, status adalah boolean (true/false), bukan string 'success'/'error'. Update parser FE kalau masih cek string.

2. Pagination format:

data[] (array langsung) + meta{}. Bukan nested paginator seperti Laravel default.

3. final_price:

Selalu gunakan final_price untuk display. Nilai sudah dihitung backend (diskusi harga diskon sudah di-apply).

4. Auto-trigger payment:

POST /orders langsung panggil Flip. Jangan double-call POST /payment/process setelah create order sukses.

5. Cancel rules:

Customer bisa cancel order selama status = pending, paid, atau processing. Setelah shipped, hanya merchant/admin yang bisa batalkan.

Maintained by: Backend Team Ovisito
Last updated: 2026-10-08
Version: 2.6

📋 Cara Pakai
Replace §6 & §7 di marketplacev2.5.md

Rename file jadi marketplacev2.6.md

Commit ke GitHub

Share ke FE web & mobile

Dokumen siap paste. Semua endpoint sudah diverifikasi dengan route list aktual, semua response sudah selaras dengan kode v2.6.

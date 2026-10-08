📚 Paket Dokumen Operasional Escrow Ovisito
Saya buatkan 5 dokumen sekaligus — padat & siap pakai untuk tim.

📄 DOKUMEN A — Kebijakan Escrow & Saldo Merchant
Versi: 1.0 | Tanggal: 2026-10-09 | Status: Draft untuk review

A.1 Tujuan
Menetapkan aturan tentang:

Kapan uang customer masuk ke saldo merchant

Kapan merchant bisa menarik saldo

Bagaimana refund diproses

Bagaimana dispute ditangani

A.2 Definisi
Istilah	Definisi
Escrow	Mekanisme penahanan uang oleh Ovisito sampai syarat tertentu terpenuhi
Saldo Tersedia	Saldo yang bisa ditarik merchant
Saldo Tertahan	Saldo yang belum bisa ditarik (dalam window hold)
Window Hold	Periode 3 hari setelah order completed untuk cek komplain
Dispute	Sengketa formal dari customer terkait order
A.3 Kebijakan Saldo
A.3.1 Kapan Uang Masuk Saldo Tertahan
Uang masuk Saldo Tertahan saat:

✅ Order status = completed (barang sudah sampai atau auto-complete)

✅ payment_status = paid

✅ Tidak ada refund pending

Saldo Tertahan TIDAK bisa ditarik — hanya bisa dilihat.

A.3.2 Kapan Uang Pindah ke Saldo Tersedia
Auto-release oleh sistem setelah 3 hari kalender dari completed_at, DENGAN syarat:

✅ Tidak ada dispute terbuka

✅ Tidak ada flag fraud

✅ Amount ≤ Rp 10.000.000 (di atas ini butuh approve admin)

✅ Merchant status = active (tidak di-blacklist)

Release otomatis tiap jam oleh scheduler.

A.3.3 Kapan Uang Ditarik (Withdraw)
Merchant bisa ajukan withdraw kapan saja dengan syarat:

✅ Saldo Tersedia ≥ Rp 10.000 (minimum)

✅ Nominal ≤ Saldo Tersedia

✅ Tidak ada withdraw pending > 3 hari (mencegah spam)

✅ Data bank lengkap

Proses: 1-3 hari kerja setelah admin approve.

A.4 Kebijakan Refund
A.4.1 Kondisi Refund Diterima
Kondisi	Refund
Barang tidak sampai	✅ Full refund
Barang rusak/tidak sesuai	✅ Full atau partial
Barang palsu	✅ Full refund + sanksi merchant
Customer berubah pikiran	❌ Tidak (kecuali belum dikirim)
A.4.2 Proses Refund
text
1. Customer buka komplain via aplikasi (dalam 3 hari)
2. Sistem flag order: is_disputed = true
3. Cron skip release → saldo tetap di Pending
4. Admin review bukti (foto, chat, dll)
5. Admin decide: refund / tidak
6. Kalau refund:
   - Ledger di-mark: cancelled
   - Refund dari escrow → customer
   - Merchant tidak dapat apa-apa
7. Kalau tidak refund:
   - Ledger di-mark: released
   - Merchant dapat saldo
A.4.3 Batas Waktu Komplain
Maksimal 3 hari setelah barang diterima.

Setelah 3 hari:

Komplain tetap dilayani tapi tidak menjamin refund

Uang sudah di merchant

Platform bantu mediasi

A.5 Fee Ovisito
Komponen	Nilai
Fee transaksi	5% dari total_amount
Fee withdrawal	Rp 0 (gratis, untuk MVP)
Fee dispute	Rp 0
Perhitungan:

text
Order: Rp 135.000
Fee Ovisito (5%): Rp 6.750
Merchant dapat: Rp 128.250
A.6 Larangan
Merchant dilarang:

❌ Menarik saldo untuk order yang masih pending

❌ Memanipulasi status order

❌ Menghindari window hold

Sanksi: suspend akun, tahan saldo, blacklist.

Customer dilarang:

❌ Komplain palsu

❌ Abuse refund

Sanksi: suspend akun, blacklist.

A.7 Force Majeure
Kalau terjadi:

Bencana alam

Gangguan payment gateway

Force majeure lain

Ovisito berhak menunda release/proses saldo dengan pemberitahuan.

📄 DOKUMEN B — Template Rekonsiliasi Harian
Untuk: Tim keuangan
Frekuensi: Setiap hari jam 09:00 WIB

B.1 Tujuan
Memastikan saldo bank riil = total kewajiban ke merchant di database.

B.2 Template Sheet
text
REKONSILIASI HARIAN — [Tanggal: DD/MM/YYYY]
=============================================

A. UANG MASUK
   Flip settlement hari ini:              Rp __________
   Order paid via Flip:                   Rp __________
   Adjustment (+):                        Rp __________
   ─────────────────────────────────────────────────
   Total masuk:                           Rp __________

B. UANG KELUAR
   Withdraw disetujui:                    Rp __________
   Refund ke customer:                    Rp __________
   Fee transfer bank:                     Rp __________
   Adjustment (-):                        Rp __________
   ─────────────────────────────────────────────────
   Total keluar:                          Rp __________

C. SALDO REKENING BANK (riil)
   Saldo awal hari:                       Rp __________
   + Total masuk:                         Rp __________
   - Total keluar:                        Rp __________
   ─────────────────────────────────────────────────
   Saldo akhir bank:                      Rp __________

D. KEWAJIBAN DI DATABASE (dari sistem)
   SUM(merchants.balance) — available:    Rp __________
   SUM(merchants.pending_balance):        Rp __________
   SUM(fee_belum_diambil):                Rp __________
   ─────────────────────────────────────────────────
   Total kewajiban:                       Rp __________

E. SELISIH
   Saldo bank (C) - Total kewajiban (D):  Rp __________

F. STATUS
   [ ] ✅ COCOK (selisih < Rp 1.000)
   [ ] ⚠️ SELISIH KECIL (Rp 1.000 - 100.000) → cek manual
   [ ] 🔴 SELISIH BESAR (> Rp 100.000) → ESCALATE
B.3 Query SQL Cek Kewajiban
sql
-- Available balance
SELECT SUM(balance) FROM user.merchants;

-- Pending balance (kalau pakai ledger)
SELECT SUM(amount) FROM payment.merchant_balance_ledger 
WHERE status = 'pending';

-- Fee belum diambil
SELECT SUM(fee_amount) FROM payment.orders WHERE fee_status = 'pending';

-- Total withdraw pending
SELECT SUM(amount) FROM payment.merchant_withdrawals 
WHERE status = 'pending';
B.4 Eskalasi
Kalau selisih > Rp 100.000:

Screenshot laporan

Kirim ke grup #finance-alert

Tag: @finance-lead @backend-lead

Investigasi dalam 24 jam

📄 DOKUMEN C — Alur Refund Detail
C.1 Skenario: Customer Komplain Barang Rusak
Timeline
text
Hari 1 (pukul 14:00) — Order dibuat
Hari 2 — Customer bayar
Hari 3 — Merchant kirim
Hari 5 (pukul 10:00) — Kurir delivered → status: completed
        → Cron 1 jam kemudian: saldo TERTAHAN +135.000
Hari 6 (pukul 09:00) — Customer BUKA KOMPLAIN
        → Ledger di-flag: is_disputed = true
        → Cron skip release
Hari 6 (pukul 14:00) — Admin review
Hari 7 — Admin decide: REFUND
        → Ledger: cancelled
        → Refund Rp 135.000 ke customer
Hari 8 — Customer konfirmasi terima refund
Hari 9 — Merchant di-notifikasi via email/WA
Step-by-Step
STEP 1 — Customer Komplain

Aplikasi customer:

Buka order

Klik "Ajukan Komplain"

Pilih alasan: Barang rusak / tidak sesuai / tidak sampai / lain

Upload bukti (foto, video, chat)

Submit

Backend:

Order: dispute_status = 'open'

Ledger: is_disputed = true

Notifikasi admin via WA/email

STEP 2 — Admin Review

Admin dashboard:

Lihat daftar komplain: /admin/disputes

Buka detail → lihat bukti

Cek history customer (sering komplain?)

Cek history merchant (sering komplain?)

STEP 3 — Keputusan Admin

3 opsi:

Opsi	Efek
Refund penuh	Rp 135.000 kembali ke customer
Refund parsial	Mis. Rp 50.000 (diskon)
Tolak komplain	Merchant tetap dapat uang
STEP 4 — Eksekusi

Kalau refund:

text
1. Payment gateway: refund Rp 135.000 → customer
2. Ledger: status = cancelled, notes = "Refund approved"
3. Merchant balance: tidak berubah
4. Notifikasi:
   - Customer: email "Refund diproses"
   - Merchant: email "Order X di-refund karena komplain"
Kalau tolak:

text
1. Ledger: is_disputed = false, status = released
2. Merchant balance += 135.000
3. Customer: email "Komplain ditolak, alasan: ..."
C.2 SLA (Service Level Agreement)
Tahap	SLA
Admin respon pertama	< 6 jam
Admin review lengkap	< 24 jam
Refund diproses	< 3 hari kerja
Notifikasi ke pihak	< 1 jam setelah keputusan
C.3 Bukti yang Diterima
Customer harus upload minimal 1:

Foto barang rusak

Video unboxing

Screenshot chat dengan merchant

Resi + foto kemasan

Kalau tidak ada bukti → auto-tolak.

📄 DOKUMEN D — Mockup Admin Dashboard Dispute
D.1 Halaman List Dispute
text
┌────────────────────────────────────────────────────────────────┐
│  DISPUTE MANAGEMENT                                🔔 3 new    │
├────────────────────────────────────────────────────────────────┤
│  Filter: [Semua ▼] [Open] [Resolved] [Rejected]   Search: ___ │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Order          Customer     Merchant    Amount    SLA   Action│
│  ─────────────────────────────────────────────────────────────│
│  SO-...A1B2    Budi S.       Kopi Aceh  135.000  🟢 4h  [▶]   │
│  SO-...C3D4    Ani Y.        Soto Lhok   75.000  🟡 20h [▶]   │
│  SO-...E5F6    Citra M.      Batik Aceh 250.000  🔴 30h [▶]   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
Warna SLA:

🟢 < 12 jam (aman)

🟡 12-24 jam (warning)

🔴 > 24 jam (urgent)

D.2 Halaman Detail Dispute
text
┌────────────────────────────────────────────────────────────────┐
│  DISPUTE #12345 — SO-20261008-A1B2C3                    [X]   │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  📦 ORDER INFO                                                  │
│  Order: SO-20261008-A1B2C3                                     │
│  Merchant: Kopi Aceh (verified)                                │
│  Customer: Budi Santoso (12 order, 2 komplain)                 │
│  Amount: Rp 135.000                                            │
│  Completed: 2026-10-08 10:00                                   │
│                                                                 │
│  ⚠️ KOMPLAIN DARI CUSTOMER                                     │
│  Alasan: Barang rusak                                          │
│  Deskripsi: "Kemasan penyok, isi tumpah"                       │
│  Bukti: [📷 Foto 1] [📷 Foto 2] [🎥 Video]                    │
│                                                                 │
│  🔍 CEK HISTORY                                                │
│  Merchant rating: ⭐ 4.8 (152 order)                           │
│  Customer komplain: 2 dari 12 order (16%)                      │
│  Status ledger: 🔴 PENDING (ditahan)                           │
│                                                                 │
│  📝 CHAT HISTORY                                               │
│  [Customer]: "Kemasan penyok"                                  │
│  [Merchant]: "Maaf, akan kami ganti"                           │
│  [Customer]: "Saya minta refund saja"                          │
│                                                                 │
├────────────────────────────────────────────────────────────────┤
│  KEPUTUSAN ADMIN                                               │
│                                                                 │
│  [ ] Setujui Refund Penuh — Rp 135.000                         │
│  [ ] Setujui Refund Parsial — Rp [____]                        │
│  [ ] Tolak Komplain                                            │
│                                                                 │
│  Catatan: ___________________________________________           │
│                                                                 │
│  [ SIMPAN KEPUTUSAN ]  [ ESCALATE KE SUPERVISOR ]              │
└────────────────────────────────────────────────────────────────┘
D.3 Halaman Ledger Balance
text
┌────────────────────────────────────────────────────────────────┐
│  BALANCE LEDGER                                  Filter: ___   │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Type              Merchant     Amount     Status    Release   │
│  ────────────────────────────────────────────────────────────  │
│  credit_pending    Kopi Aceh    135.000   🟡 pend.   +2 hari   │
│  credit_released   Soto Lhok     75.000   🟢 rel.    ✓         │
│  credit_pending    Batik Aceh   250.000   🔴 disp.   -         │
│  debit_withdraw    Kopi Aceh   -100.000   🟢 comp.   ✓         │
│                                                                 │
│  Total pending:       Rp 385.000                               │
│  Total released today: Rp 75.000                               │
│  Total disputed:      Rp 250.000                               │
│                                                                 │
│  [ EXPORT CSV ]  [ RELEASE MANUAL ]  [ RELEASE ALL BATCH ]     │
└────────────────────────────────────────────────────────────────┘
D.4 Menu Sidebar Admin
text
🏠 Dashboard
📊 Orders
👥 Customers
🏪 Merchants
💰 Balance & Escrow
    ├─ Overview
    ├─ Ledger
    ├─ Pending Release
    └─ Disputes 🔔 3
💸 Withdrawals
    ├─ Pending Approval
    ├─ History
    └─ Batch Transfer
📈 Reports
⚙️  Settings
📄 DOKUMEN E — SOP Operasional Harian
E.1 Struktur Tim
Role	Tanggung Jawab
Finance Lead	Rekonsiliasi, approve withdraw, oversight
Finance Ops	Rekonsiliasi harian, process withdraw
Customer Support	Handle dispute customer
Backend Lead	Monitor cron, error log, fix bug
Ops Manager	Oversight, decision dispute besar
E.2 SOP Harian
🌅 PAGI (08:00-10:00)
Finance Ops:

□ Buka dashboard balance → cek total pending & available
□ Rekonsiliasi bank vs DB (template B)
□ Cek withdraw pending → approve atau flag
□ Laporkan selisih (kalau ada) ke grup
Backend Lead:

□ Cek log cron merchant:release-balance semalam
□ Pastikan semua release sukses
□ Cek error log backend
□ Cek webhook Flip — semua paid order ter-proses?
☀️ SIANG (10:00-15:00)
Customer Support:

□ Cek dispute baru → assign ke CS
□ Review komplain → respon pertama < 6 jam
□ Eskalasi dispute besar ke Ops Manager
Finance Ops:

□ Process withdraw yang sudah disetujui
□ Update status di sistem
□ Notifikasi merchant via email
🌆 SORE (15:00-17:00)
Finance Lead:

□ Review withdraw hari ini
□ Approve batch transfer
□ Cek cash flow forecast minggu ini
Ops Manager:

□ Review dispute yang sudah di-decide
□ Approve refund > Rp 1jt
□ Cek metric: SLA, dispute rate, refund rate
🌙 MALAM (17:00-22:00)
Backend (on-call):

□ Cek cron
□ Cek error alert
□ Standby kalau ada issue
E.3 SOP Mingguan
Setiap Senin 09:00 — Meeting 30 menit:

Agenda	PIC	Waktu
Laporan rekonsiliasi mingguan	Finance Lead	5 min
Jumlah dispute minggu lalu	CS Lead	5 min
Error log backend	Backend Lead	5 min
Metric: order, refund, fee	Ops Manager	10 min
Action items	All	5 min
E.4 SOP Bulanan
Setiap tanggal 1, 10:00 — Review Bulanan:

□ Laporan keuangan: revenue, refund, fee, hold
□ Audit: recheck 100 sampel transaction
□ Merchant review: top 10 & bottom 10 performa
□ Customer review: komplain rate
□ Regulasi: ada update UU/POJK?
□ Roadmap: next feature
E.5 SOP Insiden
🚨 Insiden: Selisih Bank vs DB > Rp 1jt
text
1. STOP semua withdraw (manual hold di sistem)
2. Finance Lead + Backend Lead investigasi < 2 jam
3. Root cause:
   - Bug di code? → fix + deploy
   - Human error? → koreksi + training
   - Fraud? → report ke Ops Manager + legal
4. Resolve + dokumentasi
5. Post-mortem < 3 hari
🚨 Insiden: Cron release gagal > 5%
text
1. Backend Lead cek log
2. Kalau bug → fix + hotfix deploy
3. Kalau server → restart + monitor
4. Notifikasi merchant: "Release delayed, tunggu 2 jam"
5. Manual release batch untuk yang urgent
🚨 Insiden: Refund spam (> 10 komplain/hari dari 1 customer)
text
1. Suspend customer account sementara
2. CS Lead review history customer
3. Kalau fraud → blacklist
4. Kalau valid → proses normal
E.6 Kontak & Eskalasi
Level	Kontak	Respon
Level 1 — CS	cs@ovisito.com	< 6 jam
Level 2 — Finance Lead	finance@ovisito.com	< 12 jam
Level 3 — Ops Manager	ops@ovisito.com	< 24 jam
Emergency — Backend	WA grup #backend-alert	< 1 jam
📋 Ringkasan Paket Dokumen
#	Dokumen	Untuk	Halaman
A	Kebijakan Escrow & Saldo	Legal, Ops, Finance	4
B	Template Rekonsiliasi	Finance Ops	2
C	Alur Refund Detail	CS, Admin	3
D	Mockup Admin Dashboard	Product, Frontend	3
E	SOP Operasional	Semua tim	3
Total: 15 halaman — cukup ringkas untuk dibawa rapat.

🎯 Langkah Selanjutnya
Untuk rapat pertama dengan tim:
Print/screen-share Dokumen A — bahas kebijakan

Bahas Dokumen E — siapa ngapain

Putuskan: window hold, threshold, fee

Setelah rapat:
Assign PIC untuk masing-masing dokumen

Isi bagian keputusan (bagian yang kosong)

Finalisasi & publish ke Wiki/Notion

Implementasi teknis (kalau siap):
Fase 1: instant credit dulu (1 hari)

Fase 2: hold + release (1 minggu)

Fase 3: admin dashboard dispute (2 minggu)

🎬 Mau saya buatkan lagi?
F. Slide presentasi (10-15 slide) — untuk rapat direksi

G. Wireframe user-facing — halaman customer "Konfirmasi Terima Barang" + "Ajukan Komplain"

H. Template email/WA lengkap — semua notifikasi (order, paid, shipped, delivered, release, refund)

I. Skema database detail — ERD untuk merchant_balance_ledger + relasi

J. Roadmap Gantt chart — timeline implementasi 3 bulan

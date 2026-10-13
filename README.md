# 📘 General Ledger & Voucher Kas Keluar

Prototype aplikasi **General Ledger (GL) dan Voucher Entri Transaksi Kas Keluar** berbasis web yang menggunakan **HTML, CSS, JavaScript, PostgreSQL, dan Supabase**.

Proyek ini dikembangkan untuk menerapkan konsep **desain arsitektur data, normalisasi database, double-entry bookkeeping, database integrity, trigger, concurrency control, audit trail, immutability, reversing entry, serta integrasi aplikasi web dengan database cloud**.

---

## 📌 Deskripsi

Sistem dirancang untuk mengatasi permasalahan pencatatan transaksi akuntansi yang dapat menyebabkan **unbalanced balance sheet** apabila validasi hanya dilakukan pada sisi frontend.

Pada sistem ini, validasi tidak hanya dilakukan pada antarmuka pengguna, tetapi juga diperkuat pada sisi **database PostgreSQL/Supabase**.

Prinsip utama yang diterapkan:

- Setiap jurnal menggunakan sistem **double-entry bookkeeping**.
- Total Debit harus sama dengan Total Kredit sebelum jurnal diposting.
- Nilai Debit dan Kredit tidak boleh negatif.
- Setiap baris jurnal tidak boleh memiliki Debit dan Kredit sama-sama nol.
- Jurnal yang sudah `POSTED` tidak dapat diubah atau dihapus.
- Koreksi jurnal `POSTED` dilakukan menggunakan **reversing entry/storno**.
- Perubahan data dicatat dalam **audit log**.
- Integritas jurnal dapat diverifikasi menggunakan **SHA-256 checksum**.
- Mutasi persediaan dilakukan secara otomatis menggunakan **database trigger**.
- Sistem mencegah terjadinya **negative inventory**.
- Mekanisme row-level locking digunakan untuk mengurangi risiko **race condition dan overselling**.

---

# 🏗️ Arsitektur Sistem

```text
┌──────────────────────────────┐
│          Frontend            │
│      HTML / CSS / JavaScript │
└──────────────┬───────────────┘
               │
               │ Supabase API / RPC
               ▼
┌──────────────────────────────┐
│       Supabase Backend       │
│       PostgreSQL Database    │
│                              │
│ • Constraints               │
│ • Stored Functions / RPC    │
│ • Database Trigger          │
│ • Audit Trail               │
│ • Hash Verification         │
│ • Transaction Control       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        Data Akuntansi        │
│                              │
│ Chart of Accounts            │
│ Journal Headers              │
│ Journal Lines                │
│ Audit Log                    │
│ Inventory                    │
└──────────────────────────────┘

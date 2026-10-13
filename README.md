# General Ledger & Sistem Kas

## Deskripsi

Proyek ini merupakan purwarupa **General Ledger (GL)** dan **Voucher Entri Transaksi Kas Keluar** berbasis web yang dirancang untuk menerapkan prinsip akuntansi berpasangan (*double-entry bookkeeping*), integritas database, otomasi transaksi persediaan, serta mekanisme **audit trail dan immutability**.

Aplikasi menggunakan:

- **Frontend:** HTML, CSS, JavaScript
- **Backend/Database:** Supabase PostgreSQL
- **Database Logic:** SQL, Function, Trigger, Constraint Trigger, RPC
- **Audit & Integrity:** Foreign Key, CHECK Constraint, Trigger, SHA-256 Hash

Tujuan utama proyek adalah memastikan bahwa transaksi akuntansi tidak hanya tervalidasi pada sisi frontend, tetapi juga tetap aman dan konsisten ketika transaksi dikirim langsung melalui API atau database.

---

# SOAL 1 — Desain Arsitektur Data, Normalisasi, & Penegakan Asas Berpasangan

## 1.1 Entity Relationship Diagram

Sistem menggunakan tiga entitas utama:

1. `chart_of_accounts`
2. `journal_headers`
3. `journal_lines`

Relasi antarentitas:

```text
chart_of_accounts
       │
       │ 1
       │
       │ N
journal_lines
       │
       │ N
       │
       │ 1
journal_headers
```

### Relasi

- `journal_headers` → `journal_lines` = **1 : N**
- `chart_of_accounts` → `journal_lines` = **1 : N**
- `journal_lines` berfungsi sebagai tabel detail transaksi jurnal yang menghubungkan header transaksi dengan akun.

---

## 1.2 Struktur Database PostgreSQL

```sql
CREATE TABLE chart_of_accounts (
    account_id            BIGSERIAL    PRIMARY KEY,
    account_code          VARCHAR(20)  NOT NULL UNIQUE,
    account_name          VARCHAR(150) NOT NULL,
    normal_balance        CHAR(1)      NOT NULL
        CONSTRAINT chk_coa_normal_balance
        CHECK (normal_balance IN ('D','K')),
    report_classification VARCHAR(30)  NOT NULL
        CONSTRAINT chk_coa_report_class
        CHECK (
            report_classification IN (
                'ASET',
                'LIABILITAS',
                'EKUITAS',
                'PENDAPATAN',
                'BEBAN'
            )
        )
);

CREATE TABLE journal_headers (
    id             BIGSERIAL    PRIMARY KEY,
    trx_date       DATE         NOT NULL,
    ref_no         VARCHAR(50)  NOT NULL UNIQUE,
    description    TEXT         NOT NULL,
    posting_status VARCHAR(10)  NOT NULL DEFAULT 'DRAFT'
        CONSTRAINT chk_jh_status
        CHECK (posting_status IN ('DRAFT','POSTED'))
);

CREATE TABLE journal_lines (
    id         BIGSERIAL PRIMARY KEY,
    header_id  BIGINT NOT NULL,
    account_id BIGINT NOT NULL,
    debit      NUMERIC(18,2) NOT NULL DEFAULT 0,
    credit     NUMERIC(18,2) NOT NULL DEFAULT 0,

    CONSTRAINT fk_jl_header
        FOREIGN KEY (header_id)
        REFERENCES journal_headers(id)
        ON DELETE RESTRICT,

    CONSTRAINT fk_jl_account
        FOREIGN KEY (account_id)
        REFERENCES chart_of_accounts(account_id)
        ON DELETE RESTRICT,

    CONSTRAINT chk_jl_debit_nonneg
        CHECK (debit >= 0),

    CONSTRAINT chk_jl_credit_nonneg
        CHECK (credit >= 0),

    CONSTRAINT chk_jl_not_both_zero
        CHECK (
            NOT (debit = 0 AND credit = 0)
        )
);
```

### Penjelasan

`NUMERIC(18,2)` digunakan untuk nilai uang agar perhitungan akuntansi tidak mengalami masalah pembulatan seperti yang dapat terjadi pada tipe `FLOAT`.

Constraint yang digunakan:

| Constraint | Fungsi |
|---|---|
| `PRIMARY KEY` | Menjamin setiap record memiliki identitas unik |
| `UNIQUE` | Mencegah nomor referensi jurnal ganda |
| `FOREIGN KEY` | Menjaga hubungan antar tabel |
| `ON DELETE RESTRICT` | Mencegah data induk dihapus ketika masih digunakan |
| `CHECK debit >= 0` | Mencegah nilai debit negatif |
| `CHECK credit >= 0` | Mencegah nilai kredit negatif |
| `CHECK NOT(debit=0 AND credit=0)` | Memastikan setiap baris jurnal memiliki nilai |

> **Catatan:** `CHECK CONSTRAINT` biasa tidak dapat membandingkan total debit dan kredit dari beberapa baris. Oleh karena itu, validasi `Total Debit = Total Credit` diterapkan menggunakan **Constraint Trigger**.

---

## 1.3 Analisis Normalisasi

Penggunaan struktur:

```text
journal_headers
       │
       └── journal_lines
```

lebih baik dibandingkan menyimpan seluruh data jurnal dalam satu tabel datar.

Contoh tabel jurnal datar:

```text
ref_no
trx_date
description
account_code
account_name
debit
credit
nomor_baris
```

Jika primary key menggunakan:

```text
(ref_no, nomor_baris)
```

maka terdapat permasalahan ketergantungan:

- `trx_date` hanya bergantung pada `ref_no`
- `description` hanya bergantung pada `ref_no`
- `account_name` bergantung pada `account_code`

Hal tersebut dapat menyebabkan redundansi dan anomali saat data diperbarui.

Dengan memisahkan tabel:

### `journal_headers`

Menyimpan informasi yang hanya berkaitan dengan transaksi:

```text
id
trx_date
ref_no
description
posting_status
```

### `journal_lines`

Menyimpan informasi detail:

```text
id
header_id
account_id
debit
credit
```

### `chart_of_accounts`

Menyimpan master akun:

```text
account_id
account_code
account_name
normal_balance
report_classification
```

Struktur tersebut mengurangi redundansi dan lebih mendukung prinsip **3NF**.

Selain itu, struktur header-detail memberikan manfaat audit karena:

- Nomor referensi transaksi unik.
- Detail jurnal dapat ditelusuri melalui `header_id`.
- Akun selalu mengacu pada master akun.
- Status `DRAFT` dan `POSTED` tersimpan pada tingkat transaksi.
- Total debit dan kredit dapat diperiksa berdasarkan satu `header_id`.

---

# SOAL 2 — Otomasi Bisnis, Stored Procedures, & Database Trigger Persediaan

## 2.1 Trigger Persediaan

Sistem menggunakan metode **perpetual inventory**.

Struktur tabel:

```sql
CREATE TABLE inventory_items (
    item_id VARCHAR(10) PRIMARY KEY,
    item_name VARCHAR(100),
    stock_qty INT NOT NULL DEFAULT 0,
    unit_cost NUMERIC(15, 2) NOT NULL
);

CREATE TABLE inventory_transactions (
    trx_id SERIAL PRIMARY KEY,
    item_id VARCHAR(10)
        REFERENCES inventory_items(item_id),
    trx_type VARCHAR(10)
        CHECK (
            trx_type IN (
                'PURCHASE',
                'SALE',
                'ADJUSTMENT'
            )
        ),
    qty INT NOT NULL,
    trx_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Trigger digunakan untuk memperbarui stok secara otomatis.

```sql
CREATE OR REPLACE FUNCTION trg_update_stock_and_journal()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    v_delta     INT;
    v_new_stock INT;
BEGIN

    IF NEW.trx_type IN ('PURCHASE', 'SALE')
       AND NEW.qty <= 0 THEN

        RAISE EXCEPTION
            'Qty % untuk transaksi % harus lebih dari 0',
            NEW.qty,
            NEW.trx_type;

    END IF;

    v_delta :=
        CASE NEW.trx_type
            WHEN 'PURCHASE' THEN NEW.qty
            WHEN 'SALE' THEN -NEW.qty
            WHEN 'ADJUSTMENT' THEN NEW.qty
        END;

    UPDATE inventory_items
    SET stock_qty = stock_qty + v_delta
    WHERE item_id = NEW.item_id
    RETURNING stock_qty INTO v_new_stock;

    IF NOT FOUND THEN

        RAISE EXCEPTION
            'Item % tidak ditemukan',
            NEW.item_id;

    END IF;

    IF NEW.trx_type = 'SALE'
       AND v_new_stock < 0 THEN

        RAISE EXCEPTION
            'Stok item % tidak cukup. Penjualan % unit menyebabkan stok menjadi %',
            NEW.item_id,
            NEW.qty,
            v_new_stock;

    END IF;

    RETURN NEW;

END;
$$;


CREATE TRIGGER trg_inventory_after_insert
AFTER INSERT
ON inventory_transactions
FOR EACH ROW
EXECUTE FUNCTION trg_update_stock_and_journal();
```

### Mekanisme

| Jenis transaksi | Perubahan stok |
|---|---:|
| `PURCHASE` | `+qty` |
| `SALE` | `-qty` |
| `ADJUSTMENT` | mengikuti tanda `qty` |

Jika transaksi menyebabkan stok menjadi negatif, database akan menghasilkan `RAISE EXCEPTION`.

Karena trigger berjalan dalam transaksi database, ketika terjadi exception, perubahan transaksi juga akan dibatalkan.

---

## 2.2 Penanganan Concurrency

Untuk mencegah dua kasir menjual stok yang sama secara bersamaan, database harus menggunakan mekanisme penguncian baris.

Contoh:

```sql
BEGIN;

SELECT stock_qty
FROM inventory_items
WHERE item_id = 'BRG001'
FOR UPDATE;

INSERT INTO inventory_transactions (
    item_id,
    trx_type,
    qty
)
VALUES (
    'BRG001',
    'SALE',
    1
);

COMMIT;
```

`FOR UPDATE` menyebabkan baris `BRG001` dikunci selama transaksi berjalan.

Misalnya stok awal:

```text
Stok = 1
```

Kasir A melakukan penjualan:

```text
Stok 1 → 0
```

Kasir B yang mencoba transaksi pada item yang sama harus menunggu sampai transaksi A selesai. Setelah itu, transaksi B membaca stok terbaru dan tidak dapat menjual jika stok sudah tidak mencukupi.

Alternatif lainnya adalah menggunakan isolation level `SERIALIZABLE`:

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT stock_qty
FROM inventory_items
WHERE item_id = 'BRG001';

INSERT INTO inventory_transactions (
    item_id,
    trx_type,
    qty
)
VALUES (
    'BRG001',
    'SALE',
    1
);

COMMIT;
```

Jika terjadi `serialization_failure` dengan SQLSTATE:

```text
40001
```

aplikasi harus melakukan retry terhadap keseluruhan transaksi.

---

# SOAL 3 — Integrasi Purwarupa Web Full-Stack & Orkestrasi AI Tools

## 3.1 Integrasi AI Tools

### Gemini Canvas

Gemini Canvas digunakan untuk membantu membangun purwarupa antarmuka web dan menghubungkannya dengan Supabase.

Project:

```text
https://share.gemini.google/QJsOw5Iy1BxK
```

Supabase Project:

```text
https://dsdjsdvbajtjdivligue.supabase.co
```

> **Keamanan:** Supabase Publishable Key atau credential lainnya tidak dicantumkan di README publik. Credential sebaiknya disimpan melalui konfigurasi aplikasi atau environment variable dan tidak dimasukkan ke repository GitHub.

---

## NotebookLM

NotebookLM digunakan untuk membantu melakukan verifikasi terhadap aturan dan prinsip akuntansi yang digunakan dalam purwarupa.

### Hasil Verifikasi

| No | Aturan | Hasil |
|---:|---|---|
| 1 | Kas/bank dikreditkan dan akun lawan didebit sesuai substansi transaksi | SUITABLE |
| 2 | Total debit = total kredit dan total > 0 sebelum posting | SUITABLE |
| 3 | Nilai debit/kredit tidak negatif dan hanya salah satu sisi yang bernilai | SUITABLE |
| 4 | Sisi kredit harus mencakup kas/bank | NEEDS IMPROVEMENT |
| 5 | Klasifikasi kas dan setara kas | NEEDS IMPROVEMENT |
| 6 | Restricted cash | NEEDS IMPROVEMENT |
| 7 | Transaksi non-kas dipisahkan | NEEDS IMPROVEMENT |
| 8 | Informasi pendukung arus kas/direct method | NEEDS IMPROVEMENT |
| 9 | Status transaksi langsung `POSTED` | NEEDS IMPROVEMENT |
| 10 | Validasi teknologi berdasarkan standar akuntansi | Tidak dapat diverifikasi hanya dari standar |

### Kesimpulan

Purwarupa telah menerapkan prinsip dasar **double-entry bookkeeping**, terutama:

```text
Total Debit = Total Credit
Debit >= 0
Credit >= 0
Tidak boleh debit dan kredit sama-sama 0
```

Namun, purwarupa masih perlu dikembangkan untuk validasi khusus kas/bank, cash equivalent, restricted cash, klasifikasi aktivitas arus kas, serta tahapan approval sebelum transaksi menjadi `POSTED`.

---

# 3.2 Mekanisme Jejak Audit & Immutability

Mekanisme audit trail diterapkan menggunakan **PostgreSQL Function, Trigger, RPC, dan SHA-256 Hash** melalui Supabase.

## Kolom Audit pada `journal_headers`

Kolom audit yang digunakan:

```text
created_by
created_at
posted_at
reversal_of
hash_checksum
```

### `created_by`

Menyimpan identitas pengguna yang membuat jurnal.

Pada Supabase:

```sql
auth.uid()
```

digunakan untuk mendapatkan identitas pengguna yang sedang login.

### `created_at`

Waktu pembuatan jurnal diambil dari database:

```sql
now()
```

### `posted_at`

Mencatat waktu ketika jurnal berhasil diposting.

### `reversal_of`

Menyimpan referensi terhadap jurnal asli yang dibalik melalui mekanisme **reversing entry/storno**.

### `hash_checksum`

Menyimpan checksum SHA-256 dari isi jurnal yang telah dinormalisasi.

---

# Mekanisme Immutability

Jurnal memiliki dua status utama:

```text
DRAFT
   │
   │ POST
   ▼
POSTED
```

Jurnal dengan status `POSTED` tidak boleh:

```text
UPDATE
DELETE
```

Jika terdapat kesalahan pada jurnal yang sudah `POSTED`, koreksi dilakukan menggunakan:

```text
REVERSING ENTRY / STORNO
```

Jurnal asli tetap dipertahankan sehingga jejak audit tidak hilang.

---

# Constraint Trigger Balance

Validasi keseimbangan jurnal dilakukan menggunakan `CONSTRAINT TRIGGER`.

```sql
CREATE OR REPLACE FUNCTION fn_assert_journal_balanced()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_header_id BIGINT;
    v_status    VARCHAR(10);
    v_debit     NUMERIC(18,2);
    v_credit    NUMERIC(18,2);
BEGIN

    IF TG_TABLE_NAME = 'journal_headers' THEN

        v_header_id := NEW.id;

    ELSIF TG_OP = 'DELETE' THEN

        v_header_id := OLD.header_id;

    ELSE

        v_header_id := NEW.header_id;

    END IF;


    SELECT posting_status
    INTO v_status
    FROM journal_headers
    WHERE id = v_header_id;


    IF v_status = 'POSTED' THEN

        SELECT
            COALESCE(SUM(debit), 0),
            COALESCE(SUM(credit), 0)
        INTO
            v_debit,
            v_credit
        FROM journal_lines
        WHERE header_id = v_header_id;


        IF v_debit <> v_credit
           OR v_debit = 0 THEN

            RAISE EXCEPTION
                'Jurnal % tidak seimbang: total debit=% total kredit=%',
                v_header_id,
                v_debit,
                v_credit
                USING ERRCODE = '23514';

        END IF;

    END IF;

    RETURN NULL;

END;
$$;


CREATE CONSTRAINT TRIGGER trg_lines_balanced
AFTER INSERT OR UPDATE OR DELETE
ON journal_lines
DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW
EXECUTE FUNCTION fn_assert_journal_balanced();


CREATE CONSTRAINT TRIGGER trg_header_balanced
AFTER INSERT OR UPDATE OF posting_status
ON journal_headers
DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW
EXECUTE FUNCTION fn_assert_journal_balanced();
```

---

# Audit Log

Tabel `audit_log` digunakan untuk mencatat perubahan data yang berhasil dilakukan.

Informasi yang dapat dicatat meliputi:

```text
table_name
record_id
operation
old_data
new_data
changed_by
changed_at
```

Data lama dan data baru disimpan dalam format:

```text
JSONB
```

sehingga perubahan dapat ditelusuri kembali.

---

# Hash Integrity

Hash jurnal dibuat menggunakan:

```text
SHA-256
```

Data yang digunakan dalam pembuatan hash meliputi:

```text
journal id
tanggal transaksi
nomor referensi
deskripsi
created_by
created_at
reversal_of
journal lines
```

Journal lines diurutkan berdasarkan `id` agar hash bersifat deterministik.

Contoh extension PostgreSQL:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

Hash dapat dibuat dengan:

```sql
digest(data, 'sha256')
```

Kemudian hasilnya dikonversi menjadi hexadecimal string.

---

# Reversing Entry / Storno

Untuk membatalkan jurnal `POSTED`, sistem menggunakan fungsi:

```text
rpc_reverse_journal()
```

Mekanismenya:

```text
Jurnal Asli
    │
    │ STORNO
    ▼
Jurnal Pembalik
```

Contoh:

```text
Jurnal asli:

Debit  Beban       1.000.000
Kredit Kas         1.000.000
```

Jurnal pembalik:

```text
Debit  Kas         1.000.000
Kredit Beban       1.000.000
```

Jurnal asli tidak diubah.

---

# Pengujian Audit Trail & Immutability

| Pengujian | Hasil |
|---|---|
| UPDATE jurnal `POSTED` | Ditolak |
| DELETE jurnal `POSTED` | Ditolak |
| UPDATE baris jurnal `POSTED` | Ditolak |
| DELETE baris jurnal `POSTED` | Ditolak |
| TRUNCATE tabel jurnal | Ditolak |
| UPDATE `audit_log` | Ditolak |
| STORNO jurnal `POSTED` | Berhasil |
| STORNO jurnal yang sama dua kali | Ditolak |
| Verifikasi hash | Valid |
| Perubahan timezone session | Hash tetap valid |

---

# Contoh Hasil Pengujian

## UPDATE Jurnal POSTED

```text
ERROR:
Jurnal POSTED tidak dapat diubah.
Gunakan reversing entry/storno.
```

## DELETE Jurnal POSTED

```text
ERROR:
Jurnal POSTED tidak dapat dihapus.
Gunakan reversing entry/storno.
```

## Storno

```text
Original Journal:
KK-001

Status:
POSTED

Reversal:
Berhasil dibuat

Debit:
Credit dari jurnal asli

Credit:
Debit dari jurnal asli

Hash:
VALID
```

## Storno Kedua

```text
ERROR:
Jurnal tersebut sudah memiliki reversal.
```

---

# Struktur Proyek

```text
general-ledger/
│
├── backend/
│   │
│   ├── 01_schema.sql
│   ├── 02_journal_balance.sql
│   ├── 03_inventory_trigger.sql
│   └── 04_audit_immutability.sql
│
├── front end/
│   │
│   ├── HTML/
│   │   └── index.html
│   │
│   ├── CSS/
│   │   └── style.css
│   │
│   └── JS/
│       └── app.js
│
├── README.md
│
└── .gitignore
```

---

# Arsitektur Sistem

```text
                    USER
                      │
                      ▼
              ┌───────────────┐
              │    FRONTEND   │
              │ HTML/CSS/JS   │
              └───────┬───────┘
                      │
                      ▼
              ┌───────────────┐
              │    SUPABASE   │
              │      RPC      │
              └───────┬───────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │  HEADER  │ │  LINES   │ │   COA    │
    └──────────┘ └──────────┘ └──────────┘
          │           │
          └──────┬────┘
                 ▼
        ┌─────────────────┐
        │ PostgreSQL       │
        │ Trigger / RPC    │
        │ Constraint       │
        │ Audit Log        │
        │ SHA-256 Hash     │
        └─────────────────┘
```

---

# Kesimpulan

Sistem yang dirancang menerapkan beberapa lapisan pengendalian:

```text
Frontend Validation
        │
        ▼
Supabase RPC
        │
        ▼
PostgreSQL Constraint
        │
        ▼
Database Trigger
        │
        ▼
Audit Trail
        │
        ▼
Hash Verification
```

Dengan pendekatan tersebut, validasi transaksi tidak hanya bergantung pada frontend.

Sistem memastikan:

- Total debit dan kredit harus seimbang.
- Nilai debit dan kredit tidak boleh negatif.
- Satu baris jurnal tidak boleh memiliki debit dan kredit sekaligus.
- Referensi akun harus valid.
- Jurnal `POSTED` tidak dapat diubah.
- Jurnal `POSTED` tidak dapat dihapus.
- Koreksi dilakukan melalui reversing entry.
- Perubahan yang berhasil dapat dicatat dalam audit log.
- Integritas jurnal dapat diverifikasi menggunakan SHA-256.
- Perubahan stok dilakukan secara otomatis melalui database trigger.
- Transaksi persediaan dapat dilindungi dari kondisi stok negatif.
- Concurrency transaksi dapat dikendalikan menggunakan row-level locking atau isolation level `SERIALIZABLE`.

---

# Teknologi

```text
Frontend
├── HTML5
├── CSS3
└── JavaScript

Backend / Database
├── Supabase
├── PostgreSQL
├── PL/pgSQL
├── Database Trigger
├── Constraint Trigger
├── Stored Function
└── RPC

Security & Audit
├── Supabase Auth
├── auth.uid()
├── Audit Log
└── SHA-256 Hash
```

---

# Catatan Keamanan

Jangan menyimpan credential atau secret key di repository GitHub publik.

Contoh data yang **tidak boleh** dimasukkan ke `README.md`:

```text
SUPABASE_SERVICE_ROLE_KEY
PASSWORD
PRIVATE KEY
SECRET KEY
ACCESS TOKEN
```

Untuk aplikasi yang menggunakan Supabase, gunakan key yang sesuai dengan kebutuhan frontend dan terapkan **Row Level Security (RLS)** pada tabel yang diakses pengguna.

---

# Status Proyek

```text
Database Design          : DONE
Normalization            : DONE
Double-Entry Validation  : DONE
Inventory Trigger        : DONE
Concurrency Control      : DONE
Audit Trail              : DONE
Journal Immutability     : DONE
Reversing Entry          : DONE
Hash Verification        : DONE
Frontend Prototype       : DONE
Supabase Integration     : DONE
```

**Project Type:** Academic / Accounting Information System Prototype  
**Database:** Supabase PostgreSQL  
**Architecture:** Full-Stack Web Application

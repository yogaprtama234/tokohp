# Indeks PRD Modul — Sistem Manajemen Toko Handphone Multi-Cabang

Folder ini berisi pecahan spesifikasi teknis dan fungsional dari [PRD Utama (`../PRD.md`)](../PRD.md) yang dibagi menjadi **6 Modul Inti**.

Setiap dokumen modul di dalam folder ini disusun dengan struktur standar 5 bagian agar mudah dipahami oleh seluruh tim:
1. **Tujuan & Ruang Lingkup Modul** (Tabel pemetaan kode requirement, prioritas `P0`/`P1`/`P2`, dan fase rilis)
2. **User Stories & Acceptance Criteria** (Skenario *Given / When / Then* dalam format Gherkin)
3. **Skema Database & Aturan Bisnis** (Skema tabel Prisma/PostgreSQL, diagram alur state, dan penanganan *edge cases*)
4. **Spesifikasi API / Server Actions & Validasi** (Skema validasi `Zod`, daftar endpoint, hak akses role, dan kode error HTTP)
5. **Spesifikasi UI/UX & State Komponen** (Susunan layout halaman, komponen interaktif, dan penanganan *loading/empty/error state*)

---

## Daftar Dokumen Modul

| No | Nama File Modul | Fokus & Cakupan Requirement | Target Rilis |
| --- | --- | --- | --- |
| **01** | [`01-auth-branch-user.md`](./01-auth-branch-user.md) | **Autentikasi, Otorisasi, Manajemen Cabang & User**<br>`AUTH-01` s/d `AUTH-07`, `BR-01` s/d `BR-04`, `USR-01` s/d `USR-05` | Fase 1 (Fondasi) |
| **02** | [`02-catalog-barcode.md`](./02-catalog-barcode.md) | **Katalog Produk, Varian & Sistem Barcode**<br>`PRD-01` s/d `PRD-09`, `BC-01` s/d `BC-10` | Fase 2 & 3 (Master Data & Barcode) |
| **03** | [`03-inventory-stock-imei.md`](./03-inventory-stock-imei.md) | **Manajemen Stok, Inventori, Transfer Cabang & IMEI**<br>`STK-01` s/d `STK-09`, `BC-08`, `BC-10` | Fase 3 & 6 (Stok & IMEI) |
| **04** | [`04-pos-transaction-shift.md`](./04-pos-transaction-shift.md) | **Point of Sale (POS), Kasir, Shift & Retur**<br>`POS-01` s/d `POS-13`, `BC-07` | Fase 4 (POS & Transaksi) |
| **05** | [`05-finance-dashboard-reports.md`](./05-finance-dashboard-reports.md) | **Keuangan, Pengeluaran, Dashboard Analitik & Laporan**<br>`FIN-01` s/d `FIN-06`, `RPT-01` s/d `RPT-07`, Dashboard 3 Role | Fase 5 (Dashboard & Laporan) |
| **06** | [`06-crm-warranty-audit-settings.md`](./06-crm-warranty-audit-settings.md) | **CRM, Garansi IMEI, Promo, Notifikasi, Audit Log & Pengaturan**<br>`AUD-01`–`02`, `NTF-01`–`02`, `SET-01`–`02`, `CRM-01`–`02`, `WAR-01`–`02`, `PRO-01` | Fase 1–6 (Lintas Fase) |

---

## Matriks Jejak Kebutuhan (Requirement Traceability Matrix)

Tabel ini memastikan **tidak ada satu pun kebutuhan di `PRD.md` yang terlewat** di dokumen turunan:

| Kategori di PRD Induk | Kode ID | Lokasi Dokumen Modul |
| --- | --- | --- |
| **5.1 Autentikasi & Otorisasi** | `AUTH-01` s/d `AUTH-07` | [Modul 01](./01-auth-branch-user.md) |
| **5.2 Manajemen Cabang** | `BR-01` s/d `BR-04` | [Modul 01](./01-auth-branch-user.md) |
| **5.3 Manajemen User** | `USR-01` s/d `USR-05` | [Modul 01](./01-auth-branch-user.md) |
| **5.4 Katalog Produk** | `PRD-01` s/d `PRD-09` | [Modul 02](./02-catalog-barcode.md) |
| **5.5 Stok & Inventori** | `STK-01` s/d `STK-09` | [Modul 03](./03-inventory-stock-imei.md) |
| **5.6 Barcode (Fitur Inti)** | `BC-01` s/d `BC-10` | [Modul 02](./02-catalog-barcode.md) (Utama), [Modul 03](./03-inventory-stock-imei.md) (`BC-08`, `BC-10`), [Modul 04](./04-pos-transaction-shift.md) (`BC-07`) |
| **5.7 Point of Sale (Kasir)** | `POS-01` s/d `POS-13` | [Modul 04](./04-pos-transaction-shift.md) |
| **5.8 Keuangan** | `FIN-01` s/d `FIN-06` | [Modul 05](./05-finance-dashboard-reports.md) |
| **5.9 Dashboard & Laporan** | `RPT-01` s/d `RPT-07` | [Modul 05](./05-finance-dashboard-reports.md) |
| **5.10 Pelanggan, Garansi & Promo** | `CRM-01`–`02`, `WAR-01`–`02`, `PRO-01` | [Modul 06](./06-crm-warranty-audit-settings.md) |
| **5.11 Notifikasi, Audit & Pengaturan** | `NTF-01`–`02`, `AUD-01`–`02`, `SET-01`–`02` | [Modul 06](./06-crm-warranty-audit-settings.md) |

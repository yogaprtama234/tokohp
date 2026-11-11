# PRD Modul 05: Keuangan, Pengeluaran, Dashboard Analitik & Laporan

**Induk Dokumen:** [PRD.md](../PRD.md) · **Versi:** 1.0 · **Status:** Ready for Dev  
**Cakupan Requirement:** `FIN-01` s/d `FIN-06`, `RPT-01` s/d `RPT-07`, Dashboard 3 Role (`5.9`)

---

## 1. Tujuan & Ruang Lingkup Modul

Modul ini mengonsolidasikan seluruh transaksi penjualan, HPP, retur, dan biaya operasional dari seluruh cabang menjadi laporan keuangan dan dashboard real-time (target waktu muat p95 `< 2 detik` dan penyusunan laporan bulanan `< 5 menit`).

### Cakupan Kebutuhan
| ID | Fitur | Prioritas | Target Rilis |
| --- | --- | --- | --- |
| `FIN-01` | Pencatatan pengeluaran per cabang (sewa, listrik, gaji, internet, operasional) + foto bukti | P0 | MVP (Fase 5) |
| `FIN-02` | Pencatatan pemasukan otomatis dari penjualan POS dan pemasukan lain-lain secara manual | P0 | MVP (Fase 5) |
| `FIN-03` | Kalkulasi otomatis Laba Kotor ($\text{Penjualan Bersih} - \text{HPP}$) dan Laba Bersih ($\text{Laba Kotor} - \text{Pengeluaran}$) | P0 | MVP (Fase 5) |
| `FIN-04` | Rekapitulasi arus kas per metode pembayaran (Tunai, Transfer, QRIS, Kartu, Leasing) | P0 | MVP (Fase 5) |
| `FIN-05` | Rekonsiliasi kas harian per cabang (mencocokkan mutasi shift dengan setoran bank) | P1 | Rilis 1.1 |
| `FIN-06` | Target omzet & anggaran pengeluaran bulanan per cabang (progress bar pencapaian) | P2 | Rilis 1.2 |
| `RPT-01` | Laporan penjualan detail (per hari, per produk/brand, per kasir, per cabang) | P0 | MVP (Fase 5) |
| `RPT-02` | Laporan Laba Rugi (*Income Statement*) sederhana per cabang dan konsolidasi pusat | P0 | MVP (Fase 5) |
| `RPT-03` | Laporan valuasi stok & nilai aset persediaan ($\sum (\text{qty} \times \text{HPP})$ per cabang) | P0 | MVP (Fase 5) |
| `RPT-04` | Laporan mutasi/pergerakan stok (*Stock Movement Report*) | P0 | MVP (Fase 5) |
| `RPT-05` | Export seluruh laporan ke format PDF siap cetak dan Excel (`.xlsx`) / CSV | P0 | MVP (Fase 5) |
| `RPT-06` | Laporan rincian pengeluaran operasional per kategori & cabang | P0 | MVP (Fase 5) |
| `RPT-07` | Pengiriman ringkasan laporan harian/mingguan otomatis via Email | P2 | Rilis 1.2 |

---

## 2. User Stories & Acceptance Criteria (Gherkin)

### US-5.1: Pencatatan Pengeluaran Operasional Cabang (`FIN-01`, `RPT-06`)
**Sebagai** Admin Cabang, **saya ingin** mencatat biaya operasional cabang beserta foto nota buktinya **agar** perhitungan laba bersih cabang akurat dan transparan bagi Super Admin.

```gherkin
Scenario: Mencatat pengeluaran cabang dengan lampiran nota
  Given Admin Cabang "BDG-01" berada di halaman "/expenses"
  When ia menginput pengeluaran kategori "LISTRIK_AIR" sebesar Rp 1.250.000 pada tanggal "2026-10-25"
  And mengunggah foto struk token PLN lalu menekan "Simpan"
  Then pengeluaran tercatat pada branch "BDG-01"
  And Laba Bersih cabang "BDG-01" pada bulan Oktober 2026 langsung berkurang sebesar Rp 1.250.000
```

### US-5.2: Konsistensi Kalkulasi Dashboard & Laporan Laba Rugi (`FIN-03`, `RPT-01`, `RPT-02`)
**Sebagai** Super Admin, **saya ingin** melihat kartu ringkasan omzet, HPP, laba kotor, pengeluaran, dan laba bersih dengan filter rentang tanggal dan cabang **agar** dapat membandingkan performa antar cabang secara akurat.

```gherkin
Scenario: Filter multi-cabang dan verifikasi rumus Laba Rugi
  Given pada periode 1-31 Oktober 2026 di Cabang "JKT-01":
    - Total Penjualan Kotor (setelah diskon item & nota, di luar pajak) = Rp 500.000.000
    - Total Retur Disetujui = Rp 10.000.000
    - Total HPP barang terjual bersih = Rp 410.000.000
    - Total Pengeluaran Operasional = Rp 25.000.000
  When Super Admin memilih filter Cabang = "JKT-01" dan Periode = "Bulan Ini (Oktober 2026)"
  Then Dashboard dan Laporan Laba Rugi menampilkan angka yang identik:
    - Penjualan Bersih (Net Sales) = Rp 490.000.000
    - HPP (COGS) = Rp 410.000.000
    - Laba Kotor (Gross Profit) = Rp 80.000.000 (Margin Kotor: 16,33%)
    - Total Pengeluaran = Rp 25.000.000
    - Laba Bersih (Net Profit) = Rp 55.000.000 (Margin Bersih: 11,22%)
```

### US-5.3: Export Laporan ke Excel & PDF (`RPT-05`)
**Sebagai** Super Admin atau Admin Cabang, **saya ingin** mengunduh laporan yang sedang difilter ke dalam file `.xlsx` atau `.pdf` **agar** dapat dibagikan kepada pemilik/investor atau diarsipkan.

```gherkin
Scenario: Export laporan penjualan ke Excel sesuai filter aktif
  Given pengguna memilih filter tanggal "01 Okt 2026 - 25 Okt 2026" pada Laporan Penjualan
  When pengguna menekan tombol "Export Excel (.xlsx)"
  Then file Excel terunduh dengan header ringkasan filter, tabel data terformat angka Rupiah, dan baris Total di bagian bawah
```

---

## 3. Skema Database & Rumus Finansial

### 3.1 Definisi Tabel (Prisma-style Schema)

```prisma
enum ExpenseCategory {
  RENT              // Sewa ruko/tempat
  ELECTRICITY_WATER // Listrik, air, internet
  SALARY_COMMISSION // Gaji pokok & insentif kasir/staf
  MARKETING         // Iklan, spanduk, promosi lokal
  SUPPLIES          // Kertas struk, plastik/tas, ATK, kebersihan
  MAINTENANCE       // Perbaikan AC, etalase, peralatan
  OTHER             // Lain-lain
}

model Expense {
  id          String          @id @default(cuid())
  branchId    String
  category    ExpenseCategory
  title       String          @db.VarChar(150)
  amount      BigInt          // Dalam Rupiah penuh (positif)
  expenseDate DateTime        @db.Date
  receiptUrl  String?         // URL foto bukti nota/kwitansi
  notes       String?         @db.Text
  createdById String
  createdAt   DateTime        @default(now())
  updatedAt   DateTime        @updatedAt
  deletedAt   DateTime?       // Soft delete (wajib catat ke ActivityLog jika dihapus)

  @@index([branchId, expenseDate(sort: Desc)])
  @@index([category])
}

model OtherIncome {
  id          String   @id @default(cuid())
  branchId    String
  title       String   @db.VarChar(150) // Contoh: "Komisi aktivasi kartu perdana operator"
  amount      BigInt
  incomeDate  DateTime @db.Date
  notes       String?  @db.Text
  createdById String
  createdAt   DateTime @default(now())
}
```

### 3.2 Rumus Standar Perhitungan Keuangan (`FIN-03`, `RPT-02`, `RPT-03`)
Semua nilai uang disimpan dan dihitung menggunakan `BigInt` (Rupiah penuh tanpa desimal sen) untuk menghindari *floating-point rounding error*:

$$\text{Penjualan Bersih (Net Sales)} = \sum \text{Sale.grandTotal} - \sum \text{Sale.taxAmount} - \sum \text{ApprovedReturn.refundAmount}$$

$$\text{HPP Bersih (Net COGS)} = \sum (\text{SaleItem.qty} \times \text{SaleItem.costPriceAtSale}) - \sum \text{HPP Barang Retur Layak Jual}$$

$$\text{Laba Kotor (Gross Profit)} = \text{Penjualan Bersih} - \text{HPP Bersih}$$

$$\text{Laba Bersih (Net Profit)} = \text{Laba Kotor} + \sum \text{OtherIncome.amount} - \sum \text{Expense.amount}$$

$$\text{Nilai Persediaan (Inventory Valuation)} = \sum_{\text{Stok}>0} (\text{Stock.qty} \times \text{EffectiveCostPrice})$$

---

## 4. Spesifikasi API / Server Actions & Validasi Zod

### 4.1 Skema Validasi Zod

```typescript
import { z } from 'zod';

export const CreateExpenseSchema = z.object({
  branchId: z.string().min(1),
  category: z.enum([
    'RENT',
    'ELECTRICITY_WATER',
    'SALARY_COMMISSION',
    'MARKETING',
    'SUPPLIES',
    'MAINTENANCE',
    'OTHER',
  ]),
  title: z.string().min(3).max(150),
  amount: z.number().int().positive('Nominal pengeluaran harus lebih dari Rp 0'),
  expenseDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Format tanggal YYYY-MM-DD'),
  receiptUrl: z.string().url().optional(),
  notes: z.string().max(500).optional(),
});

export const ReportQueryFilterSchema = z.object({
  startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  endDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
  branchIds: z.array(z.string()).optional(), // Hanya berlaku untuk SUPER_ADMIN
  groupBy: z.enum(['day', 'week', 'month', 'branch', 'cashier', 'product']).default('day'),
  format: z.enum(['json', 'xlsx', 'pdf', 'csv']).default('json'),
});
```

### 4.2 Daftar Endpoint / Server Actions

| Action / Endpoint | Metode | Role Diizinkan | Deskripsi & Error Codes |
| --- | --- | --- | --- |
| `GET /api/dashboard/summary` | GET | Semua Role | Mengembalikan metrik sesuai role (`SUPER_ADMIN`: multi-cabang + ranking; `BRANCH_ADMIN`: cabangnya; `KASIR`: ringkasan shift & penjualan hari ini). |
| `GET /api/expenses` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` | Daftar pengeluaran dengan filter tanggal, cabang, dan kategori. |
| `POST /api/expenses` | POST | `SUPER_ADMIN`, `BRANCH_ADMIN` | Tambah pengeluaran baru. |
| `DELETE /api/expenses/:id` | DELETE | `SUPER_ADMIN`, `BRANCH_ADMIN` | Soft delete pengeluaran + simpan *snapshot* ke `ActivityLog`. |
| `GET /api/reports/sales` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` | Laporan penjualan (`RPT-01`) + opsi export (`?format=xlsx` / `pdf`). |
| `GET /api/reports/profit-loss` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` | Laporan Laba Rugi (`RPT-02`). |
| `GET /api/reports/inventory-valuation` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` | Laporan stok & nilai aset persediaan (`RPT-03`). |
| `GET /api/reports/stock-movements` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` | Laporan mutasi keluar/masuk stok (`RPT-04`). |

---

## 5. Spesifikasi UI/UX & State Komponen

### 5.1 Dashboard Berdasarkan Role (`/dashboard`)
- **Global Filter Bar (Sticky di atas Dashboard):**
  - Preset Periode Cepat: `Hari Ini`, `Kemarin`, `7 Hari Terakhir`, `Bulan Ini`, `Bulan Lalu`, `Custom Range`.
  - Pemilih Cabang (Khusus `SUPER_ADMIN`): Multi-select dropdown (`Semua Cabang`, atau pilih beberapa cabang sekaligus untuk dibandingkan).
- **Bento Grid Kartu KPI Utama (Aceternity Bento + React Bits CountUp):**
  1. **Total Omzet Bersih** (dengan badge `%` kenaikan/penurunan dibanding periode sebelumnya).
  2. **Laba Kotor & Margin (%)** (disembunyikan untuk role `KASIR`).
  3. **Laba Bersih** (setelah dikurangi pengeluaran cabang).
  4. **Total Transaksi & Unit Terjual** (beserta Rata-rata Nilai Keranjang / *Average Order Value*).
- **Visualisasi Grafik (`shadcn/ui charts` - Recharts):**
  - *Area/Bar Chart:* Tren Omzet vs Laba Bersih harian/bulanan.
  - *Horizontal Bar Chart (Khusus Super Admin):* Peringkat Performa Cabang berdasarkan Omzet & Laba.
  - *Donut Chart:* Komposisi Metode Pembayaran (`Tunai`, `QRIS`, `Transfer`, `Kartu`).
- **Widget Operasional Bawah:**
  - Tabel **10 Produk Terlaris** vs **Produk Lambat Laku (*Dead Stock > 60 hari*)**.
  - Daftar **Stok Kritis Lintas Cabang** dengan tombol cepat *"Buat Transfer"* atau *"Tambah Stok"*.
  - **Live Activity Feed:** 10 aktivitas penting terbaru (penjualan besar, perubahan harga, persetujuan retur, selisih shift).

### 5.2 Pusat Laporan (`/reports`)
- Navigasi Tab: `Penjualan`, `Laba Rugi`, `Valuasi Stok`, `Pergerakan Stok`, `Pengeluaran`, `Rekap Metode Bayar`.
- Di bagian kanan atas setiap tab terdapat tombol **"Export Excel (.xlsx)"** dan **"Cetak / Unduh PDF"** yang menggunakan filter aktif saat ini.


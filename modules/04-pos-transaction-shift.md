# PRD Modul 04: Point of Sale (POS), Kasir, Shift & Retur

**Induk Dokumen:** [PRD.md](../PRD.md) · **Versi:** 1.0 · **Status:** Ready for Dev  
**Cakupan Requirement:** `POS-01` s/d `POS-13`, `BC-07`

---

## 1. Tujuan & Ruang Lingkup Modul

Modul ini adalah ujung tombak operasional kasir di setiap cabang. Target utamanya adalah **waktu penyelesaian transaksi dari scan pertama hingga struk tercetak di bawah 45 detik**, bebas *race condition* stok, serta pencatatan kas dan HPP yang presisi.

### Cakupan Kebutuhan
| ID | Fitur | Prioritas | Target Rilis |
| --- | --- | --- | --- |
| `POS-01` | Antarmuka POS responsif (Desktop, Tablet, HP): katalog, scan, keranjang | P0 | MVP (Fase 4) |
| `POS-02` | Diskon per item dan diskon per transaksi (nominal Rp atau %) dengan batas sesuai role | P0 | MVP (Fase 4) |
| `POS-03` | Metode pembayaran: Tunai (kalkulator kembalian), Transfer Bank, QRIS (statis/manual), Kartu Debit/Kredit | P0 | MVP (Fase 4) |
| `POS-05` | Output struk: cetak printer thermal 58mm/80mm, unduh PDF, kirim struk ke WhatsApp pelanggan | P0 | MVP (Fase 4) |
| `POS-06` | Pemotongan stok otomatis & penyimpanan *snapshot* HPP dalam satu transaksi database atomik | P0 | MVP (Fase 4) |
| `BC-07` | Scan barcode di layar POS langsung menambah item ke keranjang atau menambah *qty* | P0 | MVP (Fase 4) |
| `POS-04` | Pembayaran campuran (*Split Payment*, misal sebagian Tunai + sebagian Transfer) | P1 | Rilis 1.1 |
| `POS-07` | Hold / Parkir transaksi sementara (melayani pembeli berikutnya saat pembeli pertama masih memilih) | P1 | Rilis 1.1 |
| `POS-08` | Retur dan Refund dengan persetujuan Admin Cabang; pemulihan stok dan koreksi laba otomatis | P1 | Rilis 1.1 |
| `POS-09` | Pemilihan/pendaftaran cepat pelanggan pada saat checkout (untuk garansi & riwayat) | P1 | Rilis 1.1 |
| `POS-10` | Manajemen Shift Kasir: buka shift (modal awal), rekap kas sistem vs kas fisik, tutup shift | P1 | Rilis 1.1 |
| `POS-11` | Pencatatan penjualan kredit / leasing (Home Credit, Kredivo, Indodana, dll.), DP, dan tenor | P2 | Rilis 1.2 |
| `POS-12` | Tukar tambah (*Trade-in* HP bekas masuk sebagai pengurang tagihan & stok HP second) | P2 | Rilis 1.2 |
| `POS-13` | Mode offline dasar berbasis IndexedDB (simpan antrean transaksi, sinkronisasi saat koneksi pulih) | P2 | Rilis 1.2 |

---

## 2. User Stories & Acceptance Criteria (Gherkin)

### US-4.1: Checkout Cepat dengan Scan Barcode & Pencegahan Stok Negatif (`POS-01`, `POS-06`, `BC-07`)
**Sebagai** Kasir, **saya ingin** memindai barcode barang, memilih metode bayar, dan mencetak struk dengan cepat **agar** antrean pembeli tidak menumpuk dan stok cabang langsung terpotong akurat.

```gherkin
Scenario: Transaksi penjualan tunai normal (Aksesoris + Unit HP dengan IMEI)
  Given Kasir "Rina" di cabang "JKT-01" memiliki Shift berstatus "OPEN"
  When Rina memindai barcode "Samsung A55 8/256GB" (Rp 5.999.000) dan memilih IMEI "354987009988771"
  And memindai barcode "Tempered Glass A55" (Rp 100.000) sebanyak 2 kali
  And memilih metode bayar "CASH" dengan uang diterima Rp 6.200.000 lalu klik "Bayar & Cetak Struk"
  Then dalam satu transaksi database atomic:
    1. Terbuat record Sale dengan noInvoice "INV/JKT01/20261025/0001", total = 6.199.000, kembalian = 1.000
    2. Terbuat SaleItem dengan menyalin snapshot HPP saat ini ke kolom "costPriceAtSale"
    3. Stok "Samsung A55" berkurang 1, Unit IMEI "354987009988771" berubah status menjadi "SOLD" dengan tanggal akhir garansi otomatis +12 bulan
    4. Stok "Tempered Glass A55" berkurang 2
    5. Tercatat 2 baris StockMovement bertipe "SALE_OUT"
  And dialog cetak struk 58mm/80mm serta tombol "Kirim Struk ke WhatsApp" muncul

Scenario: Concurrency / Race Condition pada barang stok terakhir (Stok = 1)
  Given stok "iPhone 15 Pro 256GB" di cabang "JKT-01" tersisa 1 unit
  When Kasir A dan Kasir B di cabang yang sama menekan tombol "Bayar" pada milidetik yang bersamaan
  Then database row-lock memastikan hanya transaksi Kasir A yang berhasil commit (Stok menjadi 0)
  And transaksi Kasir B gagal dengan error 409 "Stok untuk iPhone 15 Pro 256GB baru saja habis terjual"
  And stok tidak pernah menjadi -1
```

### US-4.2: Pembatasan Diskon Berdasarkan Role (`POS-02`)
**Sebagai** Pemilik Toko, **saya ingin** membatasi nilai diskon maksimal yang boleh diberikan Kasir tanpa otorisasi Admin **agar** margin keuntungan cabang tidak tergerus.

```gherkin
Scenario: Kasir memberikan diskon melebihi batas wewenang Kasir
  Given batas maksimal diskon mandiri untuk role "KASIR" adalah 5% atau Rp 50.000
  When Kasir menginput diskon transaksi sebesar Rp 150.000
  Then sistem meminta otorisasi PIN / Password Admin Cabang di layar POS (atau menolak jika tidak diotorisasi)
  And setiap diskon di atas batas dicatat ke dalam "ActivityLog"
```

### US-4.3: Manajemen Shift Kasir & Selisih Kas (`POS-10`)
**Sebagai** Admin Cabang, **saya ingin** setiap kasir membuka dan menutup shift dengan menghitung uang fisik di laci **agar** setiap selisih kas tunai terdeteksi per pergantian jadwal jaga.

```gherkin
Scenario: Tutup shift kasir dengan perhitungan kas tunai
  Given Kasir membuka shift pagi dengan Modal Awal = Rp 500.000
  And sepanjang shift terdapat penjualan Tunai Rp 3.000.000, penjualan QRIS Rp 4.500.000, dan Retur Tunai Rp 200.000
  When Kasir menutup shift dan menginput uang tunai fisik di laci = Rp 3.250.000
  Then sistem menghitung Kas Tunai Sistem = 500.000 + 3.000.000 - 200.000 = Rp 3.300.000
  And mencatat Selisih Kas (Discrepancy) = -Rp 50.000 (Kurang)
  And menandai shift "CLOSED" serta mengirim notifikasi ke Admin Cabang karena terdapat selisih kas
```

### US-4.4: Retur dan Refund Transaksi (`POS-08`)
**Sebagai** Kasir/Admin Cabang, **saya ingin** memproses retur barang yang cacat produksi berdasarkan nomor invoice/IMEI **agar** stok dan laporan laba bersih terkoreksi secara otomatis.

```gherkin
Scenario: Retur unit HP dengan persetujuan Admin Cabang
  Given Kasir mencari nomor invoice "INV/JKT01/20261025/0001" dan mengajukan retur 1 item dengan alasan "Layar bergaris (DOA)"
  When Admin Cabang menyetujui pengajuan retur tersebut dengan opsi "Masuk ke Stok Barang Retur/Servis" atau "Kembali ke Stok Layak Jual"
  Then status Return menjadi "APPROVED"
  And nilai refund mengurangi penjualan bersih periode berjalan
  And tercatat StockMovement bertipe "RETURN_IN"
```

---

## 3. Skema Database & Aturan Bisnis

### 3.1 Definisi Tabel (Prisma-style Schema)

```prisma
enum PaymentMethod {
  CASH
  BANK_TRANSFER
  QRIS
  DEBIT_CARD
  CREDIT_CARD
  LEASING_FINANCING
}

enum SaleStatus {
  COMPLETED
  PARTIALLY_RETURNED
  REFUNDED
  VOIDED
}

enum ReturnStatus {
  PENDING_APPROVAL
  APPROVED
  REJECTED
}

model Shift {
  id               String    @id @default(cuid())
  branchId         String
  cashierId        String
  openingCash      BigInt    // Modal awal uang tunai di laci
  expectedCash     BigInt?   // Dihitung saat tutup shift: openingCash + cashSales - cashRefunds
  actualCash       BigInt?   // Uang tunai fisik yang dihitung kasir saat tutup shift
  cashDifference   BigInt?   // actualCash - expectedCash
  notes            String?   @db.Text
  openedAt         DateTime  @default(now())
  closedAt         DateTime?

  sales            Sale[]
  @@index([branchId, cashierId, openedAt(sort: Desc)])
}

model Sale {
  id              String        @id @default(cuid())
  invoiceNumber   String        @unique @db.VarChar(40) // INV/JKT01/20261025/0001
  branchId        String
  shiftId         String?
  cashierId       String
  customerId      String?
  subtotal        BigInt        // Jumlah (qty * unitPrice) sebelum diskon
  discountAmount  BigInt        @default(0) // Total diskon item + diskon nota
  taxAmount       BigInt        @default(0)
  grandTotal      BigInt        // subtotal - discountAmount + taxAmount
  totalCost       BigInt        // Total HPP seluruh item (untuk kalkulasi cepat laba kotor)
  grossProfit     BigInt        // grandTotal - taxAmount - totalCost
  paidAmount      BigInt
  changeAmount    BigInt        @default(0)
  status          SaleStatus    @default(COMPLETED)
  notes           String?       @db.Text
  createdAt       DateTime      @default(now())

  shift           Shift?        @relation(fields: [shiftId], references: [id])
  items           SaleItem[]
  payments        Payment[]
  returns         SaleReturn[]

  @@index([branchId, createdAt(sort: Desc)])
  @@index([cashierId, createdAt(sort: Desc)])
  @@index([customerId])
}

model SaleItem {
  id              String   @id @default(cuid())
  saleId          String
  variantId       String
  unitId          String?  // Diisi jika produk memiliki IMEI (kategori HP)
  imeiSnapshot    String?  @db.VarChar(30)
  productNameSnapshot String @db.VarChar(200) // Nama produk & varian saat transaksi
  qty             Int
  unitPrice       BigInt   // Harga jual per unit saat transaksi
  discountAmount  BigInt   @default(0)
  costPriceAtSale BigInt   // WAJIB: Snapshot HPP per unit saat transaksi terjadi
  subtotal        BigInt   // (qty * unitPrice) - discountAmount

  sale            Sale     @relation(fields: [saleId], references: [id], onDelete: Cascade)
  @@index([saleId])
  @@index([variantId])
}

model Payment {
  id            String        @id @default(cuid())
  saleId        String
  method        PaymentMethod
  amount        BigInt
  referenceCode String?       @db.VarChar(100) // Nomor referensi EDC / bukti transfer / QRIS RRN
  providerName  String?       @db.VarChar(60)  // Contoh: "BCA", "Mandiri", "Kredivo"
  createdAt     DateTime      @default(now())

  sale          Sale          @relation(fields: [saleId], references: [id], onDelete: Cascade)
  @@index([saleId, method])
}

model SaleReturn {
  id             String       @id @default(cuid())
  returnNumber   String       @unique @db.VarChar(40)
  saleId         String
  branchId       String
  requestedById  String
  approvedById   String?
  reason         String       @db.Text
  refundAmount   BigInt
  refundMethod   PaymentMethod
  restockCondition String     @default("GOOD") // "GOOD" (kembali ke stok jual) atau "DEFECTIVE" (masuk stok rusak/servis)
  status         ReturnStatus @default(PENDING_APPROVAL)
  createdAt      DateTime     @default(now())
  resolvedAt     DateTime?

  sale           Sale         @relation(fields: [saleId], references: [id])
  @@index([branchId, status])
}
```

### 3.2 Aturan Bisnis Kritis POS
1. **Isolasi HPP Historis:** Kolom `SaleItem.costPriceAtSale` wajib menyalin harga modal pada detik transaksi dibuat. Perubahan harga modal di produk master pada masa depan **tidak boleh** mengubah nilai `costPriceAtSale` dan `grossProfit` pada transaksi masa lalu.
2. **Format Nomor Invoice:** `INV/{KODE_CABANG}/{YYYYMMDD}/{4_DIGIT_URUT_HARIAN}` (contoh: `INV/JKT01/20261025/0042`), di-generate secara atomik menggunakan counter harian per cabang.
3. **Validasi Pembayaran:** Jumlah total `payments[].amount` untuk pembayaran non-tunai tidak boleh melebihi `grandTotal`, dan total seluruh `payments[].amount` harus `>= grandTotal`.

---

## 4. Spesifikasi API / Server Actions & Validasi Zod

### 4.1 Skema Validasi Zod

```typescript
import { z } from 'zod';

export const CheckoutItemSchema = z.object({
  variantId: z.string().min(1),
  unitId: z.string().optional(), // Wajib jika kategori HP dan fitur IMEI aktif
  qty: z.number().int().positive(),
  unitPrice: z.number().int().positive(),
  discountAmount: z.number().int().nonnegative().default(0),
});

export const CheckoutPaymentSchema = z.object({
  method: z.enum(['CASH', 'BANK_TRANSFER', 'QRIS', 'DEBIT_CARD', 'CREDIT_CARD', 'LEASING_FINANCING']),
  amount: z.number().int().positive(),
  referenceCode: z.string().max(100).optional(),
  providerName: z.string().max(60).optional(),
});

export const CreateSaleSchema = z.object({
  customerId: z.string().optional(),
  orderDiscountAmount: z.number().int().nonnegative().default(0),
  supervisorPin: z.string().optional(), // Diperlukan jika diskon melampaui batas role Kasir
  notes: z.string().max(500).optional(),
  items: z.array(CheckoutItemSchema).min(1, 'Keranjang belanja masih kosong'),
  payments: z.array(CheckoutPaymentSchema).min(1, 'Pilih minimal 1 metode pembayaran'),
});
```

### 4.2 Daftar Endpoint / Server Actions

| Action / Endpoint | Metode | Role Diizinkan | Deskripsi & Error Codes |
| --- | --- | --- | --- |
| `POST /api/pos/checkout` | POST | `KASIR`, `BRANCH_ADMIN`, `SUPER_ADMIN` | Eksekusi transaksi penjualan atomik: validasi stok & IMEI, kurangi stok, simpan `Sale`, `SaleItem`, `Payment`, `StockMovement`. Error: `409 INSUFFICIENT_STOCK`, `409 UNIT_ALREADY_SOLD`, `403 DISCOUNT_LIMIT_EXCEEDED`. |
| `POST /api/shifts/open` | POST | `KASIR`, `BRANCH_ADMIN` | Buka shift baru dengan modal awal kas laci. Error: `409 ACTIVE_SHIFT_EXISTS`. |
| `POST /api/shifts/:id/close` | POST | `KASIR`, `BRANCH_ADMIN` | Tutup shift, hitung rekap kas tunai sistem vs fisik, simpan selisih. |
| `GET /api/sales` | GET | Semua Role | Riwayat transaksi cabang (atau semua cabang bagi Super Admin). |
| `GET /api/sales/:id/receipt` | GET | Semua Role | Data terformat untuk cetak struk 58mm/80mm, PDF, dan link pesan WhatsApp (`https://wa.me/...`). |
| `POST /api/returns` | POST | `KASIR`, `BRANCH_ADMIN` | Ajukan retur transaksi. Jika diajukan oleh `BRANCH_ADMIN`/`SUPER_ADMIN`, dapat langsung berstatus `APPROVED`. |
| `PATCH /api/returns/:id/approve` | PATCH | `BRANCH_ADMIN`, `SUPER_ADMIN` | Setujui retur, kembalikan stok (jika kondisi `GOOD`), dan catat koreksi pendapatan. |

---

## 5. Spesifikasi UI/UX & State Komponen

### 5.1 Layar Utama POS (`/pos`)
- **Layout Dua Panel Penuh Layar (`100vh` tanpa scroll halaman luar):**
  - **Panel Kiri (60% lebar — Katalog & Pencarian):**
    - Bar atas: Input pencarian / scan barcode dengan indikator hijau *"Scanner Ready"* + Pill filter kategori cepat (`Semua`, `HP`, `Casing`, `Charger`, `Audio`, `Voucher`).
    - Grid Kartu Produk: Foto, Nama, Badge Varian, Harga Jual, dan Badge sisa stok cabang. Klik kartu langsung memasukkan ke keranjang (atau membuka pemilih IMEI jika kategori HP).
  - **Panel Kanan (40% lebar — Keranjang & Pembayaran):**
    - Pemilih Pelanggan (opsional, bisa cari nomor HP/nama atau klik `+ Pelanggan Baru`).
    - Tombol **"Parkir (Hold)"** beserta badge jumlah transaksi yang sedang diparkir.
    - Daftar item di keranjang: tombol `-` / `+` qty, pill nomor IMEI terpilih, input diskon item.
    - Ringkasan harga: Subtotal, Diskon Transaksi, Pajak (jika aktif), dan **Total Bayar** berukuran besar (`text-2xl font-bold`).
    - Tombol utama **"Bayar Sekarang (`F9` / `Cmd+Enter`)"** berukuran tinggi minimal `56px` agar nyaman disentuh di layar tablet.

### 5.2 Modal Pembayaran & Struk
- Pilihan cepat nominal uang tunai (*Quick Cash Buttons*: `Uang Pas`, `Rp 50.000`, `Rp 100.000`, `Rp 200.000`, dll.) dan kalkulasi kembalian real-time.
- Tab **Split Payment** untuk menambahkan lebih dari satu metode bayar.
- Setelah transaksi sukses, tampil modal **Transaksi Berhasil** dengan 3 tombol besar: **"Cetak Struk (Enter)"**, **"Kirim WhatsApp"**, dan **"Transaksi Baru (Esc)"**.
- Template cetak struk menggunakan `@media print` khusus dengan lebar kertas `58mm` atau `80mm` sesuai pengaturan cabang, menampilkan logo toko, nama cabang, nomor invoice, daftar barang + IMEI, rincian bayar, dan ketentuan garansi di footer.


# PRD Modul 06: CRM, Garansi IMEI, Promo, Notifikasi, Audit Log & Pengaturan

**Induk Dokumen:** [PRD.md](../PRD.md) · **Versi:** 1.0 · **Status:** Ready for Dev  
**Cakupan Requirement:** `AUD-01`, `AUD-02`, `NTF-01`, `NTF-02`, `SET-01`, `SET-02`, `CRM-01`, `CRM-02`, `WAR-01`, `WAR-02`, `PRO-01`

---

## 1. Tujuan & Ruang Lingkup Modul

Modul ini menyediakan tata kelola keamanan dan kepatuhan (*Audit Trail* untuk setiap perubahan harga, stok, dan data sensitif), layanan purna jual (pelanggan & pelacakan klaim garansi IMEI), serta konfigurasi sistem terpusat.

### Cakupan Kebutuhan
| ID | Fitur | Prioritas | Target Rilis |
| --- | --- | --- | --- |
| `AUD-01` | Pencatatan otomatis Audit Log (`before` & `after` state) untuk login, harga, stok, hapus data, role | P0 | MVP (Fase 1-5) |
| `AUD-02` | Halaman eksplorasi Audit Log dengan filter user, cabang, jenis aksi, dan rentang tanggal | P0 | MVP (Fase 5) |
| `NTF-01` | Notifikasi in-app (lonceng topbar): stok menipis, transfer masuk, pengajuan retur, selisih shift | P1 | Rilis 1.1 |
| `SET-01` | Pengaturan global toko: nama brand toko, logo, mata uang, pajak default, template struk | P1 | Rilis 1.1 |
| `SET-02` | Manajemen backup database terjadwal dan ekspor snapshot data | P1 | Rilis 1.1 |
| `CRM-01` | Direktori pelanggan (nama, nomor WA, email) beserta riwayat seluruh transaksi pembelian | P1 | Rilis 1.1 |
| `WAR-01` | Pengecekan masa garansi otomatis berdasarkan scan nomor IMEI dan tanggal transaksi jual | P1 | Rilis 1.1 |
| `WAR-02` | Tiket klaim servis/garansi dengan pelacakan status (`RECEIVED`, `IN_SERVICE`, `READY`, `RETURNED`) | P2 | Rilis 1.2 |
| `CRM-02` | Sistem perolehan & penukaran poin loyalitas pelanggan | P2 | Rilis 1.2 |
| `PRO-01` | Manajemen kode promo / voucher diskon (berlaku global atau khusus cabang tertentu) | P2 | Rilis 1.2 |
| `NTF-02` | Integrasi notifikasi keluar via WhatsApp Gateway / Telegram Bot / Email | P2 | Rilis 1.2 |

---

## 2. User Stories & Acceptance Criteria (Gherkin)

### US-6.1: Jejak Audit (Audit Trail) Perubahan Sensitif (`AUD-01`, `AUD-02`)
**Sebagai** Super Admin, **saya ingin** setiap perubahan harga, penyesuaian stok manual, retur, dan perubahan user tercatat beserta nilai lama dan nilai barunya **agar** potensi kecurangan di cabang dapat dilacak dengan bukti kuat.

```gherkin
Scenario: Mencatat perubahan harga produk ke Audit Log
  Given harga jual "iPhone 15 128GB" sebelumnya adalah Rp 13.999.000
  When Admin Cabang "SBY-01" mengubah harga override cabang menjadi Rp 13.499.000
  Then sistem menyimpan baris baru di "ActivityLog" dengan:
    - action: "PRICE_OVERRIDE_UPDATED"
    - entityType: "BranchPriceOverride"
    - beforeData: { "sellPrice": 13999000 }
    - afterData: { "sellPrice": 13499000 }
    - userId, branchId: "SBY-01", ipAddress, dan timestamp
  And baris ActivityLog tersebut tidak dapat diubah atau dihapus oleh siapapun termasuk Super Admin
```

### US-6.2: Verifikasi Garansi & Klaim Servis via Scan IMEI (`WAR-01`, `WAR-02`, `BC-10`)
**Sebagai** Staf Cabang, **saya ingin** memindai nomor IMEI HP yang dibawa pelanggan **agar** langsung mengetahui tanggal pembelian, cabang penjual, nama pembeli, dan apakah masa garansi toko/resmi masih berlaku.

```gherkin
Scenario: Scan IMEI untuk verifikasi klaim garansi
  Given unit HP dengan IMEI "354987009988771" terjual pada "10 Januari 2026" di cabang "JKT-01" dengan garansi 12 bulan
  When pelanggan datang ke cabang "BDG-01" pada "11 November 2026" dan staf memindai IMEI tersebut di halaman "/warranty"
  Then sistem menampilkan status "GARANSI AKTIF (Sisa 60 Hari - berakhir 10 Januari 2027)"
  And menampilkan detail nomor invoice, tanggal beli, cabang asal pembelian, dan nama pelanggan
  And menyediakan tombol "+ Buat Tiket Klaim Garansi/Servis"
```

### US-6.3: Direktori Pelanggan & Riwayat Belanja (`CRM-01`)
**Sebagai** Kasir atau Admin Cabang, **saya ingin** mencari pelanggan berdasarkan nomor WhatsApp **agar** dapat melihat riwayat HP dan aksesoris yang pernah dibeli pelanggan tersebut.

```gherkin
Scenario: Pencarian cepat pelanggan berdasarkan nomor telepon
  Given pelanggan dengan nomor "081234567890" sudah pernah bertransaksi 3 kali di 2 cabang berbeda
  When kasir mengetik "08123456" di halaman "/customers" atau di kolom pelanggan POS
  Then sistem menampilkan profil pelanggan beserta total belanja kumulatif dan daftar unit IMEI yang pernah dibelinya
```

---

## 3. Skema Database & Aturan Bisnis

### 3.1 Definisi Tabel (Prisma-style Schema)

```prisma
enum WarrantyClaimStatus {
  RECEIVED_AT_BRANCH // Diterima dari pelanggan di cabang
  SENT_TO_SERVICE    // Dikirim ke service center / pusat
  IN_REPAIR          // Sedang diperbaiki / menunggu unit pengganti
  READY_FOR_PICKUP   // Selesai dan siap diambil pelanggan di cabang
  COMPLETED          // Sudah diserahkan kembali ke pelanggan
  REJECTED           // Ditolak (misal: cacat karena jatuh/air - human error)
}

model Customer {
  id          String   @id @default(cuid())
  name        String   @db.VarChar(100)
  phone       String   @unique @db.VarChar(30) // Dinormalisasi ke format 628xxx
  email       String?  @db.VarChar(150)
  address     String?  @db.Text
  points      Int      @default(0)
  registeredBranchId String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@index([phone])
  @@index([name])
}

model WarrantyClaim {
  id              String              @id @default(cuid())
  ticketNumber    String              @unique @db.VarChar(40) // SRV/JKT01/202611/0001
  unitId          String              // Referensi ke Unit (IMEI)
  saleId          String              // Referensi ke transaksi pembelian awal
  customerId      String?
  branchId        String              // Cabang tempat pelanggan menyerahkan klaim
  issueDescription String             @db.Text
  physicalCondition String            @db.Text // Catatan lecet/kelengkapan dus/charger saat diterima
  status          WarrantyClaimStatus @default(RECEIVED_AT_BRANCH)
  resolutionNotes String?             @db.Text
  replacementImei String?             @db.VarChar(30) // Jika diganti unit baru (swap unit)
  createdById     String
  createdAt       DateTime            @default(now())
  updatedAt       DateTime            @updatedAt
  completedAt     DateTime?

  @@index([branchId, status])
  @@index([unitId])
}

model PromoVoucher {
  id              String    @id @default(cuid())
  code            String    @unique @db.VarChar(30) // Contoh: GAJIANOKT
  name            String    @db.VarChar(100)
  discountType    String    // "PERCENTAGE" atau "FIXED_AMOUNT"
  discountValue   BigInt    // Persentase (1-100) atau nominal Rupiah
  maxDiscount     BigInt?   // Plafon maksimal potongan jika tipe PERCENTAGE
  minPurchase     BigInt    @default(0)
  branchId        String?   // null = berlaku di semua cabang
  quota           Int?
  usedCount       Int       @default(0)
  startDate       DateTime
  endDate         DateTime
  isActive        Boolean   @default(true)

  @@index([code, isActive])
}

model ActivityLog {
  id          String   @id @default(cuid())
  userId      String
  branchId    String?
  action      String   @db.VarChar(80) // Contoh: "USER_LOGIN", "PRICE_UPDATED", "STOCK_ADJUSTED"
  entityType  String   @db.VarChar(60) // Contoh: "Product", "Stock", "SaleReturn", "User"
  entityId    String   @db.VarChar(60)
  beforeData  Json?    // Nilai sebelum perubahan (fields sensitif seperti passwordHash di-redact)
  afterData   Json?    // Nilai sesudah perubahan
  ipAddress   String?  @db.VarChar(45)
  userAgent   String?
  createdAt   DateTime @default(now())

  // IMMUTABLE: Tidak boleh di-update atau di-delete
  @@index([branchId, createdAt(sort: Desc)])
  @@index([userId, createdAt(sort: Desc)])
  @@index([action, createdAt(sort: Desc)])
}

model Notification {
  id          String   @id @default(cuid())
  userId      String   // Penerima notifikasi
  branchId    String?
  type        String   @db.VarChar(50) // "LOW_STOCK", "TRANSFER_INCOMING", "RETURN_APPROVAL", "SHIFT_DIFF"
  title       String   @db.VarChar(150)
  message     String   @db.Text
  actionUrl   String?  // Deep link ke halaman terkait, misal "/inventory/transfers/trf-123"
  isRead      Boolean  @default(false)
  createdAt   DateTime @default(now())

  @@index([userId, isRead, createdAt(sort: Desc)])
}

model SystemSetting {
  key         String   @id @db.VarChar(80) // Contoh: "store.profile", "pos.discountLimits"
  value       Json
  updatedById String
  updatedAt   DateTime @updatedAt
}
```

### 3.2 Aturan Bisnis Audit & Notifikasi
1. **Sanitasi Data Sensitif di Audit Log:** Helper `logActivity()` wajib menghapus properti `password`, `passwordHash`, `twoFactorSecret`, dan `token` sebelum menyimpan objek JSON ke kolom `beforeData` / `afterData`.
2. **Normalisasi Nomor Telepon Pelanggan:** Semua input nomor HP pelanggan (`0812...`, `+62812...`, `62812...`) otomatis dinormalisasi ke standar `628...` agar tidak terjadi duplikasi akun pelanggan dan memudahkan pembuatan tautan `https://wa.me/628...`.
3. **Deduplikasi Notifikasi Stok Menipis:** Notifikasi `LOW_STOCK` untuk kombinasi `variantId + branchId` yang sama hanya dikirim maksimal 1 kali per 24 jam agar lonceng notifikasi tidak banjir setiap kali ada transaksi.

---

## 4. Spesifikasi API / Server Actions & Validasi Zod

### 4.1 Skema Validasi Zod

```typescript
import { z } from 'zod';

export const UpsertCustomerSchema = z.object({
  name: z.string().min(2).max(100),
  phone: z.string().min(9).max(20).transform((val) => {
    const digits = val.replace(/\D/g, '');
    return digits.startsWith('0') ? '62' + digits.slice(1) : digits;
  }),
  email: z.string().email().optional().or(z.literal('')),
  address: z.string().max(300).optional(),
});

export const CreateWarrantyClaimSchema = z.object({
  imei: z.string().min(8).max(30),
  issueDescription: z.string().min(5).max(1000),
  physicalCondition: z.string().min(5).max(500),
});

export const AuditLogQuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  pageSize: z.coerce.number().int().min(10).max(100).default(25),
  branchId: z.string().optional(),
  userId: z.string().optional(),
  action: z.string().optional(),
  startDate: z.string().optional(),
  endDate: z.string().optional(),
});
```

### 4.2 Daftar Endpoint / Server Actions

| Action / Endpoint | Metode | Role Diizinkan | Deskripsi & Error Codes |
| --- | --- | --- | --- |
| `GET /api/audit-logs` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` (hanya cabangnya) | Menampilkan daftar log audit dengan filter lengkap & diff JSON. |
| `GET /api/notifications` | GET | Semua Role | Daftar notifikasi in-app milik user + jumlah *unread*. |
| `PATCH /api/notifications/read-all` | PATCH | Semua Role | Tandai semua notifikasi user sebagai telah dibaca (`isRead = true`). |
| `GET /api/customers` | GET | Semua Role | Cari pelanggan berdasarkan nama atau nomor WhatsApp. |
| `POST /api/customers` | POST | Semua Role | Registrasi pelanggan baru (dari halaman CRM atau langsung dari modal POS). |
| `GET /api/warranty/lookup` | GET | Semua Role | Cek status garansi berdasarkan query `?imei=...` atau `?invoice=...`. |
| `POST /api/warranty/claims` | POST | `SUPER_ADMIN`, `BRANCH_ADMIN`, `KASIR` | Buat tiket klaim garansi/servis baru & cetak tanda terima servis. |
| `PUT /api/settings` | PUT | `SUPER_ADMIN` | Simpan konfigurasi global toko (nama toko, logo, batas diskon role, jadwal backup). |

---

## 5. Spesifikasi UI/UX & State Komponen

### 5.1 Halaman Audit Log (`/audit-log`)
- **Bar Filter:** Pemilih Cabang, Pemilih User, Dropdown Tipe Aksi (`Semua Aksi`, `Perubahan Harga`, `Penyesuaian Stok`, `Retur & Void`, `Manajemen User`, `Login`), dan Rentang Tanggal.
- **Drawer Detail Diff (`Before` vs `After`):** Saat baris log diklik, muncul panel samping yang menyoroti perbedaan nilai lama (merah) dan nilai baru (hijau) secara berdampingan, lengkap dengan alamat IP dan perangkat yang digunakan.

### 5.2 Halaman Cek Garansi & Servis (`/warranty`)
- **Kotak Pencarian Besar di Atas:** *"Scan atau ketik nomor IMEI / Nomor Invoice untuk mengecek masa garansi"*.
- **Kartu Status Garansi:**
  - Jika masih dalam masa garansi: Banner hijau **"GARANSI AKTIF"** + hitung mundur sisa hari + detail pembelian.
  - Jika sudah lewat tanggal garansi: Banner merah **"MASA GARANSI BERAKHIR"** beserta tanggal berakhirnya.
- **Tabel Klaim Servis:** Menampilkan tiket klaim yang sedang berjalan lengkap dengan fitur cetak **Struk Tanda Terima Servis** untuk pelanggan.


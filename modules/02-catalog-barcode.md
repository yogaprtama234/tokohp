# PRD Modul 02: Katalog Produk, Varian & Sistem Barcode

**Induk Dokumen:** [PRD.md](../PRD.md) · **Versi:** 1.0 · **Status:** Ready for Dev  
**Cakupan Requirement:** `PRD-01` s/d `PRD-09`, `BC-01` s/d `BC-10`

---

## 1. Tujuan & Ruang Lingkup Modul

Modul ini mengelola master data produk (HP, aksesoris, sparepart, voucher/pulsa), varian (warna, RAM/storage), harga jual global dan *override* per cabang, serta sistem pemindaian dan pencetakan barcode yang menjadi inti kecepatan operasional toko.

### Cakupan Kebutuhan
| ID | Fitur | Prioritas | Target Rilis |
| --- | --- | --- | --- |
| `PRD-01` | CRUD produk master: nama, brand, kategori, deskripsi, harga modal, harga jual default | P0 | MVP (Fase 2) |
| `PRD-02` | Varian produk (warna, RAM, storage) dengan SKU dan barcode unik per varian | P0 | MVP (Fase 2) |
| `PRD-03` | Upload multi-gambar per produk, pilih gambar utama (*primary*), kompresi WebP otomatis | P0 | MVP (Fase 2) |
| `PRD-04` | CRUD Brand dan Kategori (`HP`, `AKSESORIS`, `SPAREPART`, `VOUCHER`) | P0 | MVP (Fase 2) |
| `PRD-05` | Spesifikasi terstruktur (layar, chipset, kamera, baterai, OS, jaringan, garansi resmi/inter) | P0 | MVP (Fase 2) |
| `PRD-06` | Override harga jual per cabang (dengan pencatatan audit log) | P1 | Rilis 1.1 |
| `PRD-07` | Impor katalog produk & varian massal via CSV/Excel | P1 | Rilis 1.1 |
| `PRD-08` | Soft delete dan arsip produk/varian | P0 | MVP (Fase 2) |
| `PRD-09` | Perbandingan spesifikasi dua produk secara berdampingan (*side-by-side*) | P2 | Rilis 1.2 |
| `BC-01` | Auto-generate barcode unik global (Code128 / EAN-13) saat varian dibuat | P0 | MVP (Fase 3) |
| `BC-02` | Input/override barcode manual untuk produk yang sudah memiliki barcode pabrikan (EAN/UPC) | P0 | MVP (Fase 3) |
| `BC-03` | Cetak label stiker barcode (single & batch) ukuran thermal label standar (misal 33x15mm, 40x30mm) | P0 | MVP (Fase 3) |
| `BC-04` | Scan barcode menggunakan kamera HP/tablet/laptop (`@zxing/browser` / `html5-qrcode`) | P0 | MVP (Fase 3) |
| `BC-05` | Scan barcode menggunakan scanner fisik USB/Bluetooth (mode *keyboard wedge*) | P0 | MVP (Fase 3) |
| `BC-06` | Hasil scan langsung membuka halaman detail produk (`/p/[barcode]`) kurang dari 1 detik | P0 | MVP (Fase 3) |
| `BC-07` | Integrasi scan di POS: otomatis tambah ke keranjang atau tambah *qty* jika sudah ada | P0 | MVP (Fase 4) |
| `BC-08` | Integrasi scan di menu Stok: barang masuk, stock opname, dan transfer cabang | P1 | Rilis 1.1 |
| `BC-09` | Penanganan barcode tidak dikenal dengan opsi langsung daftarkan produk baru (khusus Admin) | P0 | MVP (Fase 3) |
| `BC-10` | Scan nomor IMEI untuk melacak status unit HP dan masa berlaku garansinya | P1 | Rilis 1.1 |

---

## 2. User Stories & Acceptance Criteria (Gherkin)

### US-2.1: Pembuatan Produk Multi-Varian & Auto-Barcode (`PRD-01`, `PRD-02`, `BC-01`, `BC-02`)
**Sebagai** Super Admin, **saya ingin** membuat produk beserta beberapa varian warna/memori sekaligus dengan barcode otomatis atau dari kotak pabrikan **agar** setiap varian dapat diidentifikasi secara unik saat discan.

```gherkin
Scenario: Membuat produk HP dengan 2 varian dan auto-generate barcode
  Given Super Admin berada di halaman "/products/new"
  When ia mengisi produk "Samsung Galaxy A55 5G", brand "Samsung", kategori "HP"
  And menambahkan varian 1 (Warna: "Awesome Iceblue", RAM: "8GB", Storage: "256GB") dengan barcode dikosongkan
  And menambahkan varian 2 (Warna: "Awesome Navy", RAM: "12GB", Storage: "256GB") dengan barcode pabrikan "8806095481234"
  Then sistem meng-generate SKU otomatis ("SAM-A55-BLU-8-256") dan barcode Code128 unik untuk varian 1
  And menyimpan barcode "8806095481234" untuk varian 2 setelah memvalidasi keunikannya
  And menginisialisasi baris Stock = 0 untuk kedua varian di seluruh cabang aktif

Scenario: Menolak duplikasi barcode antar varian
  Given barcode "8806095481234" sudah terdaftar pada varian lain
  When Super Admin menginput barcode "8806095481234" pada varian baru
  Then server menolak penyimpanan dengan kode 409 Conflict
  And menampilkan nama produk yang sudah menggunakan barcode tersebut
```

### US-2.2: Pemindaian Barcode ke Halaman Detail Lengkap (`BC-04`, `BC-05`, `BC-06`, `BC-09`)
**Sebagai** Staf Toko (Kasir/Admin), **saya ingin** memindai barcode produk via scanner USB atau kamera HP **agar** langsung melihat foto, spesifikasi, harga, stok di cabang saya, dan stok di cabang lain dalam hitungan milidetik.

```gherkin
Scenario: Scan barcode valid menampilkan detail produk dan stok lintas cabang
  Given Kasir berada di halaman "/scan" atau menekan shortcut scan global
  When barcode "8806095481234" dipindai
  Then dalam waktu < 500ms sistem mengarahkan ke "/p/8806095481234"
  And menampilkan galeri foto, spesifikasi lengkap, harga jual berlaku di cabang Kasir
  And menampilkan jumlah stok tersedia di cabang Kasir beserta badge status (Aman / Menipis / Habis)
  And menampilkan tabel read-only daftar stok di cabang lain
  And menampilkan tombol cepat "+ Tambah ke Keranjang POS"

Scenario: Scan barcode tidak terdaftar di sistem
  Given pengguna memindai barcode "9990001112223" yang belum ada di database
  When sistem mencari varian berdasarkan barcode tersebut
  Then sistem menampilkan state "Barcode 9990001112223 Tidak Ditemukan" tanpa crash
  And jika role adalah SUPER_ADMIN, tampil tombol "+ Daftarkan Produk dengan Barcode Ini" yang membuka form produk dengan field barcode terisi otomatis
```

### US-2.3: Cetak Label Stiker Barcode Batch (`BC-03`)
**Sebagai** Admin Cabang atau Staf Gudang, **saya ingin** memilih beberapa varian dan mencetak stiker barcode sesuai jumlah barang datang **agar** produk fisik di etalase memiliki label barcode dan harga yang rapi.

```gherkin
Scenario: Cetak batch stiker barcode untuk barang baru masuk
  Given Admin memilih 2 varian: "iPhone 15 128GB Black" (5 label) dan "Charger Anker 20W" (10 label)
  When Admin memilih ukuran kertas stiker "33x15mm (2 kolom)" dan klik "Cetak Label"
  Then sistem me-render halaman siap cetak berisi 15 label dengan grafik barcode tajam (SVG), nama produk singkat, SKU, dan opsi tampilkan harga jual
```

---

## 3. Skema Database & Aturan Bisnis

### 3.1 Definisi Tabel (Prisma-style Schema)

```prisma
enum CategoryType {
  HP          // Wajib lacak IMEI jika fitur IMEI aktif
  AKSESORIS
  SPAREPART
  VOUCHER
}

model Brand {
  id        String    @id @default(cuid())
  name      String    @unique @db.VarChar(80)
  slug      String    @unique @db.VarChar(80)
  logoUrl   String?
  products  Product[]
}

model Category {
  id           String       @id @default(cuid())
  name         String       @unique @db.VarChar(80)
  slug         String       @unique @db.VarChar(80)
  type         CategoryType @default(AKSESORIS)
  requiresImei Boolean      @default(false) // true untuk kategori HP
  products     Product[]
}

model Product {
  id             String           @id @default(cuid())
  name           String           @db.VarChar(150)
  slug           String           @unique @db.VarChar(180)
  brandId        String
  categoryId     String
  description    String?          @db.Text
  specifications Json             @default("{}") // { screen, chipset, ram, storage, cameraRear, cameraFront, battery, os, network, warrantyType }
  baseCostPrice  BigInt           // Harga modal default dalam Rupiah penuh (integer)
  baseSellPrice  BigInt           // Harga jual default dalam Rupiah penuh (integer)
  isActive       Boolean          @default(true)
  deletedAt      DateTime?
  createdAt      DateTime         @default(now())
  updatedAt      DateTime         @updatedAt

  brand          Brand            @relation(fields: [brandId], references: [id])
  category       Category         @relation(fields: [categoryId], references: [id])
  images         ProductImage[]
  variants       ProductVariant[]

  @@index([brandId])
  @@index([categoryId])
  @@index([isActive, deletedAt])
}

model ProductImage {
  id        String  @id @default(cuid())
  productId String
  url       String
  sortOrder Int     @default(0)
  isPrimary Boolean @default(false)

  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)
  @@index([productId, isPrimary])
}

model ProductVariant {
  id          String                @id @default(cuid())
  productId   String
  sku         String                @unique @db.VarChar(60)
  barcode     String                @unique @db.VarChar(60) // Di-index untuk lookup < 50ms
  color       String?               @db.VarChar(50)
  ram         String?               @db.VarChar(30)
  storage     String?               @db.VarChar(30)
  costPrice   BigInt?               // Opsional jika harga modal varian berbeda dari produk induk
  sellPrice   BigInt?               // Opsional jika harga jual varian berbeda dari produk induk
  isActive    Boolean               @default(true)
  deletedAt   DateTime?

  product     Product               @relation(fields: [productId], references: [id], onDelete: Cascade)
  branchPrices BranchPriceOverride[]

  @@index([barcode])
  @@index([productId, isActive])
}

model BranchPriceOverride {
  id        String         @id @default(cuid())
  variantId String
  branchId  String
  sellPrice BigInt         // Harga jual khusus di cabang ini
  updatedBy String
  updatedAt DateTime       @updatedAt

  variant   ProductVariant @relation(fields: [variantId], references: [id], onDelete: Cascade)
  @@unique([variantId, branchId])
}
```

### 3.2 Aturan Bisnis & Resolusi Harga
1. **Hierarki Resolusi Harga Jual (`EffectiveSellPrice`):**
   - Prioritas 1: `BranchPriceOverride.sellPrice` (untuk `variantId` + `branchId` terkait).
   - Prioritas 2: `ProductVariant.sellPrice` (jika diisi).
   - Prioritas 3: `Product.baseSellPrice`.
2. **Kerahasiaan Harga Modal (`HPP`):** Field `baseCostPrice` dan `costPrice` **wajib di-strip** di level server ketika endpoint dipanggil oleh role `KASIR` atau `GUDANG`.
3. **Format Auto-Barcode (`BC-01`):** Jika user tidak menginput barcode pabrikan, sistem men-generate kode 12 karakter alfanumerik (`Code128`) dengan prefix `THP` + 8 digit sequence/random + 1 check digit, diverifikasi unik di database.
4. **Deteksi Scanner USB/Bluetooth (`BC-05`):** Listener keyboard global mendeteksi karakter beruntun dengan interval antar-karakter `< 35ms` yang diakhiri tombol `Enter` (minimal 6 karakter), sehingga kasir dapat langsung scan tanpa perlu mengklik field input terlebih dahulu.

---

## 4. Spesifikasi API / Server Actions & Validasi Zod

### 4.1 Skema Validasi Zod

```typescript
import { z } from 'zod';

export const ProductSpecificationSchema = z.object({
  screen: z.string().max(100).optional(),       // Contoh: "6.6 inch Super AMOLED 120Hz"
  chipset: z.string().max(100).optional(),      // Contoh: "Exynos 1480 (4nm)"
  cameraRear: z.string().max(100).optional(),   // Contoh: "50MP OIS + 12MP + 5MP"
  cameraFront: z.string().max(100).optional(),  // Contoh: "32MP"
  battery: z.string().max(100).optional(),      // Contoh: "5000 mAh, 25W Fast Charging"
  os: z.string().max(60).optional(),            // Contoh: "Android 14, One UI 6.1"
  network: z.string().max(60).optional(),       // Contoh: "5G / NFC"
  warrantyType: z.string().max(60).optional(),  // Contoh: "Resmi SEIN 1 Tahun"
});

export const ProductVariantInputSchema = z.object({
  id: z.string().optional(),
  sku: z.string().min(3).max(60).optional(), // Auto-generated jika kosong
  barcode: z.string().min(6).max(60).regex(/^[A-Za-z0-9-]+$/).optional(), // Auto-generated jika kosong
  color: z.string().max(50).optional(),
  ram: z.string().max(30).optional(),
  storage: z.string().max(30).optional(),
  costPrice: z.number().int().nonnegative().optional(),
  sellPrice: z.number().int().positive().optional(),
});

export const UpsertProductSchema = z.object({
  name: z.string().min(3).max(150),
  brandId: z.string().min(1),
  categoryId: z.string().min(1),
  description: z.string().max(5000).optional(),
  specifications: ProductSpecificationSchema,
  baseCostPrice: z.number().int().nonnegative('Harga modal tidak boleh negatif'),
  baseSellPrice: z.number().int().positive('Harga jual harus lebih dari 0'),
  images: z.array(z.object({
    url: z.string().url(),
    isPrimary: z.boolean(),
    sortOrder: z.number().int(),
  })).min(1, 'Minimal 1 foto produk wajib diunggah'),
  variants: z.array(ProductVariantInputSchema).min(1, 'Minimal 1 varian wajib dibuat'),
}).refine((data) => data.baseSellPrice >= data.baseCostPrice, {
  message: 'Peringatan: Harga jual dasar lebih rendah dari harga modal',
  path: ['baseSellPrice'],
});
```

### 4.2 Daftar Endpoint / Server Actions

| Action / Endpoint | Metode | Role Diizinkan | Deskripsi & Error Codes |
| --- | --- | --- | --- |
| `GET /api/products` | GET | Semua Role | List katalog produk dengan pencarian nama/SKU/barcode, filter brand/kategori/stok. (`costPrice` disembunyikan untuk `KASIR`). |
| `POST /api/products` | POST | `SUPER_ADMIN` | Buat produk beserta varian, gambar, dan inisialisasi stok 0 di semua cabang. Error: `409 BARCODE_ALREADY_EXISTS`, `409 SKU_ALREADY_EXISTS`. |
| `PATCH /api/products/:id` | PATCH | `SUPER_ADMIN` | Update produk/varian dan catat perubahan harga di `ActivityLog`. |
| `DELETE /api/products/:id` | DELETE | `SUPER_ADMIN` | Soft delete (`deletedAt = now()`, `isActive = false`). Ditolak jika masih ada stok fisik `> 0` di cabang manapun (`400 STOCK_NOT_EMPTY`). |
| `GET /api/barcode/:code` | GET | Semua Role | Lookup cepat (< 500ms) berdasarkan `ProductVariant.barcode` atau `Unit.imei`. Mengembalikan detail varian, spesifikasi, stok cabang aktif, dan stok cabang lain. Error: `404 BARCODE_NOT_FOUND`. |
| `PUT /api/variants/:id/branch-price` | PUT | `SUPER_ADMIN`, `BRANCH_ADMIN` | Set atau reset *override* harga jual cabang (`PRD-06`). Wajib mencatat harga lama & baru ke `ActivityLog`. |
| `POST /api/products/import` | POST | `SUPER_ADMIN` | Impor CSV/Excel dengan validasi baris per baris (dry-run preview sebelum commit). |

---

## 5. Spesifikasi UI/UX & State Komponen

### 5.1 Halaman Scan & Detail Produk (`/scan` dan `/p/[barcode]`)
- **Mode Scan (`/scan`):**
  - Toggle antara **"Kamera Perangkat"** (viewfinder kotak dengan garis laser animasi + pilih kamera depan/belakang + tombol flash/obor) dan **"Scanner Fisik USB/Bluetooth"** (indikator status *"Siap menerima input scanner..."*).
  - Riwayat 5 barang terakhir yang baru saja discan di bawah viewfinder.
- **Halaman Detail Hasil Scan (`/p/[barcode]`):**
  - **Kolom Kiri:** Galeri foto produk (thumbnail + zoom view) dan gambar visual barcode lengkap dengan tombol *"Cetak Label"*.
  - **Kolom Tengah:** Badge Brand & Kategori, Nama Produk + Pill pemilih varian (klik varian lain langsung mengganti tampilan), Harga Jual besar berformat `Rp 5.999.000`, Grid Spesifikasi Utama dengan ikon (Layar, Chipset, Kamera, Baterai, OS, Garansi).
  - **Kolom Kanan (Panel Stok & Aksi):**
    - Kartu Stok Cabang Saat Ini dengan warna indikator: Hijau (`Aman`), Kuning (`Menipis <= minQty`), Merah (`Habis = 0`).
    - Akordion/Tabel **"Stok di Cabang Lain"** (menampilkan Nama Cabang, Kota, Telepon/WA Cabang, dan Qty tersedia) — memudahkan kasir mengarahkan pembeli atau meminta transfer cabang.
    - **Sticky Action Bar:**
      - Role `KASIR`: Tombol utama **"Masukkan ke Keranjang POS"** (`Cmd+Enter`).
      - Role `BRANCH_ADMIN`: Tombol **"+ Barang Masuk"**, **"Ubah Harga Cabang"**, **"Ajukan Transfer"**.
      - Role `SUPER_ADMIN`: Tombol **"Edit Produk Master"**.

### 5.2 Modal Cetak Label Barcode (`/products/print-labels`)
- Pengaturan Kertas: Lebar x Tinggi label (mm), jumlah kolom (1, 2, atau 3 kolom), margin, checkbox tampilkan Harga, Nama Toko, dan SKU.
- Live Preview menggunakan `react-barcode` (SVG) sehingga hasil cetak pada printer barcode thermal (Xprinter, Zebra, Blueprint) tajam dan tidak pecah.


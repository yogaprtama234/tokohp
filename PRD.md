# PRD: Sistem Manajemen Toko Handphone Multi-Cabang

**Versi:** 1.0 (Draft) · **Tanggal:** 25 Oktober 2026 · **Status:** Menunggu review

## 1. Ringkasan Eksekutif

Produk ini adalah web app fullstack untuk toko handphone yang memiliki banyak cabang. Setiap cabang memiliki akun sendiri untuk mengelola stok, penjualan, dan pengeluaran, sementara Super Admin dapat memantau dan mengelola seluruh cabang dari satu dashboard terpusat. Setiap produk memiliki barcode; memindai barcode langsung menampilkan detail produk lengkap (foto, spesifikasi, harga, stok per cabang). Dashboard dirancang bersih dan modern dengan perpaduan shadcn/ui, Aceternity UI, dan React Bits.

## 2. Latar Belakang dan Masalah

Toko dengan banyak cabang umumnya menghadapi masalah berikut:

- Data stok dan penjualan tersebar (buku, Excel, chat) sehingga pemilik sulit melihat kondisi bisnis secara real-time.
- Stok antar cabang tidak terlihat, sehingga sering terjadi kehabisan stok di satu cabang padahal cabang lain menumpuk.
- Pencatatan manual rawan salah, selisih kas, dan kecurangan.
- Tidak ada pelacakan unit (IMEI), sehingga garansi dan klaim sulit diverifikasi.
- Laporan laba rugi per cabang butuh waktu lama untuk disusun.

## 3. Tujuan, Non-Tujuan, dan Metrik Sukses

### 3.1 Tujuan

1. Memusatkan data produk, stok, penjualan, dan keuangan semua cabang dalam satu sistem.
2. Memberi setiap cabang akun dan hak akses terpisah, dengan isolasi data yang dipaksa di sisi server.
3. Mempercepat transaksi kasir lewat scan barcode.
4. Memberi Super Admin visibilitas penuh: omzet, laba, stok kritis, performa cabang.
5. Menyediakan audit trail untuk perubahan stok, harga, dan data sensitif.

### 3.2 Non-Tujuan (di luar versi 1.0)

- E-commerce publik dengan pembayaran online dan pengiriman.
- Integrasi akuntansi penuh (jurnal umum, neraca).
- Aplikasi mobile native (cukup web responsif/PWA).
- Integrasi marketplace (Tokopedia, Shopee) otomatis.

### 3.3 Metrik Sukses

| Metrik | Target |
| --- | --- |
| Waktu transaksi kasir (scan hingga struk) | di bawah 45 detik |
| Akurasi stok sistem vs fisik (opname) | minimal 98% |
| Waktu pemuatan dashboard (p95) | di bawah 2 detik |
| Waktu penyusunan laporan bulanan | dari hari menjadi di bawah 5 menit |
| Adopsi cabang (cabang aktif bertransaksi harian) | 100% dalam 30 hari setelah rilis |
| Uptime | minimal 99,5% |

## 4. Persona dan Role

| Role | Deskripsi | Cakupan Data |
| --- | --- | --- |
| **Super Admin** | Pemilik/manajemen pusat | Semua cabang |
| **Admin Cabang** | Manajer cabang | Hanya cabangnya |
| **Kasir** | Staf penjualan | Hanya POS dan data terbatas cabangnya |
| **Gudang (opsional)** | Staf stok | Stok cabangnya |

## 5. Ruang Lingkup Fitur

Prioritas: **P0** wajib untuk MVP, **P1** penting (rilis kedua), **P2** nilai tambah.

### 5.1 Autentikasi dan Otorisasi

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| AUTH-01 | Login email dan password dengan hashing (argon2/bcrypt) | P0 |
| AUTH-02 | RBAC dengan role SUPER\_ADMIN, BRANCH\_ADMIN, KASIR | P0 |
| AUTH-03 | Setiap user terikat ke satu cabang (kecuali Super Admin); filter cabang dipaksa di server | P0 |
| AUTH-04 | Logout, session expiry, refresh session | P0 |
| AUTH-05 | Reset password oleh admin | P0 |
| AUTH-06 | Rate limiting dan lockout setelah percobaan login gagal berulang | P1 |
| AUTH-07 | Two-factor authentication (TOTP) untuk Super Admin | P2 |

### 5.2 Manajemen Cabang (Super Admin)

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| BR-01 | CRUD cabang: nama, kode cabang, alamat, kota, telepon, jam operasional, status aktif | P0 |
| BR-02 | Menonaktifkan cabang (soft delete); user cabang tidak bisa login | P0 |
| BR-03 | Halaman detail cabang: ringkasan performa, daftar user, stok, transaksi terbaru | P0 |
| BR-04 | Pengaturan per cabang: info struk, pajak, nomor WhatsApp | P1 |

### 5.3 Manajemen User

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| USR-01 | Super Admin: CRUD semua user, assign role dan cabang | P0 |
| USR-02 | Admin Cabang: CRUD akun kasir di cabangnya saja | P0 |
| USR-03 | Aktif/nonaktifkan user tanpa menghapus riwayat | P0 |
| USR-04 | Profil sendiri: ubah nama, foto, password | P0 |
| USR-05 | Riwayat login dan aktivitas per user | P1 |

### 5.4 Katalog Produk

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| PRD-01 | CRUD produk: nama, brand, kategori, deskripsi, spesifikasi, harga modal, harga jual | P0 |
| PRD-02 | Varian produk (warna, RAM/storage) dengan SKU dan barcode unik per varian | P0 |
| PRD-03 | Upload banyak gambar per produk, pilih gambar utama, kompresi otomatis | P0 |
| PRD-04 | CRUD brand dan kategori (HP, aksesoris, sparepart, voucher/pulsa) | P0 |
| PRD-05 | Spesifikasi terstruktur (layar, chipset, kamera, baterai, OS) | P0 |
| PRD-06 | Harga jual bisa dioverride per cabang | P1 |
| PRD-07 | Impor produk massal via CSV/Excel | P1 |
| PRD-08 | Soft delete dan arsip produk | P0 |
| PRD-09 | Perbandingan dua produk (spesifikasi berdampingan) | P2 |

Hak akses: produk master dikelola Super Admin. Admin Cabang hanya melihat dan dapat mengusulkan produk baru (P2).

### 5.5 Stok dan Inventori

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| STK-01 | Stok per varian per cabang dengan batas minimum | P0 |
| STK-02 | Barang masuk (dari supplier atau pusat) dengan catatan dan lampiran | P0 |
| STK-03 | Riwayat pergerakan stok (masuk, keluar, transfer, retur, penyesuaian) tidak bisa dihapus | P0 |
| STK-04 | Peringatan stok menipis dan stok habis di dashboard | P0 |
| STK-05 | Transfer stok antar cabang dengan status: Diajukan, Disetujui, Dikirim, Diterima, Ditolak | P1 |
| STK-06 | Stock opname: hitung fisik, bandingkan dengan sistem, approve penyesuaian | P1 |
| STK-07 | Pelacakan IMEI/serial per unit untuk kategori HP (status: tersedia, terjual, retur, servis) | P1 |
| STK-08 | Pengecekan stok cabang lain dari halaman detail produk | P0 |
| STK-09 | Purchase order ke supplier dan manajemen hutang supplier | P2 |

Aturan bisnis: stok tidak boleh negatif; semua perubahan stok wajib tercatat di StockMovement dengan user, waktu, dan alasan.

### 5.6 Barcode (Fitur Inti)

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| BC-01 | Barcode digenerate otomatis saat varian dibuat (format Code128 atau EAN-13), unik secara global | P0 |
| BC-02 | Barcode juga dapat diinput manual jika produk sudah punya barcode pabrikan | P0 |
| BC-03 | Cetak label barcode (single dan batch) dalam ukuran stiker standar | P0 |
| BC-04 | Scan via kamera perangkat (HP/tablet) | P0 |
| BC-05 | Scan via scanner USB/Bluetooth (mode keyboard wedge) | P0 |
| BC-06 | Hasil scan membuka halaman detail produk | P0 |
| BC-07 | Di POS, scan menambahkan item ke keranjang; scan ulang menambah qty | P0 |
| BC-08 | Di menu stok, scan untuk barang masuk, opname, dan transfer | P1 |
| BC-09 | Pesan jelas jika barcode tidak ditemukan, dengan opsi mendaftarkan produk baru (Admin) | P0 |
| BC-10 | Scan IMEI untuk menemukan unit dan riwayat garansinya | P1 |

**Halaman detail hasil scan menampilkan:** galeri foto, nama dan varian, brand, kategori, deskripsi, spesifikasi, harga jual, stok cabang saat ini, stok di cabang lain (read-only), status stok (aman/menipis/habis), tombol aksi cepat sesuai role (tambah ke keranjang, tambah stok, edit).

### 5.7 Point of Sale (Kasir)

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| POS-01 | Antarmuka POS responsif (tablet dan HP): pencarian, scan, keranjang | P0 |
| POS-02 | Diskon per item dan per transaksi (nominal atau persen) dengan batas sesuai role | P0 |
| POS-03 | Metode bayar: tunai, transfer, QRIS (pencatatan manual), kartu | P0 |
| POS-04 | Pembayaran campuran (split payment) | P1 |
| POS-05 | Struk: cetak thermal 58/80mm, unduh PDF, kirim via WhatsApp | P0 |
| POS-06 | Stok berkurang otomatis saat transaksi sukses (atomic transaction) | P0 |
| POS-07 | Hold/parkir transaksi | P1 |
| POS-08 | Retur dan refund dengan persetujuan Admin Cabang; stok kembali otomatis | P1 |
| POS-09 | Data pelanggan opsional pada transaksi | P1 |
| POS-10 | Buka/tutup shift kasir, modal awal, rekap kas, selisih kas | P1 |
| POS-11 | Penjualan kredit, DP, dan cicilan | P2 |
| POS-12 | Trade-in (tukar tambah HP bekas) | P2 |
| POS-13 | Mode offline dasar (antrean transaksi, sinkron saat online) | P2 |

### 5.8 Keuangan

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| FIN-01 | Pencatatan pengeluaran per cabang (sewa, listrik, gaji, operasional) dengan kategori dan bukti | P0 |
| FIN-02 | Pemasukan otomatis dari penjualan; pemasukan lain manual | P0 |
| FIN-03 | Perhitungan laba kotor = penjualan dikurangi HPP; laba bersih = laba kotor dikurangi pengeluaran | P0 |
| FIN-04 | Rekap per metode pembayaran | P0 |
| FIN-05 | Rekonsiliasi kas harian | P1 |
| FIN-06 | Anggaran dan target penjualan per cabang | P2 |

### 5.9 Dashboard dan Laporan

**Dashboard Super Admin**

- Kartu ringkasan: total omzet, laba kotor, laba bersih, jumlah transaksi, unit terjual, rata-rata nilai transaksi, dengan perbandingan terhadap periode sebelumnya.
- Grafik tren penjualan (harian/mingguan/bulanan), perbandingan antar cabang.
- Ranking cabang berdasarkan omzet dan laba.
- Produk terlaris dan produk lambat laku.
- Stok kritis di seluruh cabang.
- Pemasukan vs pengeluaran, komposisi metode bayar.
- Feed aktivitas terbaru (transaksi, transfer stok, perubahan harga).
- Filter: periode (hari ini, 7 hari, bulan ini, custom) dan cabang (satu, beberapa, semua).

**Dashboard Admin Cabang:** seluruh metrik di atas hanya untuk cabangnya, ditambah performa per kasir, transfer stok masuk/keluar, dan shift aktif.

**Dashboard Kasir:** penjualan hari ini, shift berjalan, shortcut ke POS.

| ID | Kebutuhan laporan | Prioritas |
| --- | --- | --- |
| RPT-01 | Laporan penjualan (per hari, produk, kasir, cabang) | P0 |
| RPT-02 | Laporan laba rugi sederhana | P0 |
| RPT-03 | Laporan stok dan nilai persediaan | P0 |
| RPT-04 | Laporan pergerakan stok | P0 |
| RPT-05 | Export PDF dan Excel/CSV | P0 |
| RPT-06 | Laporan pengeluaran | P0 |
| RPT-07 | Laporan terjadwal via email | P2 |

### 5.10 Pelanggan, Garansi, dan Promo

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| CRM-01 | Data pelanggan dan riwayat pembelian | P1 |
| CRM-02 | Poin loyalitas | P2 |
| WAR-01 | Garansi otomatis berdasarkan IMEI dan tanggal jual | P1 |
| WAR-02 | Klaim servis dengan status dan riwayat | P2 |
| PRO-01 | Voucher dan promo per cabang atau global | P2 |

### 5.11 Notifikasi, Audit, dan Pengaturan

| ID | Kebutuhan | Prioritas |
| --- | --- | --- |
| NTF-01 | Notifikasi in-app: stok menipis, transfer baru, retur menunggu persetujuan | P1 |
| NTF-02 | Notifikasi WhatsApp/Telegram/email | P2 |
| AUD-01 | Audit log: login, perubahan harga, penyesuaian stok, penghapusan data, perubahan role (menyimpan nilai sebelum dan sesudah) | P0 |
| AUD-02 | Halaman audit log dengan filter user, cabang, aksi, tanggal (Super Admin) | P0 |
| SET-01 | Pengaturan global: nama toko, logo, mata uang, pajak, format struk | P1 |
| SET-02 | Backup dan pemulihan data | P1 |

## 6. Matriks Hak Akses (RBAC)

| Fitur | Super Admin | Admin Cabang | Kasir |
| --- | --- | --- | --- |
| Kelola cabang | CRUD | Lihat cabangnya | - |
| Kelola user | CRUD semua | CRUD kasir cabangnya | Profil sendiri |
| Produk master | CRUD | Lihat | Lihat |
| Harga override cabang | Ya | Ya (dengan log) | - |
| Stok | Semua cabang | Cabangnya | Lihat |
| Barang masuk / opname | Ya | Ya | - |
| Transfer stok | Setujui semua | Ajukan dan terima | - |
| POS dan transaksi | Lihat semua | Lihat cabangnya | Buat transaksi |
| Retur/refund | Ya | Setujui | Ajukan |
| Pengeluaran | Semua | Cabangnya | - |
| Dashboard | Semua cabang | Cabangnya | Ringkas |
| Laporan dan export | Semua | Cabangnya | - |
| Audit log | Semua | Cabangnya (baca) | - |
| Pengaturan sistem | Ya | - | - |

## 7. Alur Pengguna Utama

### 7.1 Transaksi Penjualan (Kasir)

1. Kasir login, membuka shift dan memasukkan modal awal.
2. Di POS, kasir scan barcode atau mencari produk; item masuk keranjang.
3. Untuk HP, kasir memindai/memasukkan IMEI unit.
4. Kasir menerapkan diskon (jika diizinkan), memilih pelanggan (opsional), dan metode bayar.
5. Sistem memvalidasi stok dan menyimpan transaksi secara atomic: Sale, SaleItem, pengurangan stok, StockMovement, ActivityLog.
6. Struk dicetak atau dikirim; dashboard cabang dan pusat ter-update.

### 7.2 Scan untuk Detail Produk

1. Pengguna membuka menu Scan dan mengarahkan kamera atau memakai scanner USB.
2. Sistem mencari varian berdasarkan barcode.
3. Jika ditemukan: tampil halaman detail produk (lihat 5.6). Jika tidak: pesan galat dengan opsi tindak lanjut sesuai role.

### 7.3 Transfer Stok Antar Cabang

1. Admin Cabang A mengajukan permintaan transfer ke Cabang B (varian, qty).
2. Admin Cabang B atau Super Admin menyetujui dan mengirim; stok Cabang B berkurang dengan status In Transit.
3. Admin Cabang A menerima dan mengonfirmasi jumlah; stok Cabang A bertambah. Selisih dicatat sebagai discrepancy.

### 7.4 Pemantauan Super Admin

1. Super Admin membuka dashboard gabungan, memilih periode dan cabang.
2. Melihat anomali (stok kritis, penjualan turun, selisih kas), lalu drill-down ke detail cabang atau transaksi.
3. Mengekspor laporan atau menindaklanjuti (setujui transfer, tegur cabang).

## 8. Model Data

| Entitas | Atribut Utama |
| --- | --- |
| **User** | id, nama, email, passwordHash, role, branchId (nullable), isActive, lastLoginAt |
| **Branch** | id, kode, nama, alamat, kota, telepon, isActive, pengaturan (JSON) |
| **Brand / Category** | id, nama, slug |
| **Product** | id, nama, brandId, categoryId, deskripsi, spesifikasi (JSON), hargaModal, hargaJual, isActive, deletedAt |
| **ProductImage** | id, productId, url, urutan, isPrimary |
| **ProductVariant** | id, productId, sku, barcode (unique), warna, storage, ram, harga override |
| **Stock** | id, variantId, branchId, qty, minQty (unique variantId + branchId) |
| **StockMovement** | id, variantId, branchId, tipe, qty, saldoSetelah, referensi, userId, catatan, createdAt |
| **StockTransfer / Item** | id, dariBranch, keBranch, status, diajukanOleh, disetujuiOleh, timestamp per status |
| **Unit (IMEI)** | id, variantId, branchId, imei, status, tanggalGaransiAkhir |
| **Sale / SaleItem** | id, noInvoice, branchId, kasirId, customerId, subtotal, diskon, pajak, total, metodeBayar, status; item: variantId, unitId, qty, harga, hpp |
| **Payment** | id, saleId, metode, jumlah, referensi |
| **Return** | id, saleId, alasan, status, disetujuiOleh, nilaiRefund |
| **Expense** | id, branchId, kategori, jumlah, tanggal, buktiUrl, userId |
| **Shift** | id, branchId, kasirId, modalAwal, kasSistem, kasFisik, selisih, buka, tutup |
| **Customer** | id, nama, telepon, email, poin |
| **ActivityLog** | id, userId, branchId, aksi, entitas, entitasId, sebelum, sesudah, ip, createdAt |
| **Notification** | id, userId, tipe, isi, isRead |

Catatan desain: gunakan integer (Rupiah penuh) atau decimal untuk uang, tidak pernah float; simpan HPP pada SaleItem agar laba historis tidak berubah ketika harga modal diubah; index pada branchId, barcode, imei, createdAt.

## 9. Kebutuhan Non-Fungsional

| Kategori | Kebutuhan |
| --- | --- |
| Performa | Dashboard p95 di bawah 2 detik; pencarian dan scan di bawah 500 ms; pagination server-side untuk semua tabel |
| Keamanan | HTTPS wajib; password di-hash; otorisasi di setiap endpoint/server action; validasi Zod di server; proteksi CSRF, XSS, SQL injection; secret via environment variable |
| Isolasi data | Query otomatis di-scope ke branchId sesuai sesi; diuji dengan tes otorisasi lintas cabang |
| Integritas | Transaksi database atomic untuk penjualan, transfer, dan retur; mencegah race condition stok (row lock atau optimistic locking) |
| Ketersediaan | Target uptime 99,5%; backup harian otomatis, retensi minimal 14 hari |
| Skalabilitas | Mendukung minimal 50 cabang, 200 user, 1 juta baris transaksi tanpa degradasi berarti |
| Responsif | Mobile-first untuk POS dan scan; dukungan Chrome, Safari, Edge terbaru |
| Aksesibilitas | Kontras WCAG AA, navigasi keyboard, label form |
| Lokalisasi | Bahasa Indonesia sebagai default, format Rupiah dan tanggal WIB/WITA/WIT |
| Observabilitas | Logging terstruktur, error tracking (Sentry), health check |
| PWA | Dapat diinstal; cache aset statis |

## 10. Arsitektur dan Tech Stack

| Layer | Pilihan |
| --- | --- |
| Framework | Next.js (App Router) + TypeScript |
| UI | Tailwind CSS, shadcn/ui (fondasi), Aceternity UI (bento grid, sidebar, spotlight), React Bits (count up, animasi teks, background) |
| State dan data | TanStack Query, TanStack Table, React Hook Form + Zod |
| Database | PostgreSQL (Neon/Supabase/VPS) |
| ORM | Prisma atau Drizzle |
| Auth | Auth.js atau Better Auth dengan session berbasis cookie httpOnly |
| File storage | Cloudinary, UploadThing, atau S3-compatible |
| Barcode | react-barcode / JsBarcode (generate), @zxing/browser atau html5-qrcode (scan) |
| Chart | Recharts via shadcn charts |
| Export | exceljs/xlsx untuk Excel, @react-pdf/renderer atau Puppeteer untuk PDF |
| Cetak struk | CSS print + ESC/POS via WebUSB/Web Bluetooth (opsional) |
| Deploy | Vercel atau VPS (Docker + Nginx), CI/CD via GitHub Actions |
| Pengujian | Vitest (unit), Playwright (E2E), tes otorisasi khusus |

**Prinsip arsitektur:** satu codebase monorepo; lapisan service terpisah dari UI; semua akses data melewati fungsi yang menerima konteks sesi (userId, role, branchId); logika bisnis kritis (penjualan, transfer, retur) dalam transaksi database.

### Struktur Rute

- `/login`
- `/dashboard` (menyesuaikan role)
- `/branches`, `/users`, `/products`, `/categories`, `/brands`
- `/inventory`, `/inventory/transfers`, `/inventory/opname`, `/inventory/units`
- `/pos`, `/sales`, `/returns`, `/shifts`
- `/expenses`, `/reports`, `/customers`, `/warranty`, `/promos`
- `/scan` dan `/p/[barcode]` (detail produk hasil scan)
- `/audit-log`, `/settings`, `/profile`

## 11. Desain UI/UX

- **Layout:** sidebar collapsible, topbar dengan pencarian global dan command palette (Cmd/Ctrl+K), pemilih cabang untuk Super Admin, toggle dark/light.
- **Prinsip clean:** ruang kosong lega, tipografi konsisten, palet netral dengan satu warna aksen, maksimal dua tingkat kedalaman visual per halaman.
- **Komponen:** kartu statistik bergaya bento grid (Aceternity) dengan angka animasi (React Bits Count Up); grafik shadcn charts; tabel TanStack Table dengan search, filter, sort, dan export.
- **Efek visual berat** (Aurora, Spotlight, Particles, Beams) hanya di halaman login dan landing; area data tetap ringan.
- **POS:** dua kolom (katalog dan keranjang), tombol besar untuk sentuh, fokus otomatis ke input scan.
- **State UI:** skeleton loading, empty state informatif, error state dengan aksi ulang, konfirmasi untuk aksi destruktif.
- **Mobile:** navigasi bawah untuk kasir, tabel berubah menjadi kartu.

## 12. Kriteria Penerimaan (Contoh Utama)

1. **Isolasi cabang:** Given Admin Cabang A login, when ia memanggil endpoint data cabang B (termasuk dengan manipulasi ID), then server menolak dengan 403.
2. **Scan barcode:** Given barcode valid, when discan, then halaman detail tampil dalam di bawah 1 detik dengan foto, spesifikasi, harga, dan stok.
3. **Barcode tidak dikenal:** Given barcode tidak terdaftar, when discan, then muncul pesan jelas dan tidak terjadi galat sistem.
4. **Penjualan:** Given item dengan stok 1, when dua kasir menjual bersamaan, then hanya satu transaksi berhasil dan stok tidak negatif.
5. **Stok otomatis:** Given transaksi sukses, then stok berkurang, StockMovement tercatat, dan dashboard ter-update.
6. **Dashboard:** Given periode dan filter cabang dipilih, then semua kartu dan grafik menyesuaikan, dan total cocok dengan laporan penjualan.
7. **Audit:** Given harga produk diubah, then audit log menyimpan nilai lama, nilai baru, pelaku, dan waktu.
8. **Retur:** Given retur disetujui, then stok kembali, nilai refund tercatat, dan laba periode menyesuaikan.

## 13. Roadmap dan Milestone

Estimasi untuk 1 sampai 2 developer fullstack.

| Fase | Cakupan | Estimasi |
| --- | --- | --- |
| **0. Persiapan** | Finalisasi PRD, desain UI (Figma), setup repo, CI, database | 1 minggu |
| **1. Fondasi** | Auth, RBAC, layout dashboard, manajemen cabang dan user | 2 minggu |
| **2. Master data** | Brand, kategori, produk, varian, upload gambar | 2 minggu |
| **3. Stok dan barcode** | Stok per cabang, barang masuk, generate dan cetak barcode, scan, halaman detail | 2 minggu |
| **4. POS dan transaksi** | Kasir, pembayaran, struk, shift, retur | 3 minggu |
| **5. Dashboard dan laporan** | Dashboard 3 role, keuangan, laporan, export | 2 minggu |
| **6. Fitur lanjutan** | Transfer stok, opname, IMEI, garansi, notifikasi, CRM, PWA | 3 minggu |
| **7. QA dan rilis** | UAT di 1 cabang pilot, perbaikan, migrasi data, pelatihan, go-live | 2 minggu |

**MVP (akhir Fase 5):** semua item P0 siap. **Rilis 1.1:** item P1. **Rilis 1.2+:** item P2.

## 14. Strategi Pengujian dan Rilis

- **Unit test** untuk kalkulasi laba, diskon, dan logika stok.
- **Integration test** untuk transaksi penjualan, transfer, dan retur.
- **Tes otorisasi** matriks role terhadap setiap endpoint.
- **E2E (Playwright):** login, scan, penjualan, laporan.
- **UAT** di satu cabang pilot selama 1 sampai 2 minggu sebelum roll-out.
- **Migrasi data:** impor produk dan stok awal via CSV; opname awal sebagai baseline.
- **Pelatihan:** panduan singkat per role dan video tutorial POS.
- **Rollback plan:** backup sebelum deploy, migrasi database yang reversible.

## 15. Risiko dan Mitigasi

| Risiko | Dampak | Mitigasi |
| --- | --- | --- |
| Kebocoran data antar cabang | Tinggi | Scope query terpusat, tes otorisasi, audit log |
| Selisih stok akibat race condition | Tinggi | Transaksi atomic, locking, pergerakan stok immutable |
| Staf menolak sistem baru | Sedang | UI sederhana, pelatihan, pilot satu cabang |
| Koneksi internet buruk di cabang | Sedang | PWA, caching, mode offline dasar (P2) |
| Scanner/printer tidak kompatibel | Sedang | Uji perangkat sejak Fase 3; dukung mode keyboard wedge dan fallback PDF |
| Scope creep fitur P2 | Sedang | Disiplin prioritas, rilis bertahap |
| Kehilangan data | Tinggi | Backup harian, uji restore berkala |

## 16. Asumsi dan Pertanyaan Terbuka

**Asumsi**

- Satu user hanya terikat ke satu cabang.
- Harga jual default global, dengan override opsional per cabang.
- Pembayaran non-tunai (QRIS/transfer) dicatat manual, belum terintegrasi payment gateway.
- Pajak (PPN) bersifat opsional dan diatur per toko.

**Perlu dikonfirmasi**

1. Berapa jumlah cabang, user, dan produk pada tahap awal?
2. Apakah IMEI wajib dicatat per unit sejak MVP, atau cukup di rilis berikutnya?
3. Apakah perlu integrasi payment gateway (Midtrans/Xendit) untuk QRIS dinamis?
4. Merek printer thermal dan scanner barcode yang dipakai cabang?
5. Apakah stok boleh dijual lintas cabang, atau harus transfer dulu?
6. Apakah perlu katalog publik (etalase online) pada versi awal?
7. Hosting yang dipilih: cloud terkelola atau server sendiri?
8. Apakah ada kebutuhan e-Faktur atau pelaporan pajak?

## 17. Glosarium

- **SKU:** kode unik internal untuk varian produk.
- **Barcode:** kode batang yang mengidentifikasi tipe/varian produk, bukan unit individual.
- **IMEI:** nomor identitas unik per unit handphone.
- **HPP:** harga pokok penjualan (harga modal barang yang terjual).
- **RBAC:** role-based access control.
- **POS:** point of sale, aplikasi kasir.
- **Stock opname:** penghitungan fisik stok untuk dicocokkan dengan sistem.
- **Soft delete:** data ditandai terhapus tanpa benar-benar dihapus dari database.

# PRD Modul 01: Autentikasi, Otorisasi, Manajemen Cabang & User

**Induk Dokumen:** [PRD.md](../PRD.md) · **Versi:** 1.0 · **Status:** Ready for Dev  
**Cakupan Requirement:** `AUTH-01` s/d `AUTH-07`, `BR-01` s/d `BR-04`, `USR-01` s/d `USR-05`

---

## 1. Tujuan & Ruang Lingkup Modul

Modul ini adalah fondasi keamanan dan struktur organisasi multi-cabang. Seluruh modul lain bergantung pada konteks sesi (`userId`, `role`, `branchId`) yang dihasilkan oleh modul ini.

### Cakupan Kebutuhan
| ID | Fitur | Prioritas | Target Rilis |
| --- | --- | --- | --- |
| `AUTH-01` | Login email dan password dengan hashing (`argon2id` / `bcrypt`) | P0 | MVP (Fase 1) |
| `AUTH-02` | RBAC dengan role `SUPER_ADMIN`, `BRANCH_ADMIN`, `KASIR`, `GUDANG` | P0 | MVP (Fase 1) |
| `AUTH-03` | Isolasi data cabang di level server (session-scoped query) | P0 | MVP (Fase 1) |
| `AUTH-04` | Logout, session expiry, silent refresh session | P0 | MVP (Fase 1) |
| `AUTH-05` | Reset password oleh Admin (Super Admin / Admin Cabang) | P0 | MVP (Fase 1) |
| `AUTH-06` | Rate limiting & akun lockout setelah 5x gagal login berturut-turut | P1 | Rilis 1.1 |
| `AUTH-07` | Two-Factor Authentication (TOTP) untuk Super Admin | P2 | Rilis 1.2 |
| `BR-01` | CRUD cabang: nama, kode, alamat, kota, telepon, jam operasional, status | P0 | MVP (Fase 1) |
| `BR-02` | Soft delete / menonaktifkan cabang (otomatis memblokir login user cabang) | P0 | MVP (Fase 1) |
| `BR-03` | Halaman detail cabang: ringkasan performa, daftar user, stok, transaksi | P0 | MVP (Fase 1) |
| `BR-04` | Pengaturan spesifik cabang: header/footer struk, pajak, WhatsApp | P1 | Rilis 1.1 |
| `USR-01` | Super Admin: CRUD semua user, assign role dan cabang | P0 | MVP (Fase 1) |
| `USR-02` | Admin Cabang: CRUD akun `KASIR` dan `GUDANG` khusus di cabangnya | P0 | MVP (Fase 1) |
| `USR-03` | Aktif/nonaktifkan user tanpa menghapus riwayat transaksi & log | P0 | MVP (Fase 1) |
| `USR-04` | Manajemen profil mandiri: ubah nama, avatar, dan password sendiri | P0 | MVP (Fase 1) |
| `USR-05` | Riwayat login dan log aktivitas spesifik per user | P1 | Rilis 1.1 |

---

## 2. User Stories & Acceptance Criteria (Gherkin)

### US-1.1: Login & Validasi Status Aktif (`AUTH-01`, `AUTH-03`, `BR-02`, `USR-03`)
**Sebagai** pengguna terdaftar, **saya ingin** masuk ke sistem menggunakan email dan password **agar** dapat mengakses fitur sesuai role dan cabang saya.

```gherkin
Scenario: Login berhasil sebagai Admin Cabang atau Kasir
  Given user memiliki akun aktif dengan role "KASIR" di cabang "CBG-JKT01"
  And cabang "CBG-JKT01" berstatus aktif (isActive = true)
  When user memasukkan email dan password yang benar di "/login"
  Then sistem membuat session cookie httpOnly yang memuat userId, role="KASIR", dan branchId="CBG-JKT01"
  And mencatat waktu login di field "lastLoginAt" serta tabel "ActivityLog"
  And mengarahkan user ke halaman "/dashboard"

Scenario: Login ditolak karena cabang dinonaktifkan
  Given user memiliki akun aktif di cabang "CBG-BDG02"
  And Super Admin telah menonaktifkan cabang "CBG-BDG02" (isActive = false)
  When user mencoba login dengan kredensial yang benar
  Then sistem menolak login dengan status 403 Forbidden
  And menampilkan pesan "Cabang Anda sedang dinonaktifkan. Hubungi manajemen pusat."

Scenario: Login ditolak karena akun dinonaktifkan atau kredensial salah
  Given user memasukkan password yang salah
  When form login dikirimkan
  Then sistem mengembalikan pesan generik "Email atau password salah" (mencegah user enumeration)
  And menambah counter failedLoginAttempts pada akun terkait
```

### US-1.2: Isolasi Data Lintas Cabang (`AUTH-02`, `AUTH-03`)
**Sebagai** Super Admin, **saya ingin** memastikan Admin Cabang dan Kasir hanya bisa mengakses data cabangnya sendiri **agar** kerahasiaan dan integritas operasional antar cabang terjaga.

```gherkin
Scenario: Manipulasi parameter branchId oleh Admin Cabang
  Given "Admin Cabang A" sedang login (session.branchId = "branch-a")
  When ia memanggil API atau Server Action dengan payload atau query "?branchId=branch-b"
  Then middleware/service layer mengabaikan parameter input "branch-b" atau menolak dengan 403 Forbidden
  And mencatat percobaan akses ilegal ke "ActivityLog"

Scenario: Context Switching Cabang oleh Super Admin
  Given "Super Admin" sedang login (session.branchId = null)
  When ia memilih filter cabang "Cabang Surabaya" di topbar
  Then seluruh query dashboard dan tabel di-scope ke "Cabang Surabaya"
  And ketika memilih "Semua Cabang", data diagregasi dari seluruh cabang
```

### US-1.3: Manajemen Cabang oleh Super Admin (`BR-01`, `BR-02`, `BR-03`)
**Sebagai** Super Admin, **saya ingin** menambah, mengubah, melihat detail, dan menonaktifkan cabang **agar** ekspansi atau penutupan toko tercatat rapi tanpa merusak data historis.

```gherkin
Scenario: Membuat cabang baru dengan kode unik
  Given Super Admin berada di halaman "/branches"
  When ia mengisi form cabang baru dengan kode "JKT-01" yang belum pernah dipakai
  Then cabang baru tersimpan dengan status aktif
  And pengaturan default struk serta pajak otomatis diinisialisasi

Scenario: Menonaktifkan cabang yang masih memiliki sesi user aktif
  Given cabang "JKT-01" memiliki 3 kasir yang sedang login
  When Super Admin mengubah status cabang "JKT-01" menjadi Nonaktif
  Then semua sesi aktif milik user di cabang "JKT-01" langsung diinvalidasi (revoke)
  And transaksi baru dari cabang tersebut ditolak oleh server
```

### US-1.4: Manajemen User Berjenjang (`USR-01`, `USR-02`, `AUTH-05`)
**Sebagai** Admin Cabang, **saya ingin** mengelola akun Kasir di cabang saya sendiri tanpa bisa membuat akun untuk cabang lain atau meningkatkan role menjadi Admin.

```gherkin
Scenario: Admin Cabang membuat akun Kasir baru
  Given Admin Cabang "BDG-01" membuka halaman "/users"
  When ia membuat user baru
  Then pilihan Role hanya menampilkan "KASIR" dan "GUDANG"
  And field Cabang terkunci otomatis ke "BDG-01"

Scenario: Admin Cabang mencoba mereset password user dari cabang lain
  Given Admin Cabang "BDG-01" mengirimkan request reset password untuk userId milik cabang "SBY-01"
  When server memvalidasi target userId
  Then server menolak request dengan kode 403 Forbidden
```

---

## 3. Skema Database & Aturan Bisnis

### 3.1 Definisi Tabel (Prisma-style Schema)

```prisma
enum Role {
  SUPER_ADMIN
  BRANCH_ADMIN
  KASIR
  GUDANG
}

model Branch {
  id              String   @id @default(cuid())
  code            String   @unique @db.VarChar(20) // Contoh: JKT-01
  name            String   @db.VarChar(100)
  address         String   @db.Text
  city            String   @db.VarChar(50)
  phone           String   @db.VarChar(30)
  operatingHours  String?  @db.VarChar(100) // Contoh: "09:00 - 21:00 WIB"
  timezone        String   @default("Asia/Jakarta") // WIB, WITA, WIT
  isActive        Boolean  @default(true)
  settings        Json     @default("{}") // { receiptHeader, receiptFooter, whatsapp, taxPercent, isTaxInclusive }
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  users           User[]
  @@index([isActive])
  @@index([city])
}

model User {
  id                  String    @id @default(cuid())
  name                String    @db.VarChar(100)
  email               String    @unique @db.VarChar(150)
  passwordHash        String
  avatarUrl           String?
  role                Role
  branchId            String?   // Wajib diisi jika role != SUPER_ADMIN
  isActive            Boolean   @default(true)
  failedLoginAttempts Int       @default(0)
  lockedUntil         DateTime?
  twoFactorSecret     String?   // P2: TOTP secret terenkripsi
  twoFactorEnabled    Boolean   @default(false)
  lastLoginAt         DateTime?
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt

  branch              Branch?   @relation(fields: [branchId], references: [id])
  sessions            Session[]

  @@index([branchId, role, isActive])
}

model Session {
  id           String   @id @default(cuid())
  userId       String
  token        String   @unique
  ipAddress    String?
  userAgent    String?
  expiresAt    DateTime
  createdAt    DateTime @default(now())

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@index([userId])
  @@index([expiresAt])
}
```

### 3.2 Aturan Bisnis & Invarian
1. **Invarian `branchId`:** Jika `role !== 'SUPER_ADMIN'`, maka `branchId` **wajib** bernilai valid (`NOT NULL`). Jika `role === 'SUPER_ADMIN'`, `branchId` bernilai `NULL`.
2. **Proteksi Super Admin Terakhir:** Sistem harus menolak penonaktifan atau penghapusan akun `SUPER_ADMIN` jika akun tersebut adalah satu-satunya `SUPER_ADMIN` aktif di database.
3. **Pencegahan Self-Lockout:** User yang sedang login tidak dapat menonaktifkan akunnya sendiri atau menurunkan role-nya sendiri.
4. **Revokasi Sesi Otomatis:** Saat `User.isActive` diubah menjadi `false`, `Branch.isActive` diubah menjadi `false`, atau password di-reset, semua baris `Session` terkait wajib dihapus dari database.
5. **Rate Limiting & Lockout (`AUTH-06`):** Maksimal 5 kali percobaan gagal dalam 15 menit per kombinasi email/IP. Jika terlampaui, `lockedUntil` diset `now() + 15 menit`.

---

## 4. Spesifikasi API / Server Actions & Validasi Zod

### 4.1 Helper Konteks Otorisasi (Wajib di Semua Service)
```typescript
export interface SessionContext {
  userId: string;
  role: 'SUPER_ADMIN' | 'BRANCH_ADMIN' | 'KASIR' | 'GUDANG';
  branchId: string | null; // null hanya untuk SUPER_ADMIN
}

// Helper yang memaksa filter cabang di semua query
export function resolveBranchFilter(ctx: SessionContext, requestedBranchId?: string): string | undefined {
  if (ctx.role === 'SUPER_ADMIN') {
    return requestedBranchId || undefined; // Bisa akses cabang tertentu atau semua cabang
  }
  if (requestedBranchId && requestedBranchId !== ctx.branchId) {
    throw new ForbiddenError('FORBIDDEN_CROSS_BRANCH_ACCESS');
  }
  return ctx.branchId!;
}
```

### 4.2 Skema Validasi Zod

```typescript
import { z } from 'zod';

export const LoginSchema = z.object({
  email: z.string().email('Format email tidak valid').trim().toLowerCase(),
  password: z.string().min(8, 'Password minimal 8 karakter'),
  totpCode: z.string().length(6).optional(),
});

export const BranchInputSchema = z.object({
  code: z.string().regex(/^[A-Z0-9-]{3,20}$/, 'Kode cabang hanya boleh huruf kapital, angka, dan strip'),
  name: z.string().min(3).max(100),
  address: z.string().min(5),
  city: z.string().min(2).max(50),
  phone: z.string().min(8).max(30),
  operatingHours: z.string().max(100).optional(),
  timezone: z.enum(['Asia/Jakarta', 'Asia/Makassar', 'Asia/Jayapura']).default('Asia/Jakarta'),
  settings: z.object({
    receiptHeader: z.string().max(200).optional(),
    receiptFooter: z.string().max(200).optional(),
    whatsapp: z.string().max(30).optional(),
    taxPercent: z.number().min(0).max(100).default(0),
    isTaxInclusive: z.boolean().default(true),
  }).optional(),
});

export const CreateUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email().trim().toLowerCase(),
  password: z.string().min(8).regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, 'Wajib kombinasi huruf besar, kecil, dan angka'),
  role: z.enum(['SUPER_ADMIN', 'BRANCH_ADMIN', 'KASIR', 'GUDANG']),
  branchId: z.string().nullable(),
}).refine((data) => data.role === 'SUPER_ADMIN' ? data.branchId === null : Boolean(data.branchId), {
  message: 'Selain Super Admin, user wajib ditempatkan pada satu cabang',
  path: ['branchId'],
});
```

### 4.3 Daftar Endpoint / Server Actions

| Action / Endpoint | Metode | Role Diizinkan | Deskripsi & Error Codes |
| --- | --- | --- | --- |
| `POST /api/auth/login` | POST | Public | Validasi kredensial, cek lockout & cabang aktif. Error: `401 INVALID_CREDENTIALS`, `403 ACCOUNT_INACTIVE`, `403 BRANCH_INACTIVE`, `429 ACCOUNT_LOCKED`. |
| `POST /api/auth/logout` | POST | Authenticated | Hapus sesi dari DB dan clear cookie. |
| `GET /api/branches` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` | Daftar cabang (Admin Cabang hanya menerima data cabangnya). |
| `POST /api/branches` | POST | `SUPER_ADMIN` | Buat cabang baru. Error: `409 BRANCH_CODE_EXISTS`. |
| `PATCH /api/branches/:id` | PATCH | `SUPER_ADMIN`, `BRANCH_ADMIN` (hanya `settings` cabangnya) | Update info cabang atau aktif/nonaktifkan cabang. |
| `GET /api/users` | GET | `SUPER_ADMIN`, `BRANCH_ADMIN` | List user dengan server-side pagination, filter role, status, cabang. |
| `POST /api/users` | POST | `SUPER_ADMIN`, `BRANCH_ADMIN` | Buat akun baru. `BRANCH_ADMIN` hanya boleh membuat `KASIR`/`GUDANG` di cabangnya. Error: `403 ROLE_ESCALATION_DENIED`, `409 EMAIL_EXISTS`. |
| `POST /api/users/:id/reset-password` | POST | `SUPER_ADMIN`, `BRANCH_ADMIN` | Reset password user dan paksa logout semua sesi target user. |
| `PATCH /api/profile` | PATCH | Semua Role | Update nama, foto profil, atau ganti password sendiri (wajib kirim `currentPassword`). |

---

## 5. Spesifikasi UI/UX & State Komponen

### 5.1 Halaman Login (`/login`)
- **Visual:** Split layout di desktop (kiri: branding toko dengan background *Aceternity Aurora/Spotlight* yang halus; kanan: card form login bersih).
- **Komponen:** Input Email, Input Password dengan toggle *show/hide*, tombol Submit dengan spinner loading, alert box jika gagal login atau terkunci.

### 5.2 Manajemen Cabang (`/branches` & `/branches/[id]`)
- **Tabel Cabang (`/branches`):** Menggunakan TanStack Table dengan kolom: Kode, Nama Cabang, Kota, Jumlah Staf Aktif, Omzet Bulan Ini, Status Badge (`Aktif` hijau / `Nonaktif` abu-abu), dan menu Aksi.
- **Detail Cabang (`/branches/[id]`):**
  - **Header:** Nama cabang, badge status, tombol "Edit Cabang", tombol "Nonaktifkan Cabang" (dengan dialog konfirmasi ketik kode cabang).
  - **Tabs:**
    1. *Ringkasan:* 4 kartu statistik (Omzet, Laba, Stok Kritis, Transaksi Hari Ini).
    2. *Staf Cabang:* Tabel user yang terikat ke cabang ini + tombol "+ Tambah Staf".
    3. *Pengaturan Struk & Pajak:* Form preview struk 58mm/80mm secara live di sisi kanan.

### 5.3 Manajemen User (`/users` & `/profile`)
- **Modal Tambah/Edit User:** Jika `role` dipilih `SUPER_ADMIN`, dropdown Cabang otomatis *disabled* dan menampilkan "Semua Cabang (Pusat)".
- **Dialog Reset Password Admin:** Dapat meng-generate password sementara acak 10 karakter dengan tombol *Copy to Clipboard*.
- **State UI Wajib:**
  - *Loading:* Skeleton table rows (5 baris).
  - *Empty State:* Ilustrasi + tombol CTA jika filter pencarian tidak menemukan user/cabang.
  - *Destructive Confirmation:* AlertDialog saat menonaktifkan cabang/user, menjelaskan dampaknya (user terkait akan otomatis ter-logout).


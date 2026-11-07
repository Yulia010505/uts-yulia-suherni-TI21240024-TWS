# LAPORAN PERANCANGAN DAN PENGUJIAN BACKEND

## Sistem Rental Kendaraan

| Identitas | Keterangan |
|---|---|
| Nama Mahasiswa | Yulia |
| NIM (3 digit terakhir) | 024 |
| Mata Kuliah | Pengembangan Backend / Pemrograman Web Lanjut |
| Teknologi | NestJS, Prisma ORM, MariaDB |
| Tanggal | 7 November 2026 |

> **Catatan autentisitas data:** Seluruh contoh data dalam laporan ini memakai
> akhiran `024` sesuai tiga digit NIM yang diberikan. ID relasi seperti `1`
> adalah contoh dan harus merujuk pada data yang benar-benar tersedia di
> database saat pengujian.
>
> **Status implementasi:** Proyek menyediakan REST API NestJS dan GraphQL
> berbasis Apollo Server pada endpoint `/graphql`, dengan integrasi Prisma
> provider MySQL yang kompatibel dengan MariaDB. Operasi GraphQL query,
> mutation, dan relasi nested diuji melalui tes e2e dengan mock Prisma. Tes
> tersebut bukan bukti operasi CRUD terhadap MariaDB. Screenshot Postman dan
> Apollo Sandbox harus diambil dari sesi pengujian yang dijalankan pengguna.

---

# BAB I: PERANCANGAN SISTEM (SYSTEM DESIGN)

## 1.1 Arsitektur Proyek & Struktur Direktori

Backend dirancang menggunakan NestJS dengan pemisahan tanggung jawab antara
controller (HTTP), service (logika aplikasi), DTO (validasi input), dan Prisma
Service (akses database). Fitur dikelompokkan per entitas; tidak ada direktori
pusat bernama `modules`. Susunan berikut menunjukkan rancangan struktur yang
diminta. Implementasi yang ada saat ini menggunakan folder fitur
`Pelanggan`, `Kendaraan`, `Penyewaan`, serta folder `prisma`.

```text
rental-kendaraan/
├── prisma/
│   └── schema.prisma
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── app.controller.ts
│   ├── app.service.ts
│   ├── prisma/
│   │   ├── prisma.module.ts
│   │   └── prisma.service.ts
│   ├── graphql/
│   │   ├── graphql.types.ts
│   │   └── graphql.inputs.ts
│   ├── pelanggan/
│   │   ├── pelanggan.module.ts
│   │   ├── pelanggan.controller.ts
│   │   ├── pelanggan.resolver.ts
│   │   ├── pelanggan.service.ts
│   │   └── dto/
│   │       ├── create-pelanggan.dto.ts
│   │       └── update-pelanggan.dto.ts
│   ├── kendaraan/
│   │   ├── kendaraan.module.ts
│   │   ├── kendaraan.controller.ts
│   │   ├── kendaraan.resolver.ts
│   │   ├── kendaraan.service.ts
│   │   └── dto/
│   │       ├── create-kendaraan.dto.ts
│   │       └── update-kendaraan.dto.ts
│   └── penyewaan/
│       ├── penyewaan.module.ts
│       ├── penyewaan.controller.ts
│       ├── penyewaan.resolver.ts
│       ├── penyewaan.service.ts
│       └── dto/
│           ├── create-penyewaan.dto.ts
│           └── update-penyewaan.dto.ts
├── test/
├── .env
├── package.json
└── tsconfig.json
```

Variabel koneksi diletakkan di `.env`, bukan ditulis langsung di source code.
Contoh format URL MariaDB:

```env
DATABASE_URL="mysql://USER:PASSWORD@HOST:3306/NAMA_DATABASE"
```

## 1.2 Perancangan Basis Data

Skema berikut menggunakan Prisma provider `mysql`, yang digunakan Prisma untuk
koneksi ke MySQL maupun MariaDB. Relasi `Pelanggan`–`Penyewaan` dan
`Kendaraan`–`Penyewaan` masing-masing adalah one-to-many. Data transaksi
menyimpan foreign key wajib ke satu pelanggan dan satu kendaraan.

```prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Pelanggan {
  id        Int         @id @default(autoincrement())
  nama      String
  penyewaan Penyewaan[]
}

model Kendaraan {
  id        Int         @id @default(autoincrement())
  platNomor String
  merk      String
  penyewaan Penyewaan[]
}

model Penyewaan {
  id           Int       @id @default(autoincrement())
  tanggal_sewa DateTime
  total_biaya  Float

  pelangganId Int
  pelanggan   Pelanggan @relation(fields: [pelangganId], references: [id])

  kendaraanId Int
  kendaraan   Kendaraan @relation(fields: [kendaraanId], references: [id])
}
```

**Catatan rancangan:** `total_biaya` dipertahankan sebagai `Float` karena
spesifikasi skema menetapkannya demikian. Untuk sistem transaksi keuangan
produksi, pertimbangkan tipe desimal (`Decimal`) agar representasi nilai uang
lebih presisi.

## 1.3 Perancangan DTO & Validasi

Contoh DTO berikut menggunakan `class-validator` dan `PartialType` dari
`@nestjs/mapped-types`. `PartialType` menjadikan seluruh properti pada DTO
Create opsional untuk operasi PATCH, sambil mempertahankan aturan validasi
properti yang diberikan. Pastikan dependensi `@nestjs/mapped-types` terpasang
sebelum memakai implementasi DTO ini.

### DTO Pelanggan

```typescript
import { IsNotEmpty, IsString } from 'class-validator';

export class CreatePelangganDto {
  @IsString()
  @IsNotEmpty()
  nama!: string;
}
```

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreatePelangganDto } from './create-pelanggan.dto.js';

export class UpdatePelangganDto extends PartialType(CreatePelangganDto) {}
```

### DTO Kendaraan

```typescript
import { IsNotEmpty, IsString } from 'class-validator';

export class CreateKendaraanDto {
  @IsString()
  @IsNotEmpty()
  platNomor!: string;

  @IsString()
  @IsNotEmpty()
  merk!: string;
}
```

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreateKendaraanDto } from './create-kendaraan.dto.js';

export class UpdateKendaraanDto extends PartialType(CreateKendaraanDto) {}
```

### DTO Penyewaan

```typescript
import {
  IsDateString,
  IsInt,
  IsNumber,
  IsPositive,
  Min,
} from 'class-validator';

export class CreatePenyewaanDto {
  @IsDateString()
  tanggal_sewa!: string;

  @IsNumber()
  @Min(0)
  total_biaya!: number;

  @IsInt()
  @IsPositive()
  pelangganId!: number;

  @IsInt()
  @IsPositive()
  kendaraanId!: number;
}
```

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreatePenyewaanDto } from './create-penyewaan.dto.js';

export class UpdatePenyewaanDto extends PartialType(CreatePenyewaanDto) {}
```

Pipe validasi global yang digunakan untuk REST sebaiknya mengaktifkan
`whitelist`, `forbidNonWhitelisted`, dan `transform`. Selain itu, `ParseIntPipe`
digunakan pada parameter ID agar ID pada path harus berupa bilangan bulat.
Format tanggal pada payload contoh adalah ISO 8601 date-only (`YYYY-MM-DD`);
`@IsDateString()` memvalidasi format tanggal ISO, tetapi bila sistem wajib
menerima date-only saja, pembatasan format itu perlu dibuat lebih spesifik.

## 1.4 Tabel Spesifikasi Endpoint REST API

Base URL pengujian lokal: `http://localhost:3000`. Endpoint aktual berada pada
`/pelanggan`, `/kendaraan`, dan `/penyewaan`.

| Modul | HTTP Method | URI | Body Payload JSON | Keterangan NIM |
|---|---|---|---|---|
| Pelanggan | POST | `/pelanggan` | `{"nama":"Yulia 024"}` | Nama menyertakan `024` |
| Pelanggan | GET | `/pelanggan` | — | Mengambil daftar; data uji yang tampil memakai nama `Yulia 024` |
| Pelanggan | GET | `/pelanggan/1` | — | ID contoh; pastikan pelanggan uji `Yulia 024` tersedia |
| Pelanggan | PATCH | `/pelanggan/1` | `{"nama":"Yulia 024 Update"}` | Nama hasil pembaruan menyertakan `024` |
| Pelanggan | DELETE | `/pelanggan/1` | — | Jalankan setelah transaksi terkait dihapus |
| Kendaraan | POST | `/kendaraan` | `{"platNomor":"DR 024 AB","merk":"Toyota 024"}` | Plat dan merk menyertakan `024` |
| Kendaraan | GET | `/kendaraan` | — | Mengambil daftar kendaraan uji beridentitas `024` |
| Kendaraan | GET | `/kendaraan/1` | — | ID contoh; pastikan kendaraan uji `DR 024 AB` tersedia |
| Kendaraan | PATCH | `/kendaraan/1` | `{"platNomor":"DR 024 CD","merk":"Toyota 024 Revisi"}` | Nilai yang diperbarui menyertakan `024` |
| Kendaraan | DELETE | `/kendaraan/1` | — | Jalankan setelah transaksi terkait dihapus |
| Penyewaan | POST | `/penyewaan` | `{"tanggal_sewa":"2026-11-07","total_biaya":150024,"pelangganId":1,"kendaraanId":1}` | Biaya menyertakan digit `024`; FK harus ada |
| Penyewaan | GET | `/penyewaan` | — | Daftar mengandung transaksi biaya `150024` dan relasi |
| Penyewaan | GET | `/penyewaan/1` | — | ID contoh; detail memuat relasi pelanggan dan kendaraan |
| Penyewaan | PATCH | `/penyewaan/1` | `{"total_biaya":175024}` | Biaya baru menyertakan digit `024` |
| Penyewaan | DELETE | `/penyewaan/1` | — | Menghapus data transaksi uji |

## 1.5 Tabel Spesifikasi Antarmuka GraphQL

**Kontrak GraphQL yang tersedia di `/graphql`.** Skema code-first dibentuk dari
GraphQL object type dan resolver; klien GraphiQL atau Apollo Sandbox dapat
mengakses operasi berikut.

| Tipe Operasi | Nama Query/Mutation | Argument | Return Type |
|---|---|---|---|
| Query | `pelangganList` | Tidak ada | `[Pelanggan!]!` |
| Query | `pelanggan` | `id: Int!` | `Pelanggan!` |
| Query | `kendaraanList` | Tidak ada | `[Kendaraan!]!` |
| Query | `kendaraan` | `id: Int!` | `Kendaraan!` |
| Query | `penyewaanList` | Tidak ada | `[Penyewaan!]!` |
| Query | `penyewaan` | `id: Int!` | `Penyewaan!` |
| Mutation | `createPelanggan` | `input: CreatePelangganInput!` | `Pelanggan!` |
| Mutation | `createKendaraan` | `input: CreateKendaraanInput!` | `Kendaraan!` |
| Mutation | `createPenyewaan` | `input: CreatePenyewaanInput!` | `Penyewaan!` |
| Mutation | `updatePenyewaan` | `id: Int!`, `input: UpdatePenyewaanInput!` | `Penyewaan!` |
| Mutation | `deletePenyewaan` | `id: Int!` | `Penyewaan!` |

Tipe `Penyewaan` pada rancangan GraphQL memuat `id`, `tanggal_sewa`,
`total_biaya`, `pelangganId`, `kendaraanId`, serta field relasi
`pelanggan: Pelanggan!` dan `kendaraan: Kendaraan!`. Field tanggal direpresentasi
sebagai `String` ISO 8601 pada kontrak ini. Query nested harus mengembalikan
relasi melalui resolver atau pemuatan relasi Prisma.

---

# BAB II: PENGUJIAN REST API (CONTROLLER)

## 2.1 Modul Pelanggan

### A. POST `/pelanggan`

**Deskripsi:** Membuat pelanggan baru. Pastikan server dan koneksi MariaDB aktif.

```http
POST http://localhost:3000/pelanggan
Content-Type: application/json
```

```json
{
  "nama": "Yulia 024"
}
```

**Hasil yang diharapkan:** HTTP `201 Created`; respons berisi `message` dan
`data` pelanggan dengan `nama` bernilai `Yulia 024`.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### B. GET `/pelanggan`

**Deskripsi:** Mengambil seluruh pelanggan.

```http
GET http://localhost:3000/pelanggan
```

**Hasil yang diharapkan:** HTTP `200 OK`; properti `data` berupa daftar
pelanggan, termasuk `Yulia 024` bila data tersebut sudah dibuat.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### C. GET `/pelanggan/1`

**Deskripsi:** Mengambil pelanggan berdasarkan ID yang tersedia.

```http
GET http://localhost:3000/pelanggan/1
```

**Hasil yang diharapkan:** HTTP `200 OK` dan detail pelanggan. Gunakan ID yang
dikembalikan oleh operasi POST, bukan mengasumsikan database selalu ber-ID `1`.
ID tidak ditemukan menghasilkan HTTP `404`.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### D. PATCH `/pelanggan/1`

**Deskripsi:** Memperbarui nama pelanggan tanpa mengganti seluruh record.

```http
PATCH http://localhost:3000/pelanggan/1
Content-Type: application/json
```

```json
{
  "nama": "Yulia 024 Update"
}
```

**Hasil yang diharapkan:** HTTP `200 OK`; nama di respons berubah menjadi
`Yulia 024 Update`.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### E. DELETE `/pelanggan/1`

**Deskripsi:** Menghapus pelanggan uji. Hapus dahulu penyewaan yang masih
mereferensikan pelanggan tersebut karena foreign key menghubungkan kedua tabel.

```http
DELETE http://localhost:3000/pelanggan/1
```

**Hasil yang diharapkan:** HTTP `200 OK` dan data pelanggan yang dihapus.
Penghapusan yang melanggar foreign key diharapkan ditolak sebagai konflik.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

## 2.2 Modul Kendaraan

### A. POST `/kendaraan`

**Deskripsi:** Mendaftarkan kendaraan baru dengan data identitas NIM.

```http
POST http://localhost:3000/kendaraan
Content-Type: application/json
```

```json
{
  "platNomor": "DR 024 AB",
  "merk": "Toyota 024"
}
```

**Hasil yang diharapkan:** HTTP `201 Created`; respons memuat plat `DR 024 AB`
dan merk `Toyota 024`.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### B. GET `/kendaraan`

**Deskripsi:** Mengambil daftar kendaraan yang tersimpan.

```http
GET http://localhost:3000/kendaraan
```

**Hasil yang diharapkan:** HTTP `200 OK`; respons berisi daftar kendaraan,
termasuk `DR 024 AB` apabila sudah dibuat.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### C. GET `/kendaraan/1`

**Deskripsi:** Mengambil detail kendaraan berdasarkan ID yang dikembalikan
operasi POST.

```http
GET http://localhost:3000/kendaraan/1
```

**Hasil yang diharapkan:** HTTP `200 OK` dan detail kendaraan uji. ID yang tidak
ditemukan menghasilkan HTTP `404`.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### D. PATCH `/kendaraan/1`

**Deskripsi:** Mengubah data kendaraan uji.

```http
PATCH http://localhost:3000/kendaraan/1
Content-Type: application/json
```

```json
{
  "platNomor": "DR 024 CD",
  "merk": "Toyota 024 Revisi"
}
```

**Hasil yang diharapkan:** HTTP `200 OK`; respons menampilkan plat dan merk
hasil pembaruan.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### E. DELETE `/kendaraan/1`

**Deskripsi:** Menghapus kendaraan uji setelah penyewaan yang mereferensikan
kendaraan tersebut sudah dihapus.

```http
DELETE http://localhost:3000/kendaraan/1
```

**Hasil yang diharapkan:** HTTP `200 OK` dan data kendaraan yang dihapus.
Kendaraan yang masih dirujuk transaksi tidak boleh dihapus.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

## 2.3 Modul Penyewaan

### A. POST `/penyewaan`

**Deskripsi:** Membuat transaksi penyewaan. Pelanggan dan kendaraan dengan ID
yang dikirim harus sudah tersedia di database.

```http
POST http://localhost:3000/penyewaan
Content-Type: application/json
```

```json
{
  "tanggal_sewa": "2026-11-07",
  "total_biaya": 150024,
  "pelangganId": 1,
  "kendaraanId": 1
}
```

**Hasil yang diharapkan:** HTTP `201 Created`; respons memuat transaksi beserta
relasi pelanggan dan kendaraan. Nilai `pelangganId` dan `kendaraanId` wajib
disesuaikan dengan ID yang benar-benar dibuat pada pengujian.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

### B. GET `/penyewaan`

**Deskripsi:** Mengambil seluruh transaksi beserta relasi pelanggan dan kendaraan.

```http
GET http://localhost:3000/penyewaan
```

**Hasil yang diharapkan:** HTTP `200 OK`; daftar pada `data` menampilkan
`total_biaya` dan objek `pelanggan` serta `kendaraan`.

[ Tempatkan Screenshot Hasil Pengujian Postman di Sini ]

---

# BAB III: PENGUJIAN GRAPHQL & APOLLO SANDBOX

> **Status:** GraphQL/Apollo sudah diaktifkan. Tes e2e memverifikasi query list
> dengan nested relation, mutation create, dan validasi input menggunakan mock
> Prisma. Hubungkan Apollo Sandbox ke `http://localhost:3000/graphql` saat
> server berjalan dan database dikonfigurasi untuk pengujian langsung.

## 3.1 GraphQL Mutation (Create Penyewaan)

**Tujuan:** Membuat transaksi dengan input tanggal date-only, biaya yang
menyertakan digit NIM `024`, dan foreign key yang merujuk record tersedia.
Contoh berikut memakai input `CreatePenyewaanInput` dan mutation
`createPenyewaan(input:)`.

```graphql
mutation CreatePenyewaan {
  createPenyewaan(
    input: {
      tanggal_sewa: "2026-11-07"
      total_biaya: 150024
      pelangganId: 1
      kendaraanId: 1
    }
  ) {
    id
    tanggal_sewa
    total_biaya
    pelanggan {
      nama
    }
    kendaraan {
      merk
      platNomor
    }
  }
}
```

**Hasil yang diharapkan:** mutation menghasilkan transaksi baru; tanggal
terserialisasi sebagai string ISO, biaya `150024`, serta relasi pelanggan dan
kendaraan sesuai foreign key. Untuk pengujian nyata, foreign key harus
menunjuk data yang sudah tersimpan.

[ Tempatkan Screenshot Hasil Pengujian Apollo Sandbox di Sini ]

## 3.2 GraphQL Query (Get Penyewaan List)

**Tujuan:** Mengambil daftar penyewaan dan menguji resolusi field nested
pelanggan serta kendaraan.

```graphql
query GetPenyewaanList {
  penyewaanList {
    id
    tanggal_sewa
    total_biaya
    pelanggan {
      nama
    }
    kendaraan {
      merk
      platNomor
    }
  }
}
```

**Hasil yang diharapkan:** field `penyewaanList` mengembalikan array transaksi
dengan tanggal, biaya, nama pelanggan, merk, dan plat nomor kendaraan.

[ Tempatkan Screenshot Hasil Pengujian Apollo Sandbox di Sini ]

---

# BAB IV: KESIMPULAN

1. Rancangan backend memisahkan tanggung jawab API, logika layanan, DTO, dan
   akses database untuk model Pelanggan, Kendaraan, dan Penyewaan.
2. Prisma menyediakan pemetaan relasi one-to-many dan integrasi database
   MariaDB melalui provider `mysql`; transaksi mensyaratkan pelanggan dan
   kendaraan yang valid.
3. DTO `class-validator` menetapkan validasi tipe nama, kendaraan, tanggal,
   biaya, dan foreign key. Untuk menerapkan DTO PATCH berbasis `PartialType`,
   dependensi `@nestjs/mapped-types` perlu ditambahkan ke proyek.
4. REST API dan GraphQL Apollo Server tersedia. Query list dengan nested
   relation dan mutation create sudah diuji pada tes e2e menggunakan mock
   Prisma; pengujian CRUD langsung terhadap MariaDB tetap perlu dilakukan.
5. Semua contoh payload memasukkan akhiran NIM `024` pada data yang relevan.
   Bukti uji Postman/Apollo harus dilengkapi dengan tangkapan layar hasil
   pengujian nyata; laporan ini tidak mengklaim tangkapan layar atau uji
   database tersebut sudah dilakukan.

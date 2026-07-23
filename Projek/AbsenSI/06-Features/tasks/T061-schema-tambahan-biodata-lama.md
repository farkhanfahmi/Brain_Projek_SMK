# T061 — Schema: Tambah Field Biodata dari Database Lama

## Depends on
T002 (schema dasar Fase 1). Tidak depend ke task Fase 2 manapun — bisa dikerjakan kapan saja, tapi WAJIB selesai sebelum T062 (ETL migrasi data).

## Objective
Tambahkan kolom yang hilang saat rebuild AbsenSI, ditemukan dari perbandingan dengan database aplikasi lama (`Projek/AbsenSI/06-Features/migrasi-database-lama.md`) — supaya data lama bisa dimigrasikan tanpa kehilangan informasi.

## Context
- **App:** `apps/api`
- **Ref:** `Projek/AbsenSI/06-Features/migrasi-database-lama.md` — baca bagian "Keputusan Final (2026-07-22)" untuk pertanggungjawaban tiap kolom yang ditambahkan

## Spec Detail

### 1. Tambah ke model `Teacher`
```prisma
model Teacher {
  // ... field existing tidak berubah ...
  gelarDepan          String?             @map("gelar_depan")
  gelarBelakang       String?             @map("gelar_belakang")
  tempatLahir         String?             @map("tempat_lahir")
  tanggalLahir        DateTime?           @map("tanggal_lahir")
  jenisKelamin        JenisKelamin?       @map("jenis_kelamin")  // enum sudah ada (dipakai Student)
  agama               Agama?                                     // enum sudah ada (dipakai Student)
  alamat              String?             @db.Text
  statusPernikahan    StatusPernikahan?   @map("status_pernikahan")
  statusKepegawaian   StatusKepegawaian   @default(guru) @map("status_kepegawaian")
}

enum StatusPernikahan {
  menikah
  belum_menikah
  pernah_menikah
}

enum StatusKepegawaian {
  guru
  karyawan
}
```
- **`no_wa` TIDAK ditambahkan sebagai field terpisah** — sesuai keputusan, digabung ke `noHp` yang sudah ada. Saat ETL (T062), data lama `pegawais.no_wa` dipetakan ke `teachers.no_hp`
- **`statusKepegawaian` default `guru`** — supaya data existing (guru yang sudah diinput manual sebelum migrasi ini) tidak perlu diisi ulang, dianggap guru kecuali migrasi data lama bilang lain

### 2. Tambah ke model `Student`
```prisma
model Student {
  // ... field existing tidak berubah ...
  alasanNonaktif AlasanNonaktif? @map("alasan_nonaktif")
  tahunLulus     Int?            @map("tahun_lulus")
}

enum AlasanNonaktif {
  lulus
  mengundurkan_diri
  lainnya
}
```
- **`alasanNonaktif` HANYA relevan kalau `status = nonaktif`** — nullable, tidak divalidasi ketat di level DB (validasi "harus diisi kalau nonaktif" ada di service layer kalau dibutuhkan, bukan scope task ini)
- **TIDAK mengubah enum `PersonStatus`** yang sudah dipakai luas di query existing — ini kolom BARU terpisah, bukan expand enum lama

### 3. Tambah ke model `Kampus`
Cek apakah sudah ada baris "Kampus 1" dari seed Fase 1 (`prisma/seed.ts`) — kalau sudah ada, task ini TIDAK perlu tambah kampus baru. Kalau belum ada, tambahkan ke seed:
```typescript
// Hanya kalau "Kampus 1" belum ada di seed existing
{ nama: "Kampus 1" }
```

## JANGAN
- ❌ JANGAN tambah field `noWa` terpisah di `Teacher` — sudah diputuskan digabung ke `noHp`, jangan buat 2 field untuk 1 konsep yang sama
- ❌ JANGAN ubah/expand enum `PersonStatus` untuk menampung status "lulus"/"mengundurkan diri" — WAJIB kolom baru terpisah `alasanNonaktif`, karena `PersonStatus` sudah dipakai di banyak query existing (alfa, dst) dan mengubahnya berisiko breaking change
- ❌ JANGAN buat kolom baru ini non-nullable — semua field yang ditambahkan task ini adalah data biodata opsional/historis, harus nullable supaya tidak memaksa isi ulang data existing yang sudah ada

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma`
- **Buat:** migration Prisma
- **Modifikasi (kalau perlu):** `apps/api/prisma/seed.ts` — pastikan "Kampus 1" ada

## Acceptance Criteria
- [ ] `prisma migrate dev` berjalan tanpa error
- [ ] Semua kolom baru nullable (kecuali `statusKepegawaian` yang punya default `guru`)
- [ ] Data `Teacher`/`Student` existing (dari seed Fase 1) tetap valid setelah migration — tidak ada data yang rusak/hilang
- [ ] `Kampus` dengan nama "Kampus 1" ada di database (baik dari seed lama atau ditambahkan task ini)

## Handoff ke T062
T062 (ETL migrasi data) akan mengisi kolom-kolom baru ini dari data lama — pastikan nama kolom Prisma di atas final sebelum T062 dimulai, perubahan nama kolom setelah T062 berarti mapping ETL harus disesuaikan ulang.

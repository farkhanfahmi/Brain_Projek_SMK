# T205 — API+Web: Tambah Field Lantai di Kelas + Perbaikan Menu Jurusan

## Depends on
Tidak ada dependency teknis ke rangkaian T203-T215 lain — **task ini bisa dikerjakan KAPAN SAJA, independen**, karena murni tambah field baru ke `Kelas` (sudah ada `ruangan` dari T169) tanpa menyentuh model jadwal yang sedang dibongkar. TAPI direkomendasikan dikerjakan LEBIH DULU dari T211 (View Read-Only Per Kelas) karena field `lantai` dipakai di situ.

## Objective
1. Tambah field `Kelas.lantai` (1/2/3) — melengkapi `Kelas.ruangan` (T169) supaya lokasi kelas lengkap (Kampus + Ruangan + Lantai).
2. Perbaiki menu Jurusan — user menyebut "perbaiki juga menu Jurusan dan Kelas" tanpa detail spesifik untuk Jurusan — **WAJIB KLARIFIKASI KE USER SAAT IMPLEMENTASI** apa yang dimaksud "perbaikan" untuk Jurusan (kemungkinan cuma dampak tidak langsung dari field Kelas baru, ATAU ada perbaikan terpisah yang belum dijelaskan detail — JANGAN ASUMSI, TANYAKAN).

## Spec Detail

### 1. Schema — tambah `Kelas.lantai`

```prisma
// Tambah ke model Kelas:
lantai Int? // 1/2/3 — opsional, KONSISTEN pola ruangan (T169) nullable, kelas lama
            // belum tentu diisi langsung
```
- Migration baru, nullable, tidak perlu backfill wajib.
- **PERTIMBANGKAN**: apakah `lantai` sebaiknya `Int` bebas (1,2,3,4,...) atau `enum` terbatas (Lantai1/2/3)? REKOMENDASI: `Int?` bebas (sekolah bisa punya gedung >3 lantai di masa depan, JANGAN batasi enum yang sulit diperluas) — VERIFIKASI dengan user kalau ragu.

### 2. Backend — DTO & form

- `CreateKelasDto`/`UpdateKelasDto` — tambah `lantai?: number` opsional (`@IsOptional() @IsInt()`).
- `KelasService` — tidak ada logic tambahan selain field pass-through.

### 3. Frontend — form Tambah/Edit Kelas + tabel

- `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx` — form Kelas tambah field "Lantai" (input number ATAU dropdown pilihan umum 1/2/3/Lainnya — PUTUSKAN saat implementasi mana yang lebih ergonomis, REKOMENDASI dropdown dengan opsi umum + input manual kalau perlu, KONSISTEN pola UX proyek untuk field terbatas tapi bisa diperluas).
- Tabel daftar Kelas — TAMBAH kolom "Lantai" (sortable, KONSISTEN aturan tabel wajib proyek), fallback "-" kalau NULL.

### 4. Menu Jurusan — klarifikasi WAJIB sebelum implementasi

- User menyebut "perbaiki juga menu Jurusan dan kelas" tanpa detail SPESIFIK untuk Jurusan (BEDA dari Kelas yang jelas: tambah lantai). **SEBELUM implementasi bagian ini, WAJIB tanya user langsung**: apakah maksudnya (a) tidak ada perbaikan terpisah untuk Jurusan, murni menyebut nama menu karena berdekatan dengan Kelas di 1 halaman (`kelas-jurusan-view.tsx` menangani KEDUANYA dalam 1 tampilan), (b) ada field/fitur spesifik yang kurang di Jurusan yang belum dijelaskan detail, atau (c) lainnya — JANGAN menebak dan mengerjakan sesuatu yang tidak diminta.

## Edge Cases
- Field `lantai` NULL untuk kelas lama — tampil "-" di tabel dan View Read-Only (T211), TIDAK error.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`Kelas` +field), `CreateKelasDto`/`UpdateKelasDto`, `apps/web/.../kelas/kelas-jurusan-view.tsx` (form + kolom tabel).
- **Buat:** migration Prisma baru.

## Acceptance Criteria
- [x] Form Tambah/Edit Kelas punya field Lantai opsional.
- [x] Tabel Kelas tampilkan kolom Lantai, sortable, fallback "-" untuk NULL.
- [x] **Klarifikasi menu Jurusan SUDAH dilakukan** dengan user SEBELUM ada perubahan kode untuk Jurusan — hasil: TIDAK ADA perbaikan terpisah, murni penyebutan menu berdekatan di 1 halaman.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] **Tanya user dulu** soal maksud "perbaikan menu Jurusan" — dijawab via AskUserQuestion SEBELUM implementasi apa pun: opsi "Tidak ada perbaikan terpisah" dipilih. TIDAK ADA kode Jurusan yang diubah di task ini.
- [x] Konfirmasi field `lantai` tidak breaking untuk data Kelas existing — nullable, migration `ALTER TABLE kelas ADD COLUMN lantai INTEGER NULL` (single line, tanpa default wajib).

## Status Eksekusi (2026-08-16)

**Selesai.**

### Klarifikasi menu Jurusan (dilakukan SEBELUM kode apa pun)

Ditanya via AskUserQuestion: user pilih **"Tidak ada perbaikan terpisah"** — konfirmasi bahwa "perbaiki juga menu Jurusan" murni menyebut nama menu karena Kelas & Jurusan ditangani di 1 halaman yang sama (`kelas-jurusan-view.tsx`). Tidak ada perubahan kode untuk Jurusan di task ini.

### Catatan koordinasi paralel

T204 (JadwalSlot/MapelGuru) sedang dikerjakan session lain BERSAMAAN dengan task ini — dicek `prisma migrate status` sebelum mulai, migration T204 (`20260816171505_t204_jadwal_slot_mapel_guru`) SUDAH applied duluan, jadi migration T205 yang di-generate HANYA berisi perubahan task ini (`ALTER TABLE kelas ADD COLUMN lantai`), tidak tercampur. Field `lantai` ditambahkan bersebelahan dengan `ruangan` di schema, TIDAK menyentuh baris `jadwalSlots JadwalSlot[]` yang sudah ditambahkan T204 di model `Kelas` yang sama.

### Implementasi

- Schema: `Kelas.lantai Int?` — `Int` bebas (bukan enum), sesuai rekomendasi spec, supaya tidak perlu migration lagi kalau sekolah punya gedung >3 lantai.
- `CreateKelasDto` — tambah `lantai?: number` (`@IsOptional() @IsInt()`), `UpdateKelasDto` otomatis ikut via `PartialType`. `KelasService.create()`/`update()` tidak diubah (pure pass-through `data: dto`, sudah generik).
- `kelas-jurusan-view.tsx` — form tambah input number "Lantai (opsional)" (BUKAN dropdown+manual — diputuskan input number polos konsisten kesederhanaan field `ruangan` di sampingnya, tidak menambah pola UI baru untuk 1 field kecil). Tabel tambah kolom Lantai sortable (numeric compare, bukan `localeCompare`, KONSISTEN pola `jumlahSiswa`), fallback "-" untuk NULL, `colSpan` empty-state disesuaikan 9→10.
- `core-types.ts` — `Kelas.lantai: number | null`.

### Verifikasi

- `tsc --noEmit` bersih di `apps/api` dan `apps/web`.
- `jest apps/api`: 494/494 pass, 29/29 suite (termasuk kerja T204 paralel, tidak ada regresi silang).
- `git diff schema.prisma`: 1 baris ditambahkan (`lantai`), tidak menyentuh baris T204 yang sudah ada.

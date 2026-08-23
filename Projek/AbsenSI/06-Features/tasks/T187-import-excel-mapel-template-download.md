# T187 — API+Web: Import Excel Mapel + Download Template

## Depends on
**WAJIB setelah T186** (skema `Mapel.jurusanId` harus ada dulu supaya template Excel bisa mengikutsertakan kolom Jurusan).

## Objective
Upgrade Import Mapel (SAAT INI sudah ada via CSV, T174) jadi **Import Excel** dengan **template file Excel yang bisa didownload** admin sebelum mengisi — mempermudah admin yang lebih familiar Excel daripada CSV mentah, dan mengurangi kesalahan format karena template sudah menyediakan struktur kolom yang benar.

## Context — Temuan Riset (2026-08-15)

`ImportService.importMapel()` (`apps/api/src/import/import.service.ts:455`) SUDAH ADA dari T174 — kolom `nama, kode` (opsional). User SEKARANG minta **versi Excel** (bukan cuma CSV) DENGAN **template downloadable**. Ini BUKAN duplikasi — CSV bisa TETAP ada sebagai opsi (kalau infrastruktur `ImportDialog`/`ImportService` support keduanya), tapi WAJIB ada jalur Excel dengan template.

**Setelah T186**: kolom Excel bertambah 1 — `jurusan` (opsional, isi nama jurusan persis kalau mapel khusus, kosongkan untuk mapel umum).

## Spec Detail

### 1. Backend — dukung parsing file Excel (.xlsx), bukan cuma CSV

- `ImportService` SAAT INI pakai `csv-parse/sync` — VERIFIKASI apakah proyek SUDAH punya library baca Excel (`exceljs` SUDAH terpasang untuk export rekap, T163 — REUSE library YANG SAMA untuk BACA .xlsx, JANGAN install library baru kalau `exceljs` sudah bisa read).
- `importMapel()` — PERLUAS terima file `.xlsx` (SELAIN `.csv` yang sudah ada, ATAU GANTI total ke .xlsx sesuai keputusan implementasi — PUTUSKAN: REKOMENDASI dukung KEDUANYA berdasarkan ekstensi file, supaya admin yang sudah terbiasa CSV tidak kehilangan opsi, TAPI Excel jadi opsi UTAMA yang direkomendasikan UI). Tambah kolom `jurusan` (opsional, lookup by nama persis seperti pola kelas/jurusan di `importStudents`, TOLAK dengan pesan jelas kalau nama jurusan tidak ditemukan).
- Endpoint BARU **download template**: `GET /import/mapel/template` — generate file `.xlsx` KOSONG (cuma header kolom: `nama, kode, jurusan`) + 1-2 baris CONTOH data (memudahkan admin paham format) menggunakan `exceljs`, return sebagai file download (`Content-Disposition: attachment`).

### 2. Frontend — tombol download template + upload Excel

- `apps/web/src/components/import-dialog.tsx` (`ImportDialog`, dipakai di banyak tempat) — TAMBAH tombol **"Download Template"** di dalam dialog (link ke endpoint template, atau fetch+trigger download) — REUSE komponen `ImportDialog` yang SAMA, TAMBAH prop opsional `templateEndpoint` (supaya modul LAIN yang pakai `ImportDialog` juga bisa dapat fitur ini kalau relevan — LIHAT T192/T193/T194 lain di rangkaian task ini yang JUGA minta fitur template download, REKOMENDASI KUAT buat fitur ini di level komponen `ImportDialog` SEKALI supaya semua modul import lain otomatis dapat manfaatnya, bukan diulang-ulang per modul).
- `mapel-view.tsx` — pasang `ImportDialog` dengan `templateEndpoint="/import/mapel/template"`.

## Edge Cases
- File Excel dengan sheet lebih dari 1 / format tidak sesuai — parse HANYA sheet pertama (atau sheet bernama tertentu kalau template didefinisikan begitu), TOLAK dengan pesan jelas kalau header kolom tidak cocok (KONSISTEN error handling `parseCsv()` yang sudah ada).
- Kolom `jurusan` diisi nama yang salah ketik/tidak ada — TOLAK per baris dengan pesan jelas (KONSISTEN pola `importStudents` untuk kelas/jurusan tidak ditemukan).

## Files
- **Modifikasi:** `apps/api/src/import/import.service.ts` (`importMapel()` dukung .xlsx + kolom jurusan), `apps/api/src/import/import.controller.ts` (endpoint download template baru), `apps/web/src/components/import-dialog.tsx` (prop `templateEndpoint` + tombol download, REUSABLE untuk modul lain), `apps/web/src/app/(admin)/mapel/mapel-view.tsx` (pasang templateEndpoint).
- **Jangan sentuh:** `importStudents`/`importTeachers`/`importCards`/method import lain (TIDAK diubah, task ini fokus Mapel + infrastruktur `ImportDialog` reusable).

## Acceptance Criteria
- [x] Tombol "Download Template" di dialog import Mapel menghasilkan file `.xlsx` dengan header benar + contoh data.
- [x] Upload file `.xlsx` terisi berhasil import Mapel (termasuk kolom jurusan, lookup by nama, tolak kalau tidak ditemukan).
- [x] CSV lama TETAP berfungsi (regresi nol) — dibedakan dari ekstensi `filename`.
- [x] Prop `templateEndpoint` di `ImportDialog` REUSABLE (opsional, tidak ada logic khusus Mapel tertanam).
- [x] Build + type-check hijau, jest baru untuk parsing .xlsx + endpoint template.

## Validasi Claudian
- [x] Konfirmasi REUSE `exceljs` yang sudah terpasang (T163), TIDAK install library Excel baru.
- [x] Konfirmasi `ImportDialog` prop `templateEndpoint` didesain REUSABLE (bukan logic khusus Mapel yang tertanam) — ini akan dipakai lagi di T192/T193/T194.

## Status Eksekusi (2026-08-16)

**Selesai.**

### Backend
- `ImportService.parseRows<T>(buffer, filename)` — helper baru, generic: `.xlsx` → `ExcelJS.Workbook().xlsx.load()` (sheet pertama, baris 1 = header), selain itu fallback ke `parseCsv()` lama. Header kosong/sheet tidak ada → `BadRequestException` pesan jelas.
- `importMapel(buffer, filename)` — signature berubah (tambah `filename`), pakai `parseRows` bukan `parseCsv` langsung. Kolom `jurusan` opsional: lookup by nama persis (pola sama `importKelas`), tidak ketemu → error jelas per baris, kosong → `jurusanId: undefined` (mapel umum).
- `generateMapelTemplate()` — `ExcelJS.Workbook` baru, sheet "Mapel", header `nama/kode/jurusan` bold + 2 baris contoh (1 umum, 1 khusus jurusan TKR).
- `GET /import/mapel/template` — endpoint baru, `Content-Disposition: attachment`, guard sama seperti `POST /mapel` (class-level `@Roles(super_admin)`).
- 8 unit test baru/diperbarui (CSV lama tetap pass, xlsx parsing, jurusan lookup ketemu/tidak ketemu, template generation) — total 441/441 pass di seluruh suite.

### Frontend
- `ImportDialog` — prop `templateEndpoint?: string` opsional. Kalau diisi: tombol "Download Template" muncul (fetch via `/api/proxy-download` yang sudah ada, generic, tidak perlu modifikasi), file input accept diperluas ke `.xlsx`+`.csv`. Kalau tidak diisi (semua pemanggil lain: siswa/guru/kartu/kelas/jurusan/jadwal/users): perilaku identik seperti sebelumnya, regresi nol.
- `mapel-view.tsx` — pasang `templateEndpoint="/import/mapel/template"`, update kolom+contoh CSV di dialog untuk mencakup `jurusan`.

### Verifikasi
- `tsc --noEmit` bersih di `apps/api` dan `apps/web`.
- `jest apps/api` full suite: 441/441 pass.
- Live-verify browser: belum dilakukan di sesi ini (menyusul, konsisten pola T186 — verifikasi manual diserahkan ke user).

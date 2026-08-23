# T197 — Web: Hapus Dropdown Filter Tahun Ajaran di Section Semester (Selalu Ikut Tahun Ajaran Aktif)

## Depends on
Tidak ada dependency teknis. Independen.

## Objective
`semesters-section.tsx` (dipakai di `(admin)/kalender/` DAN `(admin-jurnal)/admin-jurnal/kalender/` dari T188) — HAPUS dropdown filter "Semua Tahun Ajaran / [pilih tahun ajaran lain]" yang SAAT INI ADA — list Semester SELALU otomatis mengikuti Tahun Ajaran yang **AKTIF SAJA**, tanpa opsi melihat semester dari tahun ajaran lain.

## Context — Temuan Testing User (2026-08-16)

`semesters-section.tsx` (baris ~61, 103-115) punya `<Select filterAcademicYearId>` "Semua Tahun Ajaran" yang memfilter TAMPILAN list semester berdasarkan tahun ajaran manapun (bukan hanya yang aktif) — ini murni filter tampilan (constraint aktivasi semester TETAP GLOBAL, tidak terpengaruh dropdown ini).

User eksplisit: "pada menu semester jangan ada dropdown memilih semester. semester selalu mengikuti tahun ajaran yang di aktifkan" — maksudnya dropdown filter Tahun Ajaran ini DIHAPUS, list Semester SELALU tampilkan yang terkait Tahun Ajaran AKTIF saja.

## Spec Detail

### 1. Frontend — hapus dropdown, filter otomatis ke Tahun Ajaran aktif

- `semesters-section.tsx` — HAPUS `<Select filterAcademicYearId>` SEPENUHNYA dari UI.
- List Semester yang ditampilkan — GANTI logic filter dari "berdasarkan dropdown yang dipilih user" jadi **HARDCODE filter ke `academicYearId` dari Tahun Ajaran yang `isActive: true`** (ambil dari data yang SUDAH di-fetch di `academic-years-section.tsx`/parent component, REUSE, JANGAN fetch ulang).
- **Edge case KRITIKAL**: TIDAK ADA Tahun Ajaran aktif sama sekali (kondisi awal sistem/transisi) — tampilkan pesan jelas ("Belum ada Tahun Ajaran aktif — aktifkan salah satu di section Tahun Ajaran di atas") BUKAN list kosong tanpa penjelasan.

### 2. Backend — TIDAK ADA perubahan wajib

- Endpoint `GET /semesters` SUDAH BISA difilter `academicYearId` via query param (dipakai dropdown lama) — CUKUP frontend yang HARDCODE nilai ini ke tahun ajaran aktif, TIDAK PERLU perubahan endpoint.

## Edge Cases
- Create Semester BARU — form create TETAP punya dropdown pilih Tahun Ajaran (BEDA dari filter TAMPILAN yang dihapus ini — form create WAJIB tetap bisa pilih tahun ajaran MANA pun, termasuk yang belum aktif, untuk PERSIAPAN semester tahun depan sebelum diaktifkan) — JANGAN HAPUS dropdown INI, HANYA dropdown FILTER TAMPILAN yang dihapus.
- Semester dari Tahun Ajaran LAMA (bukan aktif) yang MASIH relevan dilihat admin (misal untuk audit) — TIDAK BISA dilihat lagi dari halaman ini setelah task ini (KONSEKUENSI YANG DISADARI user, sesuai permintaan eksplisit) — data TETAP ADA di database, HANYA tidak ditampilkan di UI ini.

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kalender/semesters-section.tsx` (hapus dropdown filter, hardcode ke tahun ajaran aktif).
- **Jangan sentuh:** form create Semester (dropdown pilih Tahun Ajaran DI SITU tetap ada, beda konteks), `academic-years-section.tsx` (tidak disentuh), backend `SemestersController`/`Service` (tidak ada perubahan wajib).

## Acceptance Criteria
- [x] Dropdown filter Tahun Ajaran di section Semester TIDAK ADA LAGI.
- [x] List Semester otomatis tampilkan HANYA yang terkait Tahun Ajaran aktif.
- [x] Tidak ada Tahun Ajaran aktif — pesan jelas, bukan list kosong tanpa penjelasan.
- [x] Form CREATE Semester TETAP punya dropdown pilih Tahun Ajaran (regresi nol, beda konteks dari filter yang dihapus).
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi dropdown yang DIHAPUS adalah filter TAMPILAN (`filterAcademicYearId` state + `<Select>` baris 103-115 lama), BUKAN dropdown pilih Tahun Ajaran di `SemesterForm` (baris 253-269, TIDAK disentuh) — 2 hal berbeda, tidak tertukar.

## Catatan Implementasi (2026-08-16)

- Perubahan murni di `semesters-section.tsx`: hapus state `filterAcademicYearId` + konstanta `ALL` + `<Select>` filter, ganti `filtered` jadi `activeAcademicYear = academicYears.find(ay => ay.isActive)` lalu filter semesters by `academicYearId` itu.
- Empty state dipecah 2 kondisi: (1) tidak ada tahun ajaran aktif sama sekali → pesan arahkan ke section Tahun Ajaran, (2) ada tahun ajaran aktif tapi belum ada semester untuk itu → pesan beda.
- 1 komponen shared (`SemestersSection`) dipakai `(admin)/kalender/kalender-view.tsx` — tidak ada duplikat terpisah untuk admin-jurnal (T188 reuse `KalenderView` yang sama), jadi perubahan otomatis berlaku di kedua tempat.
- Tidak ada perubahan backend (endpoint `GET /semesters?academicYearId=` sudah cukup, cuma dipakai dengan nilai hardcode sekarang).
- Verifikasi: `tsc --noEmit` bersih, `next build` sukses.

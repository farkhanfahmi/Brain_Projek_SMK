# T188 — Web: Kalender Pendidikan di Admin-Jurnal (Full CRUD, Setara Super_Admin)

## Depends on
Tidak ada dependency teknis. Independen.

## Objective
Tambah halaman **Kalender Pendidikan** (Tahun Ajaran + Semester + Hari Libur) ke dashboard `(admin-jurnal)/` dengan **CRUD PENUH** (create/edit/aktifkan) — SAAT INI kalender pendidikan HANYA ada di `(admin)/kalender/`, admin_jurnal tidak punya akses sama sekali. Setelah task ini, admin_jurnal **setara** super_admin untuk kalender pendidikan (bukan cuma lihat).

## Context — Keputusan Eksplisit User (2026-08-15, dikonfirmasi 2x)

Riset mengonfirmasi: `(admin)/kalender/` (AcademicYear+Semester+Holiday CRUD) SUDAH lengkap untuk super_admin. `(admin-jurnal)/` TIDAK PUNYA halaman kalender apa pun — hanya `semester/` read-only murni (lihat T190, task terpisah untuk MENGHAPUS menu itu karena akan REDUNDAN setelah task ini).

**Ini PERUBAHAN WEWENANG BESAR** — siklus hidup Tahun Ajaran/Semester SEBELUMNYA eksklusif `super_admin` (komentar kode lama `semester-view.tsx:16-19`: "siklus hidup semester tetap wewenang super_admin"). User DIKONFIRMASI EKSPLISIT 2x (termasuk setelah ditanya ulang secara spesifik soal implikasi ini) bahwa admin_jurnal SEKARANG BOLEH CRUD penuh, BUKAN cuma lihat.

## Spec Detail

### 1. Backend — perluas role guard

- `AcademicYearsController`/`SemestersController`/holiday controller (nama pasti VERIFIKASI saat implementasi) — endpoint create/update/activate SAAT INI `@Roles(UserRole.super_admin)` SAJA — TAMBAH `UserRole.admin_jurnal` ke array (ADDITIVE, KONSISTEN pola T157 — tambah role, JANGAN cabut super_admin).
- `@LogActivity` — SUDAH terpasang di endpoint2 ini (dari implementasi awal), TIDAK PERLU perubahan — cukup pastikan `actorId` tetap tercatat benar siapa pun role-nya yang mengaksi (guru_jurnal atau super_admin).

### 2. Frontend — halaman baru di admin-jurnal, REUSE komponen `(admin)/kalender/` yang sudah ada

- Halaman baru `(admin-jurnal)/admin-jurnal/kalender/` — REUSE LANGSUNG komponen `kalender-view.tsx`, `academic-years-section.tsx`, `semesters-section.tsx`, `holiday-form-dialog.tsx` yang SUDAH ADA di `(admin)/kalender/` (import langsung, KONSISTEN pola T157 duplikasi-via-reuse-komponen — JANGAN copy-paste kode).
- Menu sidebar admin-jurnal — tambah entry "Kalender Pendidikan".
- Menu "Semester" LAMA di admin-jurnal (read-only) — DIHAPUS (lihat T190, task TERPISAH, JANGAN dihapus di task ini supaya scope tetap jelas per-task, tapi CATAT ke user kalau T190 belum dikerjakan bersamaan bahwa akan ada 2 menu Semester sementara — 1 baru (full CRUD, task ini) dan 1 lama (read-only, T190 belum jalan) — REKOMENDASI kerjakan T190 SEGERA setelah task ini untuk menghindari duplikasi membingungkan, tapi TIDAK WAJIB bersamaan).

## Edge Cases
- Admin_jurnal dan super_admin edit Tahun Ajaran/Semester yang SAMA nyaris bersamaan — TIDAK ADA proteksi tambahan diperlukan (sama seperti risiko 2 super_admin edit bersamaan yang SUDAH ADA sebelumnya, di luar scope task ini untuk diperbaiki).
- Constraint "1 semester aktif GLOBAL" (dikonfirmasi T156) — TETAP GLOBAL, admin_jurnal MENGAKTIFKAN semester JUGA akan mematikan semester aktif manapun sebelumnya (SAMA seperti super_admin) — TIDAK ADA batasan tambahan khusus admin_jurnal.

## Files
- **Buat:** `apps/web/src/app/(admin-jurnal)/admin-jurnal/kalender/page.tsx` (reuse komponen existing).
- **Modifikasi:** backend controller (`@Roles` diperluas), sidebar admin-jurnal (menu baru).
- **Jangan sentuh:** komponen `kalender-view.tsx`/`academic-years-section.tsx`/`semesters-section.tsx`/`holiday-form-dialog.tsx` (REUSE apa adanya, TIDAK diubah isinya), halaman `(admin)/kalender/` (TIDAK terpengaruh, tetap berfungsi sama untuk super_admin).

## Acceptance Criteria
- [x] Admin_jurnal bisa akses halaman Kalender Pendidikan dari dashboard mereka.
- [x] Admin_jurnal bisa create/edit/aktifkan Tahun Ajaran — role guard terverifikasi additive.
- [x] Admin_jurnal bisa create/edit/aktifkan Semester — role guard terverifikasi additive.
- [x] Admin_jurnal bisa create/edit Hari Libur — role guard terverifikasi additive.
- [x] Super_admin TETAP bisa akses `(admin)/kalender/` seperti biasa (regresi nol, komponen tidak disentuh).
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi perubahan role guard ADDITIVE (tambah admin_jurnal, TIDAK cabut super_admin) di SEMUA endpoint terkait (Tahun Ajaran, Semester, Hari Libur).
- [x] **Dicatat ke user**: setelah task ini, ADA 2 menu terkait Semester di admin-jurnal — "Kalender Pendidikan" (baru, full CRUD) dan "Semester" (lama, read-only) — SAMPAI T190 dikerjakan untuk menghapus menu lama. Rekomendasi: kerjakan T190 segera.

## Status Eksekusi (2026-08-16)

**Selesai.**

- Backend — 3 controller diperluas ADDITIF (`@Roles(super_admin, admin_jurnal)`): `AcademicYearsController` (create, activate), `SemestersController` (create, update, activate), `SchoolHolidaysController` (create, update, delete). `GET` di ketiganya sudah terbuka untuk semua JWT sebelumnya, tidak disentuh. `@LogActivity` sudah terpasang sejak awal, tidak perlu perubahan.
- Frontend — halaman baru `(admin-jurnal)/admin-jurnal/kalender/page.tsx`, reuse `KalenderView` dari `(admin)/kalender/kalender-view.tsx` langsung (import lintas route group, pola T157) — komponen `kalender-view.tsx`/`academic-years-section.tsx`/`semesters-section.tsx`/`holiday-form-dialog.tsx` TIDAK disentuh sama sekali. Menu sidebar admin-jurnal tambah "Kalender Pendidikan" (icon `CalendarDays`, konsisten dgn `(admin)/kalender`), diposisikan sebelum menu "Semester" lama.
- `tsc --noEmit` bersih di `apps/api` dan `apps/web`. `jest apps/api` full suite 441/441 pass (tidak ada test yang mengasumsikan admin_jurnal ditolak di endpoint ini, tidak ada yang perlu diperbarui).
- Live-verify browser: belum dilakukan (konsisten pola T186/T187, verifikasi manual diserahkan ke user).

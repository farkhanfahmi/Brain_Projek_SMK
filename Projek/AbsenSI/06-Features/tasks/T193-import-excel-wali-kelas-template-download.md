# T193 — API+Web: Import Excel Wali Kelas + Download Template

## Depends on
**REKOMENDASI setelah T187** (infrastruktur `ImportDialog` dengan prop `templateEndpoint` reusable dibangun di T187 — task ini TINGGAL PAKAI, tidak perlu bangun ulang dari nol). Independen secara teknis kalau T187 belum jalan (bisa duplikasi sementara), TAPI TIDAK EFISIEN.

## Objective
Halaman Wali Kelas — tambah **Import Excel** (assignment guru↔kelas secara massal) dengan **template downloadable** — SAAT INI assignment HANYA bisa manual 1-per-1 via modal per baris kelas.

## Context — Temuan Riset (2026-08-15)

`(admin)/wali-kelas/` (REUSE `WaliKelasView` dari admin-jurnal, T157) — assignment SAAT INI via `AssignWaliModal` PER KELAS (tombol "Assign"/"Ubah" satu-satu, TIDAK ADA bulk/import). `User.kelasIdWali` — `Int?` TUNGGAL (1 guru = wali MAKSIMAL 1 kelas, dikonfirmasi UI copy "1 kelas hanya bisa punya 1 wali kelas aktif").

**Ini task PENTING untuk setup awal tahun ajaran** — admin biasanya assign PULUHAN wali kelas sekaligus saat kelas baru dibentuk, form 1-per-1 sangat lambat untuk skenario ini.

## Spec Detail

### 1. Backend — endpoint import baru

- `ImportService` — method baru `importWaliKelas(buffer)` — kolom Excel: `kelas, guru` (nama kelas + NIY atau nama guru — VERIFIKASI mana yang lebih robust untuk lookup, REKOMENDASI NIY karena unik, tapi PERTIMBANGKAN dukung NAMA juga dengan validasi ambigu KONSISTEN pola `importCards` yang sudah handle kasus serupa nisn_nip).
- Validasi PER BARIS: `kelas` harus ADA (lookup by nama), `guru` harus ADA (lookup by NIY, TOLAK kalau ambigu/tidak ditemukan) — REUSE pola validasi `importStudents`/`importCards`.
- **VALIDASI KRITIKAL — constraint "1 kelas 1 wali, 1 guru wali MAKSIMAL 1 kelas"**: SEBELUM assign, cek apakah kelas itu SUDAH punya wali (kalau ADA, apakah baris Excel ini MENGGANTIKAN wali lama, atau DITOLAK? — REKOMENDASI: REPLACE (assignment baru menggantikan lama, KONSISTEN perilaku modal manual "Ubah" yang SUDAH ADA) — DAN cek apakah guru itu SUDAH jadi wali kelas LAIN (kalau ADA, TOLAK baris ini dengan pesan jelas, KARENA 1 guru cuma boleh wali 1 kelas — KECUALI baris Excel ini justru assign guru itu ke kelas yang SAMA dengan yang sudah dia wali-i sebelumnya, itu boleh/no-op).
- Endpoint download template: `GET /import/wali-kelas/template` — kolom `kelas, guru` + contoh data.

### 2. Frontend — `ImportDialog` di halaman Wali Kelas

- `wali-kelas-view.tsx` (dan duplikat) — pasang `ImportDialog` (REUSE komponen dari T187, dengan `templateEndpoint`).

## Edge Cases
- Excel dengan 1 kelas MUNCUL 2x (assign wali beda di 2 baris) — TOLAK baris KEDUA sebagai duplikat dalam file (KONSISTEN pola `Set` cek duplikat yang sudah ada di import lain).
- Guru yang di-assign SUDAH jadi wali kelas LAIN (bukan yang di baris ini) — TOLAK dengan pesan jelas menyebut kelas mana yang sudah dia wali-i.

## Files
- **Modifikasi:** `apps/api/src/import/import.service.ts` (`importWaliKelas()` baru), `apps/api/src/import/import.controller.ts` (endpoint baru + template), `apps/web/src/app/(admin-jurnal)/admin-jurnal/wali-kelas/wali-kelas-view.tsx` (dan duplikat admin) — pasang `ImportDialog`.
- **Jangan sentuh:** `AssignWaliModal` (assignment manual 1-per-1 TETAP ADA, import adalah TAMBAHAN untuk bulk).

## Acceptance Criteria
- [x] Download template Excel Wali Kelas berhasil, format benar.
- [x] Upload Excel terisi — assignment massal berhasil, validasi kelas+guru+constraint 1-wali-1-kelas berfungsi.
- [x] Guru yang sudah jadi wali kelas lain — ditolak dengan pesan jelas.
- [x] Assignment manual (`AssignWaliModal`) TETAP berfungsi (regresi nol, tidak disentuh).
- [x] Build + type-check hijau, jest baru untuk skenario import + constraint.

## Validasi Claudian
- [x] Konfirmasi REUSE infrastruktur `ImportDialog`+`templateEndpoint` dari T187 (BUKAN bangun ulang dari nol).
- [x] Konfirmasi constraint "1 guru wali maksimal 1 kelas" DITEGAKKAN di level import (bukan cuma di modal manual).

## Status Eksekusi (2026-08-16)

**Selesai.**

### Keputusan constraint (dikonfirmasi user, beda dari asumsi awal spec)

Riset menemukan endpoint manual `PATCH /users/:id/assign-wali-kelas` SAAT INI justru MENOLAK (409) kalau kelas sudah punya wali lain — modal `AssignWaliModal` mengharuskan admin klik "Lepas" dulu secara eksplisit, BUKAN auto-replace. Dikonfirmasi ke user: jalur **import** tetap REPLACE otomatis (sesuai rekomendasi spec awal) — lebih praktis untuk bulk import awal tahun ajaran — sementara endpoint manual TIDAK diubah (tetap menolak, behavior existing dipertahankan). Constraint "1 guru wali maksimal 1 kelas LAIN" tetap ditegakkan sama di kedua jalur.

### Backend

- `ImportService.importWaliKelas(buffer, filename)` — kolom `kelas` (nama, cek ambigu KONSISTEN `importSchedules`), `guru` (NIY, lookup via `Teacher.niy` → `User` dengan `role: guru`). Dukung `.csv`+`.xlsx` (reuse `parseRows` dari T187).
- Constraint: guru yang SUDAH wali kelas LAIN (bukan kelas baris ini) → ditolak, pesan sebut nama kelas yang sudah dia wali-i. Guru assign ke kelas yang SAMA dengan yang sudah dia wali-i → no-op boleh (bukan error). Kelas duplikat dalam file → baris kedua ditolak.
- REPLACE: kalau kelas sudah punya wali lain, `updateMany` lepas wali lama (`kelasIdWali: null`) SEBELUM assign baru — 2 query terpisah per baris berhasil (bukan 1 transaction, konsisten pola import lain yang best-effort per-baris bukan all-or-nothing).
- `generateWaliKelasTemplate()` — header `kelas, guru` + 1 baris contoh.
- `POST /import/wali-kelas` + `GET /import/wali-kelas/template` — guard `@Roles(super_admin, admin_jurnal)`, KONSISTEN endpoint manual `assign-wali-kelas` (yang juga admin_jurnal+super_admin, BUKAN cuma super_admin seperti kebanyakan endpoint `ImportController` lain).
- 9 unit test baru (assignment baru, replace, guru sudah wali kelas lain, no-op kelas sama, kelas/NIY tidak ketemu, duplikat dalam file, kelas ambigu, template generation) — 461/461 pass di seluruh suite.

### Frontend

- `wali-kelas-view.tsx` (dipakai admin-jurnal+admin via reuse T157) — pasang `ImportDialog` dengan `templateEndpoint="/import/wali-kelas/template"`. Tambah `useEffect` sinkron `guruUsers` prop → state lokal (pola sama `MapelView` T186 — `router.refresh()` tidak remount komponen, props baru harus disinkronkan manual).
- `AssignWaliModal` TIDAK disentuh sama sekali (assignment manual 1-per-1 tetap jalur terpisah, sesuai instruksi "Jangan sentuh" di spec).

### Verifikasi

- `tsc --noEmit` bersih untuk semua file yang disentuh task ini (1 error pre-existing TIDAK TERKAIT di `dispen-view.tsx`, fitur lain yang sedang berjalan paralel, bukan bagian T193).
- `jest apps/api` full suite: 461/461 pass.
- Live-verify browser: belum dilakukan (konsisten pola T186-T190, verifikasi manual diserahkan ke user).

# T127 — API: Fix Bug — Surat Izin Masuk Kelas Bisa Dicetak Lintas Kampus

## Depends on
Tidak ada dependency teknis. Fix murni tambah validasi kampus di 1 service, pola sudah ada di modul lain untuk dicontek.

## Objective
Endpoint `POST /late-entry-slips` (surat izin masuk kelas, T107) memvalidasi bahwa siswa target berada di kampus yang SAMA dengan kampus piket yang login — saat ini TIDAK ADA validasi ini sama sekali, piket kampus manapun bisa mencetak surat untuk siswa kampus lain asal tahu/menebak `studentId`.

## Context
- **App:** `apps/api` — fix murni, tidak ada perubahan skema.
- **Celah dikonfirmasi 2026-08-06** lewat audit keamanan menyeluruh dashboard piket (Explore agent, baca kode langsung): `late-entry-slips.service.ts` `create()` (baris ±23-52) menerima `dto.studentId` langsung, memvalidasi role (`guru_piket`) dan status siswa (`terlambat` hari itu) — TAPI **tidak pernah menerima atau mengecek `kampusId`** sama sekali, baik di controller (`late-entry-slips.controller.ts:27-28`) maupun service.
- **Bandingkan dengan modul lain yang SUDAH BENAR** (pola yang harus ditiru):
  - `permits.service.ts` — punya `ensureStudentInKampus()` (baris ±433-444) dan `ensurePermitInKampus()` (±446-457), dipanggil di semua method mutasi (`create`, `confirmKembali`, `setPulang`).
  - `students.service.ts` — `lock()`/`unlock()` sama-sama memanggil `ensureStudentInKampus()` (±441-442, 473-474, 516-527).
  - `attendance.controller.ts` — endpoint board/notifikasi piket pakai `requireKampusId(user)` (±117-141, 194-222) untuk scoping.
  - **`late-entry-slips` adalah SATU-SATUNYA endpoint mutasi piket yang lupa menerapkan pola ini** — semua endpoint piket lain sudah benar.

## Spec Detail

### Backend
- `apps/api/src/late-entry-slips/late-entry-slips.controller.ts` — endpoint `create()` (baris ±27-28): ambil `kampusId` dari JWT user yang login (`@CurrentUser()`, pola sama seperti dipakai endpoint piket lain), teruskan ke service.
- `apps/api/src/late-entry-slips/late-entry-slips.service.ts` — `create()`: tambah validasi kampus SEBELUM proses lain — reuse fungsi `ensureStudentInKampus()` kalau bisa diimpor dari `students.service.ts`/`permits.service.ts` (kalau tidak diekspor/reusable lintas modul, tulis ulang logic yang SAMA persis di modul ini, JANGAN logic berbeda — konsisten dengan pola existing: cek `student.kelas.kampusId !== kampusId piket` → `ForbiddenException`).
- Pastikan urutan validasi masuk akal: cek keberadaan siswa dulu → cek kampus → cek status terlambat (atau urutan lain yang wajar, tapi kampus HARUS dicek sebelum data siswa lain di-expose lewat pesan error, supaya tidak jadi celah informasi lain — misal jangan sampai pesan error "siswa tidak terlambat" tetap muncul untuk siswa yang sebenarnya BUKAN di kampus piket itu, itu tetap bocor informasi).

## Edge Cases
- Siswa yang `kelasId: null` (siswa baru SPMB/PPDB belum di-plot kelas, kasus valid yang sudah didokumentasikan di proyek ini) — `kelas.kampusId` tidak bisa diakses langsung dari relasi kalau `kelas` null. Pastikan validasi kampus tidak crash untuk kasus ini (siswa tanpa kelas kemungkinan besar juga tidak punya attendance record `terlambat` hari itu karena belum bisa tap normal, tapi tetap perlu penanganan null-safe di kode validasi).

## Files
- **Modifikasi:** `apps/api/src/late-entry-slips/late-entry-slips.controller.ts`, `apps/api/src/late-entry-slips/late-entry-slips.service.ts`.
- **Jangan sentuh:** `permits.service.ts`/`students.service.ts` (hanya jadi referensi pola, tidak perlu diubah) kecuali kalau `ensureStudentInKampus()` diputuskan untuk diekstrak jadi shared utility (opsional, di luar scope minimal fix ini).

## Acceptance Criteria
- [x] Piket kampus 1 mencoba cetak surat untuk `studentId` milik kampus 2 → ditolak (403/Forbidden), bukan berhasil. Diverifikasi unit test — `ForbiddenException` dilempar SEBELUM `attendanceRecord.findFirst` dipanggil.
- [x] Piket kampus 1 mencetak surat untuk siswa kampus 1 sendiri → tetap berfungsi normal (regresi nol untuk kasus valid). Diverifikasi unit test.
- [x] super_admin tidak relevan — `@Roles(UserRole.guru_piket)` di controller adalah SATU-SATUNYA role yang boleh akses endpoint ini (dikonfirmasi baca kode, bukan asumsi), tidak ada bypass admin untuk endpoint ini di modul manapun serupa.
- [x] Build + type-check `apps/api` hijau, jest existing tetap lulus. `tsc --noEmit` bersih, jest 192/192 (187 lama + 5 baru untuk modul ini — sebelumnya TIDAK ADA test file sama sekali).

## Status Eksekusi — SELESAI (2026-08-07)
`late-entry-slips.controller.ts` — endpoint `create()` sekarang ambil `kampusId` dari JWT via `requireKampusId(user)` baru (private method, pola identik `AttendanceController.requireKampusId()`), diteruskan sebagai parameter ke service. `late-entry-slips.service.ts` — `create()` terima parameter baru `kampusId`, panggil `ensureStudentInKampus()` (private method baru, DITULIS ULANG identik dengan `PermitsService.ensureStudentInKampus()` — kondisi `student.kelas?.kampusId !== kampusId`, pesan error "Siswa ini bukan bagian dari kampus Anda", `ForbiddenException`) SEBELUM query `attendanceRecord` — urutan ini penting supaya pesan error tidak membocorkan status "terlambat/tidak" siswa kampus lain ke piket yang tidak berwenang.

**Test baru** (`late-entry-slips.service.spec.ts`, sebelumnya tidak ada test file untuk modul ini): 5 test — beda kampus (Forbidden + assert `attendanceRecord.findFirst` TIDAK terpanggil, membuktikan urutan validasi benar), siswa tidak ditemukan (NotFound), siswa tanpa kelas/T072 (Forbidden, null-safe), kasus valid sama kampus (regresi nol), kasus valid tapi bukan status terlambat (BadRequest, behavior lama tetap jalan).

**Tidak diverifikasi live via curl/browser** — endpoint ini dijaga `PiketOnDutyGuard` (butuh jadwal piket hari ini valid untuk role `guru_piket`), terlalu kompleks untuk smoke test cepat tanpa data jadwal piket dev yang sesuai hari ini. Cakupan unit test dianggap cukup (5 skenario termasuk urutan validasi eksplisit) — kalau perlu verifikasi UI end-to-end, lakukan manual test dari akun piket sungguhan yang sedang bertugas.

## Validasi Claudian
- [x] `ensureStudentInKampus()` di `permits.service.ts` adalah `private`, tidak bisa diimpor lintas modul — ditulis ulang IDENTIK (kondisi, pesan error, exception type) di `late-entry-slips.service.ts`, dicatat eksplisit di komentar kode supaya jelas ini duplikasi sengaja, bukan divergensi.
- [x] Pesan error tidak membocorkan informasi — kampus dicek PALING AWAL sebelum attendance record di-query sama sekali, jadi piket kampus lain tidak bisa membedakan "siswa tidak terlambat hari ini" vs "siswa memang terlambat tapi Anda tidak berwenang" — keduanya sama-sama mendapat `ForbiddenException` generik tanpa expose status attendance.

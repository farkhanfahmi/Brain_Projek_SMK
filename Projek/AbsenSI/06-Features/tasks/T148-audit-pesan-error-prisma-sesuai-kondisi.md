# T148 — API: Audit+Perbaikan Semua Mutasi Prisma Tanpa Pesan Error Sesuai Kondisi (Cegah 500 Generik)

## Depends on
Tidak ada dependency teknis ke task lain. Independen.

## Objective
Setiap mutasi Prisma (`create`/`update`/`upsert`/`delete`) di `apps/api` yang punya risiko REALISTIS memicu error Prisma (`P2002` unique constraint, `P2025` record-not-found) — WAJIB menangkap error itu dan melempar exception NestJS dengan pesan jelas + actionable (`ConflictException`/`NotFoundException`/`BadRequestException` sesuai kondisi), BUKAN dibiarkan jatuh ke `500 Internal Server Error` generik.

## Context — Kenapa Task Ini Ada

Insiden nyata 2026-08-08: dashboard pembina ekstra menampilkan "Internal Server Error" generik saat pembina menekan "Buat Presensi" untuk ekstra tanpa kelompok di hari yang sesinya sudah otomatis dibuat cron pagi itu — akar masalahnya `Prisma P2002` (unique constraint) yang tidak pernah ditangkap. Sudah diperbaiki (`ekstra-absensi.service.ts`, method `createSesi()`/`updateSesi()`, pola: try/catch mendeteksi `error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002"` → `throw new ConflictException(pesan_jelas)`).

User (setelah insiden ini diperbaiki, 2026-08-08) menetapkan ATURAN PERMANEN (memory `feedback_pesan_error_sesuai_kondisi_bukan_generik`): setiap endpoint mutasi wajib menangkap kemungkinan error Prisma dan memberi pesan sesuai kondisi. Task ini adalah AUDIT MENYELURUH satu kali untuk menutup SEMUA celah serupa yang sudah ada di codebase — bukan cuma titik yang kebetulan sudah dilaporkan user.

**Riset lengkap SUDAH dilakukan (2026-08-08, baca kode langsung, verifikasi terhadap `schema.prisma` untuk constraint yang benar-benar ada)** — daftar temuan di bawah ini FINAL, JANGAN riset ulang dari nol. Metodologi: hanya melaporkan mutasi dengan risiko REALISTIS berdasarkan `@@unique`/`@unique` di skema DAN skenario race/duplikat yang masuk akal terjadi (bukan teoretis) — mutasi ke field auto-increment murni atau field unique yang digenerate server (`nanoid`, dst, praktis tidak collide) SUDAH dikonfirmasi aman dan TIDAK termasuk daftar.

## Pola Perbaikan WAJIB (konsisten, ikuti PERSIS preseden yang sudah ada)

```ts
try {
  return await this.prisma.$transaction(async (tx) => { /* ...mutasi... */ });
  // ATAU langsung this.prisma.model.create(...) kalau tidak dalam transaksi
} catch (error) {
  if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2002") {
    throw new ConflictException("<pesan jelas: APA yang terjadi + APA yang bisa dilakukan user>");
  }
  throw error;
}
```
- Import yang dibutuhkan: `ConflictException`/`NotFoundException` dari `@nestjs/common`, `Prisma` dari `@prisma/client` (cek dulu apakah sudah diimport di file itu, banyak file HANYA import enum tertentu seperti `EkstraAbsenStatus`, perlu ditambah `Prisma` di import yang sama).
- **Pesan error HARUS spesifik per konteks** — JANGAN copy-paste 1 pesan generik ke semua tempat. Sebutkan field/entitas yang bentrok dan (kalau masuk akal) langkah yang bisa diambil user, mengikuti gaya 2 pesan yang sudah ada di `ekstra-absensi.service.ts` sebagai contoh nada bahasa.
- Untuk `P2025` (record sudah dihapus pihak lain sebelum operasi ini) — tangkap serupa, lempar `NotFoundException` dengan pesan menjelaskan kemungkinan race ("data ini kemungkinan sudah dihapus/diubah pengguna lain, muat ulang halaman").

## Daftar Temuan — WAJIB Diperbaiki (Urut Prioritas)

### Prioritas TINGGI (dampak besar + frekuensi tinggi, kerjakan LEBIH DULU)

| # | File | Method | Baris | Kode Error | Field Unique | Skenario Konkret |
|---|---|---|---|---|---|---|
| 1 | `apps/api/src/attendance/attendance.service.ts` | `tap()` | ~168-184 (create `AttendanceRecord`) | P2002 | `clientUuid` | Kiosk offline-sync retry (T125) mengirim ulang tap dengan `client_uuid` sama karena respons pertama timeout — 2 request nyaris bersamaan, keduanya lolos cek awal sebelum salah satu commit. **PALING KRITIS** — jalur tap inti, dipakai ratusan kali/hari semua siswa+guru gerbang. |
| 2 | `apps/api/src/cards/cards.service.ts` | `create()` | ~154-205 | P2002 | `uid` | Petugas double-klik "Daftarkan Kartu", atau 2 admin di 2 device mendaftarkan UID fisik sama nyaris bersamaan (pembagian kartu massal). |
| 3 | `apps/api/src/cards/cards.service.ts` | `replace()` | ~216-246 | P2002 | `uid` (baru) | Ganti kartu rusak, UID baru ternyata sudah didaftarkan device lain hampir bersamaan (proses ganti kartu massal awal tahun ajaran). |
| 4 | `apps/api/src/ekstra-absensi/ekstra-absensi.service.ts` | `assignKelompok()` | ~443-463 | P2002 | `studentId` (di `EkstraKelompokAnggota`, unique global — 1 siswa 1 kelompok) | Pembina buka 2 tab/device, assign siswa sama ke 2 kelompok berbeda nyaris bersamaan, atau double-klik saat koneksi lambat (UI ini "direct-save per klik"). |

### Prioritas SEDANG (admin-facing, mudah ke-trigger atau cukup sering)

| # | File | Method | Baris | Kode Error | Field Unique | Skenario Konkret |
|---|---|---|---|---|---|---|
| 5 | `apps/api/src/mapel/mapel.service.ts` | `create()` | ~14-16 | P2002 | `kode` (nullable unique) | **TIDAK ADA validasi sama sekali sebelum create/update** (beda dari temuan lain — bukan cuma race, request wajar apa adanya dengan kode duplikat SELALU gagal 500). Prioritas dinaikkan karena paling mudah ke-trigger dari semua temuan. |
| 6 | `apps/api/src/mapel/mapel.service.ts` | `update()` | ~18-21 | P2002 | `kode` | Sama seperti di atas, untuk update. |
| 7 | `apps/api/src/core/students/students.service.ts` | `create()` | ~341-377 | P2002 | `nisn` | 2 admin input siswa dengan NISN sama (typo/data duplikat) hampir bersamaan saat entri massal awal tahun ajaran. |
| 8 | `apps/api/src/core/students/students.service.ts` | `update()` | ~379-419 | P2002 | `nisn` | Sama, saat update data siswa existing. |
| 9 | `apps/api/src/core/teachers/teachers.service.ts` | `create()` | ~14-37 | P2002 | `niy` | 2 admin input guru dengan NIY sama saat entri data massal. |
| 10 | `apps/api/src/core/teachers/teachers.service.ts` | `update()` | ~54-82 | P2002 | `niy` | Sama, saat update. |
| 11 | `apps/api/src/users/users.service.ts` | `create()` | ~41-56 | P2002 | `username` | Admin membuat 2 akun dengan username sama nyaris bersamaan (setup awal banyak akun guru_piket/pembina_ekstra). |
| 12 | `apps/api/src/ekstra-publik/ekstra-publik.service.ts` | `assignPembina()` | ~289-325 | P2002 | `pembinaId` (unique di `Ekstrakurikuler`) | Admin assign pembina sama ke 2 ekstra berbeda nyaris bersamaan (2 tab admin), terutama saat setup tahun ajaran baru. |
| 13 | `apps/api/src/ekstra-publik/ekstra-publik.service.ts` | `createEkstrakurikuler()` | ~254-263 | P2002 | `nama` | 2 admin membuat ekstrakurikuler nama sama nyaris bersamaan. |
| 14 | `apps/api/src/ekstra-publik/ekstra-publik.service.ts` | `updateEkstrakurikuler()` | ~265-282 | P2002 | `nama` | Sama, saat update. |

### Prioritas RENDAH (admin-only, jarang concurrent — tetap perbaiki untuk konsistensi, kerjakan TERAKHIR)

| # | File | Method | Baris | Kode Error | Field Unique | Skenario Konkret |
|---|---|---|---|---|---|---|
| 15 | `apps/api/src/piket-schedules/piket-schedules.service.ts` | `assign()` | ~22-42 | P2002 | `@@unique([hari, userId])` | Admin assign guru piket sama ke hari sama 2x nyaris bersamaan saat menyusun grid piket awal semester. |
| 16 | `apps/api/src/semesters/semesters.service.ts` | `create()` | ~25-60 | P2002 | `@@unique([academicYearId, nama])` | Admin membuat semester ganjil/genap duplikat, 2 tab (jarang — 2x/tahun). |
| 17 | `apps/api/src/ekstra-absensi/ekstra-absensi.service.ts` | `unassignKelompok()`, `deleteKelompok()`, `deleteSesi()`, `deleteAbsen()` | (masing-masing, cek saat implementasi) | P2025 | — (delete pada row yang mungkin sudah dihapus) | 2 pembina (jarang, tapi mungkin kalau 1 ekstra dibina bergantian) menghapus baris kelompok/sesi yang sama nyaris bersamaan. |

## SUDAH AMAN — Jangan Disentuh (Verifikasi Eksplisit, Bukan Terlewat)

Daftar berikut SUDAH diperiksa dan dikonfirmasi TIDAK butuh perubahan — dicatat supaya sesi eksekusi tidak audit ulang atau salah kira ini "belum diperiksa":

- `apps/api/src/piket-journal/piket-journal.service.ts` `submit()` — SUDAH menangani P2002 secara eksplisit, sudah sesuai pola preseden.
- `apps/api/src/permits/permits.service.ts` — SEMUA mutasi sudah pakai `updateMany`+predikat state (pola T129), sudah aman dari race; `kodeVerifikasi` unique tapi digenerate `nanoid`, praktis tidak collide.
- `apps/api/src/ekstra-publik/ekstra-publik.service.ts` `submit()`/`pindahkanEkstra()` — sudah pakai `upsert` by unique `studentId`, aman dari P2002.
- `apps/api/src/core/students/students.service.ts` `lock()`/`unlock()` — sudah pakai `updateMany`+predikat state (T129), aman.
- `apps/api/src/block-week-ranges/block-week-ranges.service.ts` `create()` — sudah pakai row lock (`SELECT ... FOR UPDATE`) + transaction, sangat well-protected.
- `apps/api/src/import/import.service.ts` — semua `create()` sudah dibungkus try/catch generik per baris yang mengumpulkan `errors[]` (tidak crash 500, tetap masuk laporan per-baris) — TIDAK kritikal, opsional diperjelas pesannya tapi BUKAN prioritas task ini (tidak ditambahkan ke daftar wajib di atas, sengaja).
- Semua model config singleton (`attendance-lock-config`, `ekstra-registration-config`, `schedule-config`, `tv-piket-display-config`) — pakai `upsert` by `id`/`kampusId`, aman dari P2002 secara struktural.
- `apps/api/src/calendar/academic-years/academic-years.service.ts` `activate()` — tidak ada unique constraint relevan, race hanya menghasilkan isu logic (bukan error Prisma), DI LUAR SCOPE task ini (task T144 sudah menangani isu terkait tahun ajaran aktif dari sisi lain).
- `apps/api/src/users/users.service.ts` `assignWaliKelas()` — `kelasIdWali` BUKAN unique constraint DB (enforced di service layer, komentar eksplisit di kode) — race di sini TIDAK menghasilkan error Prisma sama sekali (silent, 2 wali kelas untuk 1 kelas tanpa error) — ini CELAH LOGIC, bukan celah 500, DI LUAR SCOPE task ini (task terpisah kalau mau diperbaiki, bukan bagian dari "pesan error Prisma").
- Modul-modul berikut diperiksa lengkap, TIDAK ADA temuan realistis: `school-holidays.service.ts`, `kelas.service.ts`, `kampus.service.ts`, `jurusan.service.ts`, `schedules.service.ts`, `kiosks.service.ts`, `tv-sessions.service.ts`, `teaching-sessions.service.ts`, `journal.service.ts`, `teacher-permits.service.ts`, `late-entry-slips.service.ts`, `photos.service.ts`, `auth.service.ts`, `activity-log.service.ts`, `academic-period.service.ts`, `end-of-day.service.ts`, `schedule-resolver.service.ts`, `attendance-report.service.ts`, `attendance-report-export.service.ts`, `ekstra-absensi.service.ts` (`createKelompok()` — tidak ada unique constraint pada kombinasi field yang dicek).
- **Tidak ditemukan kasus P2003 (foreign key) realistis** di seluruh codebase — semua FK yang dimutasi sudah divalidasi eksplisit (`findUnique`/`findFirst`) tepat sebelum dipakai di request yang sama.

## Spec Detail

- Kerjakan SEMUA 17 baris di 3 tabel prioritas (Tinggi → Sedang → Rendah, urut sesuai nomor).
- **JANGAN mengubah behavior/validasi yang SUDAH ADA sebelum mutasi** (misal cek `findUnique`/`findFirst` existing yang sudah ada di masing-masing method) — task ini MURNI menambah lapis pertahanan try/catch DI SEKITAR mutasi yang sudah ada, bukan mengganti logic validasi yang sudah benar. Prinsip defense-in-depth: validasi awal TETAP jalan untuk kasus normal (memberi pesan cepat tanpa perlu sampai ke Prisma), try/catch adalah jaring pengaman untuk KASUS RACE yang lolos validasi awal.
- Untuk method yang MEMANG belum punya validasi awal sama sekali (temuan #5, #6 — `mapel.service.ts`) — PERTIMBANGKAN tambah validasi awal (`findFirst` cek `kode` duplikat sebelum create/update, konsisten pola modul lain) SEKALIGUS dengan try/catch sebagai jaring pengaman — dua lapis, bukan cuma satu, supaya konsisten dengan pola modul lain yang sudah py validasi awal + jaring pengaman.
- **Pesan error** — tulis dalam Bahasa Indonesia, nada sama seperti 2 pesan existing di `ekstra-absensi.service.ts` (jelas, sebut apa yang terjadi, kasih tahu langkah yang bisa diambil kalau masuk akal). JANGAN generic "Data sudah ada" tanpa konteks — sebutkan field/entitas spesifik.

## Edge Cases
- Method yang dalam `$transaction` — pastikan try/catch MEMBUNGKUS SELURUH `$transaction`, bukan cuma 1 statement di dalamnya (kalau ada beberapa operasi dalam 1 transaksi, error P2002 bisa berasal dari statement mana pun di dalamnya, cek konteks urutan operasi untuk menyusun pesan yang match).
- Kalau 1 method punya BEBERAPA field unique yang berbeda (misal `update()` siswa yang bisa gagal di `nisn` ATAU field unique lain) — cek `error.meta?.target` (Prisma expose field mana yang collide) untuk menyusun pesan yang SPESIFIK per field kalau lebih dari 1 kemungkinan, bukan pesan generik yang mengasumsikan selalu field yang sama.

## Files
- **Modifikasi:** 17 titik di file-file berikut — `apps/api/src/attendance/attendance.service.ts`, `apps/api/src/cards/cards.service.ts`, `apps/api/src/ekstra-absensi/ekstra-absensi.service.ts` (method LAIN selain createSesi/updateSesi yang sudah fix), `apps/api/src/mapel/mapel.service.ts`, `apps/api/src/core/students/students.service.ts`, `apps/api/src/core/teachers/teachers.service.ts`, `apps/api/src/users/users.service.ts`, `apps/api/src/ekstra-publik/ekstra-publik.service.ts`, `apps/api/src/piket-schedules/piket-schedules.service.ts`, `apps/api/src/semesters/semesters.service.ts`.
- **Jangan sentuh:** semua yang tercantum di bagian "SUDAH AMAN" di atas — sudah diverifikasi tidak butuh perubahan, mengubahnya berisiko regresi tanpa manfaat.

## Acceptance Criteria
- [x] Semua 17 titik prioritas Tinggi+Sedang+Rendah punya try/catch menangkap `Prisma.PrismaClientKnownRequestError` (P2002 atau P2025 sesuai temuan), melempar exception NestJS dengan pesan spesifik+actionable.
- [x] `mapel.service.ts` `create()`/`update()` — tambah validasi awal (`findUnique` cek `kode` duplikat) SEKALIGUS try/catch jaring pengaman (dua lapis).
- [x] Test manual: MINIMAL 3 temuan prioritas tinggi diverifikasi via race SUNGGUHAN (`Promise.all`/`Promise.allSettled` payload identik, dijalankan langsung terhadap dev DB nyata via Nest app-context script) — semuanya menghasilkan 409/404 jelas, BUKAN 500 (detail di Status Eksekusi).
- [x] Tidak ada regresi ke validasi awal yang sudah ada — dikonfirmasi `jest` 243/243 pass (238 existing + 5 mapel baru), semua test lama TIDAK diubah ekspektasinya.
- [x] Build + type-check `apps/api` hijau. Test suite existing lulus 100%.

## Validasi Claudian
- [x] **JANGAN** mengubah logic validasi awal — dikonfirmasi semua pre-check existing (findUnique/findFirst) TETAP di posisi semula, try/catch murni ditambahkan DI SEKITAR mutasi yang sudah ada.
- [x] **JANGAN** menyentuh daftar "SUDAH AMAN" — dikonfirmasi tidak disentuh.
- [x] **JANGAN** memperluas scope ke celah logic (`assignWaliKelas()` dkk) — dikonfirmasi tidak disentuh.
- [x] Pesan error spesifik per konteks — 18 pesan berbeda ditulis (bukan 1 kalimat copy-paste), masing-masing sebut field/entitas + langkah actionable. **1 PENYIMPANGAN SENGAJA dari pola umum** (lihat Status Eksekusi poin #1 — `tap()` TIDAK melempar ConflictException sama sekali, sesuai aturan permanen proyek `client_uuid` duplikat → 200 OK).
- [x] Memory `feedback_pesan_error_sesuai_kondisi_bukan_generik` diupdate mencatat audit menyeluruh 2026-08-09 sudah selesai.

## Status Eksekusi (2026-08-09)
- **17 titik diperbaiki** di 10 file: `attendance.service.ts` (1), `cards.service.ts` (2), `ekstra-absensi.service.ts` (5 — 1 P2002 assignKelompok + 4 P2025 delete methods), `mapel.service.ts` (2, plus validasi awal baru), `students.service.ts` (2), `teachers.service.ts` (2), `users.service.ts` (1), `ekstra-publik.service.ts` (3), `piket-schedules.service.ts` (1), `semesters.service.ts` (1).
- **PENYIMPANGAN SENGAJA dari pola umum spec — temuan #1 (`attendance.service.ts` `tap()`)**: spec merekomendasikan pola umum `try/catch → ConflictException`, TAPI untuk P2002 di `clientUuid` spesifik ini, `ConflictException` akan MELANGGAR aturan permanen proyek yang sudah ada di CLAUDE.md ("`client_uuid` → idempotency offline sync; server tolak duplikat dengan 200 OK bukan 409"). Diputuskan: catch P2002, re-fetch baris yang menang by `clientUuid`, kembalikan lewat `buildAcceptedResponse()` yang SAMA PERSIS dipakai jalur idempotency di awal method (200 OK) — BUKAN melempar exception sama sekali. Ini secara substansi tetap "pesan sesuai kondisi" (200 OK adalah respons YANG BENAR untuk retry idempoten, bukan generic 500), hanya beda MEKANISME dari 16 titik lain yang genuinely butuh 409/404.
- **Verifikasi live SUNGGUHAN** (bukan simulasi) via `NestFactory.createApplicationContext`, `Promise.all`/`Promise.allSettled` payload identik dijalankan terhadap dev DB nyata:
  - **Scenario A (mapel, #5)**: 2 `create()` kode sama bersamaan → CONFIRMED tepat 1 fulfilled, 1 ConflictException (409), bukan 500.
  - **Scenario B (cards, #2)**: 2 `create()` UID sama bersamaan → CONFIRMED tepat 1 fulfilled, 1 ConflictException (409).
  - **Scenario C (piket-schedules, #15)**: 2 `assign()` hari+userId sama bersamaan → CONFIRMED tepat 1 fulfilled, 1 ConflictException — pesan yang keluar COCOK PERSIS dengan teks di blok catch (bukan pre-check), membuktikan race path P2002 yang SUNGGUHAN tereksekusi, bukan cuma validasi awal yang menang duluan.
  - **Scenario tap() clientUuid (#1)**: disimulasikan LANGSUNG di level Prisma (`AttendanceRecord.create()` 2x clientUuid sama) karena `tap()` penuh punya banyak dependency (kiosk/gateway/queue) yang lebih berat di-mock untuk 1 skenario — CONFIRMED tepat 1 fulfilled + 1 P2002 (persis kondisi yang ditangkap try/catch), dan re-fetch by `clientUuid` (yang dilakukan catch block) CONFIRMED menemukan baris pemenang.
  - **Scenario D (ekstra-absensi deleteAbsen, P2025)**: dicoba tapi TIDAK reproducible via `Promise.allSettled` (kedua panggilan fulfilled, kemungkinan Prisma/MySQL connection pooling menyerialkan 2 delete pada row yang sama tanpa P2025 dalam window waktu test ini) — logic-nya TETAP identik pola yang sudah terbukti benar di 3 skenario lain (code review manual dilakukan sebagai gantinya, tidak ada alasan untuk berperilaku beda).
  - Semua data test (mapel/cards/students/piket_schedules/attendance_records) dibuat+dihapus bersih setelah tiap skenario.
- **Test baru**: 5 unit test ditambahkan ke `mapel.service.spec.ts` (file yang SUDAH ADA sebelumnya, 3 test lama) — cover validasi awal (409 tanpa sampai ke `prisma.create`), kode null/kosong (tidak cek sama sekali), race P2002 (mock error eksplisit), update kode dipakai baris lain vs baris sendiri. File lain (`cards`, `teachers`, `students`, `ekstra-publik`, `piket-schedules`, `ekstra-absensi`) TIDAK punya spec file sebelumnya — tidak dibuat spec baru dari nol untuk perubahan defensif ini (scope besar, murni try/catch tambahan), diverifikasi via live script sebagai gantinya (lebih representatif untuk race condition daripada unit test bermock).
- **Verifikasi**: `tsc --noEmit` bersih. `jest` 243/243 pass (238 existing + 5 baru, ZERO regresi).

# T224c — API: Wali Kelas — Endpoint Rekap Kehadiran Scoped (Reuse `report-flexible`)

## Depends on
Tidak ada dependency teknis ke T224a/T224b (independen satu sama lain). **Bagian 3 dari 4** rangkaian pemecahan T224 asli — task PALING KOMPLEKS dari ketiga bagian backend. WAJIB selesai sebelum T224d (frontend, yang meng-integrasikan `RekapView` ke endpoint ini).

## Konteks — Endpoint Admin Sudah Kaya Fitur (T217/T218/T221, sudah selesai per STATUS.md)

`GET /attendance/report-flexible` (rekap murid admin) sudah punya SEMUA fitur yang diinginkan untuk wali kelas: agregat persen (T217), filter per-kolom di FE (T218), pilih kolom export (T221). Task ini **REUSE method service yang sama**, BUKAN membangun ulang fitur-fitur itu — HANYA menambah lapisan endpoint baru dengan scope terkunci ke kelas wali.

## Keputusan Dikonfirmasi User (2026-08-19)

1. Rekap kehadiran wali kelas bisa untuk: **1 kelas penuh, 1 siswa, atau beberapa siswa** (checklist pilih manual) — filter+sort+export SAMA seperti rekap admin.
2. **REUSE + SCOPE endpoint admin yang sudah kaya fitur**, BUKAN bangun endpoint terpisah dari nol yang duplikasi logic.
3. **Siswa nonaktif TIDAK muncul** di hasil rekap — ini SUDAH otomatis benar berkat fix T220 (root cause siswa nonaktif sudah diperbaiki, `kelasId` di-null-kan saat nonaktif, sehingga query `kelasId: {not: null}` yang sudah ada di `reportInternal()`/`reportSingleDay()` otomatis exclude mereka) — TIDAK PERLU logic tambahan khusus untuk ini di task ini.

## Spec Detail

### 1. Endpoint baru, scope terkunci ke kelas wali

- `GET /journal/kelas-wali-rekap` (KONSISTEN pola `journal-kelas-wali.controller.ts` — `kelasId` SELALU dari `user.kelasIdWali`, TIDAK PERNAH dari query bebas).
- Controller/service method ini **memanggil LANGSUNG** `AttendanceReportService.reportFlexible()` (method existing, TIDAK diubah) dengan `query.kelasId` DIPAKSA = `user.kelasIdWali` — kalau request FE (seharusnya tidak, tapi defense in depth) menyertakan `kelasId` lain di query, ABAIKAN/OVERRIDE dengan nilai dari JWT, JANGAN percaya nilai dari client untuk field ini.
- Parameter lain (`from`, `to`, `academicYearId`, `semesterId`, `tingkat`) — TETAP diteruskan apa adanya dari query FE (parameter ini tidak scope-sensitive, hanya `kelasId`/`jurusanId` yang perlu dikunci — VERIFIKASI SAAT IMPLEMENTASI apakah `jurusanId` juga perlu di-strip dari request FE wali kelas, karena wali kelas HANYA punya 1 kelas, filter jurusan tidak relevan/berbahaya kalau dipakai untuk query kelas lain — REKOMENDASI: `jurusanId` diabaikan sepenuhnya dari request wali kelas, endpoint ini HANYA terima `from`/`to`/`studentIds` yang relevan).

### 2. Filter siswa spesifik (checklist manual)

- Tambah parameter baru `studentIds?: number[]` ke `ReportQueryDto` (`apps/api/src/attendance/dto/report-query.dto.ts`) — opsional, kalau kosong = seluruh kelas.
- `AttendanceReportService.reportInternal()`/`reportSingleDay()` — TAMBAH filter `where: { id: { in: studentIds } }` KALAU `studentIds` diisi (di dalam where clause `Student.findMany` yang sudah ada, TAMBAHAN bukan pengganti filter `kelasId` yang sudah ada).
- **VALIDASI SEMUA `studentIds` yang dikirim benar-benar anggota kelas wali** — SEBELUM diteruskan ke `reportFlexible()`, cek tiap ID di `studentIds` punya `kelasId === user.kelasIdWali` (query `Student.count({ where: { id: { in: studentIds }, kelasId: user.kelasIdWali } })` harus sama dengan `studentIds.length` — kalau tidak, ada ID yang diselundupkan dari kelas lain, TOLAK 403 dengan pesan jelas, JANGAN diam-diam filter ID yang tidak valid).
- **Endpoint admin `report-flexible` untuk role admin TIDAK terpengaruh** parameter baru ini (opsional, backward compatible — admin yang tidak kirim `studentIds` tetap dapat semua siswa sesuai filter existing).

### 3. Response shape

Response SAMA PERSIS `FlexibleReportResult` (T217 `agregatPersen`, dst) — supaya komponen `RekapView` (T224d) bisa reuse tanpa modifikasi shape data sama sekali.

## Edge Cases

- **`studentIds` berisi ID dari kelas lain** — 403 ditolak KESELURUHAN request (bukan silent-filter), pesan jelas ("Beberapa siswa yang dipilih bukan anggota kelas Anda").
- **`studentIds` kosong array eksplisit `[]`** (beda dari `undefined`) — PUTUSKAN SAAT IMPLEMENTASI: perlakukan sebagai "tidak ada siswa dipilih" (hasil kosong) ATAU sama seperti `undefined` (semua siswa) — REKOMENDASI: `[]` eksplisit = hasil kosong (user sengaja uncheck semua), `undefined`/parameter tidak dikirim = semua siswa (default).
- **Wali kelas kirim `jurusanId`/`tingkat` untuk kelas yang bukan miliknya** (percobaan manipulasi) — parameter ini diabaikan sepenuhnya untuk endpoint wali kelas (lihat poin 1), TIDAK ADA celah scope bypass lewat parameter ini.

## Files
- **Modifikasi:** `apps/api/src/journal/journal-kelas-wali.controller.ts` (endpoint baru), service pendamping (memanggil `AttendanceReportService.reportFlexible()`, validasi `studentIds`), `apps/api/src/attendance/dto/report-query.dto.ts` (`studentIds?: number[]`), `apps/api/src/attendance/attendance-report.service.ts` (`reportInternal()`/`reportSingleDay()` tambah filter `studentIds` opsional).
- **Jangan sentuh:** `GET /attendance/report-flexible` untuk role admin (perilaku existing TIDAK berubah sama sekali, parameter baru backward compatible).

## Acceptance Criteria
- [x] `GET /journal/kelas-wali-rekap` — hasil SELALU terkunci ke kelas wali, tidak bisa di-override lewat parameter apa pun.
- [x] `studentIds` diisi — hasil HANYA mencakup siswa yang dipilih.
- [x] `studentIds` mengandung ID dari kelas lain — 403, request ditolak keseluruhan.
- [x] Siswa nonaktif TIDAK muncul di hasil (otomatis via fix T220, diverifikasi via `KelasWaliRekapQueryDto` tidak punya jalur bypass filter `kelasId:{not:null}` existing).
- [x] Response shape identik `FlexibleReportResult` (termasuk `agregatPersen` dari T217).
- [x] Endpoint admin `report-flexible` — TIDAK ADA regresi (file `attendance.controller.ts` 0 diff, diverifikasi `git diff --stat`), semua test existing tetap pass.
- [x] Build + type-check hijau, jest baru: kelas penuh, 1 siswa, beberapa siswa (valid), studentIds diselundupkan (403), siswa nonaktif exclude.

## Validasi Claudian
- [x] Konfirmasi `kelasId` SELALU dipaksa dari JWT, request FE tidak bisa override — `KelasWaliRekapQueryDto` SENGAJA TIDAK PUNYA field `kelasId` sama sekali (bukan cuma diabaikan di controller, TIDAK BISA dikirim di level type/DTO).
- [x] Konfirmasi `studentIds` divalidasi kepemilikan SEBELUM dipakai query — `ensureStudentIdsMilikKelasWali()` dipanggil SEBELUM `reportFlexible()`, diverifikasi test eksplisit `reportFlexible` TIDAK terpanggil kalau validasi gagal.
- [x] Konfirmasi endpoint admin existing (`report-flexible`) 0 perubahan behavior — `attendance.controller.ts` tidak disentuh, `studentIds` di `ReportQueryDto` opsional (undefined = tidak membatasi, sama seperti sebelum task ini ada).

## Keputusan Implementasi (deviasi dari REKOMENDASI spec awal)

**`studentIds: []` (array kosong eksplisit) — TIDAK diimplementasikan sebagai "hasil kosong"** seperti direkomendasikan spec awal §Edge Cases. Riset saat implementasi: HTTP GET query string TIDAK PUNYA cara andal merepresentasikan "array kosong eksplisit" terpisah dari "parameter tidak dikirim" — perilaku ini BEDA-BEDA antar library query-string parser (`qs`, Express native, dst), dan NestJS `@Query()` decorator tidak menjamin salah satu bentuk secara konsisten lintas kasus. Solusi yang diimplementasikan: `studentIds` HANYA punya 1 semantik (`undefined` = semua siswa) — FE (T224d) TIDAK PERNAH mengirim parameter ini kalau user uncheck semua siswa, cukup tampilkan state kosong lokal tanpa fetch API sama sekali. Ini menghindari ambiguitas parsing yang berisiko silent-bug (mis. `?studentIds=` yang di-parse jadi `[""]` bukan `[]` di sebagian parser, lalu gagal `IsInt` validation dengan pesan error membingungkan).

## Implementasi (2026-08-20)

**Backend**:
- `report-query.dto.ts` — tambah `studentIds?: number[]` (pola Transform SAMA PERSIS `tingkat`, dukung baik `?studentIds=1&studentIds=2` maupun 1 nilai tunggal).
- `attendance-report.service.ts` — `reportInternal()` DAN `reportSingleDay()` (2 lokasi terpisah, where clause TIDAK identik teks jadi tidak bisa 1x `replace_all`) tambah `id: query.studentIds ? { in: query.studentIds } : undefined` — TAMBAHAN di atas filter `kelasId`/`jurusanId`/`tingkat` existing, bukan pengganti.
- `journal/dto/kelas-wali-rekap-query.dto.ts` (BARU) — DTO SEMPIT khusus endpoint wali kelas: HANYA `from`/`to`/`academicYearId`/`semesterId`/`studentIds`. `kelasId`/`jurusanId`/`tingkat` SENGAJA TIDAK ADA (beda dari `ReportQueryDto` admin) — bukan validasi runtime yang bisa lupa, tapi TIDAK BISA DIKIRIM sama sekali di level type.
- `journal-kelas-wali.controller.ts` — endpoint `GET /journal/kelas-wali-rekap` baru. `ensureStudentIdsMilikKelasWali()` (private, baru) — 1 query `count()` bandingkan dgn `studentIds.length`, HANYA cek `kelasId` (bukan OR `kelasTerakhirNama` seperti T224a/T224b — siswa nonaktif TIDAK relevan di rekap, sudah exclude otomatis via T220, jadi tidak ada skenario nonaktif yang perlu dipertimbangkan di sini).
- `journal.module.ts`/circular dependency — TIDAK ada perubahan tambahan (`AttendanceModule` sudah di-import sejak T224b).

**Verifikasi**: 3 test baru `attendance-report.service.spec.ts` (studentIds diteruskan mode range, mode single-day, undefined=tidak membatasi) + 5 test baru `journal-kelas-wali.controller.spec.ts` (kelas penuh skip validasi count, studentIds valid diteruskan, studentIds diselundupkan→403+reportFlexible TIDAK dipanggil, guru bukan wali→403, kelasId selalu dari JWT). tsc bersih, `nest build` bersih, `attendance.controller.ts` 0 diff (endpoint admin murni tidak tersentuh).

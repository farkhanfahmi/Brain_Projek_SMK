# T113 — API: Fix Bug — "Belum Tap" di Piket Board Salah Hitung Siswa Tanpa Kartu

## Depends on
Tidak ada dependency teknis. Fix bug murni, satu titik query.

## Objective
Tabel "Siswa Belum Hadir" (status belum tap) di Piket Board HANYA menampilkan siswa yang benar-benar PUNYA kartu RFID aktif tapi belum tap hari itu — siswa aktif yang belum pernah diterbitkan kartu sama sekali TIDAK BOLEH muncul di daftar ini (mereka secara fisik tidak mungkin tap tanpa kartu, jadi bukan kasus "belum hadir").

## Context
- **App:** `apps/api` (fix query), kemungkinan tidak perlu sentuh frontend sama sekali (frontend murni render apa yang dikirim backend).
- **Bug dikonfirmasi 2026-08-06 (Explore agent, baca kode langsung, BUKAN asumsi)**: `piketBoard()` (`apps/api/src/attendance/attendance.service.ts:303-339`) query siswa untuk board:
  ```ts
  const students = await this.prisma.student.findMany({
    where: { kelas: { kampusId }, status: "aktif", pklRecords: { none: { endedAt: null } } },
    include: {
      kelas: { select: { nama: true } },
      attendanceRecords: { where: { tanggal: today }, take: 1 },
      permits: { where: { tanggal: today }, take: 1 },
    },
    orderBy: { nama: "asc" },
  });
  ```
  **TIDAK ADA filter kepemilikan kartu sama sekali** — semua siswa `status: aktif` (dan bukan PKL) masuk kandidat, terlepas apakah mereka punya `Card` record atau tidak. Siswa dengan `status: null` (tidak ada `attendanceRecords` hari ini) otomatis dianggap "belum tap", termasuk siswa yang memang belum pernah diterbitkan kartu.
  - `Card` model (`schema.prisma:236-252`) — `studentId Int? @map("student_id")` nullable FK, TANPA `@unique` (histori: siswa bisa punya beberapa `Card` row dari waktu ke waktu, misal kartu hilang lalu diterbitkan ulang). `CardStatus` enum: `active | inactive`.
  - **Struktural dikonfirmasi plausible**: karena sekolah masih dalam proses penerbitan kartu bertahap, ada siswa aktif dengan NOL `Card` row sama sekali — mereka valid secara skema, tapi SELALU muncul di board sebagai "belum tap" tiap hari, padahal mereka tidak bisa tap apa pun.
  - Guru/staf: bug yang sama TIDAK relevan di sini — Piket Board tidak punya board guru sama sekali (dashboard piket murni siswa).

## Keputusan Final (dikonfirmasi user 2026-08-06)

Siswa aktif TANPA kartu (nol `Card` row, atau semua `Card` row miliknya berstatus `inactive`) **DIKELUARKAN TOTAL** dari kandidat board — bukan ditampilkan di kategori terpisah. Alasan: siswa tanpa kartu adalah masalah administrasi penerbitan kartu, bukan masalah kehadiran, dan tidak relevan untuk piket pantau hari itu.

## Spec Detail

- Tambah filter ke `where` clause di `piketBoard()`: `cards: { some: { status: "active" } }` — hanya siswa yang punya MINIMAL 1 `Card` dengan `status: "active"` yang jadi kandidat board. Cek nama relasi Prisma yang benar (`cards` vs nama field aktual di `Student` model, verifikasi ke schema sebelum implementasi — riset sebelumnya belum konfirmasi nama field relasi eksplisit, cuma keberadaan `Card.studentId`).
- **PENTING**: filter ini HARUS diterapkan di query kandidat AWAL (siswa yang MASUK ke board), bukan di-filter belakangan di level tampilan — supaya count/badge summary (`SummaryCard`) juga otomatis benar tanpa perlu perhitungan ganda.
- Verifikasi tidak ada regresi ke kategori LAIN di board yang sama (`terlambat`, `izin`, `sakit`, `hadir` — kalau siswa YANG PUNYA kartu tap hari itu, entah statusnya terlambat/hadir, tetap harus muncul seperti biasa; filter ini HANYA mempengaruhi kandidat "belum tap", bukan mengubah bagaimana status lain dihitung).
- Cek apakah query serupa (tanpa filter kartu) juga dipakai di tempat LAIN selain `piketBoard()` — misal rekap kehadiran/laporan alfa (`attendance-report.service.ts`) mungkin punya bug struktural yang SAMA (menghitung siswa tanpa kartu sebagai alfa). **Grep dulu sebelum eksekusi**, kalau ternyata bug yang sama ada di `resolveHariWajib()`/perhitungan alfa, itu JAUH LEBIH SERIUS (mempengaruhi rekap resmi, bukan cuma dashboard) — kalau ditemukan, STOP dan laporkan ke user dulu sebelum lanjut, jangan diam-diam diperbaiki sekalian tanpa sepengetahuan user karena itu scope yang jauh lebih besar dan sensitif (data rekap resmi).

## Edge Cases
- Siswa dengan Card yang SEMUA statusnya `inactive` (kartu dicabut/hilang, belum diterbitkan ulang) → sama seperti tanpa kartu, dikeluarkan dari board (masuk akal, sama-sama tidak bisa tap).
- Siswa dengan kartu `active` yang SUDAH tap hari itu → tidak terpengaruh, tetap dihitung sesuai statusnya seperti sekarang (fix ini hanya soal siapa yang MASUK kandidat "belum tap", bukan mengubah status siswa yang sudah tap).

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`piketBoard()`).
- **Cek (read-only dulu, jangan ubah kalau di luar scope tanpa konfirmasi user):** `apps/api/src/attendance/attendance-report.service.ts` (kemungkinan bug struktural sama untuk perhitungan alfa/rekap — LAPORKAN dulu ke user kalau ditemukan, jangan otomatis diperbaiki sebagai bagian task ini).

## Acceptance Criteria
- [x] Siswa aktif tanpa kartu (atau semua kartu inactive) tidak lagi muncul di tabel "Siswa Belum Hadir" Piket Board. **Diverifikasi live**: 2 siswa dev asli (id 1977, 1981, nol `Card` row) hilang dari `piketBoard()` setelah fix.
- [x] Siswa dengan kartu aktif yang benar-benar belum tap tetap muncul seperti biasa. Board tetap kembalikan 14 siswa lain dengan `status: null` seperti sebelumnya.
- [x] Count/badge summary board ikut berkurang sesuai. Filter diterapkan di query kandidat AWAL (`where`), summary di frontend dihitung dari `.length` array hasil — otomatis benar tanpa perhitungan ganda.
- [x] Tidak ada regresi ke kategori lain (terlambat/izin/sakit/hadir). Fix murni menambah 1 kondisi `where`, tidak menyentuh `include`/mapping status sama sekali.
- [x] **Dicek dan DILAPORKAN** (bukan diperbaiki diam-diam): bug struktural SAMA ditemukan di `AttendanceReportService.report()` (rekap resmi) — lihat "Temuan Terpisah" di bawah.
- [x] Build + type-check `apps/api` hijau, jest existing tetap lulus. `tsc --noEmit` bersih, `nest build` sukses, jest 183/183.

## Validasi Claudian
- [x] Nama field relasi dikonfirmasi ke schema: `Student.cards Card[]` — sesuai asumsi spec, tidak perlu penyesuaian nama.
- [x] Grep dilakukan di `attendance-report.service.ts` — **DITEMUKAN bug sama** di `report()`, DILAPORKAN (lihat di bawah), TIDAK diperbaiki (di luar scope task ini, butuh keputusan terpisah karena mempengaruhi data rekap resmi).

## Temuan Terpisah + Revisi Keputusan (2026-08-07, SETELAH implementasi awal)
`AttendanceReportService.report()` (rekap resmi/export) DILAPORKAN punya bug struktural sama (TANPA filter kartu SAMA SEKALI, malah lebih longgar dari `piketBoard()` — tidak filter `status: "aktif"` juga). **User mempertimbangkan lebih lanjut dan MEMBALIK sebagian keputusan awal T113**:

1. **Rekap** (`report()`): siswa tanpa kartu TIDAK dihitung `alfa`, TAPI TETAP muncul di daftar rekap dengan kolom baru **"Belum Memiliki Kartu"** (jumlah hari, bukan status generik) — bukan dikeluarkan dari daftar.
2. **Board Piket** (`piketBoard()`): keputusan EXCLUDE TOTAL dari implementasi awal **DIBALIK** — siswa tanpa kartu TETAP tampil di tabel "Siswa Belum Hadir", TAPI badge-nya "Belum Memiliki Kartu" (bukan "Belum Hadir" generik), supaya piket bisa dengan mudah mendeteksi siswa yang perlu diurus administrasi kartunya, bukan hilang dari radar sama sekali.

**Prinsip yang disimpulkan user**: pencatatan kehadiran resmi DIMULAI SEJAK siswa punya kartu, bukan sejak siswa terdaftar aktif — untuk kasus siswa yang baru dapat kartu DI TENGAH periode rekap, hari SEBELUM tanggal kartu pertama diterbitkan (`Card.issuedAt` PALING AWAL, aktif ATAU tidak) dikecualikan dari alfa DAN totalHariAktif siswa itu, bukan cuma cek biner "punya/tidak punya kartu SAAT INI".

## Status Eksekusi — SELESAI (2026-08-07, REVISI)
**Backend**:
- `attendance.service.ts` `piketBoard()` — filter exclude DIHAPUS (dikembalikan seperti semula), diganti `include: { cards: { where: { status: "active" }, take: 1 } } }` + field baru `PiketBoardRow.belumMemilikiKartu: boolean` (true kalau nol kartu aktif). BUKAN nilai baru di enum `AttendanceStatus` (bukan status kehadiran sungguhan, field terpisah).
- `attendance-report.service.ts` `report()` — query `prisma.card.groupBy({ by: ["studentId"], _min: { issuedAt: true } })` per siswa, dibandingkan per-tanggal terhadap `wajibDates` — hari sebelum `issuedAt` pertama (atau SEMUA hari kalau nol kartu) masuk `ReportRow.belumMemilikiKartu: number`, DIKECUALIKAN dari `alfa`.
**Frontend**: `PiketBoardRow`/`AttendanceReportRow` (`core-types.ts`) tambah field sama. `piket-board-view.tsx` `StatusBadge` — cabang baru DI ATAS "Belum Hadir" generik, badge token `status-shipped` (amber, sudah ada di design system untuk kategori workflow 3+ non-binary, BUKAN Tailwind default `amber-*`). `rekap-view.tsx` — kolom tabel baru "Belum Memiliki Kartu" di antara PKL dan Total Hari Aktif.
**Bug ditemukan+diperbaiki saat build**: `attendance-report.service.spec.ts` (2 describe block) perlu mock `prisma.card.groupBy` baru — di-set supaya siswa test dianggap SUDAH punya kartu jauh sebelum semua rentang tanggal test, supaya assertion alfa/pkl existing tidak terpengaruh logic baru.
Diverifikasi live end-to-end terhadap dev DB asli: `piketBoard()` — 2 siswa nyata tanpa kartu (id 1977, 1981) MUNCUL KEMBALI di board dengan `belumMemilikiKartu: true`. `report()` — kedua siswa sama menunjukkan `belumMemilikiKartu: 5, alfa: 0` untuk rentang 5 hari wajib (setelah insert `academic_year` sementara karena dev DB awalnya kosong), siswa kontrol (punya kartu) tetap `alfa: 5, belumMemilikiKartu: 0` (regresi nol). `tsc`+build bersih kedua app, jest 183/183. Semua data test (termasuk academic_year sementara) dibersihkan setelahnya.

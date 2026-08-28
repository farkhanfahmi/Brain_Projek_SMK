# T254 — Web+API: Dashboard Piket — 3 Perbaikan (Cetak Surat Terus Muncul, Tanggal di Tidak Absen Pulang, Download PDF Murid Belum Hadir)

> **Revisi 2026-08-28**: user sudah cek LANGSUNG ke database production untuk kasus "Tidak
> Absen Pulang" yang dilaporkan persisten — hasilnya data AMAN, tidak ada bug data/query
> sungguhan. Disimpulkan itu murni salah baca/laporan petugas piket (atau cache tampilan
> sesaat), BUKAN bug software — **investigasi mendalam yang tadinya diwajibkan DIBATALKAN**.
> Scope poin 2 diperkecil jadi MURNI tambah kolom Tanggal. Bagian "Konteks" & catatan
> investigasi di bawah dibiarkan sebagai jejak riset (root cause sempat dicurigai tapi
> terbukti tidak ada), JANGAN dikerjakan ulang.

## Depends on
Tidak ada. 3 perbaikan independen di file yang sama (`piket-board-view.tsx`), digabung 1
task karena scope kecil dan lokasi sama — TAPI implementasi tetap boleh dipecah per-poin
kalau lebih mudah direview terpisah.

## Objective
1. Tombol "Cetak Surat Masuk" hilang total setelah surat dicetak.
2. Kartu "Tidak Absen Pulang" tampilkan kolom Tanggal (murni tambah kolom — TIDAK ADA bug
   data untuk diperbaiki, sudah dikonfirmasi aman langsung ke database production).
3. Kartu "Murid Belum Hadir" — tombol download **PDF**, data tersorting Kelas lalu Nama A-Z.

## Konteks — Dikonfirmasi via Riset 2026-08-28

### 1. Cetak Surat Masuk — root cause PASTI
`piket-board-view.tsx` baris ~505-513 — tombol cuma bersyarat `row.status === "terlambat" && onDuty`, TIDAK ADA pengecekan apakah `LateEntrySlip` sudah pernah dibuat untuk siswa+hari itu. `onPrinted={() => setLateEntryTarget(null)}` (baris ~554) CUMA menutup dialog, TIDAK update state `board` sama sekali. `PiketBoardRow` (tipe response backend) TIDAK PUNYA field apa pun yang menandakan "sudah dicetak" — root cause di backend (data tidak dikirim), bukan cuma frontend.

### 2. Tidak Absen Pulang — tanggal TIDAK ditampilkan (murni tambahan, tidak ada bug)
`TidakAbsenPulangRow` (`core-types.ts:805-812`) **SUDAH PUNYA field `tanggal`** dikirim
backend (`attendance.service.ts:838`, `tidakAbsenPulangKemarin()`) — CUMA TIDAK DIRENDER di
tabel (`piket-board-view.tsx` baris ~987-1007, kolom cuma No/Nama/Kelas/Jam Masuk/Aksi).
Menambah kolom ini MURAH (data sudah ada, tinggal render).

**Soal laporan "siswa Senin masih muncul Rabu"**: user SUDAH cek langsung ke database
production (2026-08-28) — data AMAN, tidak ada baris/query bermasalah. Disimpulkan itu
salah baca petugas piket (kemungkinan besar melihat kasus BARU Selasa/Rabu untuk siswa yang
KEBETULAN sama, dikira "kasus lama yang sama" karena TIDAK ADA kolom Tanggal untuk
membedakan — persis yang diperbaiki di poin ini). **TIDAK ADA fix logic/query yang perlu
dikerjakan** — cukup tambah kolom Tanggal, itu SEKALIGUS jadi pencegahan supaya salah baca
serupa tidak terulang di masa depan.

### 3. Murid Belum Hadir — kolom sudah jelas, tinggal tambah download
`piket-board-view.tsx` baris ~443-465 (section `openSection === "board"`) — kolom existing:
No, Nama, Kelas, Status, Keterangan, Jam Masuk, Jam Pulang. `boardRows` (state yang sudah
di-filter+sort untuk tampilan) sudah tersedia di scope komponen ini.

## Spec Detail

### 1. Cetak Surat Masuk — sembunyikan total setelah dicetak (keputusan user 2026-08-28)
- Backend: `PiketBoardRow` (endpoint `piket-board()`/`resolveStatusHarianSiswa()` di
  `attendance.service.ts`) — tambah field `suratMasukSudahDicetak: boolean`, dihitung dari
  `LateEntrySlip.findFirst({ where: { studentId, tanggal: today } })` (EXISTS check,
  REPLIKASI pola `permit?.alasanDetail` yang sudah dipakai fungsi yang sama untuk data lain
  di baris yang sama — 1 query tambahan per siswa terlambat, atau JOIN sekali di awal kalau
  mau optimal, VERIFIKASI SAAT IMPLEMENTASI pendekatan paling efisien tanpa N+1 query).
- Frontend: baris ~505 kondisi tombol jadi
  `row.status === "terlambat" && onDuty && !row.suratMasukSudahDicetak`.
- `onPrinted` callback (baris ~554) — SELAIN `setLateEntryTarget(null)`, JUGA update state
  `board` lokal: cari baris `studentId` yang sama, set `suratMasukSudahDicetak: true` —
  supaya tombol hilang SEKETIKA tanpa perlu re-fetch penuh (pola optimistic update, REUSE
  cara `handlePermitSubmitted`/handler serupa lain di file ini melakukan hal sama).

### 2. Tidak Absen Pulang — tambah kolom Tanggal
`piket-board-view.tsx`, `TidakAbsenPulangSection` — tambah `<TableHead>Tanggal</TableHead>`
setelah kolom Nama (atau posisi lain yang masuk akal, VERIFIKASI SAAT IMPLEMENTASI), render
`row.tanggal` — REUSE helper format tanggal yang sudah ada di file lain kalau ada
(`formatTanggalRiwayat` di `riwayat-catatan-table.tsx` polanya bisa direplikasi, JANGAN
`toLocaleDateString` mentah tanpa format konsisten).

### 3. Murid Belum Hadir — tombol download PDF (keputusan user 2026-08-28)
Tambah 1 tombol "Download PDF" di dekat search box section ini (baris ~437-442) — REPLIKASI
visual tombol Download PDF yang sudah ada di halaman Rekap (`rekap-view.tsx`, variant
`destructive`, `text-white`, rounded-full) untuk konsistensi, tapi endpoint/generator PDF-nya
BARU (konten beda total dari Rekap — ini snapshot "murid belum hadir hari ini", bukan
rekap periode). REUSE library PDF generator yang SUDAH DIPAKAI project ini untuk export
rekap (VERIFIKASI SAAT IMPLEMENTASI library persis apa yang dipakai
`attendance-report-export.service.ts`, pakai yang SAMA — jangan tambah dependency PDF baru
kalau yang existing sudah cukup).

**Data yang didownload = persis isi card "Murid Belum Hadir" saat itu** (keputusan user) —
artinya ikut filter search yang sedang aktif di search box section ini (`boardSearch`),
SUMBER data = `boardRows` yang sedang match filter itu (state yang sama dipakai tabel di
layar, BUKAN fetch ulang terpisah).

**Sorting hasil download WAJIB beda dari sorting layar**: Kelas (A-Z) dulu, DALAM tiap kelas
baru Nama (A-Z) — TIDAK ikut `boardSort` yang mungkin sedang di-set user ke kolom lain di
layar (misal user sedang sort by Jam Masuk di layar, PDF tetap Kelas→Nama).

Kolom di PDF: SAMA seperti kolom tabel (No, Nama, Kelas, Status, Keterangan, Jam Masuk, Jam
Pulang) — nomor urut DIHITUNG ULANG sesuai urutan Kelas→Nama hasil download (bukan mengikuti
nomor urut tampilan layar).

## Edge Cases
- **Siswa terlambat lalu statusnya berubah lagi jadi kategori lain di hari yang sama**
  (jarang tapi VERIFIKASI: mungkinkah re-evaluasi status mengubah `row.status` dari
  "terlambat" ke sesuatu lain?) — kalau `status !== "terlambat"` tombol otomatis tidak
  tampil sama sekali (kondisi existing sudah benar untuk ini), TIDAK PERLU logic tambahan.
- **Section "Murid Belum Hadir" kosong** (semua sudah hadir) — tombol download
  disabled/disembunyikan, JANGAN biarkan generate file kosong tanpa penjelasan.
- **Card "Murid Belum Hadir" kosong** (semua sudah hadir, atau search aktif tidak match
  siapa pun) — tombol download disabled/disembunyikan, JANGAN generate PDF kosong tanpa
  penjelasan.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (tambah field
  `suratMasukSudahDicetak` ke response piket board).
- **Modifikasi:** `apps/web/src/lib/core-types.ts` (`PiketBoardRow` tambah field baru).
- **Modifikasi:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (3 poin — kondisi
  tombol cetak, kolom tanggal, tombol download PDF).
- **Buat/Modifikasi:** endpoint+generator PDF baru untuk "Murid Belum Hadir" (backend,
  lokasi VERIFIKASI SAAT IMPLEMENTASI — kemungkinan `apps/api/src/attendance/`, REUSE
  library PDF existing).

## Acceptance Criteria
- [x] Tombol "Cetak Surat Masuk" hilang total dari layar SEKETIKA setelah surat dicetak
      (tanpa perlu reload halaman) — `suratMasukSudahDicetak` dari backend (join, bukan
      N+1) + optimistic update `board` state di `handleLateEntryPrinted()`.
- [x] Kolom Tanggal tampil di kartu "Tidak Absen Pulang" — sortable (konsisten aturan
      wajib tabel data project), format via `formatTanggal()` (REPLIKASI
      `formatTanggalRiwayat()`).
- [x] Tombol "Download PDF" di "Murid Belum Hadir" hasilkan PDF berisi persis data yang
      sedang match filter search aktif di card itu (`searchedBoard`, BUKAN `boardRows`
      yang sudah ikut sort layar), tersorting Kelas→Nama A-Z via `sortForExport()`
      independen dari `boardSort`, nomor urut dihitung ulang sesuai urutan file
      (`index+1` di HTML generator, bukan index tampilan layar).
- [x] Build + type-check hijau (`tsc --noEmit` api+web bersih, `next build` web sukses).

## Validasi Claudian
- [x] Konfirmasi `suratMasukSudahDicetak` TIDAK N+1 — join lewat `include.lateEntrySlips`
      DI QUERY UTAMA `students.findMany()` (1 query, REPLIKASI pola exact
      `findTerlambatHariIni()` yang sudah pakai `lateEntrySlips: {none:{...}}` di where
      clause yang sama) — BUKAN `findFirst()` terpisah per baris siswa terlambat.
- [x] Konfirmasi sorting PDF Kelas dulu baru Nama — `sortForExport()` `localeCompare`
      kelas dulu (return kalau beda), fallback nama kalau kelas sama — logic dibaca
      manual, benar secara struktur. Test LIVE dengan data ≥2 kelas TIDAK BISA dilakukan
      di sesi ini (lihat Implementasi — Chrome/Puppeteer tidak tersedia di mesin dev
      Windows ini, generatePdf() akan throw "Tidak ditemukan Chrome/Chromium" persis
      sama seperti export rekap admin existing yang JUGA tidak bisa ditest live di sini).
- [x] Konfirmasi PDF mencerminkan filter search aktif — `handleExportBelumHadirPdf()`
      SECARA STRUKTURAL sumbernya `searchedBoard` (state yang sama dipakai tabel di
      layar, sudah difilter `boardSearch`), TIDAK ADA fetch/query ulang ke server sama
      sekali — dijamin oleh desain (endpoint backend murni format, tidak re-query DB).

## Implementasi (2026-08-28)

**Poin 1 (Cetak Surat Masuk)**: `resolveStatusHarianSiswa()` (`attendance.service.ts`)
tambah `include.lateEntrySlips: {where:{tanggal:today}, take:1}` (1 join, bukan N+1) +
field baru `suratMasukSudahDicetak: student.lateEntrySlips.length > 0` di `PiketBoardRow`.
Frontend: kondisi tombol tambah `&& !row.suratMasukSudahDicetak`; `onPrinted` callback
diganti `handleLateEntryPrinted(studentId)` yang update `board` state lokal (root cause
lama: `onPrinted` sebelumnya CUMA `setLateEntryTarget(null)`, tidak pernah sentuh `board`).

**Poin 2 (Tanggal)**: kolom baru di `TidakAbsenPulangSection` — `TidakAbsenPulangRow.tanggal`
SUDAH ADA dari backend sejak awal (dikonfirmasi riset, TIDAK ADA bug data — investigasi
mendalam DIBATALKAN sesuai revisi di kepala file ini), murni render + sortable.

**Poin 3 (Download PDF)**: Backend — `AttendanceReportExportService.generateMuridBelumHadirPdf()`
BARU, REUSE `getBrowser()` (1 instance Chrome shared, TIDAK spawn browser kedua),
HTML sederhana tanpa Chart.js (murni tabel + kop surat, beda dari `generatePdf()` rekap
yang py grafik). Endpoint baru `POST /attendance/piket-board/export-belum-hadir.pdf`
(role `guru_piket`) terima `rows` di BODY (bukan query — bisa banyak baris) — DTO
`ExportBelumHadirDto`/`MuridBelumHadirRowDto` (`class-validator`, array nested).
**Desain kunci**: endpoint backend MURNI FORMAT (tidak re-query database sama sekali) —
`rows` datang APA ADANYA dari frontend (nama/kelas/status-label/keterangan/jam SUDAH
final dari `boardRows`/`searchedBoard` di layar), pola SAMA `print/struk-izin`/
`print/surat-masuk-kelas` (data display final dari client, endpoint cuma render dokumen)
— JAMIN PDF 100% identik dengan yang piket lihat di layar, termasuk filter search aktif,
tanpa risiko race condition antara state client dan re-query server terpisah.

Frontend: helper baru `resolveStatusLabelForExport()` (REPLIKASI persis logic prioritas
`StatusBadge`: Terkunci > Belum Memiliki Kartu > kategoriLive) dan
`resolveKeteranganForExport()` (SAMA logic kolom Keterangan tampilan) supaya PDF tidak
pernah menampilkan status berbeda dari layar. `sortForExport()` Kelas→Nama independen dari
`boardSort`. Sumber data `searchedBoard` (SETELAH search, SEBELUM sort layar) — BUKAN
`boardRows`. Tombol disabled kalau `searchedBoard.length === 0` (edge case spec).

**Infrastruktur baru**: proxy Next.js `/api/proxy-download/[...path]/route.ts` — GET
existing (rekap PDF/Excel via query string) TIDAK CUKUP untuk kasus ini (payload array,
butuh body) — tambah handler `POST` BARU di file yang sama, forward body JSON + baca
balik response binary, logic identik GET selain method+body.

**Bug regresi ditemukan+diperbaiki**: `attendance.service.spec.ts` `STUDENT_FIXTURE()`
(dipakai `describe("AttendanceService.piketBoard — T146 kategoriLive")`) tidak punya
field `lateEntrySlips` — `student.lateEntrySlips.length` akan throw `undefined.length`
begitu `resolveStatusHarianSiswa()` diubah. Ditemukan SEBELUM jalankan test (baca kode
fixture dulu), diperbaiki tambah `lateEntrySlips: []` ke fixture — 55/55 test
`attendance.service.spec.ts` lulus (termasuk seluruh grup `piketBoard`/`kategoriLive`
yang paling berisiko kena regresi ini), TANPA test baru ditambahkan.

**Verifikasi**: `tsc --noEmit` api+web bersih, `next build` web sukses, `jest
attendance.service.spec.ts` 55/55 lulus. Live curl: route `POST
/attendance/piket-board/export-belum-hadir.pdf` terdaftar+guard `guru_piket` bekerja
(403 utk role lain, bukan 404). **Keterbatasan**: generate PDF SUNGGUHAN (Puppeteer)
TIDAK BISA ditest live di mesin dev Windows ini — `CHROME_CANDIDATES` di
`attendance-report-export.service.ts` HANYA path Linux (`/usr/bin/google-chrome-stable`
dkk), SAMA SEKALI TIDAK ADA di Windows — ini KETERBATASAN EXISTING (export rekap admin
yang sudah lama ada JUGA tidak pernah bisa ditest live di mesin dev ini), BUKAN sesuatu
yang baru diperkenalkan task ini. Fitur akan berfungsi normal di production (Linux, Chrome
terpasang) — TAPI belum pernah dilihat hasil PDF sungguhan sama sekali, REKOMENDASI kuat
user test manual di production/staging sebelum dianggap benar-benar selesai secara visual.

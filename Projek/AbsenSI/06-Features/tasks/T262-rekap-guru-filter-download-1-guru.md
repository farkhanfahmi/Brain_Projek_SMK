# T262 — Web+API: Rekap Kehadiran Guru — Filter & Download Rentang Tanggal untuk 1 Guru Tertentu

## Depends on
Tidak ada. Independen — halaman `rekap-guru-view.tsx` terpisah dari rekap murid (T243).

## Objective
Admin bisa memfilter DAN mendownload (PDF/Excel) rekap kehadiran guru untuk **rentang
tanggal tertentu, guru tertentu SAJA** — sekarang tidak bisa sama sekali.

## Konteks — Root Cause Dikonfirmasi via Riset 2026-08-28

**Ini BUKAN cuma bug UI — backend genuinely tidak punya kapasitas filter per-guru sama
sekali.** `TeacherReportQueryDto` (`apps/api/src/teacher-attendance-report/dto/
teacher-report-query.dto.ts`) HANYA terima `from`/`to`/`statusKepegawaian` — TIDAK ADA
`teacherId`/`niy`/`search`. `TeacherAttendanceReportService` query daftar guru
(`prisma.teacher.findMany(...)`, dilihat dari pemakaian `teachers.map(t => t.id)` di banyak
tempat) TIDAK PERNAH di-filter ke 1 guru spesifik — SELALU proses SEMUA guru yang cocok
`statusKepegawaian`.

**Di frontend**, `rekap-guru-view.tsx` cuma punya `namaSearch` (Input teks bebas, baris 136)
— MURNI client-side filter tampilan tabel (`singleDaySearchedRows`/pola serupa untuk range,
baris ~398-401). `handleExport()` (baris 346-385) build `params` LEWAT `buildParams()`
(baris 296-310) yang **TIDAK PERNAH menyertakan `namaSearch`** — jadi download SELALU
берisi SEMUA guru, terlepas apa yang sedang diketik di search box. Pola root cause ini
IDENTIK dengan yang sudah diperbaiki T243 (rekap murid)/T254 (Murid Belum Hadir) — "apa
yang di layar ≠ apa yang didownload".

## Spec Detail

### 1. Backend — tambah filter `teacherId` (bukan cuma search text)
`TeacherReportQueryDto` + `TeacherReportExportQueryDto` — tambah field opsional
`teacherId?: number` (VERIFIKASI SAAT IMPLEMENTASI — REKOMENDASI `teacherId` presisi
bukan `search` text bebas, supaya "guru tertentu" benar-benar 1 guru pasti, bukan
tergantung ketepatan ketik nama yang bisa ambigu/multi-match). `TeacherAttendanceReportService`
— method yang query daftar `teachers` (baris ~97, ~173-an, dan versi single-day) TAMBAH
`id: teacherId` ke `where` clause KALAU `teacherId` dikirim (kondisional, default TETAP
semua guru kalau parameter ini tidak diisi — behavior existing untuk "lihat semua guru"
TIDAK BERUBAH).

### 2. Frontend — dropdown pilih guru (BUKAN cuma text search bebas)
`rekap-guru-view.tsx` — tambah 1 filter dropdown **"Pilih Guru"** (opsional, default "Semua
Guru") DI SAMPING filter existing (statusKepegawaian, tanggal) — REKOMENDASI: dropdown
dengan search-di-dalamnya (Command/Combobox pattern kalau sudah ada preseden di codebase,
VERIFIKASI SAAT IMPLEMENTASI — kalau tidak ada, `Select` biasa dengan daftar guru cukup
untuk skala sekolah ini, TIDAK PERLU komponen baru kompleks kalau jumlah guru masih
manageable di 1 dropdown).

`namaSearch` (Input teks bebas existing) — VERIFIKASI SAAT IMPLEMENTASI: PERTAHANKAN untuk
quick-filter tampilan cepat (use case beda dari dropdown presisi), TAPI dropdown "Pilih
Guru" adalah mekanisme UTAMA untuk kasus "download rekap 1 guru pasti" yang diminta user.

`buildParams()` — tambah `teacherId` ke params KALAU dropdown terisi (bukan "Semua Guru") —
otomatis ikut baik ke fetch tampilan MAUPUN `handleExport()` (SATU fungsi yang sama dipakai
keduanya, jadi begitu `teacherId` masuk `buildParams()`, download OTOMATIS ikut ter-filter
tanpa perlu logic terpisah).

### 3. Nama file download mencerminkan filter guru
`handleExport()` baris ~375 (`a.download = ...`) — kalau `teacherId` aktif, sertakan nama
guru di nama file (mis. `rekap-guru-Budi-Santoso-2026-08-01-2026-08-28.pdf`) — REKOMENDASI,
bukan wajib mutlak, tapi memudahkan admin yang download banyak file guru berbeda.

## Edge Cases
- **Dropdown "Pilih Guru" + `namaSearch` text AKTIF bersamaan** — VERIFIKASI SAAT
  IMPLEMENTASI interaksi keduanya (REKOMENDASI: `teacherId` dropdown adalah filter PASTI
  di level backend/query, `namaSearch` cuma quick-filter tampilan tambahan DI ATAS hasil
  yang sudah ter-filter `teacherId` — keduanya bisa aktif bersamaan tanpa konflik logis).
- **Guru dipilih tapi tidak py data di rentang tanggal itu** (belum pernah tap/tidak ada
  jadwal) — tabel+export tetap tampil 1 baris guru itu dengan angka 0/kosong, BUKAN
  kosong total tanpa penjelasan (guru yang dipilih harus tetap "ada" di hasil, meski nihil).
- **Filter `statusKepegawaian` (guru/karyawan) + `teacherId` guru dari kepegawaian BEDA**
  (kombinasi tidak masuk akal, mis. pilih guru tapi filter kepegawaian "karyawan") —
  REKOMENDASI: `teacherId` dropdown MENANG (tampilkan guru yang dipilih apa pun filter
  kepegawaian aktif), ATAU dropdown guru otomatis ter-filter sesuai kepegawaian yang aktif
  (VERIFIKASI SAAT IMPLEMENTASI mana yang lebih masuk akal UX-nya).

## Files
- **Modifikasi:** `apps/api/src/teacher-attendance-report/dto/teacher-report-query.dto.ts`
  (+`teacher-report-export-query.dto.ts` yang extends itu).
- **Modifikasi:** `apps/api/src/teacher-attendance-report/teacher-attendance-report.service.ts`
  (filter `teacherId` di query guru).
- **Modifikasi:** `apps/web/src/app/(admin)/rekap-guru/rekap-guru-view.tsx` (dropdown Pilih
  Guru, `buildParams()` sertakan `teacherId`).

## Acceptance Criteria
- [x] Admin bisa pilih 1 guru spesifik dari dropdown, tabel di layar cuma tampilkan guru itu.
- [x] Download PDF/Excel dengan guru terpilih HANYA berisi data guru itu (bukan semua guru).
- [x] Tanpa memilih guru (default) — behavior SAMA seperti sekarang (semua guru), regresi
      check.
- [x] Rentang tanggal + guru terpilih bisa dikombinasikan bebas (semua kombinasi filter
      existing tetap bekerja bersamaan dengan filter guru baru).
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi filter `teacherId` diterapkan di QUERY BACKEND (bukan cuma filter
      client-side hasil yang sudah di-fetch penuh) — supaya performa tetap baik untuk
      sekolah dengan banyak guru, bukan fetch semua lalu buang di frontend.
- [x] Konfirmasi download benar-benar ikut ter-filter — SECARA LOGIKA (lihat Implementasi):
      `buildParams()` yang sama dipakai fetch tampilan MAUPUN `handleExport()`, `teacherId`
      diteruskan lewat inheritance `TeacherReportExportQueryDto extends TeacherReportQueryDto`
      ke `reportFlexible()` yang SAMA persis dipanggil endpoint export — belum test manual
      buka file hasil download (lihat catatan "Belum diverifikasi live").
- [x] Konfirmasi tidak ada regresi ke mode "semua guru" (default, tanpa filter) — 10/10 test
      existing `teacher-attendance-report.service.spec.ts` tetap lulus tanpa perubahan
      assertion.

## Implementasi (2026-08-29)

**Backend:**
- `TeacherReportQueryDto` — tambah `teacherId?: number` opsional (`@Type(() => Number)
  @IsInt()`), otomatis diwarisi `TeacherReportExportQueryDto` (extends) — TIDAK perlu ubah
  export DTO/controller/export-service sama sekali, `teacherId` ikut mengalir ke
  `reportFlexible()` yang dipanggil `exportPdf()`/`exportExcel()`.
- `TeacherAttendanceReportService.reportSingleDay()` dan `.reportRange()` — kedua query
  `prisma.teacher.findMany()` ditambah `id: query.teacherId` di `where`. Keputusan edge case
  kombinasi `teacherId`+`statusKepegawaian` tidak sinkron (spec §Edge Cases): **`teacherId`
  MENANG** — kalau `teacherId` terisi, `statusKepegawaian` di `where` di-set `undefined`
  (tidak difilter), dan guard "karyawan dikecualikan total di mode Per Hari" (baris ~82)
  di-skip kalau `teacherId` aktif — supaya guru manapun yang dipilih tetap bisa muncul
  apa pun status kepegawaian filter yang sedang aktif.
- Edge case "guru dipilih tapi tidak ada data di rentang" — SUDAH otomatis benar untuk mode
  **Per Rentang** tanpa perubahan tambahan (`byTeacher` map di-inisialisasi utk semua
  `teacherIds` termasuk yang dipilih, row tetap muncul dengan angka 0). Mode **Per Hari**
  SENGAJA TIDAK diubah untuk edge case ini — filter "hanya guru yang py jadwal mengajar
  hari itu" adalah desain INTI mode tersebut (bukan bug), disengaja sejak awal (lihat
  komentar baris ~72-77), di luar scope task ini yang murni soal filter identitas guru.

**Frontend** (`rekap-guru-view.tsx`):
- State baru `teachers` (fetch `GET /teachers` sekali saat mount, tanpa query param =
  seluruh guru+karyawan aktif) + `selectedTeacherId` (default `"__all__"`).
- Dropdown baru **"Pilih Guru"** (`Select` dari `@absensi/ui`, style `h-10 rounded-full`
  KONSISTEN filter lain di halaman ini) diletakkan di antara search box `namaSearch` dan
  filter pill Kepegawaian — REKOMENDASI spec (Select biasa, bukan Combobox baru) dipakai
  karena skala guru 1 sekolah masih manageable di 1 dropdown.
- `buildParams()` — tambah `teacherId` ke params kalau dropdown bukan "Semua Guru" — SATU
  fungsi yang sama dipakai `useEffect` fetch tampilan DAN `handleExport()`, jadi filter
  OTOMATIS ikut ke keduanya tanpa logic terpisah (sesuai spec §2).
  `useEffect` fetch data — tambah `selectedTeacherId` ke dependency array.
- `buildFilterLabel()` — sertakan nama guru terpilih (menggantikan label kepegawaian kalau
  keduanya aktif bersamaan, konsisten keputusan "teacherId menang").
- `handleExport()` — nama file download disisipi slug nama guru kalau filter aktif
  (`rekap-kehadiran-guru-Budi-Santoso-2026-08-01-2026-08-28.pdf`), REKOMENDASI spec (bukan
  wajib) — diimplementasikan karena tidak menambah kompleksitas.
- `namaSearch` (Input teks bebas) — DIPERTAHANKAN apa adanya, tetap quick-filter tampilan
  client-side terpisah, tidak diubah perannya.

**Verifikasi:**
- `tsc --noEmit` api+web — bersih, tanpa error.
- `next build` web — sukses penuh (exit 0), route `/rekap-guru` naik sedikit (7.17→7.38 kB,
  konsisten penambahan 1 dropdown).
- `jest teacher-attendance-report.service.spec.ts` — 10/10 tetap lulus (dijalankan dengan
  `NODE_OPTIONS=--max-old-space-size=3072` karena OOM di run pertama tanpa heap limit —
  pola resource-contention yang sama berulang di mesin dev ini, bukan regresi kode). Test
  existing pakai `expect.objectContaining` untuk assert `where` clause `teacher.findMany`,
  jadi tidak pecah oleh field `id` baru yang ditambahkan.
- **Belum diverifikasi live** (DB dev naik-turun sepanjang sesi): pilih 1 guru sungguhan di
  browser, download PDF/Excel, buka filenya untuk konfirmasi isinya benar cuma 1 guru itu
  (bukan cuma percaya nama file/logika kode).

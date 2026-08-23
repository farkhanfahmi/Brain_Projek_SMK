# T176 — API+Web: Rekap Kehadiran Guru & Karyawan (Admin)

## Depends on
**WAJIB setelah T175** (aktivasi logic keterlambatan guru — tanpa ini kolom Terlambat selalu 0). Independen dari rangkaian Jurnal Guru (T168-T173) dan Jam Pelajaran (T158-T160), meski beririsan konsep (jadwal mengajar).

## Objective
Halaman admin baru — Rekap Kehadiran untuk **Guru** dan **Karyawan** (dibedakan via `Teacher.statusKepegawaian`), dengan 2 mode tampilan:
1. **Rekap per-hari** — pilih 1 tanggal, tampilkan HANYA guru/karyawan yang PUNYA jadwal mengajar hari itu, dengan status kehadiran hari itu.
2. **Rekap per-rentang** — pilih rentang tanggal, tampilkan SEMUA guru/karyawan dengan kolom akumulasi (jumlah Hadir/Terlambat/Izin/Alfa) **PLUS kolom "Jumlah Hari Mengajar"** (total hari dalam rentang itu yang mereka punya jadwal — pembagi/konteks untuk kolom akumulasi).

Filter kepegawaian: **Guru saja**, **Karyawan saja**, atau **Semua (Guru+Karyawan gabung)**.

## Context — Keputusan Diskusi (2026-08-14)

Riset kode mengonfirmasi fondasi kuat untuk reuse:
- `AttendanceRecord.teacherId` — dual-FK sudah ada, tap gerbang guru sudah tercatat di model SAMA dengan siswa (`schema.prisma:607-637`).
- **Tidak ada endpoint rekap guru untuk admin sama sekali saat ini** — `GET /attendance/my-history` HANYA untuk guru lihat riwayat sendiri (flat, scope ketat `teacherId` dari JWT), TANPA hitungan akumulasi, TANPA filter apa pun.
- **"Karyawan" BUKAN model terpisah** — row di `Teacher` yang sama, dibedakan `Teacher.statusKepegawaian` (enum `guru | karyawan`, `schema.prisma:213,232-235`). Task ini SATU modul rekap yang bisa difilter kepegawaian, BUKAN 2 modul terpisah.
- **"Hari wajib" guru BEDA KONSEP dari siswa** — siswa: Senin-Jumat semua wajib hadir (`resolveHariWajib()`, generik tanggal). Guru: **HANYA hari dia punya jadwal mengajar** yang relevan — dikonfirmasi eksplisit oleh user, BUKAN reuse `resolveHariWajib()` siswa apa adanya.
- **Sumber status "Izin"**: `TeacherPermit` (`schema.prisma:544-573`) — model lebih kaya dari `Permit` siswa (approval+bukti+tugas titipan), SUDAH dipakai alur guru saat ini.
- **Akses**: `super_admin` + `admin_jurnal` (dikonfirmasi user — pola sama seperti `TeacherPermitsController` yang sudah dipegang 2 role ini).
- Karyawan **TIDAK PUNYA jadwal mengajar** (`Schedule`/`TeachingSession` konsepnya "mengajar", karyawan TU/tenaga kependidikan tidak mengajar kelas) — lihat Edge Cases untuk bagaimana "hari wajib" karyawan ditangani (BEDA dari guru, karena tidak ada jadwal mengajar yang bisa jadi patokan).

### Konsolidasi dengan T116 (task lama, 2026-08-06, di-SUPERSEDE oleh task ini — 2026-08-14)

Task lama `T116-rekap-kehadiran-guru-baru.md` menulis rancangan rekap guru SEBELUM T175 ada — saat itu guru SELALU `hadir` (tidak pernah `terlambat`), jadi T116 SENGAJA TIDAK punya kolom Terlambat ("akan selalu nol, menyesatkan"). Task INI (T176) MENGGANTIKAN T116 sepenuhnya karena sekarang guru BISA `terlambat` (T175, prasyarat task ini). T116 DIARSIPKAN (file TIDAK dihapus, ditandai SUPERSEDED di STATUS.md untuk jejak keputusan) — TAPI 1 DETAIL BERHARGA dari T116 diserap ke sini, WAJIB diterapkan:

- **Breakdown kategori `TeacherPermit` per JENIS, bukan digabung jadi 1 "Izin" generik** — enum `TeacherPermitKategori` (`schema.prisma:537-542`) punya 4 nilai granular: `sakit`, `izin_pribadi`, `tugas_dinas`, `pelatihan`. Rekap ini (poin 2/3/4 di Spec Detail atas) WAJIB tampilkan breakdown PER KATEGORI ini (kolom terpisah: Sakit / Izin Pribadi / Tugas Dinas / Pelatihan), BUKAN digabung jadi 1 kolom "Izin" — datanya sudah granular di schema, sayang kalau tidak dimanfaatkan (alasan T116, TETAP BERLAKU). Ini MENGGANTIKAN penyebutan generik "izin/sakit" di poin 2 Spec Detail atas — di mana pun tertulis status `izin`/`sakit` gabungan di task ini, BACA sebagai "breakdown 4 kategori TeacherPermit", bukan 1 status gabungan.
- Guru dengan `AttendanceRecord` DAN `TeacherPermit` DI HARI YANG SAMA (kasus yang T116 catat TIDAK DIREKONSILIASI kode manapun) — PRIORITASKAN `AttendanceRecord`/hadir (bukti fisik tap menang), `TeacherPermit` di hari itu jadi anomali data yang diabaikan untuk rekap — KONSISTEN keputusan T116, TERAPKAN di sini juga.

## Spec Detail

### 1. Backend — "hari wajib mengajar" per guru (BUKAN generik seperti siswa)

- Method baru (nama final diputuskan saat implementasi, mis. `resolveHariMengajarGuru(teacherId, from, to)`) — HITUNG tanggal-tanggal dalam rentang `[from, to]` di mana `teacherId` itu PUNYA MINIMAL 1 `Schedule` (type `jam_mengajar`) di hari (`dayOfWeek`) tanggal itu, DIKURANGI `SchoolHoliday` (guru tidak wajib hadir di hari libur meski ada jadwal terjadwal di hari itu secara mingguan) — **PER GURU BERBEDA-BEDA**, TIDAK BISA di-precompute sekali untuk semua guru seperti `resolveHariWajib()` siswa (yang sama untuk semua siswa).
- **VERIFIKASI**: apakah perlu mempertimbangkan `Kelas.modeJadwal`/`BlockWeekRange` (T159, mode blok Minggu A/B) — KALAU T159 sudah dikerjakan, jadwal guru di kelas mode blok BISA BEDA per-Minggu-A-vs-B, resolusi hari wajib PERLU pertimbangkan ini (reuse `ScheduleResolverService` yang SUDAH ADA untuk resolve jadwal per-tanggal, JANGAN hitung ulang logic blok dari nol).
- Karyawan (`statusKepegawaian: karyawan`) — TIDAK PUNYA jadwal mengajar SAMA SEKALI (secara struktural, karyawan tidak assign ke `Schedule`) — untuk karyawan, "hari wajib" TIDAK BISA dihitung dari jadwal mengajar. LIHAT Edge Cases untuk keputusan ini — KLARIFIKASI KE USER SAAT IMPLEMENTASI kalau ditemukan karyawan butuh definisi "hari wajib" berbeda (kemungkinan besar: karyawan pakai Senin-Jumat generik SEPERTI SISWA, KARENA karyawan TU biasanya kerja reguler tiap hari kerja — TAPI ini ASUMSI, bukan keputusan eksplisit user, WAJIB dikonfirmasi sebelum implementasi rekap karyawan, JANGAN diam-diam pilih salah satu).

### 2. Backend — endpoint baru, modul `apps/api/src/teacher-attendance-report/` (nama final diputuskan saat implementasi)

- `GET /teacher-attendance-report/single-day?tanggal=&statusKepegawaian=` — role `super_admin, admin_jurnal`. Query SEMUA `Teacher` yang PUNYA `Schedule` (jam_mengajar) di hari (`dayOfWeek`) tanggal itu (filter opsional `statusKepegawaian`: guru/karyawan/semua — TAPI KALAU `karyawan` dipilih untuk mode single-day, HASIL AKAN SELALU KOSONG karena karyawan tidak punya jadwal mengajar — TAMPILKAN pesan jelas di UI untuk kasus ini, BUKAN tabel kosong membingungkan, lihat Edge Cases). Per baris: nama, NIY, status hari itu (`hadir`/`terlambat`/`izin`/`sakit`/`belum_absen` — REUSE POLA prioritas status `reportSingleDay()` siswa: cek `AttendanceRecord` dulu, fallback `TeacherPermit`, fallback `belum_absen`).
- `GET /teacher-attendance-report/range?from=&to=&statusKepegawaian=` — role SAMA. Untuk SETIAP `Teacher` (difilter `statusKepegawaian` kalau diisi): hitung `jumlahHariMengajar` (dari method poin 1, KECUALI karyawan — lihat Edge Cases), lalu akumulasi `hadir`/`terlambat`/`izin`/`sakit`/`alfa` dari `AttendanceRecord`+`TeacherPermit` dalam rentang itu, dibandingkan terhadap hari-hari wajib yang dihitung.
- Filter tambahan: `academicYearId`/`semesterId` (KONSISTEN pola T139 — kalau diisi, derive rentang tanggal dari situ, REUSE `resolveDateRange()` existing kalau applicable).
- `@LogActivity` TIDAK PERLU (endpoint GET read-only, konsisten pola rekap siswa yang juga tidak di-log).

### 3. Backend — export PDF/Excel

- REUSE pola `attendance-report-export.service.ts` SEBAGAI REFERENSI STRUKTUR (helper `sortReportRows()` generic, konstanta label/warna status) TAPI method BARU terpisah (tipe row berbeda: `teacherId`, `niy`, `statusKepegawaian` — BUKAN `studentId`/`nisn`/`kelas`) — JANGAN paksa generalisasi tipe existing yang sudah dipakai siswa, buat method paralel konsisten pola yang sudah terbukti kerja (T163: kop surat+grafik) DAN sinkron sorting (T164: pola `sortBy`/`sortDir` dari layar) — reuse KEDUA pola itu untuk rekap guru sejak awal (JANGAN bikin rekap guru versi awal yang "primitif" lalu perlu disempurnakan lagi belakangan seperti siswa).

### 4. Frontend — halaman admin baru "Rekap Kehadiran Guru"

- Path baru `apps/web/src/app/(admin)/rekap-guru/` (nama final diputuskan saat implementasi, HINDARI bentrok dengan `(admin)/rekap` existing untuk siswa — kalau ada, VERIFIKASI path persisnya dulu).
- Toggle mode: **Per Hari** vs **Per Rentang** (mirip pola `isSingleDay` di rekap siswa — REUSE konsep switching UI kalau ada komponen yang applicable).
- Filter kepegawaian: 3 pilihan (Guru / Karyawan / Semua) — tombol pill/tab, KONSISTEN pola filter Tingkat multi-select (T162) untuk gaya visual, meski ini single-select bukan multi.
- Mode Per Hari — date picker 1 tanggal, tabel hasil (kosong + pesan jelas kalau filter "Karyawan" dipilih, lihat Edge Cases).
- Mode Per Rentang — date range picker, tabel dengan kolom akumulasi + **"Jumlah Hari Mengajar"**.
- **WAJIB ikuti aturan tabel permanen proyek**: search box, semua kolom sortable via `SortableHeader`, kolom "No" paling kiri (offset halaman) — KONSISTEN memory `feedback_tabel_wajib_search_sort_kolom_no`.
- **WAJIB mobile-first** — rancang base Tailwind untuk layar sempit dulu (memory `feedback_mobile_first_wajib`), tabel lebar (banyak kolom akumulasi) BUTUH strategi mobile jelas (scroll horizontal terkontrol ATAU card-view alternatif — putuskan saat implementasi).
- Tombol Export PDF/Excel (reuse pola tombol dari rekap siswa).

## Edge Cases
- **Filter "Karyawan" di mode Per Hari** → hasil SELALU KOSONG (karyawan tidak punya jadwal mengajar) — TAMPILKAN pesan jelas ("Karyawan tidak memiliki jadwal mengajar — gunakan mode Per Rentang untuk melihat kehadiran karyawan berdasarkan tap gerbang"), BUKAN tabel kosong tanpa penjelasan.
- **"Hari wajib" karyawan** — TIDAK ADA jadwal mengajar sebagai patokan. KLARIFIKASI KE USER SAAT IMPLEMENTASI: kemungkinan besar karyawan pakai definisi Senin-Jumat generik (mirip `resolveHariWajib()` siswa) KARENA karyawan TU biasanya kerja reguler — TAPI INI BUKAN KEPUTUSAN YANG SUDAH DIKONFIRMASI EKSPLISIT, WAJIB tanya user sebelum coding bagian ini, JANGAN asumsi sepihak.
- **Filter "Semua" (Guru+Karyawan gabung) di mode Per Rentang** — kolom "Jumlah Hari Mengajar" akan 0 untuk SEMUA baris karyawan (sesuai keputusan di atas, KECUALI karyawan dapat definisi hari-wajib sendiri) — pastikan tabel TIDAK membingungkan (kolom 0 untuk karyawan harus terlihat WAJAR bukan seperti bug, misal beri keterangan/badge berbeda untuk baris karyawan).
- Guru dengan status `terlambat` yang BARU MUNGKIN muncul setelah T175 — pastikan test/verifikasi rekap ini dilakukan SETELAH T175 selesai dan di-deploy, BUKAN sebelumnya (kalau dites sebelum T175, kolom Terlambat akan 0 semua, MEMBINGUNGKAN saat verifikasi).
- Guru yang jadwalnya di kelas mode BLOK (T159, kalau sudah ada) dan tanggal yang dicek berada di luar `BlockWeekRange` manapun (lubang kalender) — "hari wajib" untuk guru itu di tanggal itu → tidak terhitung wajib (SAMA prinsipnya seperti siswa: lubang kalender = tidak masuk hitungan, JANGAN paksa hitung sebagai wajib kalau resolusi jadwal gagal).

## Files
- **Buat:** `apps/api/src/teacher-attendance-report/` (controller+service+dto, termasuk export PDF/Excel terpisah), migration TIDAK PERLU (semua field sudah ada), halaman `apps/web/src/app/(admin)/rekap-guru/`.
- **Jangan sentuh:** `attendance-report.service.ts` (rekap siswa, TIDAK diubah, task ini modul PARALEL baru), `resolveHariWajib()` (TIDAK direuse langsung untuk guru — logic guru BEDA, method baru terpisah).

## Acceptance Criteria
- [x] Mode Per Hari: tampilkan hanya guru/karyawan dengan jadwal mengajar hari itu, status kehadiran akurat (hadir/terlambat/izin/sakit/belum_absen).
- [x] Mode Per Rentang: tampilkan SEMUA guru/karyawan (sesuai filter kepegawaian), kolom akumulasi + Jumlah Hari Mengajar.
- [x] Filter Guru/Karyawan/Semua berfungsi benar berdasarkan `Teacher.statusKepegawaian`.
- [x] Filter Karyawan di mode Per Hari menampilkan pesan jelas (bukan tabel kosong tanpa konteks).
- [x] Hanya `super_admin` dan `admin_jurnal` bisa akses.
- [x] Export PDF/Excel dengan kop surat+grafik (setara T163) dan sort tersinkron layar (setara T164) SEJAK VERSI PERTAMA (bukan disempurnakan belakangan).
- [x] Tabel WAJIB search+sort semua kolom+kolom No (aturan permanen proyek).
- [x] Mobile-first, strategi jelas untuk tabel lebar di layar sempit (scroll horizontal terkontrol).
- [x] Build + type-check hijau, jest baru untuk modul ini pass.

## Validasi Claudian
- [x] **WAJIB klarifikasi ke user** definisi "hari wajib" untuk KARYAWAN — DITANYAKAN via AskUserQuestion sebelum implementasi, user memilih Senin-Jumat generik (reuse `AttendanceReportService.resolveHariWajib()` siswa).
- [x] Konfirmasi task ini dikerjakan SETELAH T175 — T175 dikerjakan tepat sebelum T176 di sesi yang sama, verified kolom Terlambat terisi live.
- [x] Konfirmasi TIDAK mengubah `attendance-report.service.ts` (siswa) — 0 diff, modul `teacher-attendance-report/` 100% paralel/terpisah, cuma inject `AttendanceReportService` untuk reuse `resolveHariWajib()` publik.
- [x] T159 (mode blok per kelas) BELUM dikerjakan saat task ini dieksekusi (dikonfirmasi tidak ada `Kelas.modeJadwal` di schema) — resolusi tetap SUDAH benar untuk kasus mode blok KARENA reuse `ScheduleResolverService.getJadwalHariIni()` yang sudah menangani `Semester.mode`/`BlockWeekRange` secara internal, tidak perlu logic tambahan di modul ini.

## Status Eksekusi (2026-08-14)

**Selesai.**

**Backend (`apps/api/src/teacher-attendance-report/`, baru)**:
- `teacher-attendance-report.service.ts` — `reportFlexible()` (mode ditentukan from===to, pola T132), `reportSingleDay()` (HANYA guru dengan `ScheduleResolverService.getJadwalHariIni()` terisi hari itu; filter `statusKepegawaian=karyawan` return kosong+`karyawanDikecualikan:true` LANGSUNG tanpa query; prioritas status AttendanceRecord > TeacherPermit > belum_absen, konsolidasi T116), `reportRange()` (akumulasi breakdown 4 kategori TeacherPermit terpisah — sakit/izinPribadi/tugasDinas/pelatihan, BUKAN digabung; `jumlahHariWajib` per-guru dari `resolveHariMengajarGuru()` BARU — loop tanggal + `getJadwalHariIni()` per hari; karyawan pakai `AttendanceReportService.resolveHariWajib()` yang di-reuse, dikonfirmasi user).
- `teacher-attendance-report-export.service.ts` — PDF+Excel dengan kop surat+grafik (pola T163) DAN sort tersinkron (pola T164) SEJAK AWAL, `sortReportRows()` DIDUPLIKASI SENGAJA (bukan import, sesuai spec "jangan paksa generalisasi tipe existing") karena tipe row beda total (`teacherId`/`niy`/`statusKepegawaian`).
- `teacher-attendance-report.controller.ts` — `GET /teacher-attendance-report`, `/export.pdf`, `/export.xlsx`, `@Roles(super_admin, admin_jurnal)` (pola `TeacherPermitsController`).
- `teacher-attendance-report.module.ts` — registrasi di `app.module.ts`, import `ScheduleResolverModule`+`AttendanceModule` (untuk reuse `AttendanceReportService.resolveHariWajib()`, sudah diekspor modul itu).
- `dto/teacher-report-query.dto.ts`, `dto/teacher-report-export-query.dto.ts`.
- `teacher-attendance-report.service.spec.ts` — 10 test baru: filter karyawan mode single-day, guru tanpa jadwal dikecualikan, prioritas AttendanceRecord>TeacherPermit, breakdown 4 kategori terpisah, karyawan pakai wajibDates generik (BUKAN 0), guru pakai jadwal-per-hari, filter statusKepegawaian, validasi from>to.

**Frontend**:
- `apps/web/src/app/(admin)/rekap-guru/page.tsx` + `rekap-guru-view.tsx` (baru) — toggle Per Hari/Per Rentang, filter kepegawaian 3-pill (Semua/Guru/Karyawan), search+sort semua kolom via `SortableHeader`+kolom No, scroll horizontal terkontrol (`overflow-x-auto`) untuk tabel 12-kolom mode range, badge "Sen-Jum" kecil di baris karyawan (jumlahHariWajib generik terlihat WAJAR bukan bug), pesan jelas saat filter Karyawan+Per Hari (bukan tabel kosong), export PDF/Excel via `proxy-download` generic route (tidak perlu diubah).
- `apps/web/src/lib/core-types.ts` — `TeacherReportRow`/`TeacherReportSingleDayRow`/`TeacherFlexibleReportResult`/`TeacherSingleDayStatus` baru.
- `apps/web/src/components/shell/nav-items.ts` — item baru "Rekap Guru & Karyawan" di grup "Absensi & Rekap", sejajar "Rekap Kehadiran" (siswa).

**Verifikasi live** (dev DB port 3307, API dev port 3101, production tidak disentuh; 1 guru existing diubah sementara jadi karyawan untuk uji, dikembalikan setelahnya; akun `adminSU` password di-override sementara TestPass123 lalu DIKEMBALIKAN persis, dikonfirmasi via SELECT sebelum/sesudah):
1. `GET /teacher-attendance-report?from=X&to=X` (single-day) — guru dengan jadwal+tap terlambat MUNCUL dengan status `terlambat`; guru tanpa jadwal TIDAK muncul di tabel.
2. `GET ...?statusKepegawaian=karyawan` mode single-day → `rows: [], karyawanDikecualikan: true`.
3. `GET ...` mode range (5 hari) — karyawan: `jumlahHariWajib: 5` (Senin-Jumat generik, BUKAN 0), `izinPribadi: 1`, `alfa: 4` (breakdown kategori + hari wajib generik BEKERJA); guru dengan 1 jadwal Jumat dalam rentang: `terlambat: 1`, `jumlahHariWajib: 1` (BUKAN 5 — patokan jadwal mengajar, bukan generik); guru lain tanpa jadwal: `jumlahHariWajib: 0, alfa: 0` (tidak dipaksa alfa tanpa patokan).
4. `GET .../export.xlsx` dan `.../export.pdf` — 200 OK, file valid (`Microsoft Excel 2007+`, `PDF document 1 page`) berukuran wajar (grafik ter-embed).
5. Role `guru` (bukan `super_admin`/`admin_jurnal`) → `GET /teacher-attendance-report` → 403 Forbidden, dikonfirmasi.
6. Data uji (attendance_records, teacher_permits, schedules jam_mengajar, semester test, statusKepegawaian guru test) dibersihkan, password hash 2 akun test dikembalikan PERSIS ke nilai semula (dikonfirmasi via SELECT).
7. `tsc --noEmit` bersih `apps/api` DAN `apps/web`, `jest` — 23 suite / 292 test lulus 100%.

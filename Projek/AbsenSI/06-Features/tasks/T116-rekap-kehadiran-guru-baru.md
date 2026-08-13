# T116 — Schema+API+Web: Rekap Kehadiran Guru (Fitur Baru dari Nol)

> **⚠️ SUPERSEDED 2026-08-14 oleh [[T176-rekap-kehadiran-guru-karyawan]]** — task ini ditulis 2026-08-06 saat guru SELALU berstatus `hadir` (tidak pernah `terlambat`), makanya SENGAJA tanpa kolom Terlambat. T175 (baru) mengaktifkan logic keterlambatan guru berdasarkan jadwal mengajar, sehingga rancangan "tanpa Terlambat" di sini SUDAH TIDAK BERLAKU. **JANGAN eksekusi task ini** — pakai T176 sebagai rancangan definitif. Detail breakdown kategori `TeacherPermit` (sakit/izin_pribadi/tugas_dinas/pelatihan) dan prioritas AttendanceRecord-vs-TeacherPermit di hari yang sama dari task ini SUDAH DISERAP ke T176. File ini dibiarkan ada (tidak dihapus) sebagai jejak riwayat keputusan.

## Depends on
**WAJIB T114 (Setting Kop Surat) selesai duluan** — sama seperti T115, task ini butuh kop surat modular. **Disarankan setelah T115 juga selesai** (bukan dependency keras, tapi T115 membangun pola export PDF/Excel/grafik yang bisa DI-REUSE langsung di sini — mengerjakan T116 sebelum T115 berarti reinvent pola yang sama 2x).

## Objective
Rekap kehadiran GURU — fitur BARU sepenuhnya (belum ada infrastruktur apa pun sebelumnya, beda dari rekap siswa yang sudah ada sejak T055) — dengan format sama seperti rekap siswa (T115): kop surat, grafik, filter, export PDF+Excel.

## Context
- **App:** `apps/api` (query+alfa-guru baru, generate PDF/Excel) + `apps/web` (halaman baru)
- **Riset 2026-08-06 (Explore agent, baca kode langsung) — konfirmasi ini benar-benar dari nol, BUKAN salah asumsi seperti rekap siswa:**
  - `AttendanceRecord.teacherId` sudah ada (dual-FK pattern, `apps/api/prisma/schema.prisma:483-506`), tap flow guru sudah mencatat record — TAPI **guru SELALU berstatus `hadir`**, tidak pernah `terlambat` (`attendance.service.ts:602-606`, `determineStatus()`). Komentar kode eksplisit: ini **SEMENTARA**, bukan keputusan permanen — "jadwal mengajar guru belum diinput user... sampai jadwal tersedia" (2026-07-28, terkait T101 yang ditunda tanpa batas). **Rekap guru versi ini TIDAK BOLEH menampilkan kolom "Terlambat" yang selalu nol** — sesuai keputusan user di bawah.
  - `TeacherPermit` (`schema.prisma:427-449`) — setara `Permit` siswa tapi lebih luas (sakit/izin_pribadi/tugas_dinas/pelatihan, bukan cuma izin/sakit). **TIDAK ADA kode yang menghubungkan `TeacherPermit` ke `AttendanceRecord`** — keduanya independen, tidak saling meniadakan. Task ini yang PERTAMA KALI menggabungkan keduanya untuk keperluan rekap.
  - **TIDAK ADA perhitungan "alfa"/hari-wajib untuk guru** di mana pun. `resolveHariWajib()` (`attendance-report.service.ts:272-305`) sebenarnya GENERIK secara logic (Senin-Jumat dikurangi `school_holidays`, scoped `academic_years` — tidak spesifik siswa) tapi SEMUA pemanggilnya hardcode ke `Student`/`StudentPkl`/`kelas`. **Task ini reuse `resolveHariWajib()` apa adanya**, tapi menulis logic BARU untuk sisi guru (belum pernah ada).
  - **Tidak ada dimensi "kelas" untuk guru** (guru bukan siswa). Grouping yang masuk akal: flat semua guru, atau via `statusKepegawaian` (guru vs karyawan, `Teacher.statusKepegawaian` enum), TIDAK ADA relasi langsung `Teacher`→`Jurusan` (cuma tidak langsung lewat `Schedule`/`TeachingSession`→`Kelas.jurusanId`, terlalu tidak langsung untuk jadi filter utama rekap).

## Keputusan Final (dikonfirmasi user 2026-08-06)

**Kategori rekap guru: Hadir vs Alfa/Tidak Hadir SAJA** — TIDAK ada kolom "Terlambat" (karena data guru saat ini selalu nol untuk kategori itu, menampilkannya akan menyesatkan/kosong tanpa makna). Kalau nanti T101 (validasi jadwal guru) selesai dan data terlambat guru mulai valid, rekap ini bisa diperluas — TAPI BUKAN scope task ini, jangan buat kolom kosong "untuk jaga-jaga masa depan".

**Kategori final:**
- **Hadir** — ada `AttendanceRecord` hari itu (status apa pun, karena guru cuma punya `hadir` sekarang).
- **Izin/Sakit/Dinas/Pelatihan** — dari `TeacherPermit`, breakdown per `kategori` field yang sudah ada di model (sakit/izin_pribadi/tugas_dinas/pelatihan) — TAMPILKAN detail kategori ini di rekap (bukan digabung jadi 1 "Izin" generik), karena datanya sudah granular di schema, sayang kalau tidak dimanfaatkan.
- **Alfa** — hari wajib (`resolveHariWajib()` reuse) TANPA `AttendanceRecord` DAN TANPA `TeacherPermit` di hari itu.

## Spec Detail

### Backend
- `apps/api/src/attendance/attendance-report.service.ts` (atau modul service baru kalau dirasa lebih bersih dipisah dari yang siswa — putuskan saat implementasi, tapi REUSE `resolveHariWajib()` yang sudah ada, jangan duplikasi logic kalender) — method baru `reportGuru(query: ReportGuruQueryDto)`:
  - Ambil semua guru aktif (`status: aktif`, filter opsional `statusKepegawaian` kalau relevan untuk bedakan guru vs karyawan — **klarifikasi ke user saat implementasi**: apakah rekap ini untuk SEMUA `Teacher` termasuk karyawan/staf non-guru, atau cuma yang `statusKepegawaian: guru`? Nama fitur "rekap kehadiran GURU" condong ke guru saja, tapi user sebelumnya juga menyinggung soal "staf karyawan jam kerjanya beda" di diskusi modularitas — mungkin relevan tapi BUKAN otomatis termasuk scope task ini kecuali dikonfirmasi).
  - Untuk tiap guru: hitung `hadir` (count `AttendanceRecord` hari wajib), breakdown `TeacherPermit` per kategori (count per `kategori` enum value), `alfa` (hari wajib dikurangi hadir dikurangi semua kategori permit).
  - Filter query: `from`/`to` (atau tahun ajaran/semester seperti rekap siswa, konsisten polanya), TIDAK ADA filter kelas/jurusan (tidak relevan untuk guru) — cukup filter individual guru (opsional, kalau admin mau lihat 1 guru saja) atau semua guru.
- Endpoint baru `GET /attendance/report-guru`, dan export `GET /attendance/report-guru/export/pdf` + `/export/excel` — **REUSE infrastruktur PDF/Excel/kop-surat dari T115** (template Puppeteer, exceljs, `LetterheadConfigService`), JANGAN bangun ulang dari nol. Kalau T115 menghasilkan fungsi generate-PDF yang generic (terima judul+kolom+data sebagai parameter), pakai itu; kalau T115 ternyata terlalu spesifik ke shape data siswa, refactor SEDIKIT supaya generic — tapi TETAP reuse kerangka utamanya (Puppeteer setup, kop surat rendering, dsb), jangan install ulang/setup ulang Puppeteer dari nol.

### Frontend
- Halaman baru `apps/web/src/app/(admin)/rekap-guru/` (route terpisah dari `/rekap` siswa, ATAU tab di halaman yang sama — putuskan berdasarkan UX yang lebih baik saat implementasi, tapi kalau terpisah tetap ikuti pola sidebar grup yang sudah ada, misal masuk grup "Rekap" bersama rekap siswa kalau sidebar admin punya struktur begitu).
- Filter: rentang tanggal/tahun ajaran-semester, opsional pilih guru tertentu.
- Tabel: Nama Guru, Hadir, breakdown kategori TeacherPermit (Sakit/Izin Pribadi/Tugas Dinas/Pelatihan — kolom terpisah per kategori sesuai keputusan), Alfa, Total Hari Wajib.
- Grafik on-screen (reuse chart library dari T115).
- 2 tombol Download PDF/Excel (reuse pola dari T115).

## Edge Cases
- Guru yang statusnya berubah jadi nonaktif di TENGAH rentang tanggal rekap → tentukan apakah tetap dihitung untuk periode dia masih aktif (kemungkinan besar YA, konsisten dengan bagaimana siswa nonaktif/keluar ditangani di rekap siswa — cek pola existing kalau ada) atau dikecualikan total — **cek dulu bagaimana rekap siswa T055 menangani siswa yang keluar/nonaktif di tengah periode, ikuti pola yang sama untuk konsistensi**.
- Guru dengan `AttendanceRecord` DAN `TeacherPermit` di hari yang SAMA (kasus yang tidak direkonsiliasi kode manapun, dikonfirmasi riset) → putuskan prioritas mana yang dihitung (kemungkinan `AttendanceRecord`/hadir menang karena itu bukti fisik tap, `TeacherPermit` di hari itu jadi anomali data yang diabaikan untuk rekap — **klarifikasi ke user** kalau ditemukan kasus ini signifikan jumlahnya saat testing).

## Files
- **Buat:** method baru di `attendance-report.service.ts` (atau service terpisah), DTO baru, endpoint baru, halaman frontend baru `apps/web/src/app/(admin)/rekap-guru/`.
- **Modifikasi:** sidebar admin (`nav-items.ts`) — tambah menu baru.
- **Jangan sentuh:** `resolveHariWajib()` internal logic (reuse fungsi yang ada apa adanya), `report()` rekap siswa (tidak perlu diubah untuk task ini).

## Acceptance Criteria
- [ ] Halaman Rekap Kehadiran Guru menampilkan Hadir/breakdown-kategori-TeacherPermit/Alfa per guru, TANPA kolom Terlambat.
- [ ] Filter tanggal/tahun ajaran-semester berfungsi, opsional filter per guru individual.
- [ ] Grafik on-screen tampil, mencerminkan data rekap.
- [ ] Export PDF dan Excel berfungsi, memakai kop surat T114, format konsisten dengan T115 (reuse infrastruktur, bukan duplikasi).
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [ ] **WAJIB klarifikasi ke user** sebelum eksekusi: apakah rekap ini untuk SEMUA `Teacher` (termasuk karyawan/staf non-guru via `statusKepegawaian`) atau HANYA guru — nama fitur ambigu terhadap ini.
- [ ] Pastikan T115 SUDAH selesai dan infrastruktur PDF/Excel/grafik/kop-surat sudah generic-reusable sebelum mulai — kalau T115 ternyata hardcode terlalu spesifik ke shape data siswa, lakukan refactor kecil dulu di T115 punya kode (bukan bikin infrastruktur PDF kedua yang terpisah).
- [ ] Cek pola penanganan siswa nonaktif-di-tengah-periode di rekap siswa (T055) sebelum putuskan pola yang sama untuk guru, demi konsistensi lintas fitur.

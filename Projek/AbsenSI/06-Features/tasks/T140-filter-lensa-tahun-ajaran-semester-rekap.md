# T140 — API+Web: "Lensa Penjelajahan" Tahun Ajaran/Semester di Halaman Rekap/Laporan

## Depends on
**WAJIB T139 selesai dan diverifikasi dulu** (kolom `academicYearId`/`semesterId` harus sudah ada dan terisi otomatis untuk data baru sebelum task ini punya kolom untuk di-query). JANGAN mulai task ini kalau T139 belum selesai.

## Objective
Halaman-halaman rekap/laporan (baca-saja, browsing histori) bisa difilter by Tahun Ajaran + Semester memakai kolom `academicYearId`/`semesterId` baru (query langsung, BUKAN date-range inference seperti `resolveDateRange()` yang dipakai sekarang) — DAN mengganti mekanisme filter tahun/semester yang SUDAH ADA di halaman Rekap (`rekap-view.tsx`) dari date-range-based ke kolom-based, TANPA mengubah operasional live sama sekali.

## Context — Batasan Arsitektur PENTING (Keputusan Final, Baca Sebelum Kerja)

Ini KELANJUTAN diskusi user soal "1 saklar tahun ajaran aktif untuk semua menu". Keputusan FINAL setelah dibahas:

**JANGAN buat 1 toggle/context global yang mempengaruhi SELURUH aplikasi** (termasuk kiosk, papan piket, lock/unlock, jurnal piket). Itu BERBAHAYA — kalau admin "mundur" ke tahun lalu untuk lihat data, lalu ada tap kartu SUNGGUHAN masuk hari ini, tap itu TIDAK BOLEH salah tercatat ke tahun yang sedang "di-browse". Operasional live HARUS SELALU pakai `isActive` sungguhan (real-time), TIDAK PERNAH bisa "digeser" oleh siapa pun.

Scope task ini SEMPIT dan SPESIFIK: **filter tahun ajaran/semester HANYA muncul di halaman-halaman rekap/laporan BACA-SAJA** (daftar lengkap di bawah) — TIDAK ADA toggle global di top bar/sidebar yang mempengaruhi menu lain. Setiap halaman rekap punya filter Tahun Ajaran + Semester SENDIRI (dropdown biasa, defaultnya ke yang `isActive` saat ini, bisa diganti admin), scope-nya HANYA halaman itu, TIDAK "menular" ke halaman lain atau ke sesi browser secara global.

**Kalau saat implementasi terasa godaan untuk membuat context/state global (React Context, localStorage, dll) yang dipakai lintas halaman** — JANGAN, itu di luar keputusan final ini. Tiap halaman rekap independen.

## Halaman yang WAJIB dapat filter Tahun Ajaran + Semester (kolom-based)

1. **`apps/web/src/app/(admin)/rekap/rekap-view.tsx`** (Rekap Kehadiran Siswa, gerbang) — SUDAH punya filter Tahun Ajaran (baris ±354) + Semester (±369), TAPI saat ini mekanismenya lewat `resolveDateRange()` (date-range inference di `attendance-report.service.ts:535-577`). **GANTI mekanisme query jadi filter kolom `academicYearId`/`semesterId` langsung** di `AttendanceRecord` — dropdown UI-nya SUDAH ADA, cukup ganti cara backend memfilternya. **INI JUGA kesempatan memperbaiki 2 pelanggaran urutan filter yang ditemukan riset 2026-08-08** (lihat Spec Detail #3 di bawah — WAJIB dikerjakan sekalian, bukan opsional).
2. **T138 (Rekap Kehadiran Ekstrakurikuler, kalau sudah dikerjakan saat T140 dimulai)** — kalau T138 SUDAH SELESAI duluan, tambahkan filter Tahun Ajaran + Semester ke halamannya juga pakai kolom baru `EkstraSesi.academicYearId`/`semesterId`. Kalau T138 BELUM dikerjakan saat T140 mulai, LEWATI poin ini (jangan blocking T140 menunggu T138) — user akan minta task susulan kecil kalau perlu nanti.
3. Halaman rekap/laporan LAIN yang mungkin ada saat T140 dieksekusi (cek ulang halaman under `(admin)/` yang punya kata "rekap"/"laporan" di nama filenya) — terapkan pola sama.

## Spec Detail

### 1. Backend — ganti mekanisme filter `attendance-report.service.ts`
- `report()` dan `report-flexible` endpoint (yang dipakai `rekap-view.tsx`, lihat T132) — ganti filter dari `WHERE tanggal BETWEEN X AND Y` (hasil `resolveDateRange()`) MENJADI `WHERE academicYearId = X AND semesterId = Y` KETIKA user memilih Tahun Ajaran+Semester spesifik dari dropdown.
- **PENTING — filter tanggal (Dari/Sampai) yang SUDAH ADA di halaman ini TETAP DIPERTAHANKAN sebagai filter TERPISAH DAN INDEPENDEN** — bukan diganti total. User bisa: (a) pilih Tahun Ajaran+Semester SAJA (tanpa rentang tanggal spesifik → tampilkan semua data semester itu), (b) pilih Tahun Ajaran+Semester DAN rentang tanggal (mempersempit lebih lanjut DALAM semester itu), atau (c) yang sudah ada sekarang tetap harus berfungsi. Putuskan kombinasi query SQL-nya (`AND` antara kedua filter) — JANGAN buang kemampuan filter tanggal granular yang sudah ada.
- Method `resolveDateRange()` — TIDAK dihapus (masih dipakai untuk kasus data LAMA yang `academicYearId`-nya `null`, lihat Edge Cases), tapi jadi FALLBACK bukan mekanisme utama untuk data baru.
- **Data lama (`academicYearId: null` dari sebelum T139)** — kalau user filter ke Tahun Ajaran/Semester yang mencakup periode SEBELUM T139 di-deploy, data itu TIDAK AKAN muncul (karena `academicYearId` mereka `null`, tidak match filter kolom). Ini KETERBATASAN YANG DIKETAHUI dan DITERIMA (bukan bug) — kalau muncul saat testing, JANGAN "diperbaiki" dengan backfill diam-diam (di luar scope, keputusan T139 sudah final tidak backfill). Cukup pastikan behavior ini tidak crash, tampilkan hasil kosong/parsial apa adanya.

### 2. Frontend — dropdown Tahun Ajaran/Semester (SUDAH ADA di rekap-view.tsx, verifikasi/sesuaikan)
- Dropdown existing (baris ±354, ±369) — cek apakah defaultnya SUDAH ke Tahun Ajaran/Semester yang `isActive`. Kalau belum, jadikan default itu (bukan kosong/harus-pilih-manual).
- Label/UX — pastikan JELAS bagi admin bahwa filter ini untuk "melihat data periode tertentu", TIDAK mengubah apa pun secara global (tidak perlu teks penjelasan panjang di UI, cukup dropdown yang berfungsi benar — tapi kalau ada keraguan istilah, gunakan "Tahun Ajaran" bukan "Tahun Aktif" supaya tidak ambigu dengan konsep `isActive` sistem).

### 3. WAJIB SEKALIAN — perbaiki 2 pelanggaran konsisten yang ditemukan riset di halaman `rekap-view.tsx` (bukan opsional, bagian dari task ini karena file yang sama sedang disentuh)
Riset 2026-08-08 menemukan `rekap-view.tsx` melanggar aturan permanen proyek (memory `feedback_filter_search_jurusan_kelas_order`, `feedback_tabel_wajib_search_sort_kolom_no`, `feedback_filter_rekap_wajib_tingkat`):
- **Urutan filter SALAH**: saat ini Tahun Ajaran → Semester → Dari/Sampai Tanggal → **Kelas (baris ±405) → Jurusan (baris ±419)** — Kelas muncul SEBELUM Jurusan, terbalik dari konvensi wajib proyek. **PERBAIKI jadi**: Tahun Ajaran → Semester → Dari/Sampai Tanggal → Search → Jurusan → Tingkat → Kelas (tambahkan Search dan Tingkat yang juga hilang, lihat 2 poin di bawah).
- **TIDAK ADA search box sama sekali** — tambahkan (cari nama siswa, pola sama halaman lain: `Input` + ikon `Search` + debounce 300ms).
- **TIDAK ADA filter Tingkat** — tambahkan (dropdown X/XI/XII, posisi antara Jurusan dan Kelas sesuai memory `feedback_filter_rekap_wajib_tingkat`).
- **Mode rentang (agregat multi-hari) TIDAK ADA kolom "No" dan TIDAK sortable sama sekali** (baris ±534-567) — perbaiki: tambahkan kolom No (offset halaman), bungkus semua kolom yang masuk akal (Nama, Kelas, dan kolom angka rekap lain) dengan `SortableHeader`.
- Mode single-day (baris ±493-507) SUDAH sebagian sortable (Nama, Kelas) tapi NISN/Status/Waktu/Keterangan TIDAK — **buat SEMUA kolom sortable** (kecuali kolom yang memang tidak masuk akal di-sort seperti tombol aksi, tapi NISN/Status/Waktu/Keterangan semua masuk akal di-sort, jadi WAJIB dibungkus `SortableHeader`).

Ini SEKALIGUS jadi bagian dari audit tabel besar (lihat T141) — TAPI karena file `rekap-view.tsx` SUDAH PASTI disentuh task ini untuk urusan filter tahun ajaran, perbaikan konsistensinya dikerjakan DI SINI (T140), BUKAN diulang lagi di T141. **T141 harus SKIP `rekap-view.tsx`** karena sudah ditangani di sini (lihat catatan di T141).

## Edge Cases
- User pilih Tahun Ajaran/Semester yang SANGAT LAMA (sebelum sistem AbsenSI dipakai sekolah ini) → hasil kosong total, tampilkan empty-state jelas ("Tidak ada data untuk periode ini"), bukan error.
- User TIDAK pilih Tahun Ajaran/Semester sama sekali (kosongkan filter) → putuskan default behavior: REKOMENDASI tampilkan periode yang `isActive` saat ini (bukan seluruh histori all-time, supaya query tidak berat) — tapi ini bisa didiskusikan ulang kalau terasa mengejutkan bagi user saat implementasi (klarifikasi ke user KALAU ambigu, JANGAN cuma menebak diam-diam mengingat instruksi user task ini harus final).
- Kombinasi Tahun Ajaran+Semester dipilih TAPI rentang tanggal manual JUGA diisi, dan rentang tanggal itu di LUAR rentang tanggal semester yang dipilih (kontradiktif) → hasil kosong (AND logic wajar), TIDAK PERLU validasi/blocking khusus, cukup query apa adanya.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (`report()`, `report-flexible`, ganti mekanisme filter), `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (urutan filter, search, Tingkat, sortable, kolom No mode rentang).
- **Jangan sentuh:** operasional live (kiosk, papan piket, lock/unlock, jurnal piket) — TIDAK ADA perubahan apa pun ke situ, task ini murni halaman rekap/laporan baca-saja.

## Acceptance Criteria
- [x] Filter Tahun Ajaran+Semester di `rekap-view.tsx` query pakai kolom `academicYearId`/`semesterId`, bukan lagi murni date-range inference untuk data baru.
- [x] Filter tanggal (Dari/Sampai) existing tetap berfungsi independen, bisa dikombinasikan dengan filter Tahun Ajaran+Semester.
- [x] Default filter Tahun Ajaran+Semester = yang `isActive` saat ini.
- [x] Urutan filter `rekap-view.tsx` jadi Tahun Ajaran→Semester→Dari/Sampai Tanggal→Search→Jurusan→Tingkat→Kelas.
- [x] Search box baru berfungsi (cari nama siswa).
- [x] Filter Tingkat baru berfungsi (X/XI/XII).
- [x] SEMUA kolom di kedua mode (single-day DAN rentang) sortable asc/desc, mode rentang dapat kolom "No".
- [x] Operasional live (kiosk, papan piket) TIDAK terpengaruh sama sekali — regresi nol, verifikasi tap kartu dan dashboard piket normal setelah perubahan.
- [x] Data lama (`academicYearId: null`) tidak menyebabkan error/crash saat difilter, cukup tidak muncul di hasil (perilaku diterima, bukan bug).
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] **JANGAN** membuat context/state global lintas halaman untuk "tahun ajaran sedang di-browse" — setiap halaman rekap independen, sesuai keputusan final.
- [x] **JANGAN** menyentuh kiosk, papan piket, atau logic `isActive` yang dipakai operasional live — task ini murni sisi baca/rekap.
- [x] Kalau ada halaman rekap/laporan LAIN yang muncul di antara T139 selesai dan T140 dikerjakan (fitur baru masuk), evaluasi apakah perlu ditambahkan ke scope filter ini — laporkan ke user sebagai temuan, jangan otomatis expand scope tanpa bilang.
- [x] Klarifikasi ke user (bukan menebak) untuk default behavior "tidak ada filter dipilih" kalau terasa ambigu saat implementasi.

## Status Eksekusi (2026-08-08)

**Selesai, diverifikasi live.**

### Temuan awal
- T138 (Rekap Ekstrakurikuler) BELUM dikerjakan saat T140 mulai — poin #2 spec (filter ekstra) DILEWATI sesuai instruksi spec, bukan diabaikan diam-diam.
- Klarifikasi WAJIB ke user (poin Validasi Claudian): default behavior "Semua Tahun Ajaran" dipilih — user memilih **Opsi A (fallback ke date-range lama)** setelah dijelaskan konsekuensi kedua opsi. Ini nol-regresi: user yang tidak pilih tahun ajaran spesifik dapat behavior identik dengan sebelum T140.

### Backend (`attendance-report.service.ts`)
- Helper baru `buildPeriodFilter(query)`: return `{}` kalau `academicYearId` tidak dikirim (fallback penuh ke date-range `resolveDateRange()` yang sudah ada), atau `{ academicYearId, semesterId? }` kalau dikirim — di-spread (AND) ke WHERE `attendanceRecord`/`permit` di `report()` DAN `reportSingleDay()`, MELENGKAPI filter tanggal (bukan menggantikan).
- Filter `tingkat` baru ditambahkan ke `ReportQueryDto` (`@IsIn(["X","XI","XII"])`) dan ke where-clause `kelas: { jurusanId, tingkat }` di kedua method (query kolom, server-side — beda dari search yang client-side).
- `resolveDateRange()` TIDAK dihapus, tetap dipakai untuk resolve rentang tanggal (DatePicker bounds, `wajibDates`, `totalHariAktif`) — sesuai spec, filter kolom MELENGKAPI bukan menggantikan mekanisme date-range.
- 15 test existing di `attendance-report.service.spec.ts` tetap lulus tanpa modifikasi (perubahan murni additive, `periodFilter` kosong `{}` kalau tidak ada `academicYearId` di query test).

### Frontend (`rekap-view.tsx`)
- Urutan filter diperbaiki: Tahun Ajaran → Semester → Dari/Sampai Tanggal → **Search** (baru) → Jurusan → **Tingkat** (baru) → Kelas (dipindah ke akhir, sebelumnya di depan Jurusan — pelanggaran konvensi proyek).
- Kelas dropdown sekarang di-cascade filter oleh Jurusan+Tingkat yang dipilih (`kelasTerfilter`), konsisten pola halaman lain.
- Default semester diperbaiki: sebelumnya SELALU reset ke "Semua Semester" tiap ganti tahun ajaran; sekarang auto-pilih semester `isActive` KHUSUS kalau tahun ajaran yang dipilih adalah default tahun ajaran aktif (bukan navigasi manual user ke tahun ajaran lain).
- Search box "Cari nama siswa..." — **client-side** (keputusan user: konsisten dengan sort yang sudah client-side, tidak perlu backend/DTO baru, instant tanpa fetch ulang).
- Mode rentang (sebelumnya TIDAK sortable sama sekali, tanpa kolom "No") — ditambahkan `rangeSort` state + `toggleRangeSort()` + kolom "No" (offset dari hasil ter-filter/ter-sort), semua 11 kolom (Nama, Kelas, Jurusan, Hadir, Terlambat, Izin, Sakit, Alfa, PKL, Belum Memiliki Kartu, Total Hari Aktif) dibungkus `SortableHeader`.
- Mode single-day — NISN/Status/Waktu/Keterangan yang sebelumnya TIDAK sortable, sekarang dibungkus `SortableHeader` juga (Nama/Kelas sudah sortable sebelumnya).
- Search filter diterapkan SEBELUM sort di kedua mode (`useMemo` chain: filter → sort), grafik (`Chart.js`) tetap pakai `result.rows` mentah (tidak terpengaruh search, sesuai scope: search cuma untuk tabel).

### Verifikasi Live
- Playwright: login admin, buka `/rekap` — urutan filter benar (Tahun Ajaran→Semester→Tanggal→Search→Jurusan→Tingkat→Kelas), semua dropdown+search render dan berfungsi (search "Gading" langsung filter ke 1 baris, No re-index ke 1), klik header "Hadir" mengaktifkan sort arrow.
- Curl matrix backend: (1) single-day mode tanpa academicYearId/tingkat → format response benar; (2) `tingkat=X` → hanya siswa kelas X ter-filter; (3) `academicYearId` spesifik (tahun ajaran dibuat sementara utk tes) → HANYA attendance record yang tertag `academicYearId` itu terhitung, record `academicYearId: null` (data lama) TIDAK muncul di hasil (`hadir: 0`, expected — keterbatasan diterima); (4) TANPA `academicYearId` (fallback) → KEDUA record (tertag maupun `null`) terhitung normal via date-range murni, konfirmasi Opsi A bekerja sesuai keputusan user.
- Operasional live: tap kartu kiosk siswa setelah semua perubahan T140 → `result: accepted`, tidak terpengaruh sama sekali — regresi nol dikonfirmasi.
- Data uji (2 `AttendanceRecord`, 1 `AcademicYear` sementara, 1 tap smoke-test) dibersihkan; kiosk `allowed_ip` direstore.

### Test Suite
- `tsc --noEmit` bersih untuk `apps/api` dan `apps/web`.
- Jest: 203 test lulus (15 suite), regresi nol — tidak ada spec yang perlu diubah karena perubahan backend murni additive/opsional.

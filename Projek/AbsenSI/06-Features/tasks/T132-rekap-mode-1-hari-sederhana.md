# T132 — API+Web: Rekap Mode 1-Hari — Kolom Sederhana (No|Nama|NISN|Status|Waktu|Keterangan)

## Depends on
Depends on T115 (export PDF/Excel/grafik rekap siswa) — **SUDAH SELESAI dan sudah live di production** (2026-08-07). Task ini MEMPERLUAS `report()`/`attendance-report-export.service.ts`/`rekap-view.tsx` yang sudah ada, bukan fitur baru dari nol. Baca ulang implementasi T115 dulu sebelum eksekusi.

## Objective
Kalau admin memilih rentang tanggal Dari=Sampai (1 hari yang sama) di halaman Rekap, tabel (dan PDF/Excel yang di-export) berubah format jadi lebih sederhana dan relevan untuk kebutuhan harian: **No | Nama | NISN | Status | Waktu | Keterangan** — 1 baris per siswa dengan status hari itu persis, BUKAN lagi kolom hitungan akumulatif (Hadir/Terlambat/Izin/Sakit/Alfa/PKL sebagai angka) yang cocok untuk rentang panjang tapi berlebihan untuk 1 hari.

## Context
- **App:** `apps/api` (query baru untuk mode 1-hari + waktu tap + keterangan izin) + `apps/web` (kondisional render tabel/grafik berdasar mode)
- **Riset 2026-08-07 (Explore agent, baca kode langsung)** — konfirmasi kondisi SEBELUM task ini:
  - `report()` (`attendance-report.service.ts:59-211`) **SELALU** menghasilkan `ReportRow` bentuk agregat (`hadir, terlambat, izin, sakit, alfa, pkl, belumMemilikiKartu, totalHariAktif` sebagai ANGKA hitungan) — **TIDAK ADA percabangan kode untuk `from === to`** sama sekali, 1 hari diperlakukan sebagai "rentang panjang 1 hari" yang tetap dihitung agregat (hasilnya technically benar tapi TIDAK BERGUNA — misal "Hadir: 1" untuk 1 hari itu kurang informatif dibanding tahu JAM BERAPA dia hadir).
  - **`waktuMasuk` TIDAK PERNAH di-fetch di `report()`** — query `AttendanceRecord` di situ sengaja cuma `select: { studentId, tanggal, status }` (baris ±89). Field ini HANYA pernah diambil di `riwayatCatatan()` (query 1 siswa spesifik, beda endpoint) — task ini perlu query BARU yang fetch `waktuMasuk` secara eksplisit untuk mode 1-hari.
  - **`Permit.alasanDetail` (field teks alasan izin/sakit) TIDAK di-fetch di `report()`** — query `Permit` di situ cuma `select: { studentId, tanggal, alasanKategori }` (baris ±91-94). Field ini ADA di schema (`schema.prisma:795`) dan sudah dipakai di `riwayatCatatan()`, tinggal ditambahkan ke `select` untuk mode 1-hari.
  - `belumMemilikiKartu` (T113/T115, SUDAH ADA) — logic pengecualian siswa yang belum pernah diterbitkan kartu dari perhitungan alfa, **REUSE logic yang sama** untuk menentukan status "Belum Memiliki Kartu" di mode 1-hari (bukan bikin logic baru).
  - `attendance-report-export.service.ts` (T115, SUDAH ADA) — PDF/Excel REUSE `ReportRow` yang sama persis dengan tampilan layar, TIDAK ADA struct kolom terpisah — pola ini perlu diperluas untuk mode 1-hari juga (kolom export ikut berubah, bukan cuma tampilan layar).

## Keputusan Final (dikonfirmasi user 2026-08-07)

1. **Trigger mode 1-hari**: `from === to` (Dari Tanggal sama dengan Sampai Tanggal) — bukan berdasarkan pilihan UI terpisah, murni deteksi otomatis dari filter tanggal yang sudah ada.
2. **7 status** (bukan 6 seperti sebutan awal, "Terlambat" ditambahkan sebagai kategori terpisah dari "Hadir"): **Hadir, Terlambat, Belum Absen, Izin, Sakit, Belum Memiliki Kartu, PKL**.
3. **Kolom Waktu**: HANYA terisi untuk status **Hadir** dan **Terlambat** (jam tap datang, `AttendanceRecord.waktuMasuk` hari itu) — status lain (Belum Absen/Izin/Sakit/Belum Memiliki Kartu/PKL) kolom ini `-` (strip).
4. **Kolom Keterangan**: HANYA terisi untuk status **Izin**/**Sakit** — sumbernya `Permit.alasanDetail` (alasan yang diinput piket saat approve izin siswa itu). Status lain kolom ini kosong/`-`.
5. **Berlaku JUGA di export PDF/Excel** — bukan cuma tampilan layar, konsisten antara yang dilihat admin di web dan file yang di-download.
6. **Grafik TETAP ada untuk mode 1-hari**, TAPI bentuknya berubah — dari "tren akumulatif Hadir/Izin/Sakit/Alfa" (cocok rentang panjang) jadi **"jumlah siswa per status hari itu"** (misal bar chart: Hadir 200, Terlambat 12, Izin 5, Sakit 3, Belum Absen 8, dst) — 7 kategori status yang sama seperti tabel.

## Spec Detail

### Backend

**Deteksi mode** — di `report()` (atau method baru khusus, putuskan saat implementasi mana yang lebih bersih): cek `from === to` (bandingkan tanggal, bukan datetime lengkap) → kalau true, panggil jalur BARU (misal `reportSingleDay()`), kalau false, `report()` existing TIDAK BERUBAH (rentang panjang tetap seperti sekarang, regresi nol).

**`reportSingleDay(tanggal, filter)` — method baru:**
- Query siswa aktif sesuai filter (kelas/jurusan/wali kelas — REUSE filter yang sama seperti `report()`, jangan reinvent).
- Untuk TIAP siswa, tentukan status hari itu (urutan prioritas evaluasi, PENTING untuk konsistensi — tentukan urutan yang benar saat implementasi, kemungkinan: cek dulu apakah siswa itu PKL aktif hari itu → cek apakah punya kartu aktif (REUSE logic `belumMemilikiKartu` dari T113) → cek `AttendanceRecord` hari itu (`hadir`/`terlambat`) → cek `Permit` hari itu (`izin`/`sakit`) → fallback `Belum Absen` kalau tidak ada apa pun).
- `AttendanceRecord` query untuk hari itu HARUS include `select: { ..., waktuMasuk: true }` (field yang SEKARANG tidak di-fetch di `report()` biasa) — untuk kolom Waktu.
- `Permit` query untuk hari itu HARUS include `select: { ..., alasanDetail: true }` (field yang SEKARANG tidak di-fetch) — untuk kolom Keterangan.
- Return shape BARU (BUKAN `ReportRow` yang sudah ada, itu untuk mode rentang) — misal `ReportSingleDayRow { no, studentId, nama, nisn, status, waktu: string | null, keterangan: string | null }`.

**Endpoint** — REUSE endpoint yang sama (`GET /attendance/report`, atau apa pun nama endpoint `report()` existing) — backend yang menentukan shape response berdasarkan `from === to`, FE cukup deteksi shape yang diterima (atau backend kirim flag eksplisit `{ mode: "single-day" | "range", rows: [...] }` — REKOMENDASI: flag eksplisit lebih aman daripada FE menebak shape dari isinya).

**Export PDF/Excel** (`attendance-report-export.service.ts`):
- Terima mode yang sama, cabangkan template kolom: mode 1-hari pakai kolom baru (No|Nama|NISN|Status|Waktu|Keterangan), mode rentang TETAP kolom lama (regresi nol).
- Grafik di PDF (dan on-screen web) untuk mode 1-hari: bar chart 7 kategori status vs jumlah siswa — REUSE chart library yang sama dari T115 (jangan install baru), cuma ganti bentuk data yang di-plot.

### Frontend
- `apps/web/src/app/(admin)/rekap/rekap-view.tsx` — deteksi `from === to` dari state filter yang sudah ada, render tabel kondisional:
  - Mode 1-hari: kolom No, Nama, NISN, Status (badge warna sesuai status — REUSE pola badge yang sudah ada di proyek untuk status kehadiran, cek `piket-board-view.tsx` `StatusBadge` untuk pola warna per status kalau relevan dicontek), Waktu, Keterangan.
  - Mode rentang: TIDAK BERUBAH dari sekarang (kolom agregat existing).
- Grafik on-screen ikut kondisional sama seperti PDF (7 kategori status untuk 1 hari, tren akumulatif untuk rentang).
- Tombol Download PDF/Excel — tetap 2 tombol yang sama, backend yang menentukan format kolom sesuai mode, FE tidak perlu logic tambahan selain kirim filter yang sama seperti sekarang.

## Edge Cases
- Siswa PKL aktif DAN entah bagaimana juga punya `AttendanceRecord`/`Permit` hari yang sama (data anomali, seharusnya tidak terjadi tapi mungkin ada data lama) — tentukan prioritas status yang menang (kemungkinan PKL menang, karena PKL adalah status administratif yang lebih definitif — putuskan saat implementasi kalau ditemukan kasus nyata).
- Siswa yang statusnya SUDAH `terlambat` DAN otomatis lock 2x-terlambat (jadi juga "Terkunci") — status "Terlambat" tetap dipakai untuk kolom Status (bukan "Terkunci", itu bukan salah satu dari 7 kategori yang diminta) — cek apakah user perlu tahu soal status lock ini juga atau cukup 7 kategori yang disebutkan, TIDAK perlu status ke-8 kecuali diminta eksplisit.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (`reportSingleDay()` baru + deteksi mode di `report()`/endpoint), `apps/api/src/attendance/attendance-report-export.service.ts` (template kolom kondisional PDF/Excel), `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (tabel+grafik kondisional).
- **Jangan sentuh:** `report()` untuk mode rentang (regresi nol, logic lama TIDAK diubah), filter existing (kelas/jurusan/wali kelas/tahun ajaran/semester — semua tetap berfungsi sama untuk kedua mode).

## Acceptance Criteria
- [x] Pilih Dari=Sampai tanggal yang sama → tabel berubah jadi kolom No|Nama|NISN|Status|Waktu|Keterangan. Diverifikasi live via Playwright.
- [x] Pilih rentang tanggal berbeda → tabel TETAP format lama (agregat Hadir/Terlambat/dst), regresi nol. Diverifikasi live — grafik+tabel T115 identik seperti sebelum T132.
- [x] Status yang muncul HANYA dari 7 kategori: Hadir, Terlambat, Belum Absen, Izin, Sakit, Belum Memiliki Kartu, PKL. Diverifikasi live (data nyata dev DB menunjukkan Belum Absen/Belum Memiliki Kartu/PKL) + 4 unit test baru meng-cover semua kategori termasuk Hadir/Terlambat/Izin/Sakit (data mock).
- [x] Kolom Waktu terisi jam tap untuk status Hadir/Terlambat, `-` untuk status lain.
- [x] Kolom Keterangan terisi `alasanDetail` untuk status Izin/Sakit, kosong untuk status lain.
- [x] Download PDF mode 1-hari menampilkan kolom sederhana yang sama seperti tabel layar. Diverifikasi live (curl 428KB PDF valid + download browser sungguhan via Playwright).
- [x] Download Excel mode 1-hari menampilkan kolom sederhana yang sama. Diverifikasi live (curl + download browser).
- [x] Grafik mode 1-hari menampilkan jumlah siswa PER STATUS (7 kategori), bukan tren akumulatif. Diverifikasi live — judul otomatis berganti "Grafik Jumlah Siswa per Status", bar chart 7 kategori tampil sesuai data nyata.
- [x] Mode rentang (PDF/Excel/grafik/tabel) semuanya regresi nol dari yang sudah ada (T115). Diverifikasi live + 11 test lama (T055/T081) tetap lulus tanpa modifikasi.
- [x] Build + type-check `apps/api` dan `apps/web` hijau. `tsc --noEmit` bersih kedua app, jest 191/191 (187 lama + 4 baru).

## Status Eksekusi — SELESAI (2026-08-07)
**Keputusan desain tambahan (tidak eksplisit di spec awal, ditentukan saat implementasi)**: `report()` endpoint lama (`GET /attendance/report`) **TIDAK disentuh sama sekali** — ternyata dipakai juga oleh dashboard Wali Kelas (`rekapKelasWali()` di controller, dikonsumsi `ringkasan-kehadiran-tab.tsx`) yang mengasumsikan `ReportRow[]` polos tanpa flag mode. Mengubah shape `report()` akan memecah dashboard itu. Solusi: endpoint BARU `GET /attendance/report-flexible` (method `AttendanceReportService.reportFlexible()`) khusus dipakai `rekap-view.tsx` (halaman Rekap admin) + endpoint export PDF/Excel — mengembalikan `{ mode: "range" | "single-day", rows: [...] }`. `report()` lama dan dashboard Wali Kelas regresi nol, tidak tersentuh sama sekali.

**Backend**: `attendance-report.service.ts` — tipe baru `SingleDayStatus`, `ReportSingleDayRow`, `FlexibleReportResult` (union). `reportFlexible()` deteksi `from===to` (compare tanggal via `dateKey()`, bukan datetime). `reportSingleDay()` (private) — query siswa+AttendanceRecord(+waktuMasuk)+Permit(+alasanDetail)+StudentPkl+Card.groupBy (REUSE logic `belumMemilikiKartu` T113 apa adanya), urutan prioritas status per siswa: PKL aktif → belum punya kartu → AttendanceRecord (hadir/terlambat) → Permit (izin/sakit) → fallback belum_absen — didokumentasikan di komentar kode. `attendance.controller.ts` — endpoint baru `GET /attendance/report-flexible`, endpoint export PDF/Excel diupdate pakai `reportFlexible()` bukan `report()`.

**Export PDF/Excel** (`attendance-report-export.service.ts`): `generatePdf()`/`generateExcel()` sekarang terima `FlexibleReportResult`, cabang template kolom sesuai mode. Method baru `buildSingleDayHtml()` (kop surat sama, subtitle "— 1 Hari", tabel 6 kolom, grafik 7 kategori status pakai Chart.js yang sama — bukan library baru). Konstanta `SINGLE_DAY_STATUS_ORDER`/`_LABEL`/`_COLOR` didefinisikan 1 tempat, dipakai tabel+grafik+Excel (tidak diulang 3x).

**Frontend** (`rekap-view.tsx`): state `result: FlexibleReportResult | null` (ganti `rows: AttendanceReportRow[] | null`), fetch dari `/attendance/report-flexible`. Tabel+grafik kondisional berdasar `result.mode`. Badge status mode 1-hari REUSE pola warna existing: `success` (hadir), `danger` (terlambat), `primary-soft` (izin/sakit), `status-shipped` (belum memiliki kartu, sama seperti piket-board-view.tsx), `status-processing` (PKL, sama seperti siswa-detail-view.tsx), `surface-subtle` (belum absen) — TIDAK ada warna baru yang belum dipakai di proyek.

**Test baru**: `attendance-report.service.spec.ts` — describe block baru "reportFlexible — T132 mode 1-hari" (4 test): deteksi mode, 7 kategori status sesuai prioritas evaluasi (termasuk hadir dengan `waktu` terisi, izin dengan `keterangan` terisi), PKL menang dari status lain saat data tumpang tindih, keterangan `null` untuk status non-izin/sakit.

**Verifikasi live**: dev API+web, JWT manual super_admin, curl endpoint `report-flexible` (kedua mode) + export PDF (428KB, 1 halaman, kop surat+tabel+grafik 7 kategori benar) + Excel, lalu verifikasi UI penuh via Playwright (login → set Dari=Sampai tanggal → mode 1-hari otomatis aktif, tabel+grafik+badge berubah sesuai data nyata dev DB → download PDF+Excel via tombol browser sungguhan → sukses). Regresi mode rentang diverifikasi terpisah (Dari≠Sampai → tabel+grafik T115 identik seperti sebelum T132).

## Validasi Claudian
- [x] Urutan prioritas evaluasi status per siswa didokumentasikan di komentar `reportSingleDay()`: PKL aktif hari itu → belum punya kartu (REUSE T113) → AttendanceRecord (hadir/terlambat) → Permit (izin/sakit) → fallback belum_absen. PKL menang karena status administratif paling definitif (siswa PKL secara fisik tidak di sekolah).
- [x] Logic `belumMemilikiKartu` REUSE persis dari T113 (`Card.groupBy` dengan `_min: { issuedAt }`, bandingkan `dateKey < firstCardDate`) — tidak ada logic baru yang berpotensi tidak konsisten dengan mode rentang.
- [x] Flag mode dikomunikasikan eksplisit via field `mode: "single-day" | "range"` di response `reportFlexible()` — FE cabang render berdasar field ini (`result.mode === "single-day"`), tidak pernah menebak dari bentuk data.

## Revisi Feedback (2026-08-07, setelah eksekusi awal) — Kolom Kelas + Sorting
User minta tabel mode 1-hari tambah kolom **Kelas** (setelah NISN), default terurut per kelas, kolom bisa diklik sort asc/desc (konvensi `SortableHeader` existing proyek).

**Frontend** (`rekap-view.tsx`): kolom "Kelas" baru setelah NISN (data `AttendanceReportSingleDayRow.kelas` sudah tersedia dari backend T132 awal, tinggal ditampilkan). Sort client-side (`useMemo`, bukan server-side seperti tabel besar lain — data mode 1-hari sudah di-fetch sekaligus tanpa pagination). State `singleDaySort` default `{ field: "kelas", dir: "asc" }` (terurut per kelas tanpa perlu klik), kolom "Nama" dan "Kelas" dibuat `SortableHeader` (REUSE komponen `components/sortable-header.tsx`, pola sama seperti tabel lain di proyek — bukan komponen baru). Nomor "No" di-renumber sesuai urutan tampilan (index setelah sort), bukan `row.no` asli dari backend.

**Backend** (`attendance-report-export.service.ts`): PDF dan Excel mode 1-hari ikut ditambah kolom Kelas + default urutan per kelas (`sort((a,b) => a.kelas.localeCompare(b.kelas, "id"))`), konsisten dengan tampilan layar — export tetap representasi 1:1 dari apa yang dilihat admin di layar (prinsip T115 yang sudah ada, dipertahankan).

Diverifikasi live: PDF (kolom Kelas muncul setelah NISN, terurut per kelas), Excel (sama), dan UI browser via Playwright — default urutan per kelas benar, klik header "Kelas" toggle ke descending dengan benar (No ikut ter-renumber). `tsc --noEmit` bersih kedua app, jest 187/187 tetap lulus (tidak ada test baru dibutuhkan — perubahan murni presentasi, logic status/urutan sudah dicover test T132 awal).

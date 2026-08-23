# T115 — API+Web: Export PDF+Excel+Grafik untuk Rekap Kehadiran Siswa (Existing)

## ⚠️ REVISI 2026-08-07 — Dikerjakan TANPA T114 (kebutuhan mendesak)

User secara eksplisit meminta T115 dieksekusi **sebelum** T114 selesai ("untuk kepentingan
mendesak"), membalik dependency asli di bawah. Keputusan sadar, bukan diam-diam:

- **Kop surat di-HARDCODE sementara** di `attendance-report-export.service.ts` (logo
  `logo-sekolah.png` dibaca+cache sekali dari `apps/web/public/`, nama sekolah string
  literal `"SMK KARTANEGARA WATES"`) — BUKAN lewat `LetterheadConfigService` (belum ada,
  T114 belum dikerjakan). **Technical debt yang disadari**: begitu T114 selesai, bagian
  kop surat di `buildHtml()` perlu diganti untuk pakai `getRenderData()`.
- **Scope dipersempit** ke rekap PER-SISWA saja (tabel yang sudah ada di `report()`) —
  filter Wali Kelas baru dan grafik tambahan untuk versi per-kelas (`Rekap.pdf` referensi)
  **TIDAK dikerjakan**, di luar scope mendesak ini. Bisa jadi task terpisah nanti.
- **Grafik Hadir/Izin/Sakit/Alfa DITAMBAHKAN ke rekap per-siswa** juga — di contoh PDF
  referensi asli (`dariDev/2. REKAPITULASI KEHADIRAN SISWA - Cetak All 2.pdf`) versi
  per-siswa TIDAK punya grafik sama sekali, tapi user eksplisit minta grafik ada di semua
  rekap sekarang.
- **`puppeteer-core`** (bukan `puppeteer` penuh) dipakai — tidak download Chromium bundled,
  pakai `/usr/bin/google-chrome-stable` yang sudah terpasang di sistem (hemat ~200MB,
  tidak perlu approve pnpm build-script). Kalau deploy ke server lain, pastikan Chrome/
  Chromium sistem tersedia (lihat `CHROME_CANDIDATES` di service) — beda dari asumsi spec
  asli yang menyinggung "headless Chrome butuh dependency sistem" untuk `puppeteer` biasa.

Depends on asli (untuk task lanjutan filter Wali Kelas + rekap per-kelas + kop surat
resmi T114) tetap berlaku untuk BAGIAN YANG BELUM dikerjakan:

## Depends on (asli, sebagian sudah di-skip — lihat revisi di atas)
**WAJIB T114 (Setting Kop Surat) selesai duluan** — task ini mengonsumsi `LetterheadConfigService.getRenderData()` dari T114, bukan hardcode kop sendiri.

## Objective
Halaman Rekap Kehadiran Siswa yang SUDAH ADA (`(admin)/rekap/`, hasil T055) mendapat kemampuan export PDF dan Excel, dengan kop surat resmi (dari T114), grafik, metadata, dan filter tambahan per Wali Kelas — melengkapi fitur yang saat ini HANYA tampil sebagai tabel di layar tanpa export sama sekali.

## Context
- **App:** `apps/api` (generate PDF/Excel) + `apps/web` (tombol export, filter wali kelas, chart on-screen)
- **Riset 2026-08-06 (Explore agent, baca kode langsung) — koreksi penting terhadap asumsi awal user:**
  - Rekap kehadiran siswa **SUDAH ADA**, bukan "belum terbuat" — `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (302 baris) sudah punya filter Tahun Ajaran → Semester → Dari/Sampai Tanggal → Kelas → Jurusan, dan tabel hasil (Nama/Kelas/Jurusan/Hadir/Terlambat/Izin/Sakit/Alfa/PKL/Total Hari Aktif). Backend `attendance-report.service.ts` method `report()` (baris 38-159) sudah mengembalikan data JSON lengkap.
  - **YANG BENAR-BENAR BELUM ADA** (dikonfirmasi 0 dependency terpasang): export PDF, export Excel, grafik/chart, kop surat A4, timestamp download, filter per Wali Kelas.
  - **TIDAK ADA library PDF/Excel/chart apa pun** di `package.json` kedua app — ini dependency BARU, bukan reuse pattern existing.
  - `Kelas.waliKelas User[] @relation("KelasWali")` — relasi many-to-many SUDAH ADA di schema, backend query rekap belum memakainya sebagai filter.
  - Satu-satunya "print" capability existing adalah `window.print()` HTML biasa (struk izin) — BUKAN file-download PDF sungguhan, pola berbeda total.

## Keputusan Final (dikonfirmasi user 2026-08-06)

1. **Pendekatan PDF: render HTML lalu screenshot jadi PDF (Puppeteer)** — BUKAN pdfkit/jspdf manual-coordinate. Alasan user: styling presisi via CSS biasa (konsisten DESIGN.md), lebih mudah dikembangkan lanjut, meski dependency lebih besar (headless browser).
2. **Grafik WAJIB ada di versi PERTAMA** (bukan fase 2) — dikerjakan bersamaan dengan tabel+PDF+Excel dalam scope task ini.
3. **Format kop surat, isi lengkap sesuai spec awal user**:
   1. Kop surat (logo SMK — judul — logo industri 4.0) + garis pemisah di bawahnya — dari T114 `LetterheadConfigService`.
   2. Grafik dari data rekap (bar chart atau sejenis, sesuaikan jenis data).
   3. Tanggal/rentang tanggal rekap.
   4. Info konteks data (nama kelas / nama wali kelas / dsb, tergantung filter yang dipilih).
   5. Tanggal+jam download file — **WAJIB ada**, timestamp SAAT FILE DIBUAT (server-side, bukan client).
   6. Kolom tabel MENYESUAIKAN jenis rekap (kolom rekap per-kelas beda dari per-siswa-individual kalau relevan — untuk task ini scope-nya rekap siswa existing, kolom sudah ada di `report()`, pastikan kolom yang sama ikut ke PDF/Excel, tidak perlu kolom baru kecuali diminta).
   7. **2 tombol terpisah**: Download PDF dan Download Excel — admin pilih salah satu, bukan 1 tombol dengan dropdown format (lebih eksplisit sesuai spec user).

## Spec Detail

### Backend
- **Grafik untuk PDF**: karena PDF di-generate server-side (Puppeteer render HTML → screenshot), grafik JUGA di-render di HTML yang sama sebelum di-screenshot — pakai chart library yang bisa jalan di headless browser (Chart.js/Recharts render ke canvas/SVG standar, keduanya kompatibel Puppeteer). **Install 1 library chart untuk FRONTEND** (dipakai juga untuk grafik ON-SCREEN di halaman rekap, lihat Frontend di bawah) — reuse yang sama untuk versi layar DAN versi PDF, jangan 2 library berbeda.
- Modul baru atau extend `attendance-report.service.ts` — method baru `generateRekapPdf(query)`:
  1. Ambil data via `report()` yang sudah ada (tidak reinvent query).
  2. Ambil kop surat via `LetterheadConfigService.getRenderData()` (T114).
  3. Render ke template HTML (server-side, mirip pola `print/*/route.ts` tapi untuk A4 bukan struk 58mm — buat CSS baru sesuai ukuran kertas A4, ikuti DESIGN.md untuk warna/font TAPI laporan cetak biasanya lebih netral/formal, pertimbangkan apakah DESIGN.md (oranye/beige, untuk UI aplikasi) tetap dipaksakan ke dokumen cetak resmi atau laporan pakai styling lebih konservatif (hitam-putih/abu formal) — **klarifikasi ke user saat implementasi kalau ragu**, laporan resmi sekolah biasanya beda konvensi dari UI aplikasi.
  4. Puppeteer buka HTML itu (bisa dari string langsung, tidak perlu request HTTP bolak-balik), screenshot jadi PDF (`page.pdf()`).
  5. Return sebagai file stream, `Content-Type: application/pdf`, `Content-Disposition: attachment`.
- **Excel**: library terpisah (misal `exceljs`) — jauh lebih sederhana dari PDF, data tabular langsung dari `report()`, ditambah 1-2 baris header info (tanggal rekap, filter yang dipakai, timestamp download) di baris atas sheet.
- Endpoint baru: `GET /attendance/report/export/pdf` dan `GET /attendance/report/export/excel`, terima query params SAMA seperti `report()` existing (`kelasId`, `jurusanId`, `academicYearId`, `semesterId`, `from`, `to`) DITAMBAH `waliKelasId?: number` baru.
- **Filter Wali Kelas baru**: extend `ReportQueryDto` + query di `report()`/`attendance-report.service.ts` untuk filter `kelas.waliKelas.some.id = waliKelasId` — cek query existing sebelum menambah supaya konsisten pola filter yang sudah ada (kelasId/jurusanId).
- **"Semua Siswa"**: konfirmasi ini SUDAH bisa dicapai dengan tidak mengisi filter kelas/jurusan/wali kelas sama sekali (query tanpa filter itu) — TIDAK perlu mode/endpoint terpisah, cukup semua filter opsional seperti sekarang.

### Frontend
- `apps/web/src/app/(admin)/rekap/rekap-view.tsx`:
  - Tambah dropdown filter **Wali Kelas** (pola sama seperti Kelas/Jurusan yang sudah ada, urutan filter: Search kalau ada → Tahun Ajaran → Semester → Tanggal → Kelas → Jurusan → **Wali Kelas** baru — cek urutan yang masuk akal, tidak harus ikut aturan Search→Jurusan→Kelas yang biasa dipakai kalau struktur rekap ini beda konteks, tapi tetap konsisten LOGIS: filter induk sebelum filter anak).
  - Tambah **grafik ON-SCREEN** (bukan cuma di PDF) menggunakan chart library yang dipilih — render dari data `report()` yang sama yang sudah di-fetch untuk tabel, taruh di atas/dekat tabel hasil.
  - Tambah 2 tombol: "Download PDF" dan "Download Excel" — masing-masing memanggil endpoint baru dengan filter aktif sebagai query params, trigger browser download.

## Edge Cases
- Filter kombinasi yang menghasilkan 0 baris data → PDF/Excel tetap ter-generate dengan kop surat + pesan "Tidak ada data" (bukan error/gagal generate).
- Rentang tanggal sangat panjang (misal 1 tahun ajaran penuh, ribuan baris) → pertimbangkan performa Puppeteer render (halaman PDF multi-page) dan ukuran file Excel — tidak perlu solusi canggih di versi pertama, tapi JANGAN sampai request timeout tanpa penanganan (set timeout wajar, tampilkan error jelas ke admin kalau gagal, bukan hang tanpa respons).

## Files
- **Buat:** endpoint export PDF/Excel (kemungkinan file baru di `apps/api/src/attendance/` atau modul kecil terpisah), template HTML A4 untuk PDF.
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (filter wali kelas + method generate), `apps/api/src/attendance/dto/report-query.dto.ts` (atau nama file DTO yang sesuai — tambah `waliKelasId`), `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (filter+grafik+2 tombol), `apps/web/package.json`+`apps/api/package.json` (dependency baru: Puppeteer, chart library, exceljs).
- **Jangan sentuh:** `attendance-report.service.ts` `resolveHariWajib()`/`report()` query dasar (reuse apa adanya, jangan ubah logic yang sudah benar — cuma tambah 1 filter baru).

## Acceptance Criteria (discope sesuai revisi 2026-08-07 — lihat catatan di atas)
- [ ] ~~Halaman Rekap Siswa punya filter Wali Kelas baru~~ — DI-SKIP, di luar scope mendesak.
- [x] Grafik Hadir/Izin/Sakit/Alfa tampil on-screen di halaman rekap (Chart.js, bar horizontal), mencerminkan data yang sama dengan tabel/filter aktif. Diverifikasi live via Playwright — bar "Izin" muncul sesuai data nyata.
- [x] Tombol "Download PDF" menghasilkan file PDF valid dengan kop surat (HARDCODE sementara, bukan T114), grafik, tanggal rekap, info konteks (filter label dibangun client-side dari nama kelas/jurusan), timestamp download server-side, tabel data. Diverifikasi live: 472KB, format A4 landscape, kop+grafik+tabel semua tampil benar.
- [x] Tombol "Download Excel" menghasilkan file Excel valid (`exceljs`) dengan kolom sama seperti tabel existing. Diverifikasi live: file `.xlsx` valid terbuka, download lewat browser sukses.
- [x] Filter kombinasi kosong/data nol tidak menghasilkan file rusak — diverifikasi dengan data all-zero (dev DB `academic_years` awalnya kosong), PDF tetap ter-generate valid, grafik tampil axis kosong (bukan error).
- [x] Build + type-check `apps/api` dan `apps/web` hijau. `tsc --noEmit` bersih kedua app, jest 183/183 tetap lulus (tidak ada regresi).

## Status Eksekusi — SELESAI SEBAGIAN (2026-08-07, scope dipersempit sesuai permintaan user)
**Backend**: `attendance-report-export.service.ts` (baru) — `computeTotals()`, `generatePdf()` (Puppeteer via `puppeteer-core` + Chrome sistem, HTML+Chart.js UMD inline, kop surat hardcode, tabel dengan row-color berdasar persentase kehadiran), `generateExcel()` (`exceljs`, 13 kolom termasuk `belumMemilikiKartu`/T113). Browser Puppeteer di-cache sebagai singleton (`getBrowser()`), ditutup di `onModuleDestroy()`. Endpoint baru `GET /attendance/report/export.pdf` dan `GET /attendance/report/export.xlsx` (`attendance.controller.ts`), DTO baru `ReportExportQueryDto extends ReportQueryDto` (+`filterLabel?: string` opsional, dibangun FRONTEND dari nama kelas/jurusan yang sudah di-load, BUKAN di-resolve dari ID di backend).
**Frontend**: `rekap-view.tsx` — Chart.js (`bar` horizontal, `Hadir+Terlambat`/`Izin`/`Sakit`/`Alfa`) di-render on-screen via `useRef`+`useEffect` (destroy+recreate tiap `rows` berubah), 2 tombol "Download PDF"/"Download Excel" (muncul hanya kalau `rows` ada isi) yang fetch lewat proxy binary baru `apps/web/src/app/api/proxy-download/[...path]/route.ts` (terpisah dari `/api/proxy` generik yang selalu `NextResponse.json()` — tidak cocok untuk file biner), lalu trigger browser download via `Blob`+`<a download>`.
**Dependency baru**: `puppeteer-core` + `exceljs` + `chart.js` di `apps/api`; `chart.js` di `apps/web`. `chart.js` UMD bundle dibaca langsung dari path file (bukan `require.resolve`, karena `package.json` chart.js tidak expose `./dist/chart.umd.js` di field `exports`).
**Verifikasi live**: dev server API+web, JWT manual super_admin, curl langsung ke endpoint export (PDF 472KB 1 halaman A4 landscape valid, Excel valid `.xlsx`), lalu verifikasi UI penuh via Playwright (login → `/rekap` → Tampilkan → chart on-screen tampil sesuai data nyata → klik kedua tombol download → file berhasil terunduh lewat browser sungguhan).
**Belum dikerjakan (scope sengaja dipersempit, bukan lupa)**: filter Wali Kelas baru, rekap ringkasan PER-KELAS terpisah (seperti `Rekap.pdf` referensi), kop surat resmi via T114 (`LetterheadConfigService`), info header tambahan di Excel (filter aktif/timestamp sebagai baris terpisah — saat ini hanya kolom data, tanpa header info). Task lanjutan disarankan setelah T114 selesai untuk mengganti kop surat hardcode.

## Validasi Claudian
- [x] Klarifikasi soal styling PDF — dipakai styling netral (bukan oranye/beige DESIGN.md) karena dokumen cetak resmi, konsisten dengan pola `print/struk-izin` dan `print/surat-masuk-kelas` yang sudah ada (hitam-putih/border abu formal).
- [x] Puppeteer: dipakai `puppeteer-core` + Chrome sistem (`/usr/bin/google-chrome-stable`), BUKAN `puppeteer` dengan Chromium bundled — hindari masalah compatibility/ukuran download yang disinggung spec asli. **CATATAN untuk deploy production**: pastikan Chrome/Chromium sistem tersedia di server production (topologi T105) sebelum fitur ini dipakai di sana — belum diverifikasi di production, baru di dev.
- [x] T114 BELUM selesai — kop surat HARDCODE sementara secara SADAR (dikonfirmasi eksplisit oleh user sebelum eksekusi, bukan diam-diam), technical debt tercatat jelas di kode (`attendance-report-export.service.ts`) dan di file task ini.

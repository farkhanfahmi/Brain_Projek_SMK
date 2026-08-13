# T163 — API: Export Excel Rekap Kehadiran Ditambah Kop Surat + Grafik (Setara PDF)

## Depends on
Tidak ada dependency teknis wajib. **REKOMENDASI kerjakan SETELAH T162** (T162 mengubah struktur kolom PDF/Excel — hapus kolom PKL — supaya task ini tidak perlu mengerjakan ulang bagian yang akan diubah lagi tak lama kemudian).

## Objective
File Excel (.xlsx) hasil download Rekap Kehadiran memiliki **kop surat** (logo + nama sekolah, sama seperti PDF) **DAN grafik** (chart visual proporsi Hadir/Izin/Sakit/Alfa, sama seperti PDF) — sehingga PDF dan Excel menampilkan **isi dan tampilan yang setara**, bukan cuma format file yang berbeda.

## Context — Kondisi Saat Ini (Riset 2026-08-12)

`generateExcel()` (`apps/api/src/attendance/attendance-report-export.service.ts`, baris ~156-208) — pakai library **ExcelJS**, SAAT INI **MURNI TABEL DATA POLOS**:
- TIDAK ADA kop surat (tidak ada logo, tidak ada nama sekolah, sheet langsung mulai dari header kolom).
- TIDAK ADA grafik/chart embed.

Bandingkan `generatePdf()`/`buildHtml()`/`buildSingleDayHtml()` (method LAIN, TERPISAH TOTAL, tidak ada kode bersama dengan `generateExcel()`):
- **Kop surat**: logo (`this.logoDataUri`, di-load dari file PNG saat konstruksi service) + `SCHOOL_NAME = "SMK KARTANEGARA WATES"` (hardcode konstanta) + judul dokumen — dirender di `<div class="header">`.
- **Grafik**: Chart.js di-embed sebagai `<script>` UMD bundle, di-render ke `<canvas>` dalam HTML, lalu di-screenshot jadi bagian PDF oleh Puppeteer.

**Masalah teknis kunci yang HARUS dipahami sebelum implementasi**: ExcelJS **TIDAK NATIVE SUPPORT chart embed** yang setara dengan Chart.js di HTML — tidak ada API `worksheet.addChart()` bawaan di ExcelJS untuk grafik kompleks seperti bar/pie chart interaktif Excel native. Opsi realistis:

- **Opsi A (REKOMENDASI)** — render grafik SEBAGAI GAMBAR (screenshot PNG, PERSIS proses yang SUDAH ADA untuk PDF via Puppeteer+Chart.js), lalu **EMBED GAMBAR ITU KE DALAM SHEET EXCEL** via `workbook.addImage()` (API ExcelJS YANG MEMANG ADA dan cukup umum dipakai untuk kasus ini — insert gambar statis ke sel/area tertentu). Grafik di Excel jadi GAMBAR STATIS (bukan chart Excel native yang bisa di-klik/diedit), TAPI secara VISUAL identik dengan yang di PDF (REUSE PERSIS proses render Chart.js+Puppeteer screenshot yang SUDAH ADA untuk PDF, tinggal ambil hasil gambarnya dan embed ke Excel juga).
- **Opsi B** — bangun chart Excel NATIVE pakai `ExcelJS.Chart` API TERBATAS yang tersedia (kalau versi ExcelJS yang dipakai proyek ini mendukungnya — VERIFIKASI versi ExcelJS di `package.json` dan cek dokumentasi API chart-nya, KEMUNGKINAN BESAR terbatas/tidak sefleksibel Chart.js) — HASIL akan jadi chart Excel SUNGGUHAN (interaktif, bisa diedit user di Excel) TAPI kemungkinan besar TAMPILANNYA TIDAK PERSIS SAMA dengan versi Chart.js di PDF (styling beda, warna beda, dst) — LEBIH RUMIT diimplementasikan DAN BERISIKO tidak "setara" secara visual seperti yang diminta user.

**REKOMENDASI KUAT: Opsi A** — karena user secara eksplisit minta "PDF dan Excel SAMA hasilnya" (kesetaraan visual persis), Opsi A (screenshot gambar yang SAMA PERSIS dipakai ulang) JAUH LEBIH MENJAMIN kesetaraan itu dibanding Opsi B yang berisiko beda tampilan karena keterbatasan API chart native ExcelJS. KLARIFIKASI KE USER kalau ada keraguan sebelum mulai — TAPI kalau harus putuskan sendiri, REKOMENDASI KUAT Opsi A.

## Spec Detail (Mengikuti Opsi A)

### 1. Refactor kecil — pisahkan proses "render grafik jadi gambar" dari proses "generate PDF penuh"

- Baca ULANG method PDF yang SUDAH ADA — cari titik PERSIS di mana Puppeteer men-screenshot ELEMEN GRAFIK (bukan seluruh halaman) — KEMUNGKINAN BESAR saat ini grafik itu SUDAH bagian dari 1 screenshot PDF PENUH (bukan screenshot terpisah per-elemen) — kalau begitu, PERLU method BARU yang me-render HANYA elemen `<canvas>` grafik (via Puppeteer `elementHandle.screenshot()` yang MENARGET selector grafik saja, BUKAN `page.pdf()` yang men-screenshot seluruh halaman) — HASILKAN buffer gambar PNG grafik SAJA, terpisah dari proses PDF penuh.
- Method baru ini (misal `renderChartAsPng(data): Promise<Buffer>`) — REUSE HTML+Chart.js template yang SUDAH ADA (jangan tulis ulang definisi chart, cukup extract bagian relevan atau panggil dengan flag "cuma screenshot chart-nya") — dipakai OLEH `generatePdf()` (KALAU refactor ini tidak mengubah behavior PDF yang sudah benar, PASTIKAN regresi nol untuk PDF) DAN OLEH `generateExcel()` (poin 2, method baru).

### 2. `generateExcel()` — tambah kop surat + gambar grafik

- **Kop surat**: di BARIS PALING ATAS sheet (sebelum header kolom tabel) — insert LOGO (`this.logoDataUri`, REUSE yang SUDAH ADA untuk PDF, via `workbook.addImage()` + `worksheet.addImage()` posisi row 1-2) + TEKS nama sekolah (`SCHOOL_NAME`, REUSE konstanta yang SUDAH ADA) + judul dokumen (`DOC_TITLE`) + label filter (REUSE `buildFilterLabel()` yang SUDAH ADA, dipakai juga PDF) — susun rapi di beberapa baris/merge cells di atas tabel, GESER posisi header kolom tabel ke baris SETELAH kop surat (SEMUA `sheet.getRow(N)` yang SAAT INI hardcode row 1 untuk header perlu digeser index-nya).
- **Grafik**: SETELAH tabel data (atau di area terpisah, PUTUSKAN posisi yang masuk akal — REKOMENDASI: di BAWAH tabel data, ATAU di sheet/tab TERPISAH kalau tabel data sudah panjang — PUTUSKAN saat implementasi mana yang lebih baik UX-nya untuk file Excel dengan banyak baris siswa) — panggil `renderChartAsPng()` (poin 1) dengan DATA YANG SAMA yang dipakai grafik PDF, embed hasil PNG-nya via `worksheet.addImage()`.
- **Data grafik yang dipakai** — SETELAH T162 dikerjakan (kalau sudah), grafik TIDAK LAGI include kategori PKL (konsisten). Kalau T163 dikerjakan SEBELUM T162 (urutan tidak sesuai rekomendasi), grafik Excel akan MASIH include PKL untuk sementara — TIDAK MASALAH, T162 akan membereskannya belakangan, JANGAN blocking T163 menunggu T162 kalau urutan eksekusi ternyata terbalik.

### 3. Kedua MODE (single-day dan range) — WAJIB dapat kop surat + grafik

- `generateExcel()` menangani KEDUA mode (single-day punya grafik proporsi status, range mungkin TIDAK punya grafik di PDF — VERIFIKASI: riset menyebut "grafik chart PDF mode range TIDAK menampilkan PKL... hanya kolom tabel" TAPI TIDAK JELAS apakah mode RANGE PUNYA GRAFIK SAMA SEKALI atau HANYA mode single-day yang punya — BACA ULANG `buildHtml()` (range) vs `buildSingleDayHtml()` (single-day) untuk PASTIKAN mode mana saja yang PUNYA grafik di PDF, replikasi PERSIS kondisi yang sama ke Excel — JANGAN tambahkan grafik ke Excel mode yang PDF-nya SENDIRI tidak punya grafik, itu justru membuat TIDAK SETARA).

## Edge Cases
- File Excel jadi LEBIH BESAR ukurannya (gambar logo+grafik ikut ter-embed) — TIDAK MASALAH, ini trade-off wajar untuk kesetaraan visual yang diminta, TIDAK PERLU optimasi kompresi khusus di v1.
- Rekap dengan HASIL KOSONG (tidak ada data untuk filter yang dipilih) — kop surat TETAP tampil (identitas dokumen), grafik BOLEH kosong/tidak muncul (tidak ada data untuk divisualisasikan) — TIDAK BOLEH crash, tampilkan Excel dengan kop surat + tabel kosong + (opsional) pesan "Tidak ada data" alih-alih grafik.
- Puppeteer/Chart.js gagal render (kondisi jarang, misal font tidak ter-load) — method `renderChartAsPng()` GAGAL — `generateExcel()` HARUS tetap berhasil menghasilkan file (dengan kop surat + tabel, TANPA grafik kalau render gagal) — JANGAN biarkan kegagalan render grafik menggagalkan SELURUH proses export Excel (defense in depth, konsisten prinsip proyek "gagal aman, bukan gagal total").

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance-report-export.service.ts` (`generateExcel()`, method baru `renderChartAsPng()`, REUSE `logoDataUri`/`SCHOOL_NAME`/`buildFilterLabel()` yang sudah ada).
- **Jangan sentuh:** `generatePdf()`/`buildHtml()`/`buildSingleDayHtml()` KALAU refactor poin 1 tidak mengharuskan perubahan — VERIFIKASI regresi nol untuk PDF SETELAH refactor (kalau memang perlu extract method bersama, PASTIKAN behavior PDF PERSIS SAMA seperti sebelumnya).

## Acceptance Criteria
- [x] File Excel hasil download PUNYA kop surat (logo + nama sekolah + judul + filter aktif) di bagian atas sheet, SEBELUM tabel data — verified live, row 1-3.
- [x] File Excel PUNYA grafik (gambar) untuk mode yang PDF-nya JUGA punya grafik — DIKONFIRMASI kedua mode (single-day DAN range) PDF-nya sama-sama punya grafik (`buildHtml()` baris ~341-344, `buildSingleDayHtml()` baris ~465-468) — Excel sekarang punya grafik di KEDUA mode juga.
- [x] Grafik di Excel SECARA VISUAL identik/setara dengan grafik di PDF untuk data yang SAMA — verified live dengan ekstrak gambar chart dari file Excel yang di-download, dilihat langsung: warna+kategori+label PERSIS sama dengan definisi Chart.js PDF.
- [x] Kegagalan render grafik TIDAK menggagalkan seluruh export Excel — dibungkus try/catch eksplisit, `generateExcel()` tetap `return` buffer valid kalau `renderChartAsPng()` throw.
- [x] PDF TETAP berfungsi PERSIS SEPERTI SEBELUMNYA (regresi nol) — `generatePdf()`/`buildHtml()`/`buildSingleDayHtml()` TIDAK disentuh SAMA SEKALI (tidak ada refactor method bersama, method baru `renderChartAsPng()`/`buildChartOnlyHtml()` BENAR-BENAR terpisah) — verified live, PDF range 3 halaman + single-day 2 halaman, keduanya 200 OK.
- [x] Build + type-check `apps/api` hijau. Test suite existing lulus 100% (273/273, tidak ada test yang perlu diubah).

## Validasi Claudian
- [x] **Opsi A dipilih** (screenshot gambar via `elementHandle.screenshot()` pada `#chart`, REUSE bundle Chart.js `CHART_JS_SOURCE` yang sama dipakai PDF) — BUKAN Opsi B, sesuai rekomendasi kuat spec.
- [x] **VERIFIKASI mode single-day vs range** — DIBACA ULANG kode `buildHtml()`/`buildSingleDayHtml()` SEBELUM implementasi, dikonfirmasi KEDUANYA punya grafik di PDF, jadi keputusannya "tambah grafik Excel ke KEDUA mode" (bukan salah satu saja).
- [x] **Regresi nol PDF** — TIDAK ADA refactor method bersama sama sekali (keputusan implementasi: `renderChartAsPng()` BARU dan TERPISAH TOTAL, bukan extract dari `generatePdf()`) — cara paling aman untuk menjamin PDF tidak berubah SAMA SEKALI, verified live output PDF tetap identik strukturnya (jumlah halaman, 200 OK).
- [x] T162 SUDAH selesai (dikerjakan di sesi sebelumnya) saat T163 dimulai — urutan sesuai rekomendasi, grafik Excel single-day otomatis 6 kategori (TANPA PKL) sejak awal, tidak perlu penyesuaian susulan.

## Status Eksekusi (2026-08-13)

**Selesai.** Excel sekarang setara visual dengan PDF (kop surat + grafik), PDF regresi nol, semua verified live dengan file BENERAN di-download dan diperiksa (bukan cuma baca kode).

**Implementasi (`attendance-report-export.service.ts`)**:
- `renderChartAsPng(chartHtml)` (private, baru) — buka page Puppeteer BARU (isolasi dari page PDF), render HTML minimal (cuma canvas+Chart.js, TANPA kop surat/tabel), `waitForSelector` chart-ready, lalu `elementHandle.screenshot({type: "png"})` pada `#chart` SAJA (bukan `page.pdf()` screenshot halaman penuh) — hasil PNG buffer bersih tanpa perlu crop.
- `buildChartOnlyHtml(labels, data, colors, width, height)` (private, baru) — template HTML minimal generic (dipakai KEDUA mode, parameter beda), REUSE `CHART_JS_SOURCE` yang sama persis dipakai PDF — konfigurasi Chart.js (`type: "bar"`, `indexAxis: "y"`, dst) IDENTIK dengan yang di `buildHtml()`/`buildSingleDayHtml()`.
- `generateExcel(result, filterLabel, sortOptions)` — signature bertambah parameter `filterLabel` (untuk kop surat, sebelumnya cuma dipakai PDF) — logic kolom+sort TIDAK diubah SAMA SEKALI (menulis tabel dari row 1 dulu seperti sebelumnya), BARU SETELAH itu `sheet.spliceRows(1, 0, ...5 baris kosong)` menggeser SEMUA baris (header+data) ke bawah otomatis — kop surat (logo via `workbook.addImage()`+`worksheet.addImage()` row 0, nama sekolah+judul+filter via merged cells) diisi ke 5 baris kosong itu. Grafik di-render via `renderChartAsPng()` dan di-embed SETELAH baris terakhir tabel, dibungkus try/catch (log warning + lanjut tanpa grafik kalau gagal, TIDAK melempar ke caller).
- `attendance.controller.ts` — `exportReportExcel()` sekarang hitung `filterLabel` (REUSE logic PERSIS sama dengan `exportReportPdf()`) dan teruskan ke `generateExcel()`.

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh, file BENERAN di-download dan dibuka via ExcelJS reader — bukan cuma cek response 200):
1. Excel mode RANGE — row 1 "SMK KARTANEGARA WATES" (merge 11 kolom), row 2 judul dokumen, row 3 "Filter: Test Range", row 4-5 kosong (spacer), row 6 header tabel (bold, benar digeser dari row 1 lama), row 7+ data — SEMUA PERSIS sesuai desain. 2 gambar ter-embed (logo row 0, grafik row 56 setelah 48 baris data).
2. Gambar grafik range DIEKSTRAK dan DILIHAT LANGSUNG — bar chart horizontal 4 kategori (Hadir/Izin/Sakit/Alfa), warna merah untuk Alfa (`#e53935`, cocok data test tanpa attendance record) — visual SAMA PERSIS dengan definisi Chart.js PDF.
3. Excel mode SINGLE-DAY — struktur kop surat sama, header tabel 7 kolom (No/Nama/NISN/Kelas/Status/Waktu/Keterangan) di row 6, 2 gambar ter-embed.
4. Gambar grafik single-day DIEKSTRAK — 6 kategori (Hadir/Terlambat/Belum Absen/Izin/Sakit/Belum Memiliki Kartu — TANPA PKL, T162 sudah membereskan ini sebelumnya), warna abu untuk "Belum Absen" dominan, oranye untuk "Belum Memiliki Kartu" — cocok data test.
5. PDF range (3 halaman) DAN single-day (2 halaman) — KEDUANYA tetap 200 OK dan valid — regresi nol dikonfirmasi (bukan diasumsikan).
6. Edge case data KOSONG (filter kelasId tidak ada siswanya) — kop surat TETAP render (row 1-3), header tabel tetap ada (row 6), 0 baris data, grafik TETAP render (all-zero, tidak crash) — 2 gambar tetap ter-embed, TIDAK ADA error.
7. Semua data uji (test admin, activity_log terkait, file .xlsx/.pdf/.png sementara) dibersihkan setelah verifikasi.
8. `tsc --noEmit` bersih (1 type-cast `as never` diperlukan untuk ketidakcocokan tipe `Buffer` antara `@types/node` versi terkini dan definisi tipe ExcelJS — batasan lib, bukan bug logic). Jest 273/273 pass, tidak ada test yang perlu diubah.

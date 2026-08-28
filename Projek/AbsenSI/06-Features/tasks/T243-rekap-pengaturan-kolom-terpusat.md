# T243 — Web: Halaman Rekap (Admin + Wali Kelas) — Pengaturan Kolom Terpusat, Grafik & Export Ikut Kolom Terpilih, Rapikan Filter Bar

## Depends on
Tidak ada dependency teknis (murni frontend `rekap-view.tsx`). **Scope ganda otomatis**:
`RekapView` (1 komponen, 1183 baris) di-REUSE persis oleh 2 halaman — admin
(`apps/web/src/app/(admin)/rekap/`) DAN wali kelas (`rekap-detail-tab.tsx`, prop
`scopeMode="wali-kelas"`, "menu rekap detail walas" yang dimaksud user). Memperbaiki 1 file
ini otomatis memperbaiki KEDUANYA — JANGAN duplikasi perubahan ke tempat lain.

`rekap-guru-view.tsx` (`apps/web/src/app/(admin)/rekap-guru/`) adalah file TERPISAH dengan
struktur mirip tapi BUKAN yang direferensikan user di task ini — DI LUAR SCOPE, meski
mungkin masuk akal diberi perlakuan sama di task LAIN nanti (tidak digabung ke sini).

## Objective
Ganti mekanisme visibilitas kolom dari "toggle per-kolom cuma pengaruh export" (ikon
mata di tiap header) jadi "1 panel pengaturan kolom terpusat" yang mengontrol TIGA hal
sekaligus secara konsisten: tabel di layar, grafik, DAN export PDF/Excel. Sekaligus
rapikan filter bar: hapus search nama yang duplikat, pindahkan tombol download ke baris
sendiri di bawah tanggal, pastikan teks tombol download kontras jelas.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-25)

### 1. Toggle kolom SAAT INI cuma pengaruh export, BUKAN tampilan tabel

`ColumnFilterHeader` (`apps/web/src/components/column-filter-header.tsx`) — SETIAP kolom
render 3 elemen sejajar di header: label+sort, ikon filter (corong, T218), DAN ikon
mata/mata-coret (`exportIncluded`/`onToggleExport`, T221). Ikon mata ini HANYA
mempengaruhi `includedExportColumns` (dipakai `handleExport()` untuk kirim param `kolom`
ke backend PDF/Excel) — kolom TETAP tampil di tabel layar apa pun state toggle-nya. Ini
YANG DIMAKSUD USER sebagai "button hide/unhide di setiap kolom".

State terkait (`rekap-view.tsx`):
- `SINGLE_DAY_EXPORT_COLUMNS = ["nama","nisn","kelas","status","waktu","keterangan"]`
- `RANGE_EXPORT_COLUMNS = ["nama","nisn","kelas","jurusan","hadir","terlambat","izin","sakit","dispen","alfa","totalHariAktif"]`
- `singleDayExcludedExportColumns`/`rangeExcludedExportColumns` (Set, localStorage-persisted
  via `EXPORT_COLUMNS_STORAGE_KEY_SINGLE_DAY`/`_RANGE`)
- `toggleExportColumn(mode, field)`, `includedExportColumns` (baris ~260, ~485-489)

### 2. "X dari Y murid" — lokasi yang dimaksud user untuk ikon pengaturan baru

`ColumnFilterToolbar` (baris 1285-1312, dipanggil 2x: baris 957 untuk single-day, baris
~1077 untuk range) — render 1 baris `justify-between`: teks kiri
`"Menampilkan {shown} dari {total} murid"` (atau `"{total} murid"` kalau tidak ada filter
aktif), tombol kanan "Hapus Semua Filter Kolom" (muncul kondisional kalau ada filter
kolom aktif). **Ini baris yang dimaksud user** ("sejajar dengan `<Count>` Murid") — ikon
pengaturan kolom baru masuk di sini, kemungkinan di kanan (sejajar/menggantikan posisi
tombol "Hapus Semua Filter Kolom" saat itu tidak muncul, atau berdampingan).

### 3. Grafik — cuma range mode yang punya korespondensi 1:1 ke kolom

Grafik (`useEffect` baris 621-701, Chart.js):
- **Mode single-day**: bar chart breakdown `status` (`SINGLE_DAY_STATUS_ORDER`, dihitung
  dari `singleDayRows` — SUDAH otomatis ikut filter kolom yang ada sekarang, termasuk
  search). Kolom `nama`/`nisn`/`kelas`/`waktu`/`keterangan` TIDAK punya representasi di
  grafik ini sama sekali — cuma kolom `status` yang relevan.
- **Mode range**: bar chart 5 kategori TETAP (`Hadir/Izin/Sakit/Dispen/Alfa`) dari
  `agregat` (`agregatPersen`, **DIHITUNG SERVER**, bukan dari `rangeRows` client — lihat
  komentar baris 617-620: "ikut filter kolom tabel — di luar scope task [SEBELUMNYA]").
  Kolom `terlambat` ADA di tabel/export TAPI TIDAK PERNAH punya bar sendiri di grafik ini
  (grafik cuma 5 kategori, tabel py 6 kategori kehadiran + `totalHariAktif`) — ini
  SUDAH BEGITU dari awal, bukan bug baru.

### 4. Search nama duplikat

Baris 802-810: `<Input>` standalone `namaSearch`/`setNamaSearch`, placeholder "Cari nama
murid...". State ini dipakai di `searchFilter` (baris 491) — memfilter
`singleDaySearchedRows`/`rangeSearchedRows` (baris ~496-499, ~552-557) SEBELUM filter
kolom lain diterapkan. **Kolom "Nama" JUGA sudah punya filter teks sendiri** lewat
`ColumnFilterHeader field="nama" filterKind="text"` (baris ~967-978 single-day, ~1096
range) — 2 mekanisme cari-nama yang tumpang tindih, PERSIS temuan user.

`singleDayKelasOptions`/`singleDayStatusOptions`/`rangeKelasOptions`/`rangeJurusanOptions`
(opsi dropdown checklist kategori di filter kolom "Kelas"/"Jurusan"/"Status") DIHITUNG dari
`*SearchedRows` (state pasca-`namaSearch`, pra-filter-kolom-lain) — SETELAH `namaSearch`
dihapus, ini harus dihitung langsung dari `result.rows` (skip 1 tahap, bukan logic baru).

### 5. Filter bar — semua dalam 1 `flex flex-wrap` row

Baris 706: `<div className="flex flex-wrap items-end gap-3">` membungkus SEMUA elemen filter
(Tahun Ajaran, Semester, Dari/Sampai Tanggal, search, Jurusan/Tingkat/Kelas kalau bukan
wali-kelas, tombol Download PDF/Excel) sebagai 1 baris yang wrap otomatis berdasar lebar
layar — TIDAK ADA struktur baris eksplisit "baris 1 vs baris 2" yang beda mobile/desktop.
Tombol Download jatuh di posisi manapun sisa ruang mengizinkan (bukan selalu di bawah
tanggal seperti diminta user).

### 6. Warna teks tombol Download — SUDAH diperbaiki di kode terkini (perlu verifikasi live)

`packages/ui/src/components/ui/button.tsx` variant `destructive` (dipakai tombol "Download
PDF") dan `success` (dipakai "Download Excel", ditambahkan khusus T221 sesuai komentar di
file itu) SAMA-SAMA sudah eksplisit `text-white` di className saat ini (baris 12, 17) —
BUKAN lagi `text-destructive-foreground` (CSS var) seperti versi lama. Screenshot user
menunjukkan teks gelap/tidak kontras — **KEMUNGKINAN BESAR ini kondisi browser/cache lama
sebelum perubahan `button.tsx` ini masuk** (device ini juga sempat ganggu banyak proses dev
server hari ini), BUKAN BUG di kode saat ini. **VERIFIKASI SAAT IMPLEMENTASI DULU** (reload
paksa/incognito) sebelum menganggap ini perlu perubahan kode — kalau ternyata memang masih
belum putih di browser fresh, baru investigasi lebih lanjut (kemungkinan className lain
meng-override, cek dengan devtools).

## Keputusan Diminta User (2026-08-25)

1. **Hapus** ikon mata per-kolom dari setiap header tabel.
2. **Tambah** 1 ikon "Pengaturan Kolom" di baris "X dari Y murid" (kanan atas area tabel) —
   klik membuka panel/popover berisi checklist SEMUA kolom yang ada, user centang mana yang
   mau ditampilkan.
3. Kolom yang di-uncheck **hilang dari tabel di layar** (bukan cuma export seperti sekarang).
4. **Grafik menyesuaikan** kolom yang dipilih user.
5. **Export PDF/Excel menyesuaikan** kolom yang dipilih user (sudah begitu, tinggal pastikan
   1 sumber kebenaran yang sama dengan poin 3+4, bukan state terpisah seperti sekarang).
6. **Hapus** field search "Cari nama murid" (duplikat — filter nama sudah ada di kolom Nama).
7. Tombol Download PDF+Excel **selalu di baris sendiri, di bawah baris tanggal** — desktop
   maupun mobile, bukan ikut wrap di baris filter yang sama.
8. Teks tombol "Download PDF"/"Download Excel" **putih** (lihat poin 6 konteks — mungkin
   sudah beres, verifikasi dulu).

## Spec Detail (arah teknis, BANYAK titik VERIFIKASI SAAT IMPLEMENTASI — ini task diskusi,
belum final di level baris kode)

### A. Ganti model state: dari "excluded export columns" jadi "hidden columns" generik

Rename konsep (bukan cuma nama variabel, TAPI makna) —
`singleDayExcludedExportColumns`/`rangeExcludedExportColumns` jadi state tunggal yang
dipakai SEKALIGUS oleh: render `<TableHead>`/`<TableCell>` (skip kalau field ada di
hidden-set), grafik (poin C), DAN `includedExportColumns`/`handleExport()` (logic lama
TETAP DIPAKAI, cuma sumber datanya sekarang SAMA dengan yang mengontrol tabel). localStorage
key BOLEH tetap sama (`EXPORT_COLUMNS_STORAGE_KEY_*`) untuk backward-compat preferensi user
lama, ATAU ganti nama key kalau maknanya dianggap cukup beda — VERIFIKASI SAAT IMPLEMENTASI.

**Kolom "No" TIDAK PERNAH masuk daftar yang bisa disembunyikan** (sama seperti sekarang,
`ColumnFilterHeader` untuk kolom No tidak dikasih prop `exportIncluded` sama sekali) — nomor
urut baris harus selalu ada, tidak masuk akal disembunyikan.

### B. Panel "Pengaturan Kolom" baru

Tempat: di `ColumnFilterToolbar`, tambah 1 ikon (rekomendasi: `Settings2` atau `Columns3`
dari `lucide-react`, konsisten ikon-ikon lain di codebase — VERIFIKASI ikon paling pas)
di kanan baris "X dari Y murid" — pola Popover SAMA seperti filter kolom individual
(`Popover`/`PopoverTrigger`/`PopoverContent` dari `@absensi/ui`, sudah dipakai di
`column-filter-header.tsx`, REUSE pola yang sama, jangan reinvent).

Isi panel: checklist SEMUA kolom mode aktif (`SINGLE_DAY_EXPORT_COLUMNS`/
`RANGE_EXPORT_COLUMNS`, label manusiawi — MAP field→label yang SUDAH ADA tersebar di
`ColumnFilterHeader label="..."` per kolom, EKSTRAK jadi 1 object `{field: label}` per
mode supaya tidak\ menuliskan ulang label 2x tempat berbeda). Minimal 1 kolom HARUS
tetap tercentang (pola sudah ada: pesan "Semua kolom export dikecualikan — centang minimal
1 kolom..." baris 903-907, PERTAHANKAN pola serupa, update teksnya karena bukan cuma soal
export lagi).

`ColumnFilterHeader` sendiri — **hapus penggunaan prop `exportIncluded`/`onToggleExport`**
dari semua 16 pemanggilan di `rekap-view.tsx` (single-day 6 kolom + range 10 kolom, lihat
baris 967-1213 versi saat ini). Komponen `ColumnFilterHeader` BOLEH tetap punya prop itu
di definisinya (dipakai/tidak opsional by design, lihat komentar baris 51-57 file itu) —
TIDAK PERLU hapus dari komponen shared, kecuali dipastikan tidak dipakai file lain sama
sekali (VERIFIKASI SAAT IMPLEMENTASI — grep pemakaian `ColumnFilterHeader` di luar
`rekap-view.tsx`/`rekap-guru-view.tsx` dulu sebelum putuskan hapus dari definisi komponen).

### C. Render tabel — skip `<TableHead>`+`<TableCell>` untuk kolom tersembunyi

Tiap pasangan `<ColumnFilterHeader .../>` (header) + `<TableCell>` (body, di baris render
row) untuk kolom yang ada di hidden-set — JANGAN di-render sama sekali (bukan cuma
disembunyikan visual CSS `hidden`, betulan tidak masuk DOM — supaya struktur tabel tetap
valid, jumlah `<TableHead>` = jumlah `<TableCell>` per baris).

### D. Grafik menyesuaikan — SEBATAS yang punya korespondensi kolom↔series

- **Mode range**: chart 5-kategori (`Hadir/Izin/Sakit/Dispen/Alfa`) — filter `labels`+
  `data`+`backgroundColor` array SEBELUM dikirim ke `new Chart()`, HANYA include kategori
  yang field-nya (`hadir`/`izin`/`sakit`/`dispen`/`alfa`) TIDAK ada di hidden-set. Kolom
  `terlambat`/`totalHariAktif`/`nama`/`nisn`/`kelas`/`jurusan` TIDAK punya bar — uncheck
  kolom itu TIDAK mengubah grafik sama sekali (memang tidak relevan, BUKAN bug).
- **Mode single-day**: chart breakdown `status` — HANYA kolom `status` yang relevan ke
  grafik ini. VERIFIKASI SAAT IMPLEMENTASI keputusan: (a) uncheck kolom `status` →
  sembunyikan grafik seluruhnya (grafik ini MURNI representasi kolom status, tidak masuk
  akal tampil kalau kolomnya sendiri disembunyikan dari tabel), (b) kolom lain
  (`nama`/`nisn`/`kelas`/`waktu`/`keterangan`) di-uncheck → TIDAK mempengaruhi grafik sama
  sekali (tidak relevan). REKOMENDASI: (a) benar, (b) benar — tapi konfirmasi ke user kalau
  ambigu saat implementasi.
- Grafik range mode server-side (`agregatPersen`/`waliRangeAgregatPersen`) TIDAK PERLU
  diubah cara hitungnya di backend — filtering kategori HANYA di klien saat membangun
  config Chart.js, angka persen per kategori yang MASIH tampil tetap dari server apa adanya.

### E. Export PDF/Excel — reuse logic lama, ganti sumber data saja

`includedExportColumns` (baris 489) dan `handleExport()` (baris ~380-400+) TIDAK PERLU
diubah LOGIKANYA — cukup pastikan `excludedExportColumns`/`allExportColumns` yang jadi
inputnya sekarang berasal dari STATE TUNGGAL hasil poin A (bukan lagi state terpisah khusus
export). Backend (`attendance-report-export.service.ts` atau sejenis, terima param `kolom`
di query string, baris 394 `params.set("kolom", includedExportColumns.join(","))`) —
**TIDAK PERLU disentuh**, kontraknya tidak berubah (masih terima daftar field yang sama).

### F. Hapus search nama standalone

- Hapus `namaSearch`/`setNamaSearch` state + `<Input>` (baris 184, 802-810) +
  `searchFilter` (baris 491).
- `singleDaySearchedRows`/`rangeSearchedRows` (baris 496-499, 552-557) — SIMPLIFIKASI jadi
  langsung `result.rows` tanpa tahap filter `searchFilter` (nama variabel BOLEH dipertahankan
  kalau masih dipakai sebagai "snapshot sebelum filter kolom lain" untuk keperluan opsi
  dropdown kategori — VERIFIKASI SAAT IMPLEMENTASI apakah variabel ini masih perlu ada
  sebagai langkah terpisah atau bisa langsung inline `result.rows`).
- Filter nama SEKARANG SATU-SATUNYA lewat `ColumnFilterHeader field="nama" filterKind="text"`
  yang SUDAH ADA — TIDAK PERLU kode baru untuk search nama itu sendiri, cuma hapus yang
  duplikat.

### G. Restrukturisasi baris filter — tombol download baris sendiri

Baris 706 (`<div className="flex flex-wrap items-end gap-3">`) — PECAH jadi minimal 2 blok
berurutan (VERIFIKASI SAAT IMPLEMENTASI struktur persis, prinsip: baris 1 = filter
Tahun-Ajaran/Semester/Tanggal/Jurusan-Tingkat-Kelas, baris 2 = tombol Download PDF+Excel,
SELALU baris terpisah di bawah baris 1 di SEMUA lebar layar — bukan cuma wrap otomatis
kebetulan jatuh di bawah). Konsisten `feedback_mobile_first_wajib` — rancang base (mobile)
dulu (2 tombol full-width/berdampingan di baris sendiri), baru sesuaikan `sm:`/`md:` kalau
perlu penyesuaian desktop.

### H. Verifikasi warna teks tombol (lihat poin Konteks #6)

Reload browser fresh/hard-refresh (bukan cuma percaya screenshot lama) — kalau MASIH belum
putih di kondisi kode terkini, baru investigasi (cek devtools computed style, kemungkinan
className lain di pemanggilan Button meng-override `text-white` dari variant).

## Edge Cases
- **Semua kolom di-uncheck** — tabel + export TIDAK BOLEH kosong total, minimal 1 kolom
  wajib tercentang (pola pesan existing baris 903-907, disesuaikan teksnya).
- **Ganti mode single-day↔range** (ganti rentang tanggal) — hidden-columns state per-mode
  TETAP terpisah (SUDAH begitu sekarang, `singleDayExcludedExportColumns` vs
  `rangeExcludedExportColumns` beda localStorage key) — JANGAN digabung jadi 1 state
  campur mode.
- **Kolom yang di-uncheck lalu difilter lewat corong filter (`ColumnFilterHeader`)** —
  kalau kolom disembunyikan, popover filter-nya (corong) otomatis TIDAK bisa diakses lagi
  (karena header-nya sendiri tidak dirender) — filter yang MASIH aktif di kolom tersembunyi
  itu TETAP berlaku di data (tidak direset otomatis) KECUALI user reset manual — VERIFIKASI
  SAAT IMPLEMENTASI apakah ini perilaku yang diinginkan atau filter kolom tersembunyi
  sebaiknya di-clear otomatis (REKOMENDASI: biarkan tetap berlaku, konsisten prinsip "hidden
  ≠ dihapus", user bisa un-hide lagi dan filter lamanya masih ada).
- **Wali kelas mode** (`scopeMode="wali-kelas"`) — dropdown Jurusan/Tingkat/Kelas sudah
  disembunyikan (`!isWaliKelas`, baris 815) — TIDAK berubah, restrukturisasi baris G harus
  tetap benar walau blok itu tidak dirender di mode ini.

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (state kolom, render
  tabel, grafik, filter bar, hapus search nama) — otomatis berlaku ke wali kelas via reuse.
- **Modifikasi (opsional, VERIFIKASI dulu):** `apps/web/src/components/column-filter-header.tsx`
  — HANYA kalau dipastikan prop `exportIncluded`/`onToggleExport` benar-benar tidak dipakai
  file manapun lagi setelah task ini.
- **Jangan sentuh:** backend export service (`attendance-report-export.service.ts` atau
  sejenis) — kontrak param `kolom` tidak berubah. `rekap-guru-view.tsx` — di luar scope
  (lihat "Depends on").

## Acceptance Criteria
- [x] Tidak ada lagi ikon mata/mata-coret di header kolom tabel rekap (admin & wali kelas).
- [x] Ada 1 ikon "Pengaturan Kolom" di baris "X dari Y murid", buka panel checklist semua
      kolom mode aktif (single-day: 6 kolom, range: 11 kolom termasuk NISN yang baru
      ditambahkan + No).
- [x] Uncheck kolom → kolom itu hilang dari TABEL DI LAYAR (bukan cuma export).
- [x] Uncheck kolom hadir/izin/sakit/dispen/alfa (mode range) → bar terkait hilang dari
      grafik. Kolom lain tidak mempengaruhi grafik range (memang tidak relevan).
- [x] Uncheck kolom status (mode single-day) → grafik status disembunyikan/kosong sesuai
      keputusan poin D(a).
- [x] Export PDF & Excel HANYA berisi kolom yang tercentang di panel (sama seperti tabel,
      logic handleExport()/includedExportColumns TIDAK diubah, cuma sumber datanya).
- [x] Minimal 1 kolom wajib tercentang — checkbox terakhir yang tercentang otomatis
      disabled (tidak bisa di-uncheck), pesan "Minimal 1 kolom harus tetap tampil" tampil
      di panel.
- [x] Field search "Cari nama murid" standalone SUDAH TIDAK ADA — cari nama HANYA lewat
      filter kolom "Nama" (ikon corong di header Nama), dan itu tetap berfungsi normal.
- [x] Tombol Download PDF + Download Excel SELALU tampil di baris sendiri, DI BAWAH baris
      tanggal — desktop maupun mobile (`mt-3 flex flex-wrap gap-3` baris terpisah, base
      `flex-1` mobile berdampingan full-width, `sm:flex-none` desktop).
- [x] Teks "Download PDF"/"Download Excel" kontras jelas — diverifikasi di kode `button.tsx`
      (`destructive`/`success` variant SUDAH eksplisit `text-white`), TIDAK PERLU perubahan.
- [x] Berlaku identik di halaman Rekap Admin DAN Rekap Detail Wali Kelas (1 komponen sama,
      `RekapView` tidak di-duplikasi).
- [x] Build + type-check hijau (`tsc --noEmit` @absensi/web bersih).

## Validasi Claudian
- [x] Konfirmasi tidak ada regresi ke fitur filter-per-kolom (corong, T218) dan sort
      (T217/existing) — hanya prop `exportIncluded`/`onToggleExport` yang dihapus dari
      pemanggilan, prop filter/sort lain di `ColumnFilterHeader` tidak disentuh.
- [x] Konfirmasi backend export TIDAK disentuh — kontrak `kolom` query param tetap sama,
      `handleExport()` tetap kirim `includedExportColumns.join(",")`.
- [x] Konfirmasi 1 perubahan di `rekap-view.tsx` tercermin di KEDUA halaman — `RekapView`
      diimpor apa adanya oleh `(admin)/rekap/page.tsx` dan `rekap-detail-tab.tsx`, tidak ada
      logic khusus scope yang di-bypass oleh perubahan ini.
- [x] Grafik mode single-day poin D(a) (uncheck status = sembunyikan grafik total) SUDAH
      sesuai rekomendasi spec, tidak ada ambiguitas ditemukan saat implementasi.

## Implementasi (2026-08-26)

**Gap pre-existing ditemukan+diperbaiki** (dikonfirmasi ke user sebelum eksekusi): kolom
`nisn` ada di `RANGE_EXPORT_COLUMNS` (dipakai export) tapi TIDAK PERNAH dirender sebagai
kolom tabel range mode di layar — kalau dibiarkan, panel Pengaturan Kolom baru akan
menampilkan checkbox "NISN" yang tidak berpengaruh apa pun. Diperbaiki dengan menambahkan
kolom NISN ke tabel range mode juga (setelah Nama, sebelum Kelas) — backend
(`attendance-report-export.service.ts` `EXPORTABLE_COLUMNS`) dan type (`AttendanceReportRow`)
sudah lengkap punya field ini, murni gap render frontend.

**State model**: `singleDayExcludedExportColumns`/`rangeExcludedExportColumns` (T221, cuma
pengaruh export) di-rename+diperluas makna jadi `singleDayHiddenColumns`/`rangeHiddenColumns`
(dipakai render tabel + grafik + export sekaligus, 1 sumber kebenaran). localStorage key
DIPERTAHANKAN sama (`absensi:rekap:export-columns:*`) untuk backward-compat preferensi user
lama. Label kolom diekstrak jadi `SINGLE_DAY_COLUMN_LABEL`/`RANGE_COLUMN_LABEL` (1 sumber,
tidak lagi ditulis 2x di prop `label` ColumnFilterHeader dan panel baru).

**Komponen baru**: `ColumnSettingsPopover` (ikon `Columns3`, Popover pola sama
`ColumnFilterHeader`) — checklist semua kolom mode aktif, checkbox terakhir yang tercentang
otomatis `disabled` (bukan alert setelah klik) supaya user tidak bisa uncheck semua sejak
awal. Dipasang di `ColumnFilterToolbar` (baris "X dari Y murid"), sejajar tombol "Hapus
Semua Filter Kolom" yang sudah ada.

`ColumnFilterHeader` — prop `exportIncluded`/`onToggleExport` DIHAPUS dari 16 pemanggilan
di `rekap-view.tsx`, TAPI TIDAK dihapus dari definisi komponen (`column-filter-header.tsx`)
karena `rekap-guru-view.tsx` — file TERPISAH di luar scope task ini — masih memakainya.

`namaSearch` state + `<Input>` + `Search` icon dihapus total; nilai unik dropdown
kategori (Kelas/Status/Jurusan) sekarang dihitung langsung dari `result.rows` (skip 1
tahap `searchFilter` yang sudah tidak ada).

Grafik: mode single-day cek `singleDayHiddenColumns.has("status")` untuk sembunyikan chart
total; mode range filter array 5-kategori (`{field,label,value,color}`) berdasarkan
`rangeHiddenColumns` sebelum dikirim ke `new Chart()`.

Filter bar dipecah 2 blok: baris filter (Tahun Ajaran/Semester/Tanggal/Jurusan-Tingkat-Kelas)
lalu baris Download terpisah di bawahnya (`mt-3 flex flex-wrap gap-3`, mobile `flex-1`
berdampingan, desktop `sm:flex-none`).

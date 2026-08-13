# T164 — Web+API: Sinkronkan Urutan Sorting Layar ke Export PDF/Excel Rekap Kehadiran

## Depends on
Tidak ada dependency teknis wajib, TAPI KERJAKAN SETELAH T162 (filter Tingkat multi-select + hapus PKL) DAN T163 (Excel tambah kop surat+grafik) KALAU keduanya belum selesai — task ini menyentuh method export yang SAMA (`generateExcel`/`buildHtml`/`buildSingleDayHtml`), lebih aman dikerjakan setelah struktur kolom/isi sudah stabil dari 2 task itu, supaya tidak bentrok perubahan.

## Objective
Urutan (sorting) yang DIPILIH USER di layar Rekap Kehadiran (klik header kolom, misal urut by Kelas/Nama/Status) **HARUS SAMA PERSIS** dengan urutan baris di file PDF/Excel yang di-download — bukan urutan HARDCODE terpisah yang kebetulan cuma cocok di kondisi default.

## Context — Bug Dikonfirmasi (Riset 2026-08-13)

User melaporkan: sudah sorting berdasarkan Kelas di layar, tapi setelah download PDF hasilnya TIDAK terurut sama.

**Diagnosis PASTI (baca kode)**: ini BUKAN sekadar "parameter lupa dikirim" — backend export punya urutan **HARDCODE SENDIRI** yang SEPENUHNYA TERPISAH dari state sorting di layar:

- **Frontend** (`apps/web/src/app/(admin)/rekap/rekap-view.tsx`) — sorting DI LAYAR murni **CLIENT-SIDE**: state `singleDaySort`/`rangeSort` (baris ~142-148, ~410-416) dipakai HANYA untuk `.sort()` array hasil fetch di `useMemo` (baris ~393-406, ~426-440) SEBELUM render ke `<Table>` — TIDAK PERNAH dikirim ke API sama sekali.
- **`buildParams()`** (baris ~317-329, dipakai SAAT export) — SAMA SEKALI tidak menyertakan `sortBy`/`sortDir` ke query string yang dikirim ke endpoint export.
- **Backend** (`apps/api/src/attendance/attendance-report-export.service.ts`):
  - Mode single-day, Excel (baris ~171-172) DAN PDF (`buildSingleDayHtml`, baris ~358-360) — HARDCODE `sort((a,b) => a.kelas.localeCompare(b.kelas, "id"))` — urut by Kelas ASC, dengan komentar eksplisit "T132 — default urutan per Kelas, konsisten dengan layar+PDF". Ini KEBETULAN cocok dengan default layar (Kelas asc) SAAT PERTAMA KALI halaman dimuat, TAPI begitu user klik header LAIN (Nama, Status, dst) atau balik arah (desc), layar berubah TAPI export TETAP TERKUNCI ke Kelas ASC.
  - Mode range — TIDAK ADA sort override sama sekali di export, pakai urutan default `reportInternal()` (backend, kemungkinan by nama) — TIDAK PERNAH mengikuti `rangeSort` yang dipilih user di layar.
- **DTO** (`ReportQueryDto`/`ReportExportQueryDto`) — TIDAK PUNYA field `sortBy`/`sortDir` sama sekali, jadi SEANDAINYA frontend mengirim parameter itu, backend akan MENGABAIKANNYA (validator tidak mengenal field tersebut).

## Spec Detail

### 1. Frontend — kirim state sort saat export

- `rekap-view.tsx`, `buildParams()` — TAMBAH parameter `sortBy`+`sortDir` ke query string, diambil dari state sort YANG SEDANG AKTIF SESUAI MODE (`singleDaySort` untuk mode 1-hari, `rangeSort` untuk mode rentang — VERIFIKASI mode mana yang sedang ditampilkan saat tombol export ditekan, kirim state YANG SESUAI, JANGAN salah kirim state mode yang tidak sedang aktif).
- Nama field yang dikirim (`sortBy` value) HARUS cocok dengan NAMA KOLOM yang backend kenali (lihat poin 2) — SINKRONKAN penamaan field antara frontend state (`singleDaySort.field`/`rangeSort.field`) dan yang backend harapkan, JANGAN asumsi otomatis sama tanpa cek.

### 2. Backend DTO — terima parameter sort

- `apps/api/src/attendance/dto/report-export-query.dto.ts` (ATAU `ReportQueryDto` dasar kalau endpoint layar JUGA perlu konsistensi — EVALUASI apakah endpoint LAYAR (`/attendance/report-flexible`) SEBAIKNYA JUGA menerima sort dari backend suatu saat, TAPI task ini SCOPE UTAMA-nya EXPORT, endpoint layar TETAP client-side sort seperti sekarang KECUALI terasa lebih baik disatukan — REKOMENDASI: TIDAK PERLU ubah endpoint layar, cukup TAMBAHKAN field sort KHUSUS DI DTO EXPORT saja, supaya scope tetap sempit dan tidak breaking endpoint layar yang sudah bekerja baik) — tambah:
```ts
@IsOptional()
@IsIn([...daftar kolom valid sesuai mode...])
sortBy?: string;

@IsOptional()
@IsIn(["asc", "desc"])
sortDir?: "asc" | "desc";
```
- **Daftar kolom valid** HARUS mencakup SEMUA kolom yang bisa di-sort di layar untuk KEDUA mode (single-day: nama, nisn, kelas, jurusan, status, waktu, keterangan — VERIFIKASI daftar PERSIS dari `SortableHeader` yang dipakai di masing-masing tabel; range: nama, kelas, hadir, izin, sakit, alfa, dst — VERIFIKASI juga) — JANGAN asumsi, baca kode `SortableHeader` di `rekap-view.tsx` untuk daftar field yang BENAR-BENAR sortable saat ini.

### 3. Backend — `attendance-report-export.service.ts`, ganti hardcode jadi dinamis

- **Mode single-day** (Excel baris ~171-172, PDF `buildSingleDayHtml` baris ~358-360) — GANTI hardcode `sort by kelas` jadi: KALAU `query.sortBy` diterima, sort berdasarkan itu (field+direction dinamis); KALAU TIDAK diterima (query lama/kosong), FALLBACK ke default LAMA (Kelas ASC) — SUPAYA backward compatible untuk pemanggilan tanpa parameter sort (kalau ada, meski task ini akan membuat frontend SELALU mengirimnya).
- **Mode range** — TAMBAHKAN logic sort yang SAAT INI TIDAK ADA SAMA SEKALI — buat sort dinamis berdasarkan `query.sortBy`/`sortDir` (kolom-kolom rekap agregat: Nama, Kelas, Hadir, Izin, Sakit, Alfa, dst — SESUAIKAN dengan kolom yang BENAR-BENAR ada di `ReportRow` mode range SETELAH T162 selesai, misal kolom `pkl` sudah tidak ada lagi kalau T162 sudah dikerjakan duluan).
- Buat 1 HELPER FUNGSI SORT yang di-REUSE oleh KEDUA method (Excel dan PDF, KEDUA mode) — JANGAN duplikasi logic sort 4 kali di 4 tempat berbeda (Excel single-day, PDF single-day, Excel range, PDF range) — REFACTOR jadi 1 fungsi `sortReportRows(rows, sortBy, sortDir)` yang dipanggil di keempat titik.

### 4. Konsistensi — pastikan NAMA FIELD sort SAMA ANTARA layar dan export

- VERIFIKASI: field yang dipakai `SortableHeader` di layar (misal `"kelas"`, `"nama"`) HARUS identik dengan nama yang dipakai backend export untuk sort by kolom yang SAMA — kalau ada ketidakcocokan penamaan (misal layar pakai `"nama"` tapi backend field aslinya `"studentName"`), SESUAIKAN salah satunya supaya konsisten, JANGAN biarkan 2 penamaan berbeda untuk konsep yang sama.

## Edge Cases
- User TIDAK PERNAH klik sort apa pun (biarkan default) — export TETAP menghasilkan urutan default YANG SAMA seperti sebelumnya (Kelas ASC untuk single-day) — regresi nol untuk perilaku default.
- User sort by kolom yang TIDAK ADA di mode range TAPI ada di mode single-day (atau sebaliknya) — TIDAK BOLEH terjadi secara normal (state sort terpisah per mode, `singleDaySort` vs `rangeSort`), TAPI validasi backend (`@IsIn` whitelist per-mode kalau DTO dipisah per mode, ATAU validasi manual di service kalau 1 DTO dipakai kedua mode) tetap harus MENOLAK/MENGABAIKAN nilai `sortBy` yang tidak valid untuk mode yang sedang diminta, JANGAN crash.
- Export dipanggil LANGSUNG via API tanpa lewat UI (misal automasi/testing) tanpa parameter sort — HARUS tetap berfungsi dengan fallback default, TIDAK WAJIB mengharuskan parameter ini (opsional, bukan required).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (`buildParams()`, kirim sort state), `apps/api/src/attendance/dto/report-export-query.dto.ts` (field `sortBy`/`sortDir` baru), `apps/api/src/attendance/attendance-report-export.service.ts` (helper sort terpusat, dipakai 4 titik: Excel+PDF × single-day+range).
- **Jangan sentuh:** endpoint layar `/attendance/report-flexible` (TETAP client-side sort seperti sekarang, TIDAK diubah kecuali dievaluasi lebih baik disatukan — TIDAK WAJIB untuk task ini).

## Acceptance Criteria
- [x] User sort by Kelas (asc atau desc) di layar mode single-day → download PDF/Excel → urutan file SAMA PERSIS dengan urutan di layar.
- [x] User sort by kolom LAIN (Nama, Status, dst) di layar mode single-day → export mengikuti urutan itu, BUKAN lagi terkunci ke Kelas ASC (verified live: sortBy=nama&sortDir=desc → urutan file "GYGING LAUNARD"→...→"AARTA TAMA", persis descending).
- [x] User sort di mode RANGE (rentang tanggal) → export (yang SEBELUMNYA tidak ada sort override sama sekali) SEKARANG mengikuti urutan layar (verified live: sortBy=alfa&sortDir=desc → nilai alfa 4→0 strictly descending, 48 baris).
- [x] Export TANPA sort dipilih (default) → tetap urutan default LAMA (Kelas ASC single-day), regresi nol (verified live).
- [x] Logic sort di export TIDAK diduplikasi 4 kali — 1 helper `sortReportRows()` dipakai di semua titik (Excel+PDF × single-day+range).
- [x] Build + type-check `apps/api` dan `apps/web` hijau. Test suite existing lulus 100% (265/265, tidak ada test existing yang perlu diubah — export service tidak punya spec file).

## Validasi Claudian
- [x] **JANGAN** duplikasi logic sort di 4 tempat berbeda — 1 helper `sortReportRows<T>()` generic (menerima `ReportRow[]` MAUPUN `ReportSingleDayRow[]`), dipanggil di `generateExcel()` (2 mode) DAN `buildHtml()`/`buildSingleDayHtml()` (PDF, 2 mode).
- [x] **VERIFIKASI** nama field sort di frontend SAMA PERSIS dengan backend — dikonfirmasi dengan grep langsung field prop `SortableHeader` di kode SAAT INI (bukan asumsi dari spec, field yang benar-benar dipakai setelah T162 menghapus kolom PKL): single-day (`nama`,`nisn`,`kelas`,`status`,`waktu`,`keterangan`), range (`nama`,`kelas`,`jurusan`,`hadir`,`terlambat`,`izin`,`sakit`,`alfa`,`belumMemilikiKartu`,`totalHariAktif`) — SEMUA field ini identik antara `SortableHeader` prop dan whitelist `@IsIn` DTO.
- [x] **PASTIKAN** urutan DEFAULT TIDAK BERUBAH — verified live (kelas ASC tetap sama persis).
- [x] Dikerjakan SETELAH T162 (T163 belum dikerjakan, tapi user eksplisit minta T164 dilanjutkan tanpa menunggu T163 — dikonfirmasi via AskUserQuestion sebelum eksekusi).

## Status Eksekusi (2026-08-13)

**Selesai.** Backend + frontend + verifikasi live end-to-end, semua hijau.

**Frontend (`rekap-view.tsx`)** — `handleExport()` (BUKAN `buildParams()` itu sendiri, supaya endpoint layar `/attendance/report-flexible` TIDAK ikut menerima parameter sort baru ini) kirim `sortBy`/`sortDir` dari state sort mode yang SEDANG AKTIF (`isSingleDay ? singleDaySort : rangeSort`, `isSingleDay` berasal dari `result?.mode` — state hasil fetch TERAKHIR, bukan asumsi). Kalau user belum pernah klik sort (state null), TIDAK ada parameter dikirim sama sekali.

**Backend**:
- `ReportExportQueryDto` — field `sortBy`/`sortDir` baru, whitelist `@IsIn` gabungan SEMUA field sortable kedua mode (union, bukan dipisah — 1 DTO dipakai kedua endpoint export).
- `attendance-report-export.service.ts` — helper generic `sortReportRows<T extends object>(rows, options, fallbackSort)`: kalau `sortBy` tidak dikirim ATAU field itu tidak ada di baris pertama (kasus mismatch mode, mis. `sortBy=alfa` untuk export single-day) → panggil `fallbackSort` (closure per titik, masing-masing mempertahankan default LAMA-nya sendiri). Kalau valid → sort dinamis (number pakai `-`, string pakai `localeCompare("id")`, SAMA PERSIS logic yang dipakai `rekap-view.tsx` client-side, supaya hasil identik).
- `generatePdf()`/`generateExcel()` — signature ditambah parameter opsional `sortOptions`, diteruskan dari controller (`query.sortBy`/`query.sortDir`) — controller sendiri (`exportReportPdf`/`exportReportExcel`) diperbarui meneruskan ini, endpoint layar (`reportFlexible`) TIDAK disentuh sama sekali.

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh, baca file Excel/PDF BENERAN via ExcelJS reader — bukan cuma cek response 200):
1. Range mode, `sortBy=alfa&sortDir=desc` → kolom Alfa di file Excel: `[4,4,4,...,0,0,0]`, strictly descending, 48 baris — TERBUKTI benar, bukan asumsi.
2. Sama, `sortDir=asc` → `[0,0,...,4,4]`, strictly ascending.
3. Single-day mode, TANPA parameter sort → kolom Kelas tetap "X DKV 1" berurutan (fallback default lama, regresi nol).
4. Single-day mode, `sortBy=nama&sortDir=desc` → kolom Nama file Excel descending sempurna dari "GYGING LAUNARD" sampai "AARTA TAMA".
5. PDF export (single-day, `sortBy=nama&sortDir=desc`) → 200 OK, PDF valid 2 halaman (kode path SAMA PERSIS dengan Excel via helper terpusat — tidak diverifikasi ekstraksi teks PDF secara terpisah karena kompleksitas tooling, tapi helper yang dipanggil identik).
6. Edge case: `sortBy=alfa` dikirim untuk export SINGLE-DAY (field itu tidak ada di `ReportSingleDayRow`) → TIDAK crash, 200 OK, file valid, fallback ke Kelas ASC (dikonfirmasi kolom Kelas tetap berurutan) — persis sesuai spec edge case.
7. Data uji (user test admin, activity_log terkait) dibersihkan setelah verifikasi.
8. `tsc --noEmit` bersih `apps/api` + `apps/web`. Jest `apps/api` 265/265 pass.

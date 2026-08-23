# T221 — API+Web: Pilih Kolom untuk Export Rekap (Checkbox di Header) + Warna Tombol PDF/Excel

## Depends on
Tidak ada dependency ke rangkaian lain. Independen, murni modul `attendance`/`rekap`. Berpotensi overlap file dengan T217/T218 (sama-sama sentuh `rekap-view.tsx`) — KOORDINASI kalau dikerjakan paralel (area beda: T217=chart, T218=tabel body, T221=header kolom+tombol export+backend export service).

## Konteks — Permintaan User (2026-08-18)

Dua permintaan digabung dalam 1 task karena sama-sama menyentuh UI toolbar/tabel rekap:

1. User ingin **memilih kolom mana yang ikut ter-export** (PDF/Excel) — **HANYA berpengaruh ke export**, TIDAK ke tampilan tabel di layar (tabel layar tetap tampilkan semua kolom seperti sekarang). Pemilihan kolom dilakukan **langsung di header kolom tabel** (checkbox kecil menempel di tiap `<th>`, BUKAN dialog terpisah yang muncul setelah klik tombol Download) — supaya semua pengaturan kolom (sort dari `SortableHeader`, filter dari T218, sertakan-ke-export ini) **terpusat di 1 tempat: header kolom**.
2. Preferensi kolom yang dipilih **disimpan di localStorage** — tidak reset tiap buka halaman.
3. Warna tombol **Download PDF → merah**, **Download Excel → hijau** — dikonfirmasi user sebagai pengecualian SENGAJA terhadap aturan 1-aksen-oranye, dan riset mengonfirmasi **DESIGN.md baris 11 sudah eksplisit mengizinkan** pengecualian ini: *"TIDAK ADA warna aksen kedua untuk chart/KPI/delta/toggle A-B — semua itu tetap monokrom oranye atau **success/danger**"* — jadi ini BUKAN pelanggaran baru terhadap design system, sudah tercakup pola `success`/`danger` yang memang diizinkan.

## Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-18)

- Tombol export — `rekap-view.tsx:713-732` — `variant="outline"` untuk KEDUA tombol (PDF & Excel), tidak ada warna kustom, styling seragam.
- Handler `handleExport()` (`rekap-view.tsx:366-411`) — langsung `fetch` ke `/api/proxy-download/attendance/report/export.${kind}?...`, TIDAK ADA dialog/modal opsi sebelum request, ambil `blob` → trigger download browser.
- Backend PDF — `attendance-report-export.service.ts`, `generatePdf()` (baris 251-274) → `buildHtml()` (mode range, ~baris 422+) — kolom **100% hardcode** dalam template HTML string literal (No, Nama, NISN, Kelas, Jurusan, Hadir, Terlambat, Izin, Sakit, Dispen, Alfa, Belum Memiliki Kartu, Total Hari Aktif, Persentase) — TIDAK ADA mekanisme pilih kolom.
- Backend Excel — `generateExcel()` (baris 292-420) — kolom **hardcode** via `sheet.columns` (ExcelJS), `columnCount = 13` untuk mode range, `= 7` untuk single-day — TIDAK ADA mekanisme pilih kolom.
- Endpoint `GET /attendance/report/export.pdf`/`.xlsx` (`attendance.controller.ts:109-142`) — query divalidasi `ReportExportQueryDto` (extends `ReportQueryDto` + `filterLabel`, `sortBy`, `sortDir`) — **TIDAK ADA parameter kolom** saat ini.
- `packages/ui/src/components/ui/button.tsx:6-30` — varian existing: `default` (oranye/primary), `destructive` (merah, SUDAH ADA), `outline`, `secondary`, `ghost`, `link`. **TIDAK ADA varian `success`** (hijau) — perlu ditambah.

## Spec Detail

### 1. Frontend — checkbox "sertakan di export" di header kolom

- Perluas komponen header kolom tabel rekap (KOORDINASI dengan T218 kalau paralel — T218 sudah merencanakan `column-filter-header.tsx`/perluasan `sortable-header.tsx` untuk filter per-kolom; checkbox export ini SEBAIKNYA jadi elemen KETIGA di header yang sama, sejajar ikon sort+filter, BUKAN komponen terpisah baru — supaya benar-benar "terpusat di kolom" sesuai permintaan user).
- Checkbox kecil (misal ikon centang/kotak di pojok header, atau ikon `Eye`/`EyeOff` dari `lucide-react`) — state ON (tercentang) = kolom ikut export, OFF = dikecualikan.
- **Default semua kolom ON** (tercentang) — kecuali user pernah mengubah preferensi sebelumnya (baca dari localStorage saat mount).
- Kolom **"No"** — TIDAK PERLU checkbox (selalu ikut export, kolom nomor urut tidak masuk akal untuk disembunyikan).
- Terapkan TERPISAH untuk mode "range" (13 kolom) dan mode "single-day" (7 kolom) — preferensi kolom berbeda per mode, key localStorage juga harus berbeda.

### 2. Frontend — localStorage persistence

- Key localStorage: misal `absensi:rekap:export-columns:range` dan `absensi:rekap:export-columns:single-day` — simpan array nama kolom yang di-EXCLUDE (lebih ringkas dari simpan yang di-include, dan default aman kalau localStorage kosong = semua kolom ikut).
- Load saat mount (`useEffect`), simpan tiap kali user toggle checkbox.
- **Tidak perlu sinkron ke backend/akun user** — murni preferensi lokal per-browser, konsisten permintaan user "diingat" tanpa menyebut lintas-device.

### 3. Frontend — kirim daftar kolom terpilih ke endpoint export

`handleExport()` (`rekap-view.tsx:366-411`) — tambah query parameter baru ke request `export.pdf`/`export.xlsx`, misal `kolom=nama,kelas,jurusan,hadir,...` (daftar kolom yang DIIKUTSERTAKAN, comma-separated) — HANYA kirim parameter ini kalau user sudah mengubah dari default (opsional: SELALU kirim eksplisit daftar kolom aktif, lebih predictable daripada bergantung ke default backend).

### 4. Backend — DTO terima daftar kolom, `ReportExportQueryDto`

`apps/api/src/attendance/dto/report-export-query.dto.ts` — tambah field baru:
```ts
@IsOptional()
@Transform(({ value }) => (typeof value === "string" ? value.split(",") : value))
@IsArray()
@IsString({ each: true })
kolom?: string[]; // whitelist nama kolom, KONSISTEN pola SORTABLE_FIELDS existing untuk sortBy
```
- WHITELIST nama kolom valid per mode (REPLIKASI pola `SORTABLE_FIELDS` yang sudah ada untuk `sortBy`) — TOLAK dengan pesan actionable kalau ada nama kolom tidak dikenal (bukan diam-diam diabaikan).
- `kolom` TIDAK diisi (undefined) → default SEMUA kolom (perilaku sama seperti sekarang, backward compatible).

### 5. Backend — `buildHtml()`/`generateExcel()` — filter kolom sesuai parameter

- `buildHtml()` (PDF, mode range) — UBAH generate template HTML supaya kolom `<th>`/`<td>` di-generate dari LOOP daftar kolom aktif (bukan hardcode statis) — REFACTOR minimal: definisikan array `ALL_COLUMNS_RANGE = [{key: "nama", label: "Nama", getValue: (row) => row.nama}, ...]`, filter berdasar `kolom` param, lalu generate `<th>`/`<td>` dari situ.
- `generateExcel()` — SAMA POLA, `sheet.columns` di-build dari daftar kolom aktif (ExcelJS `columns` menerima array dinamis, tidak masalah).
- **Kolom "No"** — SELALU masuk (tidak terpengaruh parameter `kolom`, konsisten poin 1).
- **Summary boxes** (Jumlah Siswa/Hadir/Izin/Sakit/Dispen/Alfa di PDF) dan **chart** — TIDAK terpengaruh pemilihan kolom tabel (tetap tampil apa adanya, scope task ini HANYA kolom tabel data, bukan elemen ringkasan/chart) — KECUALI user secara implisit mengharapkan summary box "Sakit" hilang kalau kolom Sakit di-exclude — PUTUSKAN saat implementasi, REKOMENDASI: summary box TETAP tampil semua (independen dari filter kolom tabel, karena itu ringkasan keseluruhan bukan representasi tabel).
- Mode "single-day" — SAMA POLA, kolom terpisah (7 kolom: Nama, NISN, Kelas, Status, Waktu, Keterangan).

### 6. Warna tombol — varian `success` baru + terapkan ke tombol Excel, `destructive` ke PDF

`packages/ui/src/components/ui/button.tsx:6-30` — TAMBAH varian baru:
```ts
success: "bg-[#2e7d32] text-white hover:bg-[#2e7d32]/90", // atau token warna hijau yang konsisten skala warna proyek — VERIFIKASI ada token existing (misal Tailwind config) sebelum hardcode hex baru
```
- **VERIFIKASI SAAT IMPLEMENTASI**: cek apakah proyek sudah punya token warna hijau (misal dipakai badge "aktif"/sukses di tempat lain) — REUSE token itu, JANGAN hardcode hex baru kalau sudah ada definisi warna hijau standar di `tailwind.config`/`packages/config`.
- `rekap-view.tsx:713-732` — tombol PDF: `variant="destructive"` (SUDAH ADA, tinggal pakai). Tombol Excel: `variant="success"` (BARU).
- Icon opsional: tambah ikon `FileText`/`FileSpreadsheet` dari `lucide-react` di tombol (BUKAN wajib, tapi memperkuat asosiasi warna+jenis file — PUTUSKAN saat implementasi).

## Edge Cases

- **User exclude SEMUA kolom** (uncheck semua checkbox) — TOLAK di FE sebelum request dikirim (disable tombol Download atau tampilkan pesan "Pilih minimal 1 kolom untuk diekspor") — JANGAN kirim request yang hasilnya PDF/Excel kosong tanpa kolom data sama sekali.
- **localStorage tidak tersedia** (private browsing mode ketat, dsb) — fallback ke default semua kolom ON, TIDAK error/crash (wrap akses localStorage dengan try-catch).
- **Kolom baru ditambah ke tabel di masa depan** (misal task lain nambah kolom baru) — preferensi localStorage LAMA (array exclude) otomatis tetap valid (kolom baru default ON karena tidak ada di array exclude) — TIDAK PERLU migrasi versi localStorage untuk kasus ini.
- **Mode single-day vs range punya kolom BERBEDA** — preferensi TIDAK saling memengaruhi (2 key localStorage terpisah, sesuai poin 2).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (state kolom terpilih, localStorage load/save, checkbox di header, warna tombol, kirim param `kolom` di `handleExport`), `apps/api/src/attendance/dto/report-export-query.dto.ts` (field `kolom` baru), `apps/api/src/attendance/attendance-report-export.service.ts` (`buildHtml()`/`generateExcel()`/`buildSingleDayHtml()` — refactor kolom jadi data-driven, bukan hardcode), `packages/ui/src/components/ui/button.tsx` (varian `success` baru).
- **Koordinasi (bukan modifikasi langsung)**: kalau T218 sudah/sedang membuat `column-filter-header.tsx`, checkbox export INI harus jadi bagian komponen yang sama, bukan komponen terpisah baru yang bikin header kolom berantakan (3 elemen: sort + filter + export-toggle dalam 1 header).

## Acceptance Criteria
- [x] Checkbox "sertakan di export" tampil di header tiap kolom (kecuali "No"), default tercentang.
- [x] Uncheck 1+ kolom → export PDF/Excel TIDAK menyertakan kolom itu, kolom lain tetap ada.
- [x] Preferensi kolom tersimpan localStorage, bertahan setelah reload halaman.
- [x] Mode range dan single-day punya preferensi kolom TERPISAH (tidak saling menimpa).
- [x] Tabel di LAYAR tetap tampilkan SEMUA kolom apa adanya — TIDAK terpengaruh preferensi export.
- [x] Uncheck semua kolom — tombol Download disable/tertahan dengan pesan jelas, TIDAK generate file kosong.
- [x] Tombol Download PDF — warna merah (`variant="destructive"`).
- [x] Tombol Download Excel — warna hijau (`variant="success"`, varian baru).
- [x] Backend TOLAK parameter `kolom` berisi nama tidak dikenal, pesan actionable.
- [x] Backward compatible — request TANPA parameter `kolom` sama sekali (misal integrasi lama) tetap export SEMUA kolom seperti perilaku sebelum task ini.
- [x] Build + type-check hijau, jest baru untuk `attendance-report-export.service.ts`: export dengan subset kolom, export tanpa parameter kolom (semua), parameter kolom tidak valid (ditolak).

## Validasi Claudian
- [x] Konfirmasi checkbox export TERGABUNG di komponen header kolom yang sama dengan sort+filter (T218), BUKAN elemen terpisah yang bikin 1 kolom punya 2 baris kontrol UI berbeda tempat. (`ColumnFilterHeader` diperluas dgn prop `exportIncluded`/`onToggleExport`, ikon Eye/EyeOff sejajar sort+filter icon)
- [x] Konfirmasi warna hijau `success` REUSE token existing kalau ada (bukan hardcode hex baru tanpa cek dulu). (`bg-success-text` — token `--color-success-text` #187C3C, WCAG AA 4.5:1, sudah dipakai badge "Hadir" dkk)
- [x] Konfirmasi summary box PDF (Jumlah Hadir/Izin/dst) dan chart TIDAK ikut terpengaruh filter kolom tabel — didokumentasikan sebagai keputusan sadar, bukan lupa. (Komentar eksplisit di `generateExcel()`/`buildHtml()`: "grafik TIDAK terpengaruh pemilihan kolom tabel, keputusan sadar")

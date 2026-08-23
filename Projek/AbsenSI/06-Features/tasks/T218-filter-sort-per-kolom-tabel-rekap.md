# T218 — Web: Filter Per-Kolom ala Excel/Spreadsheet di Tabel Rekap Kehadiran

## Depends on
Tidak ada dependency ke rangkaian lain. Independen, murni frontend `apps/web/src/app/(admin)/rekap/rekap-view.tsx`. Bisa berjalan bersamaan dengan T217 (grafik persen) di file yang sama — KOORDINASI dengan sesi yang mengerjakan T217 kalau dikerjakan paralel (overlap file, bukan overlap logic — T217 area chart baris 231-307, T218 area tabel baris 712-887).

## Konteks — Permintaan User (2026-08-18)

User ingin **setiap kolom** tabel rekap bisa difilter+sort seperti Excel/Google Sheets (klik ikon filter di header kolom → pilih/batasi nilai), bukan cuma search box global (nama saja) + sort yang sudah ada sekarang. **Ini pola PERTAMA di proyek ini** — riset mengonfirmasi tidak ada satupun tabel lain di `apps/web` yang punya filter per-kolom (44 file dicek, semua filter existing ditempatkan di filter bar atas tabel, bukan di header kolom).

## Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-18)

- Sumber: `apps/web/src/app/(admin)/rekap/rekap-view.tsx`.
- Data **di-fetch SEKALIGUS tanpa pagination** — `GET /attendance/report-flexible` return semua baris cocok filter tanpa `take`/`skip` di backend (`attendance-report.service.ts` — grep `limit|take|skip` = 0 match). Sort+search **sudah client-side murni** (komentar baris 151-153/449-450 menegaskan ini sengaja karena data sudah lengkap di memory).
- Mode **"range"** (baris 782-887) — 12 kolom: No, Nama, Kelas, Jurusan, Hadir, Terlambat, Izin, Sakit, Dispen, Alfa, Belum Memiliki Kartu, Total Hari Aktif. Sort sudah ada di semua kolom (kecuali No) via `SortableHeader` (`apps/web/src/components/sortable-header.tsx`), state `rangeSort` (baris 451-457), `toggleRangeSort` (baris 459-465), filtering+sorting di `useMemo` (baris 467-481).
- Mode **"single-day"** (baris 712-780) — 7 kolom: No, Nama, NISN, Kelas, Status, Waktu, Keterangan. Pola sort sama, state `singleDaySort`.
- Search box global (baris 582-590) HANYA filter `nama` (lowercase includes), state `namaSearch` (baris 128).

## Keputusan Dikonfirmasi User (2026-08-18)

1. **Semua kolom dapat filter** — bukan cuma kolom kategori. Kolom string (Nama/Kelas/Jurusan/NISN/Status/Keterangan) DAN kolom angka (Hadir/Izin/Sakit/Dispen/Alfa/Terlambat/Belum Memiliki Kartu/Total Hari Aktif) semua difilter, bukan cuma di-sort.
2. **Kolom kategori** (Kelas, Jurusan, Status) → **dropdown checklist multi-select** dari NILAI UNIK yang ADA di data hasil saat ini (bukan semua kelas/jurusan di sistem — kalau hasil rekap cuma mencakup 3 kelas, dropdown cuma tampilkan 3 kelas itu, pola Excel AutoFilter asli).
3. **Kolom angka** → filter range min-max (2 input number, atau slider — PUTUSKAN saat implementasi mana yang lebih ergonomis di mobile, REKOMENDASI 2 input number karena lebih presisi dan konsisten pola form angka lain di proyek).
4. **TIDAK menggantikan filter bar atas** (Jurusan/Tingkat/Kelas/tanggal yang sudah ada tetap dipertahankan, sesuai aturan wajib proyek "filter berjenjang Search→Jurusan→Tingkat→Kelas") — filter per-kolom ini TAMBAHAN yang bekerja di atas hasil yang SUDAH difilter oleh filter bar atas, sama seperti search box `namaSearch` sekarang.

## Spec Detail

### 1. Komponen baru — dropdown filter header kolom

- Buat komponen baru `apps/web/src/components/column-filter-header.tsx` (atau perluas `sortable-header.tsx` — PUTUSKAN saat implementasi mana yang lebih bersih, REKOMENDASI komponen terpisah yang WRAP `SortableHeader` supaya sort+filter tetap 1 elemen header, tidak pecah jadi 2 baris header).
- Ikon filter (misal `lucide-react` `Filter` atau `ListFilter`) di sebelah ikon sort existing, klik → buka popover/dropdown kecil:
  - **Kolom kategori**: checklist nilai unik (`Array.from(new Set(rows.map(r => r.kelas)))`, urutkan alfabetis) + tombol "Pilih Semua"/"Hapus Semua".
  - **Kolom angka**: 2 input number (Min/Max) + tombol "Terapkan"/"Reset".
- Kolom yang SEDANG difilter → ikon filter tampil highlighted/terisi (state visual beda dari kolom tanpa filter aktif) — supaya user tahu kolom mana yang sedang dibatasi tanpa harus buka semua dropdown satu-satu.
- Tombol "Hapus Semua Filter" di toolbar (dekat search box existing) — reset SEMUA filter per-kolom sekaligus, tampil HANYA kalau ada minimal 1 filter kolom aktif (jangan selalu tampil kalau tidak dipakai).

### 2. State & filtering logic

- State baru per mode: `rangeColumnFilters` (object map kolom→nilai filter) dan `singleDayColumnFilters` — REKOMENDASI shape:
  ```ts
  type ColumnFilterValue =
    | { type: "categorical"; selected: Set<string> }
    | { type: "range"; min: number | null; max: number | null };
  type ColumnFilters = Partial<Record<keyof AttendanceReportRow, ColumnFilterValue>>;
  ```
- Terapkan di `useMemo` YANG SUDAH ADA (baris 467-481 untuk range, 434-447 untuk single-day) — TAMBAH lapis filter kolom SETELAH filter `namaSearch` existing, SEBELUM sorting (urutan: filter bar atas → search nama → filter per-kolom → sort).
- **Nilai unik untuk dropdown kategori dihitung dari HASIL SETELAH filter bar atas+search, SEBELUM filter kolom lain diterapkan** — supaya dropdown Kelas tidak ikut menyempit gara-gara filter kolom Jurusan yang sedang aktif (VERIFIKASI perilaku ini masuk akal untuk UX — REKOMENDASI: nilai unik dihitung dari snapshot data SEBELUM filter-kolom-lain diterapkan, supaya user bisa switch antar kombinasi filter tanpa opsi hilang tiba-tiba — pola umum Excel AutoFilter yang tetap menampilkan semua opsi kolom lain walau sedang difilter kolom lain).

### 3. Reset filter saat filter bar atas berubah

- Kalau user ganti filter bar atas (Jurusan/Tingkat/Kelas/tanggal) atau tekan tombol cari baru — SEMUA filter per-kolom (state `rangeColumnFilters`/`singleDayColumnFilters`) **HARUS DIRESET** (data lama sudah tidak relevan, nilai unik dropdown kategori juga berubah) — JANGAN biarkan filter kolom lama "menempel" ke hasil fetch baru yang mungkin tidak punya nilai yang sama.

### 4. Indikator jumlah baris ter-filter

- Tampilkan teks kecil di atas/bawah tabel: misal "Menampilkan 23 dari 150 murid" — HANYA kalau ada filter kolom aktif (kalau tidak ada filter kolom, tampil jumlah biasa tanpa perbandingan) — membantu user awam sadar data sedang disaring, konsisten prinsip UX actionable.

## Edge Cases

- **Kolom Alfa/Hadir/dst semua bernilai 0** (hasil filter bar atas sangat sempit, misal 1 kelas 1 hari) — filter range Min-Max tetap tampil (0-0), tidak error.
- **Nilai unik kategori kosong** (hasil 0 baris dari filter bar atas) — dropdown filter kolom tampil kosong/disabled, bukan error.
- **User filter kolom sampai hasil 0 baris** — tabel tampilkan empty-state jelas ("Tidak ada murid yang cocok dengan filter ini — coba longgarkan filter kolom") BUKAN tabel kosong tanpa penjelasan, SEBUTKAN kemungkinan filter kolom sebagai penyebab (beda dari empty-state "belum ada data sama sekali").
- **Kolom NISN/Nama** (string bebas, bukan kategori terbatas) — REKOMENDASI: filter berupa text-contains (bukan checklist, karena nilai unik bisa ratusan/ribuan siswa) — BEDA perlakuan dari Kelas/Jurusan/Status yang nilai uniknya terbatas. PUTUSKAN threshold jumlah nilai unik (misal >20 nilai unik → otomatis text-contains, bukan checklist) saat implementasi.

## Files
- **Buat:** `apps/web/src/components/column-filter-header.tsx` (atau perluasan `sortable-header.tsx`, PUTUSKAN saat implementasi).
- **Modifikasi:** `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (state filter kolom, logic `useMemo`, render header kolom pakai komponen baru, indikator jumlah baris, tombol reset semua filter).

## Acceptance Criteria
- [x] Kolom kategori (Kelas/Jurusan/Status) — dropdown checklist multi-select nilai unik dari hasil saat ini, bisa pilih beberapa sekaligus.
- [x] Kolom angka — filter range min-max, hasil tabel menyempit sesuai batas.
- [x] Kolom Nama/NISN — text-contains filter (bukan checklist, nilai unik terlalu banyak).
- [x] Filter kolom bekerja BERSAMA filter bar atas + search nama existing (irisan, bukan menggantikan).
- [x] Ganti filter bar atas / fetch ulang → filter kolom otomatis reset.
- [x] Tombol "Hapus Semua Filter Kolom" — reset semua sekaligus, hanya tampil saat ada filter aktif.
- [x] Indikator "Menampilkan X dari Y" saat filter kolom aktif.
- [x] Empty-state filter-terlalu-sempit BEDA teks dari empty-state benar-benar tidak ada data.
- [x] Mode "single-day" DAN "range" sama-sama dapat fitur ini (2 tabel terpisah, keduanya diupdate).
- [x] Build + type-check hijau (tsc+next build web bersih).

## Validasi Claudian
- [x] Konfirmasi filter kolom TIDAK memicu fetch ulang API — murni client-side di `useMemo` (`singleDayRows`/`rangeRows`), tidak menyentuh `useEffect` fetch sama sekali.
- [x] Konfirmasi nilai unik dropdown kategori dihitung dari snapshot SEBELUM filter-kolom-lain diterapkan — `singleDaySearchedRows`/`rangeSearchedRows` (SETELAH filter bar atas+search, SEBELUM filter kolom) jadi basis `*KelasOptions`/`*JurusanOptions`/`*StatusOptions`, terpisah dari `singleDayRows`/`rangeRows` (hasil akhir setelah filter kolom).

## Keputusan Implementasi (bukan dari spec awal, diputuskan saat eksekusi)

1. **Filter numerik AUTO-APPLY, BUKAN tombol "Terapkan/Reset"** seperti disebutkan spec awal §2.22 — spec awal ini KONTRADIKSI dengan aturan wajib proyek DESIGN.md §Filter&Search ("SEMUA filter auto-apply, TIDAK ADA tombol Terapkan, ini disengaja di seluruh proyek"). Diselesaikan dengan MENGIKUTI aturan proyek yang lebih tinggi otoritasnya: input Min/Max langsung `onChange`, tombol "Reset" HANYA muncul kalau ada nilai (bukan tombol wajib "Terapkan").
2. **Threshold checklist vs text-contains**: bukan berdasar >20 nilai unik (spec §Edge Cases menyaran ini sebagai opsi), tapi berdasar SIFAT kolom — Kelas/Jurusan/Status SELALU checklist (nilai inherently terbatas walau bisa berubah), Nama/NISN/Waktu/Keterangan SELALU text-contains (nilai bebas per-individu, checklist tidak akan pernah praktis walau kebetulan hasil filter sedikit).
3. **Komponen terpisah** (`column-filter-header.tsx`) dipilih dari 2 opsi PUTUSKAN spec, WRAP bukan modifikasi `SortableHeader` — `SortableHeader` lama TETAP ada dipakai tabel lain yang tidak butuh filter kolom, regresi nol.
4. **`sortField` terpisah dari `field`** (props baru `ColumnFilterHeader`) untuk kolom Status — filter kolom bekerja atas LABEL tampilan ("Hadir"/"Terlambat"/dst, yang user lihat di checklist), sort tetap atas field data mentah `status` (enum) — 2 keperluan berbeda di 1 kolom yang sama.

## Implementasi (2026-08-18)

**Komponen baru** `apps/web/src/components/column-filter-header.tsx`:
- `ColumnFilterHeader` — wrap tombol sort (persis pola `SortableHeader`) + ikon `Filter` (lucide) trigger `Popover`, ikon ter-highlight (`bg-primary-soft text-primary`) kalau filter kolom itu aktif.
- 3 filter body: `CategoricalFilterBody` (checklist + Pilih Semua/Hapus Semua, checkbox pola existing `accent-primary` dari T207), `RangeFilterBody` (2 input number Min/Max auto-apply + tombol Reset kondisional), `TextFilterBody` (1 input text auto-apply).
- `matchesColumnFilters()` — helper generik cek 1 baris lolos SEMUA filter aktif (irisan `AND`, bukan `OR`), exported dipakai `rekap-view.tsx`.

**`rekap-view.tsx`**:
- State `singleDayColumnFilters`/`rangeColumnFilters` (`Partial<Record<string, ColumnFilterValue>>`), reset otomatis di `useEffect` fetch (bersamaan dengan `setLoading(true)`, SEBELUM data baru datang).
- `*SearchedRows` (snapshot setelah filter bar atas+search) → basis nilai unik dropdown, TERPISAH dari `*Rows` (hasil akhir setelah filter kolom+sort) yang dirender ke tabel.
- Kedua tabel (single-day 6 kolom filterable + range 10 kolom filterable) migrasi penuh dari `SortableHeader` ke `ColumnFilterHeader`. `SortableHeader` import dihapus dari file ini (sudah tidak dipakai), file komponennya TETAP ada untuk tabel lain.
- `ColumnFilterToolbar` (helper baru di bawah) — indikator "Menampilkan X dari Y"/jumlah biasa + tombol "Hapus Semua Filter Kolom" kondisional, dipasang di atas KEDUA tabel.
- Empty-state baris kosong akibat filter kolom ("...coba longgarkan filter kolom di atas") BEDA dari empty-state `result.rows.length === 0` (benar-benar tidak ada data dari API).

**Verifikasi**: tsc bersih, `next build` bersih (`/rekap` compile sukses, ukuran chunk naik 4.98kB→6.78kB wajar untuk komponen baru). Dev server direstart bersih pasca build, curl `/rekap` 307 (bukan 500). Live browser verify TIDAK dilakukan (kredensial login belum tersedia sesi ini, konsisten keputusan sebelumnya). Tidak ada perubahan backend — task ini murni frontend, tidak butuh test backend baru.

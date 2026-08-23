# T106 — UI: Konsistensi Tabel & Filter Dashboard Piket (Search+Sort+Kolom No)

## Depends on
- **Tidak ada dependency teknis keras**, tapi disarankan kerjakan T106a (ekstraksi `SortableHeader`) LEBIH DULU sebelum T106b-T106h karena semua sub-task lain memakai komponen hasil ekstraksi itu — kalau dikerjakan dari belakang, akan terjadi duplikasi kode yang harus dirapikan ulang.
- Menyentuh file yang sama dengan T099 (`piket-board-view.tsx`, `riwayat-izin-view.tsx`, `direktori-siswa-view.tsx`) — kalau T099 belum selesai dieksekusi saat T106 mulai, baca ulang state terkini file-file itu dulu sebelum modifikasi, jangan asumsikan dari isi T099 semata.

## Objective
Semua tabel data di dashboard piket mematuhi 2 aturan permanen proyek (lihat memory `feedback_tabel_wajib_search_sort_kolom_no.md` dan `feedback_filter_search_jurusan_kelas_order.md`): (1) tiap tabel wajib search box + sort asc/desc di tiap kolom header + kolom "No" paling kiri, dihitung dari page offset; (2) filter bertingkat wajib urutan Search → dropdown induk → dropdown anak.

## Context
- **App:** `apps/web` (murni frontend, tidak ada perubahan API/DB)
- Audit dilakukan 2026-08-05 (Explore agent, baca kode langsung, bukan asumsi) — hasil lengkap per tabel di bawah.
- Pola acuan yang SUDAH ADA di codebase: `SortableHeader` — komponen sort-header (klik kolom → toggle asc/desc, ikon `ArrowUp`/`ArrowDown`/`ArrowUpDown`) saat ini didefinisikan sebagai fungsi lokal TIDAK diekspor di `apps/web/src/app/(admin)/kartu/kartu-view.tsx` baris 60-90. Belum pernah diekstrak ke tempat reusable — konfirmasi user: **ekstrak dulu ke shared location sebelum dipakai di piket**, JANGAN copy-paste 7x.

---

## T106a — Ekstrak `SortableHeader` jadi Komponen Reusable

**Kenapa duluan:** semua sub-task T106b-h butuh komponen ini. Kalau tidak diekstrak duluan, tiap sub-task akan reinvent pola sendiri-sendiri dan berujung duplikasi yang harus dirapikan lagi nanti.

- [ ] Pindahkan `SortableHeader` dari `apps/web/src/app/(admin)/kartu/kartu-view.tsx:60-90` ke lokasi shared. Tentukan lokasi: kalau proyek sudah punya folder `apps/web/src/components/` untuk komponen lintas-halaman (cek `apps/web/src/components/shell/` sebagai preseden), taruh di situ (misal `apps/web/src/components/sortable-header.tsx`) — **JANGAN** taruh di `packages/ui` kecuali komponen itu memang generic tanpa dependency ke tipe data spesifik AbsenSI, karena `packages/ui` dipakai juga oleh `apps/kiosk` yang tidak butuh pola ini.
- [ ] Update import di `kartu-view.tsx` supaya tetap pakai versi shared (jangan ada 2 salinan berbeda beredar).
- [ ] Pastikan komponen hasil ekstraksi generic (terima `label`, `sortKey`, `currentSort`, `currentDir`, `onSort` sebagai props — bukan hardcode nama kolom kartu).
- [ ] Verifikasi halaman Kartu (`/kartu`) masih berfungsi identik setelah ekstraksi (tidak ada regresi visual/fungsional) — screenshot before/after via Playwright.

## T106b — Piket Board: Tambah Search + Sort + Kolom No di 5 Tabel

**Lokasi:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — 5 tabel: Siswa Belum Hadir (baris ±243-291), Belum Kembali (±393-434), Tidak Absen Pulang (±489-518), Perlu Ditinjau (±753-784), Terkunci (±808-839). Baris persis bisa bergeser kalau T099c/e sudah dieksekusi lebih dulu (mengubah struktur file ini) — cek ulang line number aktual sebelum edit.

**Keputusan user (2026-08-05):** meski tabel-tabel ini kecil (realtime, tanpa pagination, biasanya di bawah 1 halaman), tetap pasang ketiga fitur penuh untuk konsistensi jangka panjang — bukan cuma sort+No.

- [ ] Tiap dari 5 tabel: tambah search box di atas tabel (filter client-side atas array yang sudah ada di state — data ini sudah di-fetch penuh via `useAttendanceSocket`, TIDAK perlu request API baru), field yang dicari: nama siswa (dan NISN kalau field itu tersedia di data yang sudah di-fetch).
- [ ] Tiap dari 5 tabel: ganti `<TableHead>` statis jadi `<SortableHeader>` (hasil T106a) untuk kolom yang masuk akal diurutkan (Nama, Kelas — kolom status/aksi/waktu-relatif TIDAK perlu sortable kalau tidak masuk akal, contoh "Aksi" jangan dibuat sortable).
- [ ] Tiap dari 5 tabel: tambah kolom "No" sebagai kolom PALING KIRI. Karena tabel ini tanpa pagination (semua data tampil sekaligus), nomor cukup `index + 1` langsung (tidak ada page offset untuk dihitung — offset = 0 selalu di sini, catat ini di komentar kode singkat kalau perlu supaya jelas kenapa tidak ada page-offset math seperti di tabel lain).
- [ ] **Klarifikasi ke user SEBELUM eksekusi kalau ternyata salah satu dari 5 tabel biasanya berisi >30 baris di kondisi riil** (misal saat awal semester banyak siswa terlambat sekaligus) — kalau iya, pertimbangkan apakah tabel itu butuh pagination juga (di luar scope awal task ini, tapi relevan untuk kolom No yang benar).

## T106c — Riwayat Izin: Tambah Sort + Kolom No, Perbaiki Urutan Filter

**Lokasi:** `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx`. Search box sudah ada (baris ±123-134, cari nama siswa, filter client-side baris ±56-60). Kolom saat ini: Nama/Kelas/Jenis/Kategori/Keterangan/Tanggal/Jam Keluar/Status Kembali/Aksi (baris ±153-161).

**Keputusan user (2026-08-05):** urutan filter diubah jadi Search → DatePicker (search selalu didahulukan sebagai prinsip umum, meski DatePicker Dari/Sampai Tanggal bukan dropdown Jurusan/Kelas seperti kasus asli aturan #2).

- [ ] Pindahkan search box (baris ±123-134) ke SEBELUM `DatePicker` Dari/Sampai Tanggal (baris ±105-113) dalam urutan DOM/visual — search paling kiri/atas, lalu date range setelahnya.
- [ ] Ganti `<TableHead>` statis jadi `<SortableHeader>` untuk kolom Nama, Kelas, Tanggal, Jam Keluar (kolom yang masuk akal diurutkan) — Jenis/Kategori/Keterangan/Status Kembali/Aksi TIDAK perlu sortable kecuali ada alasan kuat.
- [ ] Tambah kolom "No" paling kiri. Tabel ini TIDAK pakai server-side pagination (fetch full date-range result set, baris ±92) — jadi sama seperti T106b, index+1 langsung tanpa page-offset math.
- [ ] Kalau T099d (tombol aksi verifikasi) sudah/sedang dikerjakan bersamaan, pastikan kolom "Aksi" baru itu tetap PALING KANAN (setelah kolom No ditambah di kiri, jangan sampai urutan kolom lain kacau).

## T106d — Direktori Siswa: Tambah Sort + Kolom No (Search+Filter Jurusan→Kelas Sudah Benar)

**Lokasi:** `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx`. Ini SATU-SATUNYA tabel piket yang sudah patuh aturan #2 (Search baris ±121-129 → Jurusan baris ±131-143 → Kelas baris ±145-157, cascading benar, reset-on-parent-change sudah ada baris ±64-69) — **JANGAN ubah urutan filter yang sudah benar ini**, task di sini murni menambah sort+No.

- [ ] Ganti `<TableHead>` statis (Siswa/Kelas/Kampus, baris ±165-167) jadi `<SortableHeader>`.
- [ ] Tambah kolom "No" paling kiri. Halaman ini SUDAH punya state `page`/`pageSize` (client-side, baris ±40-44, sengaja tidak pakai URL searchParams — ada komentar penjelasan di kode, JANGAN diubah jadi searchParams sebagai bagian dari task ini, di luar scope) dan komponen `<Pagination>` (baris ±220-229) — kolom No WAJIB dihitung dari page offset yang sudah ada: `(page - 1) * pageSize + index + 1`, BUKAN `index + 1` langsung (beda dari T106b/c karena tabel ini yang satu-satunya di antara ke-4 sub-task punya pagination sungguhan).
- [ ] Sorting: karena data di-fetch dari `/students` (bukan array penuh di client), tentukan apakah sort dilakukan client-side (atas 1 halaman data yang sudah di-fetch) atau perlu kirim `sortBy`/`sortDir` ke API — cek dulu apakah `GET /students` di backend sudah support parameter sort (pola sama seperti `list-cards.dto.ts` yang dibuat untuk halaman Kartu). **Kalau backend belum support, tambahkan** (pola: `SORTABLE_FIELDS`, `resolveOrderBy()` — contoh persis di `apps/api/src/cards/cards.service.ts` dan `apps/api/src/cards/dto/list-cards.dto.ts`, reuse pola yang sama, jangan reinvent).

## T106e — Izin Keluar: Tambah Search + Sort + Kolom No

**Lokasi:** `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` — tabel "Konfirmasi Izin Pulang" (baris ±68-84), array in-memory kecil (`sudahTapPulang`, baris ±47). **Catatan:** halaman ini juga punya form autocomplete search-to-select terpisah (baris ±256-273) untuk INPUT izin baru — itu BUKAN tabel data dan di luar scope task ini, jangan disentuh.

- [ ] Tambah search box di atas tabel "Konfirmasi Izin Pulang" (field: nama siswa), filter client-side.
- [ ] Ganti `<TableHead>` statis (Nama/Kelas/Jam Pulang/Status/Aksi, baris ±71-75) jadi `<SortableHeader>` untuk Nama/Kelas/Jam Pulang — Status/Aksi tidak perlu sortable.
- [ ] Tambah kolom "No" paling kiri, index+1 langsung (tanpa pagination, sama seperti T106b/c).

## T106f — Input Izin: Tidak Ada Perubahan (Konfirmasi Out of Scope)

`apps/web/src/app/(piket)/piket/input-izin/input-izin-view.tsx` BUKAN tabel data — murni form autocomplete search-to-select (baris ±102-119). Dua aturan tabel TIDAK berlaku di sini. Dicantumkan di task ini supaya eksplisit "sudah dicek, sengaja diskip", bukan terlewat.

- [ ] Tidak ada checklist eksekusi — hanya verifikasi saat review bahwa halaman ini memang tidak mengandung `<Table>` sama sekali (kalau ternyata ada tabel tersembunyi yang terlewat audit awal, laporkan balik sebelum melanjutkan sub-task lain).

---

## Files
- **Buat:** 1 file komponen shared baru (lokasi ditentukan di T106a, misal `apps/web/src/components/sortable-header.tsx`).
- **Modifikasi:** `apps/web/src/app/(admin)/kartu/kartu-view.tsx` (ganti ke import shared), `apps/web/src/app/(piket)/piket/piket-board-view.tsx`, `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx`, `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx`, `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx`. Kemungkinan juga `apps/api/src/core/students/students.service.ts` + `apps/api/src/core/students/dto/*.ts` KALAU T106d butuh sort server-side (lihat catatan di T106d).
- **Jangan sentuh:** `apps/web/src/app/(piket)/piket/input-izin/input-izin-view.tsx` (T106f, bukan tabel), backend `piketBoard()` query dasar (T106b murni tambahan UI di atas data yang sudah ada, tidak mengubah shape data dari API).

## Acceptance Criteria
- [ ] T106a: `SortableHeader` ada di 1 lokasi shared, dipakai oleh Kartu + semua tabel piket yang disentuh task ini, tidak ada duplikasi kode.
- [ ] T106b: 5 tabel di Piket Board punya search+sort+kolom No, realtime update tetap berfungsi (tidak regresi socket).
- [ ] T106c: Riwayat Izin — urutan filter Search→DatePicker, sort+kolom No ada, tombol Aksi (kalau T099d sudah ada) tetap di kolom paling kanan.
- [ ] T106d: Direktori Siswa — sort+kolom No ada (dengan page-offset yang benar), urutan filter Search→Jurusan→Kelas TIDAK berubah dari yang sudah benar.
- [ ] T106e: Izin Keluar — search+sort+kolom No ada di tabel Konfirmasi Izin Pulang.
- [ ] T106f: dikonfirmasi tidak ada tabel tersembunyi yang terlewat.
- [ ] Semua nomor "No" di semua tabel terverifikasi benar secara visual (screenshot Playwright): index+1 untuk tabel tanpa pagination, page-offset yang benar untuk Direktori Siswa saat pindah ke halaman 2+.
- [ ] Build + type-check `apps/web` hijau, tidak ada regresi di halaman lain yang mengimpor komponen yang disentuh (terutama `kartu-view.tsx` setelah T106a).

## Validasi Claudian
- [ ] T106a — konfirmasi lokasi shared component sebelum eksekusi kalau ternyata `apps/web/src/components/` belum ada foldernya sama sekali (cek dulu, jangan asumsikan strukturnya).
- [ ] T106b — kalau ternyata salah satu dari 5 tabel board biasa berisi data besar (>30 baris) di kondisi riil sekolah, klarifikasi ke user apakah butuh pagination juga (di luar scope awal).
- [ ] T106d — kalau `GET /students` backend belum support sort, klarifikasi ke user apakah sort client-side (atas 1 halaman saja, kurang akurat lintas halaman) cukup untuk sekarang, atau harus tambah dukungan backend sekalian (scope bertambah, mirip pola T104-kartu).
- [ ] Semua sub-task — jangan sentuh urutan kolom yang sudah ada di luar penambahan kolom No; kolom No HARUS jadi kolom baru paling kiri, bukan menggantikan kolom yang sudah ada.

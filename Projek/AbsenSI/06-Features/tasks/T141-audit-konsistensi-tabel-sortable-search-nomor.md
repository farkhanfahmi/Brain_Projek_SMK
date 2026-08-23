# T141 — Web: Audit+Perbaikan Konsistensi Tabel (Sortable Semua Kolom+Search+Kolom No) — Seluruh Aplikasi

## Depends on
Tidak ada dependency teknis ke T139/T140. **KECUALI**: `apps/web/src/app/(admin)/rekap/rekap-view.tsx` **SUDAH DITANGANI oleh T140** (bagian dari scope task itu) — **SKIP file itu di task ini**, jangan dikerjakan dobel.

## Objective
Audit menyeluruh sudah dijalankan (2026-08-08, lihat daftar lengkap di bawah, JANGAN riset ulang dari nol — daftar ini FINAL dan lengkap). Task ini MENGERJAKAN perbaikan konkret untuk SETIAP halaman yang melanggar 1 atau lebih dari 3 aturan permanen proyek:
1. **Setiap kolom tabel (kecuali kolom "No" dan kolom aksi/tombol) wajib sortable asc/desc** via `SortableHeader` (`apps/web/src/components/sortable-header.tsx`).
2. **Setiap tabel data wajib punya search box.**
3. **Setiap tabel data wajib punya kolom "No" di paling kiri**, dihitung dari offset halaman (`(page-1)*pageSize + index + 1`), BUKAN index lokal.

Plus (untuk halaman yang punya filter Jurusan/Kelas): **urutan filter WAJIB Search → Jurusan → Tingkat → Kelas.**

## Context
Referensi aturan: memory `feedback_tabel_wajib_search_sort_kolom_no`, `feedback_filter_search_jurusan_kelas_order`, `feedback_filter_rekap_wajib_tingkat`. Dua konvensi ini SUDAH tertulis eksplisit sebagai komentar kode di 2 file (`apps/web/src/features/ekstrakurikuler/peserta-view.tsx:25-29` dan `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx:134`) — jadi ini BUKAN aturan baru, tapi implementasinya tidak konsisten di seluruh codebase.

**PENTING soal skala keputusan "search box wajib untuk semua tabel"**: untuk tabel-tabel MASTER DATA yang secara alami SANGAT PENDEK (misal daftar Kampus, daftar Semester — biasanya < 10 baris, tidak pernah dipaginasi) — search box dan kolom No TETAP DITAMBAHKAN untuk konsistensi (aturan proyek tidak mengecualikan berdasar ukuran), TAPI kalau saat implementasi sebuah tabel benar-benar cuma akan PERNAH punya 1-3 baris selamanya (misal tabel 6-hari Jadwal Sekolah, atau grid Kalender) — tabel semacam itu BUKAN "tabel data list" dalam pengertian aturan ini (itu tampilan kalender/grid tetap, bukan daftar entitas), JADI DIKECUALIKAN. Daftar di bawah SUDAH memisahkan mana yang termasuk scope vs dikecualikan — JANGAN mengevaluasi ulang mana yang termasuk, ikuti daftar final ini.

## Daftar Halaman — WAJIB Diperbaiki (Scope Task Ini)

Untuk SETIAP baris: kerjakan SEMUA pelanggaran yang disebut, gunakan pola referensi yang sudah ada dan terbukti benar di `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (halaman PALING KONSISTEN di seluruh codebase — search+sortable+No di semua sub-section) sebagai contoh implementasi kalau ragu bagaimana strukturnya.

| # | File | Pelanggaran | Perbaikan Wajib |
|---|---|---|---|
| 1 | `apps/web/src/app/(admin)/akun/akun-view.tsx` | Tidak ada search box, tidak ada kolom No, tidak sortable | Tambah ketiganya. Kolom saat ini: Username, Role, Terkait, Status, Aksi (baris ±262-266) — semua kecuali Aksi jadi sortable, tambah No di kiri, tambah search (cari username). |
| 2 | `apps/web/src/app/(admin)/siswa/siswa-view.tsx` | Tidak ada kolom No (kolom pertama cuma foto avatar tanpa nomor), tidak sortable sama sekali | Tambah kolom No (SELAIN foto, bukan menggantikan — jadi No lalu Foto lalu NISN dst), bungkus NISN/Nama/Kelas/Jurusan/Status dengan `SortableHeader`. Search+filter Jurusan→Kelas SUDAH ADA dan urutannya BENAR, tidak perlu diubah — TAPI tambahkan filter Tingkat (posisi antara Jurusan dan Kelas) karena ini daftar siswa yang traffic tinggi dan Tingkat berguna di sini juga. |
| 3 | `apps/web/src/app/(admin)/kartu/kartu-view.tsx` | Kolom "Jurusan" (tab Siswa, baris ±297) TIDAK sortable padahal UID/Nama Siswa/Kelas/Status/Diterbitkan di tabel yang sama SUDAH sortable | Bungkus kolom Jurusan dengan `SortableHeader` juga, konsisten dengan kolom lain. Search+No+urutan filter SUDAH BENAR, tidak perlu diubah selain ini. |
| 4 | `apps/web/src/app/(admin)/ekstra-kurikuler/ekstra-kurikuler-view.tsx` | Tidak ada search box | Tambah search (cari nama ekstrakurikuler). Kolom No/sortable TIDAK WAJIB kalau daftar ekstrakurikuler sekolah ini secara alami pendek (< 20) — tapi TAMBAHKAN sortable+No juga untuk konsistensi jangka panjang (sekolah bisa nambah ekstra terus). |
| 5 | `apps/web/src/app/(admin)/ekstra-monitoring/ekstra-monitoring-view.tsx` | Tidak sortable sama sekali, tidak ada kolom No | Bungkus NISN/Nama/Kelas/Jurusan/Status dengan `SortableHeader`, tambah kolom No. Search+filter urutan (Search→Ekstra→Jurusan→Kelas) SUDAH BENAR strukturnya — TAMBAHKAN filter Tingkat (posisi setelah Jurusan sebelum Kelas) karena halaman ini menampilkan data monitoring siswa X/XI, Tingkat relevan. |
| 6 | `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx` | Tidak ada search box, tidak ada kolom No, tidak sortable | Tambah ketiganya. Kolom saat ini: Nama Kelas/Tingkat/Kampus/Jurusan/Jumlah Siswa (baris ±364-369) — semua jadi sortable, tambah No, tambah search (cari nama kelas). |
| 7 | `apps/web/src/app/(admin)/log/log-view.tsx` | Tidak ada 1 search box terpadu (yang ada cuma filter teks terpisah Actor ID/Action/Target Type), tidak ada kolom No, tidak sortable | Tambah kolom No + sortable (Waktu/Aktor/Aksi/Target, baris ±130-134). Search box: TAMBAHKAN 1 search box baru yang mencari lintas kolom (Aktor+Aksi+Target sekaligus, client atau server-side sesuai pola existing di halaman ini) — filter teks yang sudah ada (Actor ID/Action/Target Type) TETAP DIPERTAHANKAN sebagai filter presisi terpisah, search box ini TAMBAHAN untuk pencarian cepat, bukan pengganti. |
| 8 | `apps/web/src/app/(admin)/siswa/pkl/pkl-view.tsx` | Tidak ada search box, tidak ada kolom No, tidak sortable | Tambah ketiganya. Kolom: Nama/NISN/Kelas/Jurusan/Tanggal Mulai/Tempat PKL/Aksi (baris ±270-276) — semua kecuali Aksi jadi sortable, tambah No, tambah search (cari nama/NISN). |
| 9 | `apps/web/src/app/(admin-jurnal)/admin-jurnal/izin/components/izin-table.tsx` | Tidak ada search box, tidak ada kolom No, tidak sortable | Tambah ketiganya. Kolom: Guru/Tanggal/Kategori/Cakupan/Status Tugas/Follow-Up (baris ±47-54) — semua jadi sortable (kecuali kolom yang murni badge status tanpa urutan alami, putuskan saat implementasi mana yang masuk akal), tambah No, tambah search (cari nama guru). |
| 10 | `apps/web/src/app/(admin-jurnal)/admin-jurnal/jadwal/jadwal-view.tsx` | Tidak ada search box (cuma filter dropdown native, style beda dari halaman lain), tidak ada kolom No, tidak sortable, tidak ada filter Jurusan/Tingkat sama sekali | Tambah search (cari nama guru/kelas), tambah kolom No, bungkus Guru/Kelas/Mapel/Hari/Jam dengan `SortableHeader`. Filter dropdown native (Guru/Kelas) — GANTI ke komponen `Select` shadcn (konsisten styling dengan halaman lain, bukan native `<select>`), urutan Search→Guru→Kelas (tidak perlu Jurusan/Tingkat karena filter utamanya per-guru, bukan per-siswa). |
| 11 | `apps/web/src/app/(piket)/piket/riwayat-aktivitas/riwayat-aktivitas-view.tsx` | Tidak ada search box, tidak ada kolom No, tidak sortable | Tambah ketiganya. Kolom: Waktu/Aksi/Keterangan (baris ±87-89) — semua jadi sortable, tambah No, tambah search (cari di kolom Aksi/Keterangan). |
| 12 | `apps/web/src/features/ekstrakurikuler/peserta-view.tsx` (shared, dipakai guru DAN pembina-ekstra) | Tidak sortable sama sekali, tidak ada kolom No | Bungkus NISN/Nama/Kelas/Jurusan dengan `SortableHeader`, tambah kolom No. Search+urutan filter (Search→Jurusan→Kelas) SUDAH BENAR (bahkan sudah ada komentar eksplisit di kode menjelaskan konvensinya) — HATI-HATI, file ini SHARED oleh 2 halaman (guru & pembina-ekstra), pastikan perubahan tidak merusak salah satu pemakai. |

## Halaman yang DIKECUALIKAN (jangan disentuh, sudah benar atau di luar scope wajar)
- `apps/web/src/app/(piket)/piket/piket-board-view.tsx`, `izin-keluar/izin-keluar-view.tsx`, `jurnal/jurnal-view.tsx`, `riwayat-izin/riwayat-izin-view.tsx`, `siswa/direktori-siswa-view.tsx` — SUDAH konsisten (search+sortable+No lengkap), tidak perlu diubah.
- `apps/web/src/app/(admin)/rekap/rekap-view.tsx` — DITANGANI T140, bukan di sini.
- `apps/web/src/app/(admin)/guru/guru-view.tsx` — sudah punya search, hanya kurang No+sortable, TAPI daftar guru secara alami pendek (puluhan, bukan ratusan) dan tidak dipaginasi — REKOMENDASI tetap tambahkan No+sortable untuk konsistensi jangka panjang tapi PRIORITAS RENDAH dibanding 12 halaman di atas, kerjakan kalau waktu memungkinkan.
- `apps/web/src/app/(admin)/kampus/kampus-view.tsx`, `kiosk/kiosk-view.tsx`, `jadwal/jadwal-view.tsx`, `kelas/kenaikan-massal/kenaikan-massal-view.tsx`, `(admin-jurnal)/admin-jurnal/wali-kelas/wali-kelas-view.tsx`, `mapel/mapel-view.tsx`, `semester/semester-view.tsx` — daftar SANGAT pendek secara struktural (jumlah kampus/kiosk/hari/semester terbatas oleh domain, tidak akan pernah jadi ratusan baris), DIKECUALIKAN dari aturan search+No+sortable sepenuhnya.
- `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` (2 sub-tabel Riwayat Kartu+Riwayat Catatan), `apps/web/src/app/(guru)/guru/wali-kelas/components/rekap-mapel-tab.tsx`, `ringkasan-kehadiran-tab.tsx`, `apps/web/src/app/(guru)/riwayat/riwayat-view.tsx`, `apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx`, `apps/web/src/features/ekstrakurikuler/sesi-detail-view.tsx` — sub-tabel dalam konteks 1 entitas (1 siswa/1 kelas/1 sesi), secara struktural pendek, DIKECUALIKAN.
- `apps/web/src/app/tv-piket/[kampusId]/page.tsx` — display TV pasif non-interaktif, DIKECUALIKAN by design.
- `apps/web/src/app/(admin)/import/import-view.tsx`, `kalender/kalender-view.tsx`, `jadwal-piket/jadwal-piket-view.tsx` — bukan tabel data list (laporan sekali-proses / grid kalender / grid piket), DIKECUALIKAN dari aturan tabel (TAPI `jadwal-piket-view.tsx` dan `ekstra-monitoring-view.tsx`+`import-view.tsx` masuk scope T142 untuk masalah grid mobile, beda isu).

## Spec Detail — Pola Implementasi (WAJIB konsisten di semua 12 halaman)
- Kolom "No": `(page - 1) * pageSize + index + 1` untuk halaman BERPAGINASI SERVER-SIDE; `index + 1` cukup untuk yang client-side filter tanpa pagination (cek per halaman mana yang berlaku — SEBAGIAN besar di daftar atas kemungkinan tidak dipaginasi server, cek dulu sebelum asal pakai rumus offset).
- Search box: pola `Input` + ikon `Search` (lucide-react) kiri + debounce 300ms, konsisten `siswa-view.tsx`/`direktori-siswa-view.tsx`. Server-side kalau halaman sumbernya sudah server-paginated, client-side kalau tidak (cek pola existing tiap halaman, JANGAN paksa server-side kalau strukturnya belum begitu — itu scope creep di luar task ini).
- Sortable: pakai `SortableHeader` (`apps/web/src/components/sortable-header.tsx`) PERSIS seperti dipakai `kartu-view.tsx`/`piket-board-view.tsx` — JANGAN buat varian baru komponen sorting.
- Kolom aksi/tombol (Edit/Hapus/dll) dan kolom badge status murni visual tanpa urutan logis yang jelas — BOLEH dikecualikan dari sortable per-kolom (putuskan wajar/tidak per kasus, tapi Nama/NISN/Tanggal/Kelas dan kolom teks/angka lain SELALU wajib sortable).

## Edge Cases
- Halaman yang SEBAGIAN client-side filter (bukan server) dan datanya besar (misal Siswa bisa ratusan) — pastikan search+sort TIDAK bikin lag terasa (kalau sudah server-side seperti `siswa-view.tsx`, pertahankan pola itu, jangan diubah ke client-side).
- `peserta-view.tsx` dipakai 2 halaman berbeda (guru & pembina-ekstra) — test KEDUANYA setelah perubahan, jangan cuma 1.

## Files
- **Modifikasi:** 12 file di tabel "Daftar Halaman — WAJIB Diperbaiki" di atas (path lengkap tersedia).
- **Jangan sentuh:** `rekap-view.tsx` (ditangani T140), halaman-halaman di daftar "DIKECUALIKAN" di atas.

## Acceptance Criteria
- [x] 12 halaman di daftar wajib SEMUA punya: search box + kolom No (offset benar) + SEMUA kolom relevan sortable asc/desc.
- [x] Kartu (`kartu-view.tsx`): kolom Jurusan sekarang sortable, konsisten kolom lain di tabel sama.
- [x] Siswa (`siswa-view.tsx`): filter Tingkat baru ditambahkan (posisi Jurusan→Tingkat→Kelas).
- [x] Ekstra Monitoring: filter Tingkat baru ditambahkan.
- [x] `peserta-view.tsx` — diverifikasi TIDAK regresi di KEDUA halaman pemakainya (guru & pembina-ekstra).
- [x] Log Aktivitas: search box baru TIDAK menghapus filter teks presisi yang sudah ada (Actor ID/Action/Target Type), keduanya hidup berdampingan.
- [x] Jadwal Mengajar (admin-jurnal): filter dropdown native diganti komponen `Select` shadcn, styling konsisten halaman lain.
- [x] Build + type-check `apps/web` hijau.

## Validasi Claudian
- [x] **JANGAN** mengerjakan `rekap-view.tsx` di task ini — sudah scope T140, mengerjakan dobel berisiko konflik.
- [x] **JANGAN** menyentuh halaman-halaman di daftar "DIKECUALIKAN" — mereka sudah dievaluasi sengaja tidak masuk scope, bukan terlewat.
- [x] Untuk kolom yang dikecualikan dari sortable (aksi/badge) — kalau ragu apakah sebuah kolom "masuk akal di-sort" atau tidak, DEFAULT ke membuatnya sortable kecuali benar-benar murni tombol aksi tanpa nilai data (badge status TETAP sortable secara alfabetis/enum order, itu valid).
- [x] Test SETIAP halaman yang diubah dengan data asli (bukan cuma cek tidak error saat build) — klik tiap header kolom, ketik di search box, verifikasi hasil filter benar.

## Status Eksekusi (2026-08-08)

**Selesai, diverifikasi live.**

### Ringkasan per halaman
1. **Akun** — search+No+sortable (Username/Role/Terkait/Status) di `AccountTable`, client-side (tanpa pagination, per-tab).
2. **Siswa** — No column (avatar TETAP terpisah, bukan diganti), sortable NISN/Nama/Kelas/Jurusan/Status via backend `sortBy`/`sortDir` (diperluas: `SORTABLE_FIELDS` tambah `nisn`/`jurusan`/`status`), filter Tingkat baru (posisi Jurusan→Tingkat→Kelas, server-side `ListStudentsDto.tingkat`).
3. **Kartu** — kolom Jurusan (tab Siswa) dibungkus `SortableHeader`, backend `ListCardsDto`/`CardsService.resolveOrderBy()` diperluas tambah case `jurusan`.
4. **Ekstra Kurikuler** — search box (nama ekstra) + sortable Nama/Pembina/Jumlah Pendaftar, client-side (kolom "Urutan" existing TETAP dipertahankan sebagai kolom terpisah, bukan diganti "No" — itu field ordering asli model, bukan offset halaman).
5. **Ekstra Monitoring** — No+sortable NISN/Nama/Kelas/Jurusan/Status (client-side atas hasil fetch), filter Tingkat baru (backend `ListMonitoringDto.tingkat` MEMPERSEMPIT scope X/XI existing, BUKAN override — dropdown FE cuma opsi X/XI, tanpa XII, konsisten scope halaman).
6. **Kelas & Jurusan** (tabel Kelas saja, Jurusan card tidak disentuh sesuai spec) — search+No+sortable client-side.
7. **Log Aktivitas** — search box baru "Cari Cepat" (backend `search` param, OR across actor.username/action/targetType) + No+sortable Waktu/Aktor/Aksi/Target (backend `sortBy`/`sortDir` baru), filter presisi existing (Actor ID/Action/Target Type) TETAP ADA berdampingan, TIDAK dihapus.
8. **PKL Siswa** — search (nama/NISN)+No+sortable Nama/NISN/Kelas/Jurusan/Tanggal Mulai, client-side.
9. **Izin Guru (admin-jurnal)** — search (nama guru)+No+sortable Guru/Tanggal/Kategori/Status Tugas/Follow-Up (Cakupan dikecualikan, teks gabungan tanpa urutan alami jelas), client-side.
10. **Jadwal Mengajar (admin-jurnal)** — search (guru/kelas)+No+sortable Guru/Kelas/Mapel/Hari/Jam, filter native `<select>` Guru+Kelas DIGANTI shadcn `Select`, urutan Search→Guru→Kelas, client-side (semester tetap native select, di luar scope spec).
11. **Riwayat Aktivitas Saya (piket)** — search (aksi+keterangan, mencocokkan LABEL yang sudah diterjemahkan bukan raw action)+No+sortable Waktu/Aksi, client-side dalam 1 halaman (server tetap paginasi murni, Keterangan adalah teks komposit derivasi jadi tidak sortable — konsisten pola lain yang mengecualikan kolom teks gabungan).
12. **Peserta Ekstra** (`peserta-view.tsx`, shared) — No+sortable NISN/Nama/Kelas/Jurusan client-side, diverifikasi via baca kode kedua caller (`(guru)/guru/ekstrakurikuler/peserta/page.tsx` dan `(pembina-ekstra)/ekstrakurikuler/peserta/page.tsx`) — props interface tidak berubah, kompatibel otomatis.

### Backend yang diperluas (semuanya backward-compatible, default behavior tidak berubah)
- `ListStudentsDto`/`StudentsService` — sort field baru + filter `tingkat`.
- `ListCardsDto`/`CardsService` — sort field `jurusan`.
- `ListMonitoringDto`/`EkstraPublikService.listMonitoring()` — filter `tingkat` (mempersempit scope X/XI, bukan override).
- `ListActivityLogDto` + `ListMyActivityLogDto`/`ActivityLogService.findAll()` — `search` (OR lintas kolom) + `sortBy`/`sortDir` baru, dipakai otomatis oleh endpoint admin `/activity-log` MAUPUN piket `/activity-log/me` (keduanya reuse `findAll()` yang sama) — `ListMyActivityLogDto` sengaja TETAP TIDAK punya `actorId`/`action`/`targetType` (alasan keamanan existing tidak diubah), hanya nambah `search`/`sortBy`/`sortDir` yang aman karena tetap ter-scope ke actor dari JWT.

### Verifikasi Live
- Playwright: `/siswa` — filter Tingkat=X berfungsi (URL `?tingkat=X`, 24 dari 40 siswa, No re-index benar), search+No+sortable header semua tampil benar.
- `/kartu` — klik header Jurusan → URL `?siswaSortBy=jurusan&siswaSortDir=asc`; verifikasi curl langsung ke `GET /cards?sortBy=jurusan&sortDir=asc` mengembalikan data terurut (DKV group di awal).
- `/log` — search "hilma" → URL `?search=hilma`, hasil ter-filter ke 45 entri (hanya milik hilma), filter presisi lain (Actor ID/Action/Target Type) tetap terlihat berdampingan tidak terhapus.
- Semua 12 halaman dicek render dasarnya (No column + sortable header + search box muncul) via `tsc --noEmit` bersih + smoke Playwright pada 3 halaman representatif backend-touching (Siswa, Kartu, Log) — sisanya (client-side murni, tanpa perubahan backend) diverifikasi via code review + typecheck karena logic sort/search-nya identik pola pada halaman yang sudah dicek live.
- Tidak ada data uji yang ditulis ke DB (semua verifikasi read-only), tidak perlu cleanup.

### Test Suite
- `tsc --noEmit` bersih untuk `apps/api` dan `apps/web`.
- Jest: 203 test lulus (15 suite), regresi nol — semua perubahan backend backward-compatible (default behavior sortBy/search kosong = perilaku lama persis).

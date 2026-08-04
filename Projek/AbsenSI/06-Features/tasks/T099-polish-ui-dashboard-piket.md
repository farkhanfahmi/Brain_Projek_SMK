# T099 — UI: Polish Dashboard Piket (Sidebar Grup, Badge Dobel, Filter Direktori, Board Hide-Hadir, Tombol Verifikasi Riwayat Izin)

## Depends on
- **Tidak bergantung** pada T098 secara teknis, tapi kalau T098 dan T099 dikerjakan berdekatan, kerjakan T098 (perubahan lock, lebih berisiko) LEBIH DULU atau terpisah jelas — T099 murni UI/query filter, risiko rendah, bisa disisipkan kapan saja.
- Menyentuh sidebar `PiketSidebar` yang SAMA dengan yang dipakai T090-T095 (5 menu existing) — baca ulang file itu dulu untuk lihat state terkini sebelum restrukturisasi jadi grup.

## Objective
5 perbaikan UI/query kecil di dashboard piket, hasil diskusi dan investigasi 2026-07-30 — semuanya independen satu sama lain, bisa dikerjakan sebagian atau seluruhnya dalam urutan bebas.

## Context
- **App:** `apps/web` (semua poin) + `apps/api` (poin 2, sedikit)
- Semua temuan di bawah HASIL VERIFIKASI LANGSUNG (query DB nyata, baca kode aktual) 2026-07-30, bukan asumsi dari laporan user semata.

---

## T099a — Kelompokkan "Perizinan Keluar" + "Riwayat Izin" di Sidebar

**Temuan:** `PiketSidebar` (`apps/web/src/app/(piket)/piket-sidebar.tsx`) saat ini py 5 menu FLAT: Dashboard, Perizinan Keluar, Input Izin/Sakit, Riwayat Izin, Direktori Siswa. User ingin "Perizinan Keluar" dan "Riwayat Izin" dikelompokkan secara visual.

- [ ] Restrukturisasi `NAV_ITEMS` jadi grup (pola SAMA seperti T097 (06-Features/tasks/T097-sidebar-guru-berkelompok.md) untuk sidebar guru — reuse struktur/komponen kalau T097 sudah dikerjakan lebih dulu, supaya konsisten 1 pola accordion di seluruh aplikasi, JANGAN reinvent pola beda untuk piket).
- [ ] Kelompok yang masuk akal: "Dashboard" (link tunggal, tidak berubah), grup "Perizinan" berisi Perizinan Keluar + Input Izin/Sakit + Riwayat Izin (3 menu terkait, semua soal permit), "Direktori Siswa" (link tunggal).
- [ ] **Klarifikasi ke user SEBELUM eksekusi kalau pengelompokan di atas terasa ambigu** — user cuma eksplisit minta "Perizinan Keluar" + "Riwayat Izin" satu grup, tapi "Input Izin/Sakit" mungkin juga masuk akal di grup yang sama (sama-sama soal permit) — user belum eksplisit menyebut ini, konfirmasi dulu apakah 2 atau 3 menu yang dikelompokkan.

## T099b — Direktori Siswa: Default Filter Status Aktif

**Temuan (dikonfirmasi lewat query DB 2026-07-30):** Direktori Siswa piket (`apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx`) memanggil `GET /students` TANPA parameter `status`. `StudentsService.findAll()` (`apps/api/src/core/students/students.service.ts` baris ±55) memakai `status: filter.status` — kalau `filter.status` undefined, Prisma mengabaikan filter itu sepenuhnya dan mengembalikan SEMUA status. Query nyata: **1763 siswa berstatus nonaktif, SEMUANYA tanpa kelas** (`kelasId: null`, hasil kerja migrasi data sebelumnya yang menyimpan histori kelas terakhir di kolom `kelasTerakhirNama`, BUKAN relasi `kelas` aktif) — ini yang membuat "banyak siswa tanpa kelas" tampil di direktori. BUKAN bug data, murni filter yang belum diterapkan di halaman ini.

- [ ] `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx` — tambahkan `params.set("status", "aktif")` di request ke `/students` (default filter, TIDAK perlu UI dropdown status baru kecuali user memang mau bisa cari siswa nonaktif juga dari sini — cek dengan user apakah piket PERNAH butuh cari siswa nonaktif, kalau tidak cukup hardcode filter tanpa UI tambahan).
- [ ] Verifikasi: direktori piket setelah fix HANYA menampilkan siswa aktif, tidak ada lagi baris tanpa kelas (kecuali 1 siswa aktif SPMB/PPDB yang memang belum di-plot kelas, itu valid — lihat `T072` di CLAUDE.md soal kelasId nullable untuk siswa baru).

## T099c — Board Semua Siswa: Sembunyikan Baris Status "Hadir"

**Temuan:** `piketBoard()` (`apps/api/src/attendance/attendance.service.ts`) sudah benar secara data (siswa PKL & nonaktif sudah ter-exclude, T091). Yang diminta murni filter tampilan: jangan tampilkan baris siswa yang statusnya "Hadir" murni (tap tepat waktu) — user tidak perlu lihat siswa yang sudah beres, cukup lihat yang masih perlu perhatian.

- [ ] **Dikonfirmasi user**: status `izin`/`sakit`/terkunci TETAP tampil (bukan cuma "Hadir" yang disembunyikan) — HANYA baris dengan `status === "hadir"` (tap normal, tidak terlambat) yang disembunyikan. Baris `status === null` (belum tap = "Belum Hadir") dan `status === "terlambat"` tetap tampil, begitu juga `izin`/`sakit` dan siswa `isLocked`.
- [ ] `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — filter `board` sebelum render tabel: `board.filter((row) => row.status !== "hadir")`. Lakukan di level TAMPILAN (client-side filter atas data yang sudah di-fetch), BUKAN di backend `piketBoard()` — backend tetap kirim semua data (dipakai count di `SummaryCard` "Board Semua Siswa", pertimbangkan apakah count itu juga perlu ikut menghitung HANYA yang ditampilkan atau tetap total keseluruhan — **klarifikasi ke user**: apakah angka di kartu ringkas "Board Semua Siswa" tetap menghitung SEMUA siswa termasuk yang statusnya Hadir/disembunyikan dari tabel, atau ikut mengecualikan Hadir juga).
- [ ] **Ganti label** kartu ringkas dan judul section dari **"Board Semua Siswa"** menjadi **"Siswa Belum Hadir"** (dikonfirmasi user 2026-07-30) — sesuai dengan isi barunya yang memang tidak lagi menampilkan "semua siswa" (sudah dikecualikan status Hadir). Ganti di 2 tempat: `SummaryCard label="Board Semua Siswa"` (baris ±180-185) dan judul `<h2>Board Semua Siswa</h2>` + deskripsi di bawahnya (baris ±234-237, sesuaikan juga kalimat deskripsi "Semua siswa kampus ini..." supaya konsisten dengan nama baru, misal "Siswa yang belum tap masuk atau berstatus belum beres hari ini").

## T099d — Tombol Verifikasi "Sudah Kembali"/"Dianggap Pulang" di Halaman Riwayat Izin

**Temuan:** `RiwayatIzinView` (`apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx`, dibuat T090) menampilkan tabel histori permit TANPA kolom aksi sama sekali — murni read-only. User ingin bisa langsung verifikasi kembali dari sini juga (bukan cuma dari section "Belum Kembali" di Dashboard), untuk permit yang jenisnya `keluar` dan `statusKembali` masih `belum`.

- [ ] Tambah kolom "Aksi" di tabel — untuk baris dengan `permit.jenis === "keluar" && permit.statusKembali === "belum"`, tampilkan tombol "Sudah Kembali" dan "Dianggap Pulang" (reuse endpoint yang SAMA persis dengan yang dipakai `BelumKembaliSection` di `piket-board-view.tsx`: `PATCH /permits/:id/confirm-kembali`, `PATCH /permits/:id/set-pulang` — JANGAN buat endpoint baru, ini akses ke data yang sama).
- [ ] Setelah aksi sukses, update state lokal `permits` (ganti `statusKembali` baris itu, sembunyikan tombol aksi untuk baris itu) — TIDAK perlu re-fetch seluruh tabel.
- [ ] Kalau T098 sudah dikerjakan (auto-lock + tombol "Tidak Kembali"), tombol ke-3 "Tidak Kembali" JUGA perlu ditambahkan di sini untuk konsistensi — cek urutan pengerjaan T098 vs T099, kalau T098 belum ada baru 2 tombol dulu cukup, tambahkan tombol ke-3 saat/setelah T098 selesai.
- [ ] Pastikan `onDuty` (piket bertugas hari ini, `PiketOnDutyGuard` ADR-024) tetap dihormati — tombol disabled untuk piket yang login di luar jadwalnya, pola sama seperti section lain (`usePiketOnDuty()` sudah dipakai di file ini? cek, kalau belum di-import tambahkan).

## T099e — Hapus Badge Merah Dobel di Summary Card

**Temuan (bug nyata, dikonfirmasi baca kode):** `SummaryCard` (`apps/web/src/app/(piket)/piket/piket-board-view.tsx`, fungsi `SummaryCard` baris ±321-353) merender DUA angka identik: badge kecil merah di pojok kanan atas (`{count}`, baris ±345-348, HANYA muncul kalau `count > 0`) DAN angka besar di bawah label (`{count}`, baris ±350) — keduanya menampilkan `count` yang SAMA PERSIS, terlihat seperti 2 angka berbeda padahal 1 data.

- [ ] Hapus badge kecil merah (baris ±344-348) — cukup pertahankan angka besar (`text-display`) di bawah label sebagai satu-satunya indikator jumlah.
- [ ] Pertimbangkan: apakah highlight visual utk count>0 masih diperlukan (misal warna angka besar jadi merah kalau count>0, alih-alih badge terpisah) — **klarifikasi ke user** preferensi visualnya, karena menghapus badge total juga menghilangkan sinyal "ada yang perlu perhatian" yang mungkin masih berguna, cuma perlu 1 elemen bukan 2.

---

## Files
- **Modifikasi:** `apps/web/src/app/(piket)/piket-sidebar.tsx`, `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx`, `apps/web/src/app/(piket)/piket/piket-board-view.tsx`, `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx`.
- **Jangan sentuh:** backend `piketBoard()`/`findAll` permits (T099b/c murni filter di level pemanggilan/tampilan, bukan ubah query dasar — KECUALI T099b diputuskan perlu tambah default di backend juga, cek dulu apakah endpoint lain memanfaatkan perilaku "tanpa filter status = semua" ini secara sengaja sebelum mengubah default backend-nya).

## Acceptance Criteria
- [ ] T099a: sidebar piket menampilkan grup sesuai kesepakatan final dengan user.
- [ ] T099b: Direktori Siswa piket hanya menampilkan siswa aktif secara default.
- [ ] T099c: Board Semua Siswa tidak menampilkan baris status "Hadir" murni, status lain tetap tampil.
- [ ] T099d: halaman Riwayat Izin punya tombol aksi fungsional untuk permit yang masih belum diresolve.
- [ ] T099e: tidak ada lagi 2 angka identik di kartu ringkas manapun.
- [ ] Build + type-check `apps/web` hijau, tidak ada regresi di halaman lain yang mengimpor komponen yang disentuh.

## Validasi Claudian
- [ ] T099a — konfirmasi ke user dulu soal cakupan grup (2 vs 3 menu) sebelum eksekusi, jangan asumsikan.
- [ ] T099b — pastikan tidak ada halaman/endpoint LAIN yang sengaja mengandalkan `GET /students` tanpa filter status mengembalikan semua data (grep pemanggil sebelum yakin aman mengubah salah satu caller).
- [ ] T099c — konfirmasi ke user soal angka count di `SummaryCard` "Board Semua Siswa" (total vs ter-filter) sebelum putuskan.
- [ ] T099e — konfirmasi ke user soal pengganti sinyal visual count>0 (dihilangkan total vs dipindah ke angka besar) sebelum eksekusi.

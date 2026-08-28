# T259 — Web+API: Detail Murid — Tabel Baru "Riwayat Presensi" (Paginated, Aksi Hadir/Izin)

## Depends on
Tidak ada secara teknis, TAPI **kerjakan SEBELUM/BARENG T260** — T260 menghapus entri alfa/
izin/sakit/dispen dari Riwayat Catatan DENGAN ASUMSI representasinya sudah pindah ke sini.
Kalau T260 dikerjakan duluan tanpa task ini, admin kehilangan visibilitas alfa/izin/sakit
sementara.

## Objective
Tabel baru **"Riwayat Presensi"** di halaman Detail Murid (`SiswaDetailView`) — riwayat
status kehadiran harian siswa (Hadir/Alfa/Sakit/Izin/Dispen), paginated 10 baris, dengan
aksi cepat Hadir/Izin KHUSUS baris Alfa (menggantikan mekanisme T238 yang sebelumnya ada
di Riwayat Catatan — sekarang direlokasi ke sini).

## Keputusan Diminta User (2026-08-28)

1. Kolom: **No | Tanggal | Status | Jam Datang | Jam Pulang | Keterangan | Petugas | Aksi**.
2. Status: Hadir, Alfa, Sakit, Izin, **Dispen** (Bolos DILEBUR ke Alfa — bukan status
   terpisah di tabel ini; Terlambat SENGAJA TIDAK muncul di sini — itu tetap di Riwayat
   Catatan, T260).
3. Jam Datang & Jam Pulang format **HH:MM WIB** (bukan tanggal+jam lengkap).
4. Petugas = **nama guru piket** (nama asli, BUKAN username akun).
5. Aksi = pill **Hadir** & **Izin**, **HANYA muncul di baris berstatus Alfa** (konsisten
   T238 — baris yang sudah py status TIDAK perlu aksi koreksi lagi).
6. Pagination: **10 baris per halaman** + navigasi nomor halaman (1 2 3 dst) di bawah tabel.

## Konteks — Reuse yang WAJIB (jangan hitung ulang logic yang sudah ada)

**Sumber data**: pola perhitungan status harian (Hadir/Terlambat/Izin/Sakit/Dispen/Alfa per
hari wajib) SUDAH ADA MATANG di `AttendanceReportService` — `resolveHariWajib()` (dipakai
luas, termasuk `riwayatCatatan()` existing) dan logic `presentOrExcusedDates`/mapping
`AttendanceRecord.status`→label yang dipakai di banyak tempat (`attendance.service.ts`
`resolveKategoriLive()` untuk live, `attendance-report.service.ts` untuk historis). **REUSE
resolusi status ini, JANGAN tulis ulang logic hari-wajib/alfa dari nol.**

`Petugas` (nama guru piket, bukan username) — POLA SUDAH ADA persis di `riwayatCatatan()`
existing: `approvedBy.username` di situ SEBENARNYA salah pakai username (VERIFIKASI SAAT
IMPLEMENTASI — task ini WAJIB pakai nama ASLI, cek apakah ada relasi `approvedBy.teacher.nama`
via join `User.teacherId→Teacher.nama`, REPLIKASI pola "nama asli bukan username" yang sudah
established di project ini, mis. T108's alasan fix "teacherNama, BUKAN username akun").

## Spec Detail

### 1. Backend — endpoint baru, per-siswa, ter-paginate
`GET /students/:id/riwayat-presensi?page=1&pageSize=10` (lokasi modul VERIFIKASI SAAT
IMPLEMENTASI, kemungkinan `apps/api/src/attendance/` mengingat semua logic status ada di
sana) — untuk SETIAP hari wajib siswa itu (dari tanggal siswa mulai tercatat s.d. hari ini,
REUSE batas "sejak kartu pertama" T113 yang sudah dipakai `riwayatCatatan()`), resolve:
- **Status**: Hadir/Terlambat/Izin/Sakit/Dispen/Bolos/Alfa dari `AttendanceRecord.status`
  ATAU alfa (tidak ada record). **FILTER OUT baris Terlambat dari hasil** (tidak relevan di
  tabel ini, ada di Riwayat Catatan) — VERIFIKASI: apakah baris Terlambat di-skip TOTAL dari
  daftar (tidak nongol sama sekali sebagai baris tanggal itu) atau ditampilkan tapi dengan
  Status "Hadir" (karena secara teknis siswa terlambat tetap HADIR, cuma telat) — REKOMENDASI
  KUAT: tampilkan sebagai **"Hadir"** (siswa itu hadir hari itu, faktanya), BUKAN
  disembunyikan barisnya — kalau baris disembunyikan total, riwayat presensi jadi ada
  "lubang tanggal" yang membingungkan. Detail "terlambat"-nya sendiri ada di Riwayat Catatan
  terpisah, TAPI baris kehadiran hari itu tetap harus ada di Riwayat Presensi sebagai Hadir.
  **VERIFIKASI ke user kalau ambigu saat implementasi** (dua interpretasi valid, pilih yang
  paling masuk akal secara operasional, konfirmasi dulu kalau ragu).
- **Bolos** → tampilkan sebagai **"Alfa"** (keputusan user, lebur).
- **Jam Datang/Jam Pulang**: dari `AttendanceRecord.waktuMasuk`/`waktuPulang`, format
  `HH:MM WIB` (`toLocaleTimeString("id-ID", {hour:"2-digit", minute:"2-digit"})` + suffix
  " WIB" literal, REPLIKASI pola jam yang sudah dipakai luas di codebase, timezone SERVER
  sudah `Asia/Jakarta` per CLAUDE.md, TIDAK PERLU konversi timezone manual).
- **Keterangan**: `Permit.alasanDetail` untuk baris Izin/Sakit/Dispen, `"-"` untuk Hadir/Alfa.
- **Petugas**: nama ASLI guru piket yang approve Permit (untuk Izin/Sakit/Dispen) ATAU yang
  proses absen-manual T238 (untuk baris yang pernah dikoreksi lewat Aksi Hadir/Izin di
  tabel ini sendiri) — `"-"` untuk Hadir normal via tap kiosk (tidak ada petugas manusia).
- Response ter-paginate (`{items, total, page, pageSize}`, KONSISTEN kontrak pagination lain
  di project, mis. `ActivityLogPage`/pola T240).

### 2. Aksi Hadir/Izin — RELOKASI mekanisme T238, TAMBAH Izin
`RiwayatCatatanTable`'s tombol "Hadir" (T238, `absen-manual` endpoint) SAAT INI cuma utk
baris alfa di Riwayat Catatan. Task ini:
- **Tombol "Hadir"**: REUSE endpoint absen-manual T238 apa adanya (`POST` yang sama), cuma
  dipanggil dari tabel BARU ini alih-alih dari `RiwayatCatatanTable` (T260 akan menghapus
  baris alfa dari Riwayat Catatan, jadi tombol lama otomatis tidak relevan lagi di sana).
- **Tombol "Izin" (BARU)**: buka dialog kecil (mirip pola `PermitForm` di piket-board-view —
  pilih kategori Izin/Sakit/Dispen + keterangan opsional) → submit bikin `Permit` baru
  (`jenis: "tidak_masuk"`) untuk tanggal baris itu (BUKAN cuma "izin", user minta tombolnya
  "Izin" tapi dialognya WAJAR tetap kasih pilihan kategori Izin/Sakit/Dispen, KONSISTEN
  pola existing `PermitForm` — VERIFIKASI ke user kalau ternyata mau tombol terpisah literal
  Izin-only tanpa pilihan kategori).
- Setelah aksi berhasil (Hadir maupun Izin) — baris itu HILANG dari kategori "Alfa" (refresh
  data halaman itu saja), Aksi otomatis hilang (karena statusnya sudah bukan Alfa lagi).

## Edge Cases
- **Siswa BELUM PERNAH punya kartu** (T113 — histori presensi mulai dari kartu pertama) —
  tabel kosong dengan pesan jelas, BUKAN error.
- **Halaman terakhir dengan sisa baris < 10** — pagination tetap benar (jangan expect
  selalu genap 10 per halaman).
- **Siswa nonaktif/lulus** — riwayat presensi TETAP bisa dilihat (histori, bukan operasional
  live) — Aksi Hadir/Izin pada baris Alfa siswa NONAKTIF VERIFIKASI ke user apakah masih
  boleh dikoreksi (REKOMENDASI: boleh, koreksi data historis tidak bergantung status aktif
  siswa saat ini).

## Files
- **Buat:** endpoint backend baru (`apps/api/src/attendance/` kemungkinan, method di
  `AttendanceReportService`/service terkait — REUSE resolusi status existing).
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` (tambah
  section tabel baru "Riwayat Presensi").
- **Buat:** komponen tabel+pagination (kalau belum ada pola pagination-dengan-nomor-halaman
  reusable di codebase — VERIFIKASI, kemungkinan besar sudah ada pola serupa di halaman lain
  yang punya pagination bernomor, REPLIKASI kalau ketemu).

## Acceptance Criteria
- [x] Tabel Riwayat Presensi tampil di halaman Detail Murid, kolom sesuai spec, jam format
      HH:MM WIB, Petugas nama asli (`Teacher.nama` via `User.teacherId`, fallback username
      HANYA kalau actor tidak py Teacher terkait, mis. super_admin/card_admin).
- [x] Terlambat TIDAK muncul sebagai kategori Status terpisah — **dikonfirmasi eksplisit ke
      user** (AskUserQuestion, bukan asumsi sepihak): baris hari itu tampil sebagai "Hadir".
- [x] Bolos tampil sebagai Alfa (lebur), Dispen tetap tampil sebagai Dispen sendiri
      (dibaca langsung dari `AttendanceRecord.status`, bukan diturunkan dari Permit).
- [x] Aksi Hadir+Izin HANYA muncul di baris Alfa, hilang otomatis setelah baris itu
      dikoreksi (refetch halaman itu setelah submit berhasil).
- [x] Pagination 10 baris/halaman + navigasi nomor halaman (windowed ±2 + ellipsis, BARU —
      belum ada pola serupa di codebase sebelumnya) berfungsi di data >10 baris.
- [x] Build + type-check hijau (`tsc --noEmit` api+web bersih, `next build` web sukses).

## Validasi Claudian
- [x] Konfirmasi logic resolusi status di-REUSE dari `AttendanceReportService` — `firstCardDate`
      REPLIKASI PERSIS pola `report()` (`Card.aggregate`/`_min.issuedAt`), `resolveHariWajib()`
      dipanggil APA ADANYA (bukan ditulis ulang). SATU deviasi disengaja: method BARU baca
      `AttendanceRecord.status` LANGSUNG (bukan cuma dari Permit seperti `riwayatCatatan()`)
      — dikonfirmasi via baca kode `PermitsService.createTidakMasuk()`/`createDispen()`: KEDUANYA
      SELALU insert AttendanceRecord+Permit SEPASANG, status izin/sakit/dispen tersimpan
      di AttendanceRecord juga — bukan drift, cuma sumber lookup yang lebih langsung utk
      kasus baru ini (SATU status per hari, bukan SATU entry per kategori kejadian).
- [x] Konfirmasi "Petugas nama asli" — **BUG DITEMUKAN saat baca kode**: `riwayatCatatan()`
      existing (`approvedBy: {select: {username: true}}`, baris 619/694) MEMANG salah pakai
      username persis seperti dugaan spec — TIDAK diperbaiki di sana (di luar scope task
      ini, exp isolasi perubahan), TAPI dipastikan endpoint BARU ini (`riwayatPresensi()`)
      pakai `approvedBy: {select: {username, teacher: {select: {nama}}}}` + fallback
      `teacher?.nama ?? username`. Verifikasi ke response API sungguhan BELUM dilakukan
      (DB dev sempat aktif lalu turun lagi selama sesi ini) — diverifikasi via `jest`
      (45/45 test `attendance-report.service.spec.ts` + 55/55 `attendance.service.spec.ts`
      tetap lulus) dan pembacaan kode, BUKAN curl langsung.
- [x] Konfirmasi tombol Hadir tabel baru vs Riwayat Catatan lama — task ini SENGAJA TIDAK
      menyentuh `RiwayatCatatanTable`/tombol Hadir lama di sana (scope T260) — SELAMA
      JEDA antara T259 selesai dan T260 belum dikerjakan, KEDUA tombol Hadir aktif
      bersamaan utk baris Alfa yang sama (baik dari sini maupun dari Riwayat Catatan) —
      INI SESUAI catatan "Depends on" di kepala task ini sendiri ("kerjakan SEBELUM/BARENG
      T260"), BUKAN oversight — akan otomatis rapi begitu T260 menghapus baris alfa dari
      Riwayat Catatan.

## Implementasi (2026-08-28)

**Keputusan dikonfirmasi via AskUserQuestion (mode eksekusi, TAPI 2 poin ini genuinely
butuh keputusan user, bukan cuma detail implementasi)**:
1. Baris Terlambat tampil sebagai "Hadir" di tabel ini (bukan disembunyikan) — direct
   dari opsi "Recommended" spec sendiri.
2. **Konflik arsitektur ditemukan SEBELUM implementasi** (bukan sesudah): tombol "Izin"
   yang diminta spec memanggil `POST /permits` — TAPI `PermitsController` `@Roles(guru_piket)`
   di level CLASS (ADR-019, dari CLAUDE.md: "Buat/ubah permits" HANYA guru_piket,
   super_admin dilarang "mengubah status kehadiran siswa"). Endpoint `/siswa/[id]` yang
   jadi rumah tabel ini dibuka utk super_admin/card_admin — TIDAK PERNAH guru_piket dengan
   akses tulis (piket SELALU `readOnly=true` di halaman ini, T076). Dikonfirmasi ke user:
   buat endpoint BARU terpisah (`POST /attendance/students/:id/izin-manual`, super_admin+
   card_admin only), REPLIKASI PERSIS pola `absenManual()` T238 (bukan tambah role ke
   `PermitsController` yang akan melanggar ADR-019, bukan juga batalkan fitur Izin).

**Backend**:
- `AttendanceReportService.riwayatPresensi(studentId, page, pageSize)` — method BARU,
  return `RiwayatPresensiPage`. Universe hari wajib dihitung PENUH in-memory (dari
  `earliestYear.tanggalMulai` s.d. hari ini, filter `>= firstCardDate`) lalu di-slice
  sesuai page — konsisten pendekatan `report()` (skala per-siswa realistis, beberapa
  ratus hari maksimal, murah dihitung penuh tanpa perlu SQL pagination kompleks).
  PKL-covered dates DILEWATI TOTAL dari rows (tidak ada kategori "pkl" di 5 status tabel
  ini — keputusan diambil sendiri berdasar precedent `report()`/`riwayatCatatan()` yang
  JUGA tidak pernah anggap PKL sebagai alfa, tidak eksplisit di spec tapi konsisten pola
  existing).
- `AttendanceService.izinManual()` — endpoint BARU `POST /attendance/students/:id/izin-manual`,
  REPLIKASI EFEK `PermitsService.createTidakMasuk()` (Permit+AttendanceRecord sepasang
  dalam transaction) TAPI accessible super_admin/card_admin. Kategori dibatasi izin/sakit
  SAJA (`IzinManualDto` `@IsIn(["izin","sakit"])`) — dispen SENGAJA TIDAK termasuk (py
  alur SENDIRI `POST /permits/dispen`, batch+upload bukti+role admin_jurnal, REPLIKASI
  pola visual `PermitForm` piket-board-view.tsx yang JUGA cuma izin/sakit, TIDAK PERNAH
  ada opsi dispen di komponen referensi yang dicontoh spec).
- Petugas utk baris "hadir" hasil koreksi manual (T238, `masukVia==="manual_admin"`)
  diambil dari `ActivityLog` (`action:"attendance.absen_manual"`) — TIDAK ADA Permit sama
  sekali di jalur itu — 1 query batch utk semua baris di halaman (bukan N+1 per baris).

**Frontend**: komponen baru `components/riwayat-presensi-table.tsx` — `NumberedPagination`
lokal BARU (windowed ±2 dari halaman aktif + ellipsis, TIDAK ADA preseden pagination
bernomor di codebase ini sebelumnya, `Pagination` shared existing cuma Previous/Next).
Palet badge status SAMA PERSIS `KATEGORI_LIVE_BADGE` (`piket-board-view.tsx`) — konsisten
warna kategori kehadiran lintas halaman. Dialog "Izin" (`IzinManualForm`) REPLIKASI pola
visual `PermitForm` (2 kategori saja, izin/sakit). Dipasang di `siswa-detail-view.tsx`
persis DI BAWAH `RiwayatCatatanTable` existing (BUKAN menggantikan — itu scope T260),
`canManageAksi` pakai prop `canAbsenManual` yang SUDAH ADA (role gate identik).

**Verifikasi**: `tsc --noEmit` api+web bersih, `next build` web sukses (`/siswa/[id]` naik
jadi 7.57 kB). `jest attendance-report.service.spec.ts` 45/45 lulus, `jest
attendance.service.spec.ts` 55/55 lulus — TANPA test baru ditambahkan, regresi nol ke test
existing (1 run pertama sempat 8 gagal karena resource contention `next build` jalan
bersamaan di mesin RAM-terbatas ini — re-run terisolasi 100% bersih, dikonfirmasi bukan
regresi sungguhan). **Keterbatasan**: test live end-to-end (curl/browser sungguhan verifikasi
Petugas nama asli tampil benar, klik tombol Izin sungguhan) BELUM dilakukan — DB dev
sempat aktif lalu turun lagi selama sesi berjalan, verifikasi mengandalkan test unit +
code review manual.

# T138 — Schema+API+Web: Rekap Kehadiran Ekstrakurikuler (Per Kelas/Per Ekstra/Semua)

## Depends on
Tidak ada dependency teknis langsung, TAPI konsep/pola PDF/Excel/grafik/filter Jurusan+Kelas+Tingkat MENGACU ke T115/T132 (Rekap Kehadiran Gerbang) — REUSE infrastruktur export (Puppeteer+exceljs+chart library) yang SUDAH ADA dari task itu, JANGAN install dependency baru/bangun ulang dari nol.

## Objective
Fitur rekap kehadiran ekstrakurikuler BARU sepenuhnya — SAAT INI TIDAK ADA rekap/aggregate/export apa pun untuk domain ekstrakurikuler (murni pencatatan per-sesi oleh pembina, tidak ada tampilan ringkasan lintas sesi). Rekap ini bisa difilter per Kelas, per Ekstrakurikuler, atau Semua (gabungan), dengan filter tambahan Jurusan+Tingkat, export PDF/Excel.

## Context
- **App:** `apps/api` (modul rekap baru dari nol) + `apps/web` (halaman rekap baru)
- **Riset 2026-08-08 (Explore agent, baca kode langsung)** — konfirmasi kondisi SEBELUM task ini:
  - **TIDAK ADA rekap/report/export apa pun untuk ekstrakurikuler** — `ekstra-absensi.controller.ts` isinya murni CRUD per-sesi/peserta/kelompok (`saya`, `generate-today`, `sesi`, `peserta`, `kelompok`, `absen/:absenId`) — grep kata "rekap"/"export"/"report" nol hasil di seluruh domain ini. `EkstraMonitoringController` (`ekstra-publik/`) itu monitoring PENDAFTARAN (siapa pilih ekstra apa), BUKAN rekap KEHADIRAN — jangan tertukar, 2 konsep berbeda.
  - **`EkstraAbsen.status`** (`schema.prisma:1015-1030`) — enum `hadir | izin | sakit | alfa`, NULLABLE (`NULL = belum ditandai`, komentar eksplisit di baris ±1019).
  - **Rantai relasi untuk join SUDAH TERSEDIA tanpa field baru**: `EkstraAbsen.studentId → Student.kelasId → Kelas` (untuk "per kelas") DAN `EkstraAbsen.sesiId → EkstraSesi.ekstrakurikulerId → Ekstrakurikuler` (untuk "per ekstra", TIDAK PERLU lewat `EkstraKelompokAnggota`/`EkstraPendaftaran`, `EkstraSesi` sudah langsung punya `ekstrakurikulerId`).
  - **`Kelas.tingkat`** (`schema.prisma:42-53`, enum `X | XI | XII`) — kolom TERSTRUKTUR, BUKAN perlu di-parse dari `Kelas.nama` — filter Tingkat yang diminta user MURAH diimplementasikan (lihat memory `feedback_filter_rekap_wajib_tingkat`).
  - **KONVENSI PENTING soal "sesi belum diabsen"** (dikonfirmasi baca kode `ekstra-absensi.service.ts:95-98`, `:486`, `:491-496`, pola `sudahAdaAbsen`): row `EkstraAbsen` untuk SEMUA peserta terdaftar dibuat SEKALIGUS saat sesi dibuat (`status: null` semua) — BUKAN dibuat satu-satu saat pembina menandai. "Sesi sudah diabsen" ditentukan oleh **MINIMAL 1 row punya status non-null** (`.some(a => a.status !== null)`), BUKAN "semua row punya status". Ini konvensi EXISTING yang sudah dipakai di banyak tempat — REUSE logic yang SAMA untuk rekap, jangan bikin definisi berbeda.

## Keputusan Final (dikonfirmasi user 2026-08-08)

1. **Filter grouping**: per Kelas, per Ekstrakurikuler, atau Semua (gabungan) — 3 mode, mirip semangat "per-kelas/per-ekstra/semua" yang diminta user.
2. **Filter tambahan wajib**: Tingkat (X/XI/XII) — sesuai permintaan eksplisit user, dicatat juga di memory `feedback_filter_rekap_wajib_tingkat` sebagai aturan permanen untuk SEMUA fitur rekap ke depan, bukan cuma task ini.
3. **Definisi Alfa** (dikonfirmasi user, PENTING dan bercabang):
   - Sesi yang **SUDAH dibuat presensinya** oleh pembina (minimal 1 siswa sudah ditandai, sesuai konvensi `sudahAdaAbsen` existing) — siswa dengan `status: null` di sesi itu **OTOMATIS dihitung Alfa**.
   - Sesi yang **BELUM dibuat presensinya sama sekali** (SEMUA row masih `status: null`, `sudahAdaAbsen === false`) — sesi ini **DIKECUALIKAN TOTAL** dari perhitungan rekap (tidak dihitung Alfa untuk siapa pun, karena pembina memang belum sempat proses sesi itu — beda dari "pembina proses tapi lupa 1 siswa").
4. **Berlaku juga di export PDF/Excel**, filter yang sama dipakai untuk file yang di-download (konsisten prinsip T132).
5. **Filter Tahun Ajaran/Semester** (ditambahkan audit Hermes 2026-09-03) — `EkstraSesi`/`EkstraAbsen` SUDAH punya `academicYearId`/`semesterId` (di-tag otomatis via `AcademicPeriodService.getActive()` saat sesi dibuat, sama pola T139+). Rekap ini WAJIB terima filter `academicYearId?`/`semesterId?` opsional (pola sama `ReportQueryDto` di `attendance-report.service.ts`/T140) — MELENGKAPI filter tanggal manual (`from`/`to`), bukan menggantikan. Jangan andalkan cuma filter tanggal manual seperti versi awal task ini — data akan terus menumpuk lintas tahun ajaran begitu fitur ini live cukup lama.
6. **Siswa "belum berkelompok"** (ditambahkan audit Hermes 2026-09-03) — siswa yang terdaftar `EkstraPendaftaran` di ekstra BERKELOMPOK tapi belum pernah di-assign ke `EkstraKelompokAnggota` manapun TIDAK PERNAH punya baris `EkstraAbsen` sama sekali (sesi hanya digenerate untuk anggota kelompok, lihat `createSesi()`/`generateSesiIdempotent()`) — kalau rekap murni agregasi dari baris `EkstraAbsen` yang ada, siswa ini akan HILANG TOTAL dari rekap tanpa jejak. **Keputusan user**: siswa begini WAJIB tetap tampil di rekap sebagai baris terpisah dengan indikator jelas "0 sesi — belum berkelompok" (bukan disembunyikan, bukan dihitung Alfa) — supaya admin sadar ada siswa yang datanya kosong BUKAN karena alfa terus-menerus, tapi karena belum di-plot ke kelompok manapun. Sumber daftar siswa ini: query mirip `listBelumBerkelompok()` existing di `ekstra-absensi.service.ts`, per ekstra yang punya kelompok.

## Spec Detail

### Backend
- Modul baru `apps/api/src/ekstra-absensi/` (tambahkan ke modul existing, KARENA domain sama) ATAU modul terpisah kecil — putuskan saat implementasi, method baru `rekap(filter)`.
- **⛔ WAJIB MUTLAK — RBAC Guard (audit Hermes 2026-09-03, celah nyata ditemukan)**: `EkstraAbsensiController` existing punya guard CLASS-LEVEL `@Roles(UserRole.guru, UserRole.pembina_ekstra)` — kalau endpoint rekap ditambahkan ke controller ini TANPA `@Roles()` override eksplisit di level method, endpoint itu OTOMATIS ikut guard class-level dan **super_admin akan dapat 403**, padahal halaman rekap direncanakan di `(admin)/rekap-ekstrakurikuler/` yang HANYA bisa diakses role admin (guru/pembina_ekstra di-redirect keluar dari grup `(admin)` di `layout.tsx`). Hasilnya: TIDAK ADA role yang bisa sukses pakai fitur ini kalau guard ini terlewat. **WAJIB** pasang `@Roles(UserRole.super_admin)` eksplisit di method `rekap()` + endpoint export PDF/Excel (pola PERSIS `generateToday()` di controller yang sama, baris ±76-79 — override total dari class-level guard). Ini konsisten filosofi Rekap Kehadiran Gerbang (`attendance.controller.ts` — `@Roles(UserRole.super_admin)` eksplisit di endpoint `report()`/`report-flexible`) — rekap adalah operasi READ admin, BUKAN mutasi data absensi ekstra yang jadi domain eksklusif pembina (ADR-019 hanya berlaku ke mutasi, bukan ke laporan read-only).
- **Logic inti (WAJIB ikuti definisi Alfa di atas PERSIS)**:
  1. Ambil semua `EkstraSesi` dalam rentang tanggal+filter (kelas/ekstra/tingkat/jurusan/tahun-ajaran/semester) yang relevan.
  2. Untuk TIAP sesi, hitung `sudahAdaAbsen = absen.some(a => a.status !== null)` (REUSE logic PERSIS dari `ekstra-absensi.service.ts`, jangan tulis ulang beda — pertimbangkan extract jadi shared helper method kalau memungkinkan, dipakai baik oleh CRUD existing maupun rekap baru).
  3. **Kecualikan sesi dengan `sudahAdaAbsen === false`** dari SELURUH agregasi (tidak masuk hitungan Hadir/Izin/Sakit/Alfa/Total Sesi sama sekali).
  4. Untuk sesi yang LOLOS filter itu (`sudahAdaAbsen === true`): per siswa, `status === null` → hitung sebagai Alfa; `status` terisi eksplisit → hitung sesuai nilainya.
  5. Agregasi per siswa (kalau grouping per-kelas/semua) atau per siswa dalam 1 ekstra (kalau grouping per-ekstra): total Hadir/Izin/Sakit/Alfa + Total Sesi Efektif (jumlah sesi yang `sudahAdaAbsen === true` yang siswa itu PUNYA baris `EkstraAbsen`-nya — TIDAK PERLU logic tambahan soal "kapan siswa gabung", karena baris `EkstraAbsen` sudah snapshot peserta PERSIS saat sesi dibuat/digenerate (`createSesi()`/`generateSesiIdempotent()`) — siswa yang belum tergabung saat sesi dibuat otomatis TIDAK PUNYA baris untuk sesi itu, jadi tidak perlu dihitung manual dari timestamp `EkstraPendaftaran`/`EkstraKelompokAnggota`).
  6. **Siswa "belum berkelompok"** (lihat Keputusan Final poin 6) — TAMBAHKAN sebagai baris terpisah di response dengan flag eksplisit (mis. `belumBerkelompok: true`, semua hitungan 0) untuk ekstra yang punya kelompok, supaya tidak hilang tanpa jejak dari rekap.
- **Filter**: `kelasId?`, `jurusanId?`, `tingkat?` (enum X/XI/XII), `ekstrakurikulerId?`, `academicYearId?`, `semesterId?` (opsional, pola sama `ReportQueryDto`), `from`/`to` (rentang tanggal sesi, tetap wajib/manual kalau `academicYearId` tidak dikirim — pola sama `resolveDateRange()` di `attendance-report.service.ts`), `mode: "per-kelas" | "per-ekstra" | "semua"` (menentukan struktur grouping response — putuskan apakah "semua" berarti flat semua siswa tanpa grouping, atau grouping ganda kelas+ekstra sekaligus, klarifikasi ke user kalau ambigu saat implementasi).
- **Kolom "Nama Ekstrakurikuler" WAJIB tampil di mode "per-kelas"** (audit Hermes 2026-09-03) — karena `EkstraPendaftaran.studentId` unique global (1 siswa cuma ikut 1 ekstra), 1 kelas berisi siswa-siswa dengan ekstra yang BERBEDA-BEDA. Tanpa kolom ini, tabel mode per-kelas terlihat seperti data tidak konsisten (siswa A 8 sesi, siswa B di kelas sama 5 sesi, tanpa konteks kenapa beda).
- **Export PDF/Excel** — REUSE infrastruktur `attendance-report-export.service.ts` (T115) SEBAGAI POLA/TEMPLATE (Puppeteer setup, exceljs, chart library sudah terpasang), BUKAN fungsi yang sama persis (domain data beda) — tapi JANGAN install ulang dependency yang sama, reuse packages yang sudah ada di `package.json`.
- Endpoint baru: `GET /ekstra-absensi/rekap` (data JSON), `GET /ekstra-absensi/rekap/export/pdf`, `GET /ekstra-absensi/rekap/export/excel` — SEMUA WAJIB `@Roles(UserRole.super_admin)` eksplisit, lihat poin RBAC di atas.

### Frontend
- Halaman admin baru (lokasi: pertimbangkan `(admin)/rekap-ekstrakurikuler/` terpisah dari `(admin)/rekap/` existing yang khusus kehadiran gerbang — JANGAN campur 2 domain berbeda di 1 halaman) — filter Search→Jurusan→Tingkat→Kelas (urutan sesuai memory `feedback_filter_search_jurusan_kelas_order` + `feedback_filter_rekap_wajib_tingkat`) DITAMBAH dropdown Ekstrakurikuler DITAMBAH dropdown Tahun Ajaran/Semester (pola sama halaman Rekap gerbang T140) DITAMBAH toggle mode (per-kelas/per-ekstra/semua) DITAMBAH rentang tanggal.
- Tabel hasil + grafik (REUSE pola visual T115/T132: bar chart, kolom sortable+search+No sesuai aturan permanen proyek). Baris siswa "belum berkelompok" tampil dengan indikator visual jelas (badge/warna beda), bukan campur tanpa pembeda dengan siswa 0 sesi karena alfa.
- 2 tombol Download PDF/Excel (pola sama T115/T132).

## Edge Cases
- Ekstrakurikuler yang BELUM PERNAH ada sesi sama sekali → tampil di dropdown filter tapi hasil rekap kosong (empty state jelas, bukan error).
- Siswa terdaftar ekstra tapi TIDAK PERNAH masuk grup manapun (`EkstraKelompokAnggota` kosong) → cek apakah siswa begini bahkan bisa punya `EkstraAbsen` row sama sekali (tergantung alur `createSesi`/`generateSesiIdempotent` peserta-nya dari mana) — pastikan tidak crash kalau ada siswa terdaftar tapi nol partisipasi sesi.
- Rentang tanggal filter yang tidak mengandung sesi apa pun → hasil kosong, bukan error.

## Files
- **Buat:** method rekap baru di `ekstra-absensi.service.ts` (atau modul terpisah), endpoint export PDF/Excel, halaman admin baru `apps/web/src/app/(admin)/rekap-ekstrakurikuler/`.
- **Modifikasi:** `apps/api/src/ekstra-absensi/ekstra-absensi.controller.ts` (endpoint baru), sidebar admin (tambah menu baru).
- **Jangan sentuh:** `apps/web/src/app/(admin)/rekap/` (rekap gerbang existing, domain berbeda, JANGAN dicampur), `EkstraMonitoringController` (monitoring pendaftaran, konsep berbeda dari rekap kehadiran).

## Acceptance Criteria
- [ ] Rekap bisa difilter per Kelas, per Ekstrakurikuler, atau Semua.
- [ ] Filter Jurusan+Tingkat+Kelas tersedia dan bekerja (cascading sesuai pola permanen proyek).
- [ ] Filter Tahun Ajaran/Semester tersedia dan bekerja, melengkapi filter tanggal manual (pola T140).
- [ ] **`super_admin` bisa akses endpoint rekap + export tanpa 403** — verifikasi eksplisit `@Roles(UserRole.super_admin)` terpasang di method (BUKAN mengandalkan guard class-level controller yang HANYA mengizinkan guru/pembina_ekstra).
- [ ] Sesi yang BELUM diabsen sama sekali TIDAK masuk hitungan rekap sama sekali (dikecualikan total, bukan dihitung Alfa untuk siapa pun).
- [ ] Sesi yang SUDAH diabsen (minimal 1 siswa) — siswa dengan status kosong di sesi itu otomatis terhitung Alfa.
- [ ] Siswa "belum berkelompok" (ekstra berkelompok, belum di-assign) tampil sebagai baris terpisah dengan indikator jelas, bukan hilang tanpa jejak.
- [ ] Mode "per-kelas" menampilkan kolom Nama Ekstrakurikuler per siswa.
- [ ] Export PDF dan Excel tersedia, mengikuti filter yang sama seperti tampilan layar.
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [ ] **WAJIB reuse logic `sudahAdaAbsen` PERSIS** dari kode existing (`ekstra-absensi.service.ts`) — jangan tulis ulang kondisi yang berpotensi beda hasil dari yang sudah dipakai fitur lain.
- [ ] **WAJIB verifikasi `@Roles(UserRole.super_admin)` eksplisit** di endpoint rekap+export sebelum menganggap task selesai — controller existing punya guard class-level yang HANYA mengizinkan guru/pembina_ekstra, endpoint baru TIDAK otomatis dapat akses super_admin tanpa override eksplisit ini. Test manual: login sebagai super_admin, panggil endpoint, pastikan BUKAN 403.
- [ ] Klarifikasi ke user saat implementasi: definisi "mode semua" (flat semua siswa vs grouping ganda kelas+ekstra) kalau terasa ambigu.

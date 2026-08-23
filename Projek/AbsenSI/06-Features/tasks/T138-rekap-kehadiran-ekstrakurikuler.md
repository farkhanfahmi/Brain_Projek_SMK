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

## Spec Detail

### Backend
- Modul baru `apps/api/src/ekstra-absensi/` (tambahkan ke modul existing, KARENA domain sama) ATAU modul terpisah kecil — putuskan saat implementasi, method baru `rekap(filter)`.
- **Logic inti (WAJIB ikuti definisi Alfa di atas PERSIS)**:
  1. Ambil semua `EkstraSesi` dalam rentang tanggal+filter (kelas/ekstra/tingkat/jurusan) yang relevan.
  2. Untuk TIAP sesi, hitung `sudahAdaAbsen = absen.some(a => a.status !== null)` (REUSE logic PERSIS dari `ekstra-absensi.service.ts`, jangan tulis ulang beda — pertimbangkan extract jadi shared helper method kalau memungkinkan, dipakai baik oleh CRUD existing maupun rekap baru).
  3. **Kecualikan sesi dengan `sudahAdaAbsen === false`** dari SELURUH agregasi (tidak masuk hitungan Hadir/Izin/Sakit/Alfa/Total Sesi sama sekali).
  4. Untuk sesi yang LOLOS filter itu (`sudahAdaAbsen === true`): per siswa, `status === null` → hitung sebagai Alfa; `status` terisi eksplisit → hitung sesuai nilainya.
  5. Agregasi per siswa (kalau grouping per-kelas/semua) atau per siswa dalam 1 ekstra (kalau grouping per-ekstra): total Hadir/Izin/Sakit/Alfa + Total Sesi Efektif (jumlah sesi yang `sudahAdaAbsen === true` DAN relevan untuk siswa itu — kalau siswa baru gabung ekstra di tengah semester, "sesi efektif" untuk dia mungkin cuma sebagian, PERTIMBANGKAN ini saat implementasi berdasarkan `EkstraPendaftaran`/`EkstraKelompokAnggota` timestamp kalau relevan).
- **Filter**: `kelasId?`, `jurusanId?`, `tingkat?` (enum X/XI/XII), `ekstrakurikulerId?`, `from`/`to` (rentang tanggal sesi), `mode: "per-kelas" | "per-ekstra" | "semua"` (menentukan struktur grouping response — putuskan apakah "semua" berarti flat semua siswa tanpa grouping, atau grouping ganda kelas+ekstra sekaligus, klarifikasi ke user kalau ambigu saat implementasi).
- **Export PDF/Excel** — REUSE infrastruktur `attendance-report-export.service.ts` (T115) SEBAGAI POLA/TEMPLATE (Puppeteer setup, exceljs, chart library sudah terpasang), BUKAN fungsi yang sama persis (domain data beda) — tapi JANGAN install ulang dependency yang sama, reuse packages yang sudah ada di `package.json`.
- Endpoint baru: `GET /ekstra-absensi/rekap` (data JSON), `GET /ekstra-absensi/rekap/export/pdf`, `GET /ekstra-absensi/rekap/export/excel`.

### Frontend
- Halaman admin baru (lokasi: pertimbangkan `(admin)/rekap-ekstrakurikuler/` terpisah dari `(admin)/rekap/` existing yang khusus kehadiran gerbang — JANGAN campur 2 domain berbeda di 1 halaman) — filter Search→Jurusan→Tingkat→Kelas (urutan sesuai memory `feedback_filter_search_jurusan_kelas_order` + `feedback_filter_rekap_wajib_tingkat`) DITAMBAH dropdown Ekstrakurikuler DITAMBAH toggle mode (per-kelas/per-ekstra/semua) DITAMBAH rentang tanggal.
- Tabel hasil + grafik (REUSE pola visual T115/T132: bar chart, kolom sortable+search+No sesuai aturan permanen proyek).
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
- [ ] Sesi yang BELUM diabsen sama sekali TIDAK masuk hitungan rekap sama sekali (dikecualikan total, bukan dihitung Alfa untuk siapa pun).
- [ ] Sesi yang SUDAH diabsen (minimal 1 siswa) — siswa dengan status kosong di sesi itu otomatis terhitung Alfa.
- [ ] Export PDF dan Excel tersedia, mengikuti filter yang sama seperti tampilan layar.
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [ ] **WAJIB reuse logic `sudahAdaAbsen` PERSIS** dari kode existing (`ekstra-absensi.service.ts`) — jangan tulis ulang kondisi yang berpotensi beda hasil dari yang sudah dipakai fitur lain.
- [ ] Klarifikasi ke user saat implementasi: definisi "mode semua" (flat semua siswa vs grouping ganda kelas+ekstra) kalau terasa ambigu.
- [ ] Pertimbangkan/tanyakan ke user: apakah "Total Sesi Efektif" per siswa perlu memperhitungkan kapan siswa itu BERGABUNG ke ekstra/kelompok (kalau baru gabung di tengah semester), atau cukup hitung dari SEMUA sesi ekstra itu tanpa peduli kapan siswa gabung — ini bisa mempengaruhi keakuratan rekap untuk siswa yang pindah kelompok/gabung telat.

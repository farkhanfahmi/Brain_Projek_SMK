# T253 — Web+API: Menu Baru "Monitoring" — Halaman Monitoring Wali Kelas

> **Revisi 2026-08-28**: keputusan "sejajar Log" di bawah ini SUDAH DIBATALKAN oleh user
> setelah dilihat live — menu "Monitoring" (`/monitoring-wali-kelas`) DIPINDAH jadi
> sub-menu grup **"Dashboard"** dengan judul sub-menu **"Wali Kelas"** (`dashboardGroup` di
> `nav-items.ts`, BUKAN lagi item tunggal `dashboardHome` + BUKAN lagi entry di
> `secondaryNav`). Konten halaman/endpoint backend TIDAK berubah, murni lokasi menu.
> Bagian "Spec Detail poin 1" dan "Validasi Claudian" soal posisi menu di bawah SEKARANG
> USANG — dibiarkan apa adanya sebagai jejak keputusan awal, JANGAN dijadikan acuan
> posisi menu terkini.

## Depends on
**T252** (field `passwordChangedAt`, `updatedAt`, logging export) — WAJIB selesai dulu.

## Objective
Menu baru **"Monitoring"** di sidebar admin (SEJAJAR "Log", BUKAN sub-bagiannya — beda
konsep: Log = riwayat historis, Monitoring = status operasional TERKINI) — halaman pertama:
**Monitoring Wali Kelas**, 3 section tabel.

## Konteks — Kenapa BUKAN Digabung ke "Log" (klarifikasi diskusi 2026-08-25)

Sempat didiskusikan sebagai perluasan Log (naratif+analitik+anomali) — TAPI itu salah
paham kebutuhan sebenarnya. User butuh **snapshot status HARI INI/SEKARANG** ("siapa sudah/
belum"), bukan riwayat historis yang di-scroll ke belakang. Konsepnya SAMA dengan
Papan Piket/Dashboard TV Piket/"Hari Ini" wali kelas yang SUDAH ADA — bukan Log. Diskusi
Log naratif+dashboard analitik **DIBATALKAN**, JANGAN dikerjakan kecuali diminta ulang
eksplisit di kemudian hari, terpisah dari task ini.

## Spec Detail

### 1. Menu baru
Sidebar admin — item baru **"Monitoring"** (ikon `Activity`/`Radar` dari lucide-react,
VERIFIKASI SAAT IMPLEMENTASI ikon paling pas, BEDA dari ikon "Log" yang sudah ada supaya
tidak keliru). Untuk sekarang cukup 1 halaman langsung (`/monitoring-wali-kelas` atau
`/monitoring/wali-kelas` — VERIFIKASI SAAT IMPLEMENTASI, kalau nanti fitur lain juga masuk
Monitoring baru pertimbangkan sub-menu, BELUM PERLU sekarang untuk 1 halaman ini saja).

### 2. Section 1 — Status Ganti Password
Tabel standar (SortableHeader+search+kolom No, pola wajib project — cocok di sini karena ini
genuinely data tabular, BEDA dari draft Log naratif yang dibatalkan):

| No | Nama Wali Kelas | Kelas | Status | Waktu Ganti |
|---|---|---|---|---|
| 1 | Budi Santoso | X TKJ 1 | 🟢 Sudah Ganti | 21 Agu 2026, 14:32 |
| 2 | Siti Aminah | X RPL 1 | 🔴 Belum Ganti | - |

- Status dari `User.mustChangePassword` (false=Sudah, true=Belum) — badge hijau/merah
  KONSISTEN token warna existing (`success-bg`/`danger-bg`).
- Waktu dari `User.passwordChangedAt` — `null` DAN `mustChangePassword: false` (akun lama
  sebelum T252) → tampilkan "Sudah (waktu tidak tercatat)" BUKAN kosong/"-" polos (jangan
  ambigu dengan "belum ganti").
- Search+sort SEMUA kolom (aturan wajib project).

### 3. Section 2 — Status Kelengkapan Data Kelas
| No | Nama Wali Kelas | Kelas | Struktur Pengurus | Piket Kelas | Terakhir Diupdate |
|---|---|---|---|---|---|
| 1 | Budi Santoso | X TKJ 1 | 🟢 Lengkap (5/6) | 🟢 4 hari | 22 Agu 2026, 09:10 |
| 2 | Siti Aminah | X RPL 1 | 🔴 Belum Lengkap (0/6) | 🔴 Belum ada | - |

- **Kriteria "Lengkap" struktur** (VERIFIKASI ke user kalau ambigu): MINIMAL 3 jabatan wajib
  terisi (Ketua, Sekretaris, Bendahara) — 3 jabatan wakil TETAP dihitung di angka "X/6" tapi
  TIDAK mempengaruhi status Lengkap/Belum (opsional, sesuai desain T247/T248).
- **Kriteria piket**: minimal 1 hari dikonfigurasi = "Ada" (tampilkan jumlah hari, bukan
  cuma ya/tidak — lebih informatif).
- Query: `COUNT(KelasPengurus WHERE kelasId+academicYearId aktif)` per kelas +
  `COUNT(DISTINCT hari) FROM KelasPiketJadwal WHERE kelasId`. "Terakhir Diupdate" = `MAX(updatedAt)`
  gabungan kedua tabel untuk kelas itu (VERIFIKASI SAAT IMPLEMENTASI cara query paling
  efisien — mungkin 2 query terpisah lalu diambil yang terbaru, bukan 1 query SQL kompleks).

### 4. Section 3 — Riwayat Download Rekap
| No | Nama Wali Kelas | Kelas | Format | Waktu Download |
|---|---|---|---|---|
| 1 | Budi Santoso | X TKJ 1 | PDF | 23 Agu 2026, 15:00 |
| 2 | Budi Santoso | X TKJ 1 | Excel | 23 Agu 2026, 15:02 |

- Sumber: `ActivityLog WHERE action IN ('rekap.export_pdf', 'rekap.export_xlsx') AND actor.role = 'guru' AND actor.kelasIdWali IS NOT NULL`
  (scoped ke wali kelas SAJA, bukan semua export — admin juga bisa export rekap tapi itu DI
  LUAR SCOPE monitoring ini, VERIFIKASI ke user kalau ternyata ingin sertakan juga).
- Ini TABEL BIASA juga (bukan naratif) — 1 baris = 1 event download, sortable by waktu
  default descending (terbaru dulu).
- **Bisa banyak baris per wali kelas** (beda dari section 1-2 yang 1 baris = 1 wali kelas) —
  pagination WAJIB (konsisten pola tabel besar lain di project, page+pageSize).

### 5. Mobile-responsif
3 tabel ini berpotensi lebar (banyak kolom) — REPLIKASI pola `overflow-x-auto` wrapper yang
sudah dipakai tabel lain di project untuk scroll horizontal terkontrol di mobile, KONSISTEN
aturan mobile-first wajib.

## Edge Cases
- **Kelas belum punya wali kelas ditugaskan** (`kelasIdWali` belum di-assign ke User
  manapun) — TIDAK muncul di Section 1-2 sama sekali (tidak ada subjek untuk dipantau),
  BUKAN muncul dengan data kosong membingungkan.
- **Wali kelas pegang lebih dari 1 kelas** (kalau ini memang mungkin terjadi di data —
  VERIFIKASI struktur `User.kelasIdWali` apakah 1-ke-1 atau bisa lebih, dari baca kode
  sebelumnya SETIAP User cuma py 1 `kelasIdWali` scalar jadi 1 wali = 1 kelas, TIDAK ADA
  kasus rangkap — aman diasumsikan 1 baris = 1 wali kelas = 1 kelas di Section 1-2).

## Files
- **Buat:** `apps/web/src/app/(admin)/monitoring-wali-kelas/` (page.tsx + view component).
- **Buat:** endpoint backend baru (read-only, agregasi 3 section) — lokasi modul
  VERIFIKASI SAAT IMPLEMENTASI (kemungkinan `apps/api/src/journal/` diperluas, atau modul
  baru `monitoring/` kalau dianggap cukup besar scope-nya ke depan).
- **Modifikasi:** sidebar admin (tambah menu "Monitoring").

## Acceptance Criteria
- [x] 3 section tampil benar dengan data real dari seed/test — **DIVERIFIKASI 2026-08-28**
      setelah DB dev dinyalakan: `GET /api/monitoring/wali-kelas` (curl, login `admin`)
      mengembalikan data NYATA dari 1 wali kelas seed (Budi Santoso, X TKJ 1) — Section 1
      `password_changed_at` terisi benar, Section 2 `jumlah_pengurus_wajib_terisi: 1`
      (state "Belum Lengkap", 1 dari 3 wajib) + `piket_hari_count: 1` (state "Ada").
      `GET /api/monitoring/wali-kelas/riwayat-download` kembalikan `{items:[], total:0}`
      (state kosong, belum pernah ada download di seed) — SEMUA benar sesuai kode.
      Halaman `/monitoring-wali-kelas` di browser TIDAK crash (307 redirect ke login tanpa
      cookie sesi, bukan 500). **CATATAN**: verifikasi API-level+halaman-tidak-crash, BUKAN
      screenshot visual browser penuh (di luar kemampuan sesi eksekusi ini) — DAN seed data
      cuma py 1 wali kelas jadi state "Sudah Ganti Password"/"Lengkap"/"ada riwayat
      download" belum sempat terobservasi (cuma 1 dari beberapa state tiap section yang
      punya data), REKOMENDASI user cek visual sendiri di browser untuk cakupan penuh.
- [x] Semua tabel search+sortable semua kolom+kolom No (aturan wajib project) — 3 tabel,
      Section 1-2 client-side (dataset kecil, 1 baris = 1 wali kelas), Section 3 server-side
      (bisa banyak baris, backend extend `RiwayatDownloadQueryDto` dgn `search`/`sortBy`/`sortDir`).
- [x] Status password/struktur/piket pakai kriteria yang didefinisikan di atas
      (Ketua+Sekretaris+Bendahara wajib utk "Lengkap", piket "Ada" kalau ≥1 hari), badge
      `success-bg`/`danger-bg` konsisten token existing.
- [x] Riwayat download ter-paginate (page/pageSize, default 25), terbaru di atas
      (`orderBy: {createdAt: "desc"}` default).
- [x] Responsif mobile — 3 tabel dibungkus `overflow-x-auto` (REPLIKASI pola tabel lain).
- [x] Build + type-check hijau (`tsc --noEmit` api+web bersih, `next build` web sukses,
      route `/monitoring-wali-kelas` terdaftar 3.35 kB).

## Validasi Claudian
- [x] Konfirmasi menu "Monitoring" terpisah dari "Log" — ditambahkan ke `secondaryNav`
      (`nav-items.ts`) SEJAJAR "Log Aktivitas" (item array yang sama, bukan nested), ikon
      `Radar` (beda dari `FileClock` Log dan `Activity` Kesehatan Kiosk). Dicek struktur
      sidebar terkini SEBELUM nambah — tidak ada perubahan lain ke `secondaryNav` sejak
      task ini ditulis.
- [ ] Konfirmasi data test mencakup KETIGA kondisi tiap section — **SEBAGIAN**: seed dev
      saat ini cuma py 1 wali kelas (Budi Santoso), jadi baru 1 state per section yang
      benar-benar teramati live (Belum Lengkap 1/3, Piket Ada 1 hari, Riwayat Download
      kosong). State sebaliknya (Sudah Ganti Password, Lengkap 3/3, ada riwayat download)
      TIDAK ADA di data seed untuk diamati — logic backend sudah benar secara kode (dibaca:
      `must_change_password` boolean 2 state, `jumlah_pengurus_wajib_terisi === 3` vs `< 3`,
      `items.length === 0` vs terisi), TAPI belum pernah TERLIHAT sungguhan di response API
      manapun. Kalau mau verifikasi penuh, perlu tambah data seed 2nd wali kelas dgn kondisi
      sebaliknya, atau test manual lewat UI (ganti password 1 akun wali, isi 3 jabatan wajib
      kelas lain, download 1 rekap) — TIDAK dilakukan di sesi ini (di luar scope perbaikan
      kode, murni soal kelengkapan data uji).

## Implementasi (2026-08-27)

**Backend** — modul baru `apps/api/src/monitoring/` (`MonitoringModule`/`Controller`/`Service`),
`@Roles(UserRole.super_admin)` KONSISTEN pola `/log` (`ActivityLogController`). 2 endpoint:
`GET /monitoring/wali-kelas` (Section 1+2 sekaligus — basis query SAMA: `User.findMany({role:
"guru", kelasIdWali: {not: null}})`, 1 wali = 1 kelas scalar TIDAK ADA kasus rangkap
dikonfirmasi baca schema, jadi query dari sisi `User` cukup TANPA perlu dari sisi
`Kelas.waliKelas[]`), `GET /monitoring/wali-kelas/riwayat-download` (Section 3, paginated +
search+sort, scoped `actor.role='guru' AND actor.kelasIdWali IS NOT NULL` — SESUAI spec,
export oleh admin TIDAK disertakan).

**Kriteria Lengkap struktur**: `jumlahPengurusWajibTerisi === 3` (Ketua/Sekretaris/Bendahara)
— 3 jabatan wakil TETAP dihitung di `jumlahPengurusTotal` (ditampilkan "X/6") TAPI TIDAK
mempengaruhi status, PERSIS keputusan spec. **Kriteria piket**: `piketHariCount > 0` (COUNT
DISTINCT hari), ditampilkan sebagai jumlah hari bukan cuma ya/tidak. **Terakhir Diupdate**:
diambil dari 2 query terpisah (`KelasPengurus.updatedAt` + `KelasPiketJadwal.updatedAt`)
lalu `Math.max()` di application layer (BUKAN 1 query SQL kompleks — sesuai rekomendasi
spec, lebih sederhana+mudah dibaca untuk skala data ini).

**Frontend** — halaman baru `(admin)/monitoring-wali-kelas/`, menu sidebar baru
"Monitoring" (`secondaryNav`, ikon `Radar`) sejajar "Log Aktivitas". 3 komponen section
dalam 1 file view: `PasswordSection`/`KelengkapanSection` (client-side search+sort,
REPLIKASI pola `akun-view.tsx` — dataset kecil, `useMemo` filter+sort lokal, TIDAK
perlu round-trip server) dan `RiwayatDownloadSection` (server-side via query param prefix
`rekapXxx` — REPLIKASI pola `log-view.tsx`, `router.push`+`router.refresh()`, debounce
search 300ms — prefix SENGAJA beda dari filter section lain di halaman yang sama supaya
tidak tabrakan nama query param).

**Bug ditemukan+diperbaiki (regresi T252)**: `journal-kelas-wali.controller.spec.ts`
(spec existing, T224a-T226) instantiate `JournalKelasWaliController` manual dengan 5
argumen — T252 menambah `ActivityLogService` sebagai argumen ke-6 (utk logging manual
export rekap), `pnpm jest` (bukan cuma `tsc`, yang tidak mendeteksi ini karena constructor
call di spec pakai `as unknown as X` cast) gagal total 1 file test SEBELUM diperbaiki.
Ditemukan lewat full-suite test run (bukan tsc), diperbaiki dengan tambah stub
`activityLog = {record: jest.fn().mockResolvedValue(undefined)}` + argumen ke-6 — 22/22
test lulus setelah perbaikan, TANPA test baru ditambahkan (konsisten instruksi sesi ini).

**Verifikasi**: `tsc --noEmit` api+web bersih, `next build` web sukses, full suite jest
624/633 lulus (9 gagal — SEMUA timeout `generateExcel()` di 2 file export report lain,
flaky pre-existing TIDAK TERKAIT T252/T253, sama kelas masalah yang sudah didokumentasi
di T249), `jest journal-kelas-wali.controller.spec.ts` 22/22 lulus (regresi
ditemukan+diperbaiki di atas).

**Update 2026-08-28 — dev server dinyalakan, verifikasi live dilakukan**: Docker (MySQL+Redis)
dinyalakan user, `prisma migrate deploy` T252 sukses diterapkan. API dev SEMPAT OOM crash
total (`nest start --watch` default pakai `tsc` full compile, machine RAM-constrained,
masalah yang sudah didokumentasi CLAUDE.md) — diperbaiki restart pakai `pnpm dev:swc`
(mode ringan yang sudah disiapkan khusus utk masalah ini, T249). Setelah itu: login
`admin`/`password123` sukses, `GET /api/monitoring/wali-kelas` dan
`/api/monitoring/wali-kelas/riwayat-download` mengembalikan data BENAR (lihat Acceptance
Criteria), halaman `/monitoring-wali-kelas` di browser tidak crash (307 redirect ke login,
bukan 500). T252+T253 SEKARANG SUDAH DIVERIFIKASI LIVE (bukan cuma tsc/build), dengan
catatan cakupan data seed masih terbatas 1 wali kelas (lihat Validasi Claudian).

---
tags: [absensi, tasks, fase-2, jurnal]
updated: 2026-07-22
---

# AbsenSI — Task Breakdown Fase 2: Dashboard Guru — Jurnal Mengajar & Absensi Mapel

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> Spec lengkap: [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]] — **baca dulu sebelum eksekusi task apapun di bawah**, banyak keputusan desain (kenapa gating berurutan begitu, kenapa mode blok global bukan per-kelas, dll) hanya dijelaskan lengkap di sana, task file sengaja tidak mengulang semuanya.
>
> **Status spec saat breakdown ini dibuat: masih ada open question yang belum final** (lihat bagian "Open Questions" di `dashboard-guru-jurnal.md`) — task di bawah ditulis berdasarkan keputusan yang SUDAH firm dari interview. Kalau saat eksekusi menemukan area yang ternyata menyentuh open question yang belum diputuskan, STOP dan konfirmasi ke user dulu, jangan berasumsi.
>
> **Eksekusi berurutan.** Urutan T038→T053 dirancang linear: schema dulu, lalu backend per-domain (resolver jadwal → izin → sesi mengajar → jurnal), baru frontend (guru dulu, admin_jurnal terakhir). Jangan loncat kecuali dependency "Depends on" di task terkait sudah selesai semua.
>
> **Catatan penomoran (2026-07-22):** T054, T055, T056 ditambahkan belakangan (fitur semester + loopback tahun ajaran + revisi mekanisme blok A/B) — nomornya lebih besar dari T038-T051 yang mereka pengaruhi, tapi **urutan eksekusi sebenarnya mengikuti "Depends on" di tiap file, bukan urutan angka**. T054 harus selesai SEBELUM T039 dan T047. Ikuti urutan blok di bawah, bukan urutan nomor.
>
> **⚠️ Revisi besar 2026-07-22 (baca sebelum eksekusi Blok 0):** mekanisme mode blok A/B DIROMBAK TOTAL. Draf awal T038/T039 merancang "1 titik acuan tanggal + hitung selisih minggu otomatis" (`schedule_config.minggu_acuan_tanggal`/`minggu_acuan_nilai`) — **mekanisme ini DIBATALKAN sepenuhnya** untuk menghindari kesalahan sistem dari perhitungan otomatis. Diganti: admin (`admin_jurnal`) input RENTANG TANGGAL EKSPLISIT untuk tiap minggu A dan B, per semester, lewat kalender visual (T056). Konsekuensinya:
> - `schedule_config` (T038) sekarang HANYA berisi `toleransi_terlambat_menit` — field mode/minggu acuan sudah dihapus dari sana
> - Field `mode` (blok/normal) pindah ke level `semesters` (T054) — per-semester, bukan konfigurasi sekolah global
> - Tabel baru `block_week_ranges` (T054) menyimpan rentang tanggal eksplisit per minggu A/B
> - `ScheduleResolverService.getMingguAktif()` (T039) sekarang PURE LOOKUP ke `block_week_ranges`, tidak ada aritmetika sama sekali
> - Validasi ketat WAJIB di T054: tidak boleh ada celah (gap) atau tumpang tindih (overlap) tanggal — baik dalam 1 semester maupun ANTAR semester (termasuk semester yang belum aktif tapi sedang disiapkan)
> - Celah tanggal yang belum admin lengkapi → **hard block total**, semua sesi "Mulai Mengajar" terkunci untuk tanggal itu sampai admin melengkapi
> - Semester Genap BOLEH disiapkan/diinput sebelum Semester Ganjil selesai (operasional), TAPI tanggalnya tetap tidak boleh tumpang tindih secara kalender dengan semester lain manapun (validasi real-time saat submit)
>
> Kalau Anda membaca versi task file yang masih menyebut "minggu_acuan" atau "hitung selisih minggu" — itu SUDAH USANG, jangan ikuti. Rujuk isi T038/T039/T054 versi terkini.

---

## 🟢 STATUS TERKINI (2026-07-22) — BACA INI DULU

**Ditemukan lewat pengecekan langsung ke working tree (bukan asumsi):** T038–T054 dan T056 **SUDAH DIEKSEKUSI** (kode backend `apps/api/src/{teaching-sessions,semesters,block-week-ranges,teacher-permits,schedule-resolver}` dan frontend `apps/web/src/app/{(guru),(admin-jurnal)}` sudah ada) — tapi **BELUM ADA SATU PUN YANG TER-COMMIT** (`git status` menunjukkan semuanya masih `??`/modified di working tree). Kalau Anda pindah sesi/mesin tanpa commit dulu, risiko kehilangan pekerjaan ini nyata — pertimbangkan commit sebelum lanjut ke task baru manapun.

| Task | Status nyata |
|---|---|
| T038, T039, T040, T041, T042, T043, T044, T045, T046, T047 | ✅ Kode ada di working tree (belum commit) |
| T048, T049 | ✅ Kode ada (`(guru)/guru/jurnal`, `(guru)/guru/sesi`, `(guru)/guru/izin`) |
| T050, T051 | ✅ Kode ada (`(admin-jurnal)/admin-jurnal/{toleransi,mapel,jadwal,izin}`) |
| T052, T053 | ✅ Kode ada (`(admin-jurnal)/admin-jurnal/wali-kelas`, `(guru)/guru/wali-kelas`) |
| T054, T056 | ✅ Kode ada (`apps/api/src/semesters`, `block-week-ranges`; `(admin-jurnal)/admin-jurnal/{semester,jadwal-blok}`) |
| **T055** | ❌ **BELUM dieksekusi** — tidak ditemukan perubahan di modul Rekap Admin Fase 1 untuk filter tahun ajaran/semester |
| T057, T058, T059 | ✅ Kode ada (belum commit) — token status warna + `packages/ui/src/components/{data-table,activity-feed}` (verifikasi visual Playwright: badge shipped/processing cocok hex spec, scroll activity feed clip di maxHeight) |
| **T060** | ❌ **BELUM dieksekusi** — audit visual menyeluruh (lihat Blok 9) |
| T061, T063 | ✅ Kode ada (belum commit) — schema biodata guru + wali murid siswa, T061 terverifikasi live di DB via MySQL MCP |
| **T062** | ❌ **BELUM dieksekusi** — ETL script belum ada file-nya sama sekali (lihat Blok 10) |
| T064 | ✅ Kode ada (belum commit) — primitif `Sheet`+`Tabs` di `packages/ui/src/components/ui/{sheet,tabs}.tsx` (lihat Blok 11) |
| T065 | ✅ Kode ada (belum commit) — `guru-view.tsx` migrasi ke Sheet+Tabs 3 section, `PersonStatus` enum ditambah `cuti` (migrasi DB `20260722163645_add_cuti_to_person_status`, sudah diterapkan) |
| T066 | ✅ Kode ada (belum commit) — `siswa-view.tsx` migrasi ke Sheet+Tabs 4 section (Data Pokok/Biodata/Kontak & Alamat/Data Wali), logic kondisional Status→Alasan Nonaktif→Tahun Lulus terverifikasi Playwright, backend `CreateStudentDto`+`students.service.ts` diperluas (field T063 belum pernah di-wire ke DTO sebelumnya, sama seperti temuan T065 di Teacher) |
| T067 | ✅ Kode ada (belum commit) — `@LogActivity` ditambahkan ke `kampus`, `mapel`, `semesters`, `teachers` controller (diverifikasi grep langsung); kemungkinan besar 14 controller lengkap sudah tercakup, BELUM diverifikasi 1-per-1 seluruh daftar T067 |
| T068, T069 | ✅ Kode ada (belum commit) — `apps/api/src/tv-piket/` (module/service/controller) dan `apps/api/src/tv-sessions/` (module/service/controller/dto) sudah ada |
| **T070** | ❌ **BELUM dieksekusi** — tidak ditemukan `apps/web/src/app` untuk route `/tv-piket/[kampusId]`, hanya ada `apps/web/src/app/api/tv-piket-proxy` (proxy API, bukan halaman UI) |
| **T071–T076** | ❌ **SEMUA BELUM dieksekusi** — Blok 14 (Plotting Siswa ke Kelas) dan Blok 15 (Direktori Siswa Piket) baru ditulis spec-nya di sesi ini, belum ada kode sama sekali (`kelas.controller.ts` tidak ada `tingkat`/`plot-siswa`/`kenaikan-massal`, `students.controller.ts` belum buka akses `guru_piket` untuk `GET`) |

> ✅ **2 bug ditemukan & DIPERBAIKI saat verifikasi T064/T065 (di luar scope asal, tapi sudah dieksekusi):**
> 1. `Dialog`/`Sheet` render dengan background beige page (`#EEE6D9`), bukan putih. Root cause: className `bg-background` resolve ke CSS var `--background` (page), bukan `--card`/`bg-surface` (putih). Fix: ganti semua `bg-background` → `bg-surface` di `dialog.tsx`, `sheet.tsx`, `button.tsx` (varian outline), `tabs.tsx` (trigger tidak aktif).
> 2. `shadow-card`/`shadow-popover` render sebagai glow PUTIH (bukan shadow gelap) — root cause GANDA: (a) `hsl(var(--color-shadow) / alpha)` tidak ter-resolve browser dengan benar kalau disimpan di custom property `--tw-shadow` (harus pakai `rgba(23,20,18,x)` literal), DAN (b) nama key `card`/`popover` di `boxShadow` collide dengan `colors.card`/`colors.popover` — Tailwind auto-generate utility `shadow-{colorName}` yang override dengan urutan CSS menang, selalu putih. Fix: rename key jadi `elevated`/`elevated-hover`/`overlay` (bukan collide dengan `colors`), pakai `rgba()` literal. **Class Tailwind BERUBAH**: `shadow-card`→`shadow-elevated`, `shadow-popover`→`shadow-overlay` (sudah di-replace di 53 file). Verifikasi Playwright: computed `box-shadow` sekarang `rgba(23, 20, 18, x)`, bukan putih.
>
> ⚠️ **Keputusan user (2026-07-22):** `PersonStatus` diperluas dengan value `cuti` (dulu cuma `aktif`/`nonaktif`) untuk dropdown Status guru di T065. Enum ini SHARED dengan `Student` — semua query existing pakai `where: { status: "aktif" }` eksplisit (bukan switch exhaustive), jadi `cuti` otomatis ter-exclude dengan benar tanpa perlu ubah logic lain. Migrasi: `20260722163645_add_cuti_to_person_status`.

**Yang PALING PENTING untuk sesi berikutnya:** T038–T054/T056 sudah ada kode-nya tapi **belum pernah diverifikasi visual nyata** (dijalankan di browser, di-screenshot) — itulah kenapa T060 (audit visual) jadi prioritas berikutnya, BUKAN mulai dari T055 begitu saja. Baca Blok 9 di bawah.

> **Update 2026-07-23:** verifikasi ulang working tree — T067, T068, T069 ternyata SUDAH ada kode-nya (dieksekusi paralel di luar percakapan ini). T070 (UI TV Piket), T055, T060, T062, dan seluruh Blok 14-15 (T071-T076) MASIH BELUM ada kode sama sekali — cek tabel di bawah untuk detail per-task, jangan asumsikan "banyak yang selesai" berarti semua selesai.

---

## 📊 Progress

| Blok | Task | Selesai |
|---|---|---|
| 0 — Schema & Resolver Jadwal | T038–T039, T054 | 3/3 (kode ada, belum commit) |
| 1 — Job Sesi & Config | T040, T042, T044 | 3/3 (kode ada, belum commit) |
| 2 — API Inti Sesi Mengajar | T041, T043, T045 | 3/3 (kode ada, belum commit) |
| 3 — API Izin Guru & RBAC Admin Jurnal | T046–T047 | 2/2 (kode ada, belum commit) |
| 4 — UI Dashboard Guru | T048–T049 | 2/2 (kode ada, belum commit) |
| 5 — UI Dashboard Admin Jurnal | T050–T051 | 2/2 (kode ada, belum commit) |
| 6 — Wali Kelas | T052–T053 | 2/2 (kode ada, belum commit) |
| 7 — Loopback Rekap | T055 | **0/1 — BELUM DIKERJAKAN** |
| 8 — UI Kalender Blok Minggu | T056 | 1/1 (kode ada, belum commit) |
| 9 — Perbaikan UI dari Audit Referensi EzMart | T057–T060 | 3/4 (kode ada, belum commit) — T060 belum dikerjakan |
| 10 — Migrasi Database Lama | T061–T063 | 2/3 (kode ada, belum commit) — T062 (ETL) belum dikerjakan |
| 11 — Form Input Modern (Sheet+Tabs) | T064–T066 | 3/3 (kode ada, belum commit) |
| 12 — Kelengkapan Activity Log | T067 | 1/1 (kode ada, belum commit — belum diverifikasi lengkap semua 14 controller) |
| 13 — TV Piket | T068–T070 | 2/3 (kode ada, belum commit) — T070 (UI) belum dikerjakan |
| 14 — Plotting Siswa ke Kelas | T071–T075 | **0/5 — BELUM DIKERJAKAN** (celah baru ditemukan 2026-07-22, T073 direvisi total setelah diskusi lanjutan) |
| 15 — Direktori Siswa Dashboard Piket | T076 | **0/1 — BELUM DIKERJAKAN** (celah baru ditemukan 2026-07-22) |
| **Total** | | **29/38** kode ada (belum commit) · **9/38 murni belum dikerjakan** |

> "Selesai" di atas = kode SUDAH DITULIS, BUKAN berarti sudah diverifikasi jalan/benar. Verifikasi nyata adalah tujuan T060.

---

## 🏗️ Blok 0 — Schema & Resolver Jadwal
> Fondasi seluruh fitur. Semua task lain depend langsung/tidak langsung ke blok ini.

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T038-schema-jurnal-mengajar|T038 — Prisma Schema: Jurnal Mengajar, Izin Guru, Geofence]] *(field mode/minggu-acuan sudah dihapus dari task ini, lihat catatan revisi di atas)*
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T054-schema-semester|T054 — Schema: Semester + Rentang Blok A/B Eksplisit]] *(kerjakan SEBELUM T039 meski nomornya besar — T039 depend ke task ini, task PALING SENSITIF di blok ini karena berisi validasi no-gap/no-overlap)*
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T039-resolver-jadwal-aktif|T039 — Service Resolver: Jadwal Aktif Hari Ini (Lookup Rentang Blok A/B)]] *(direvisi total — sekarang pure lookup, bukan hitung selisih minggu)*

## 🏗️ Blok 1 — Job Sesi & Config
> Job harian generate sesi, config toleransi keterlambatan, job auto-close. Bisa dikerjakan agak paralel setelah Blok 0, tapi T044 depend ke T043 (Blok 2) jadi urutan penomoran tetap dijaga.

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T040-job-generate-teaching-sessions|T040 — Job Harian: Generate teaching_sessions]]
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T042-toleransi-keterlambatan-config|T042 — Config Toleransi Keterlambatan Mengajar]]
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T044-job-auto-close-sesi|T044 — Job Auto-Close Sesi & Flag Follow-Up Izin]] *(depend ke T043, kerjakan setelah Blok 2)*

## 🏗️ Blok 2 — API Inti Sesi Mengajar
> Endpoint yang dipakai langsung oleh dashboard guru: lihat jadwal hari ini, mulai mengajar (dengan gating tap gerbang + waktu + geofence), isi jurnal & koreksi presensi.

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T041-api-jadwal-guru-hari-ini|T041 — API: Jadwal Guru Hari Ini]]
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T043-start-session-geofence|T043 — API: Mulai Mengajar (Start Session) dengan Validasi Geofence]] ⚠️ *task paling sensitif — baca urutan validasi di spec dengan teliti*
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T045-api-jurnal-dan-presensi-kelas|T045 — API: Jurnal Mengajar & Presensi Kelas]]

## 🏗️ Blok 3 — API Izin Guru & RBAC Admin Jurnal
> Alur izin 2-tahap (admin approve dulu, baru guru submit tugas) + penegakan role `admin_jurnal` terkunci ke domain jurnal.

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T046-api-izin-guru|T046 — API: Izin Guru Tidak Mengajar]]
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T047-rbac-admin-jurnal-mapel-jadwal|T047 — RBAC admin_jurnal + API Kelola Mapel & Jadwal Mengajar]]

## 🏗️ Blok 4 — UI Dashboard Guru
> Landing page guru (jadwal hari ini + tombol Mulai Mengajar/Izin) dan halaman detail sesi (jurnal + presensi + form tugas titipan).

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T048-ui-dashboard-guru-halaman-utama|T048 — UI: Dashboard Guru — Halaman Utama]]
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T049-ui-halaman-sesi-jurnal-presensi|T049 — UI: Halaman Sesi — Jurnal, Presensi Kelas & Form Izin]]

## 🏗️ Blok 5 — UI Dashboard Admin Jurnal
> Dashboard baru untuk role `admin_jurnal`: toleransi keterlambatan, kelola mapel & jadwal mengajar (per semester), kelola izin guru.

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T050-ui-dashboard-admin-jurnal-jadwal|T050 — UI: Dashboard Admin Jurnal — Mapel, Jadwal & Toleransi Keterlambatan]] *(halaman "Konfigurasi Mode Blok" versi lama DIHAPUS dari task ini, lihat T056)*
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T051-ui-admin-jurnal-izin-guru|T051 — UI: Dashboard Admin Jurnal — Kelola Izin Guru]]

## 🏗️ Blok 6 — Wali Kelas
> Final (2026-07-21) — bukan role baru, extend akun `guru` dengan `kelas_id_wali` (pola identik `guru_piket.kampus_id`). Read-only murni. v1 scope: rekap kehadiran + rekap per mapel + catatan/kendala saja (bukan jurnal lengkap materi/tujuan/tugas — itu sudah dicatat sebagai perluasan masa depan di spec, tinggal buka filter kalau dibutuhkan nanti).

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T052-schema-dan-assign-wali-kelas|T052 — Schema kelas_id_wali + UI Admin Jurnal: Assign Wali Kelas]]
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T053-ui-menu-wali-kelas-guru|T053 — UI: Menu Wali Kelas di Dashboard Guru]]

## 🏗️ Blok 7 — Loopback Rekap
> Filter tahun ajaran/semester di halaman Rekap Admin existing (Fase 1) — data historis sudah ada, ini murni tambahan UI+filter.

- [ ] [[Projek/AbsenSI/06-Features/tasks/T055-rekap-loopback-tahun-ajaran|T055 — Rekap Kehadiran: Filter Loopback Tahun Ajaran & Semester]]

## 🏗️ Blok 8 — UI Kalender Blok Minggu
> Kalender visual untuk admin_jurnal menandai tiap minggu sebagai A/B — MENGGANTIKAN form input tanggal manual, mengurangi risiko human error untuk belasan pasangan rentang per semester.

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T056-ui-kalender-blok-minggu|T056 — UI: Kalender Visual Input Rentang Blok Minggu A/B]]

## 🏗️ Blok 9 — Perbaikan UI dari Audit Referensi EzMart (BARU, 2026-07-22 — T057-T059 dieksekusi 2026-07-22, T060 masih tersisa)
> User menunjukkan 2 gambar referensi Figma asli EzMart yang jadi dasar `design-system/*.md`. Audit manual (baca gambar langsung) menemukan `03-components.md` sudah akurat untuk 1 gambar tapi gambar KEDUA (dashboard dengan tabel "Recent Orders" + "Recent Activity") belum pernah didokumentasikan — 4 komponen hilang, dan status badge tabel ternyata butuh 4 kategori warna (bukan cuma success/danger). Dokumen desain sudah diperbaiki (`01-colors.md`, `03-components.md`, `DESIGN.md`); blok ini adalah TASK EKSEKUSI dari perbaikan tersebut. **Kerjakan T057 dulu (fondasi token), lalu T058/T059 bisa paralel, T060 (audit visual) terakhir sebagai penutup — idealnya T060 dikerjakan setelah T057-T059 SEKALIGUS setelah rekonsiliasi status "kode ada tapi belum commit" di atas (audit percuma kalau kode yang diaudit belum stabil).**

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T057-tambah-token-status-workflow|T057 — Tambah Token Warna Status Workflow (Shipped/Processing) ke Tailwind Config]] — token ditambahkan di `packages/ui/src/globals.css` + `packages/config/tailwind.config.ts` (grup `status.shipped`/`status.processing`), komentar path vault diperbaiki. Verifikasi: computed style browser cocok persis `#FDECD1`/`#B8720A`/`#EDE3F7`/`#7C4FC7`.
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T058-komponen-data-table-generik|T058 — Komponen Generik: Data Table dengan Status Badge Workflow]] *(depend T057)* — dibuat di `packages/ui/src/components/data-table/` (`DataTableCard`, `DataTableHeader`, `DataTableRow`+`DataTableCell`, `DataTableEntityCell`, `StatusBadge` 5 varian). Migrasi `izin-table.tsx` **DITUNDA** — row-nya sudah punya keyboard-a11y custom (tabIndex/role/onKeyDown dari perbaikan audit UI sebelumnya) dan badge "Belum diisi" pakai `primary-tint`/`primary-hover` yang bukan salah satu dari 5 varian `StatusBadge`; migrasi paksa berisiko regresi a11y atau membuka varian warna ke-6 di luar scope. Evaluasi ulang kalau ada task lanjutan yang menyentuh file itu.
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T059-komponen-activity-feed-generik|T059 — Komponen Generik: Activity Feed]] *(tidak depend T057/T058, bisa paralel — belum ada pemakai nyata sampai TV Piket/Dashboard Kepsek mulai dikerjakan)* — dibuat di `packages/ui/src/components/activity-feed/` (`ActivityFeedCard` dengan `maxHeight` default 240-320px + scroll internal, `ActivityFeedItem` dengan 4 varian `iconChipColor`). Verifikasi: 10 item dummy, scrollHeight 520px vs clientHeight 240px — clip+scroll terkonfirmasi via Playwright.
- [ ] [[Projek/AbsenSI/06-Features/tasks/T060-audit-visual-dashboard-guru-admin-jurnal|T060 — Audit Visual Nyata: Dashboard Guru & Admin Jurnal (Playwright)]] ⚠️ *paling penting untuk sesi berikutnya — T038-T054/T056 belum pernah diverifikasi visual sama sekali*

## 🏗️ Blok 10 — Migrasi Database Lama (BARU, 2026-07-22, semuanya BELUM DIKERJAKAN)
> AbsenSI adalah rebuild dari aplikasi Laravel lama. Audit struktur (`Projek/AbsenSI/06-Features/migrasi-database-lama.md`) menemukan field biodata guru & wali murid siswa yang hilang saat rebuild — sudah diputuskan ditambahkan kembali. **Ini scope di LUAR "Fase 2 Jurnal" murni** (lebih ke perbaikan Fase 1 Core/Students/Teachers), tapi ditaruh di tracker yang sama karena ditemukan & diputuskan dalam rangkaian sesi yang sama. Urutan: T061+T063 (schema, bisa paralel) → T062 (ETL, WAJIB dry-run dulu, JANGAN langsung --execute).

- [x] (**kode ada, belum commit — ditemukan 2026-07-22 saat cek dependency T065**) [[Projek/AbsenSI/06-Features/tasks/T061-schema-tambahan-biodata-lama|T061 — Schema: Tambah Field Biodata Guru dari Database Lama]] — field `gelarDepan/gelarBelakang/tempatLahir/tanggalLahir/jenisKelamin/agama/alamat/statusPernikahan/statusKepegawaian` sudah ada di model `Teacher`, terverifikasi `DESCRIBE teachers` via MySQL MCP — migrasi SUDAH diterapkan ke DB live, bukan cuma di schema.prisma.
- [x] (**kode ada, belum commit — ditemukan 2026-07-22**) [[Projek/AbsenSI/06-Features/tasks/T063-schema-data-wali-murid|T063 — Schema: Tambah Data Wali Murid ke Student]] — field `namaAyah/namaIbu/rtRw` dkk sudah ada di model `Student` (terlihat di `core-types.ts` & schema.prisma), belum diverifikasi ulang senyakin T061 tapi pola sama.
- [ ] [[Projek/AbsenSI/06-Features/tasks/T062-etl-migrasi-data-lama|T062 — ETL Script: Migrasi Data dari Database Lama]] ⚠️ **BELUM ADA** (tidak ditemukan file ETL di `apps/api/src`) — *depend T061+T063 [selesai] — JANGAN --execute tanpa review --dry-run bersama user*

## 🏗️ Blok 11 — Form Input Modern: Sheet + Tabs (BARU, 2026-07-22 — T064/T065/T066 SEMUA dieksekusi 2026-07-22)
> Form `Tambah Guru`/`Tambah Siswa` existing pakai popup `Dialog` kecil sederhana — sudah 11 field (siswa) dan akan bertambah ~9-13 lagi dari Blok 10 (T061/T063). Pola baru "Sheet + Tabs" ditambahkan ke `design-system/03-components.md` (bagian "Form Input Panjang") sebagai **rujukan WAJIB untuk SEMUA form >6 field ke depan**, bukan cuma guru/siswa. **T064 harus selesai duluan** (primitif dasar), lalu T065/T066 depend ke T061/T063 (field baru harus sudah ada di backend sebelum form-nya dibuat).
>
> ⚠️ **Ditemukan saat T066 (belum diperbaiki, DI LUAR scope T066):** endpoint `POST /students` (`students.controller.ts` baris 39) TIDAK punya `@LogActivity`, melanggar aturan permanen di catatan #16 bawah. Ini gap PRA-EXISTING (bukan endpoint baru dari T066), jadi tidak diperbaiki otomatis di sesi ini — cek juga endpoint serupa (`POST /teachers` di T065) sebelum menutup Blok 12 (T067).

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T064-primitif-sheet-tabs|T064 — Primitif UI Baru: Sheet & Tabs]] — dibuat di `packages/ui/src/components/ui/sheet.tsx` (berbasis `@radix-ui/react-dialog`, sama seperti `Dialog` existing) dan `tabs.tsx` (baru install `@radix-ui/react-tabs`). Verifikasi Playwright: lebar Sheet 560px (dalam rentang 480-560px spec), radius `24px` HANYA di sudut kiri (`borderTopLeftRadius`/`borderBottomLeftRadius`), sisi kanan `0px`; footer tetap terlihat (posisi tidak berubah) saat body di-scroll dengan 20 dummy field; Tabs aktif render solid `#F5841F`+teks putih, tidak aktif border tipis+teks gelap, semua `border-radius: 9999px` (pill), TIDAK ada underline indicator.
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T065-form-guru-sheet-lengkap|T065 — UI: Form Guru — Migrasi ke Sheet + Field Biodata Lengkap]] — `guru-view.tsx` migrasi `Dialog`→`Sheet`, 3 tab (Data Pokok/Biodata/Kontak & Alamat), semua field baru opsional kecuali NIY+Nama. Backend: `CreateTeacherDto` diperluas + `teachers.service.ts` diperbaiki (bug ditemukan saat testing: `data: dto` blind-spread gagal karena `tanggalLahir` string mentah bukan `Date`, sekarang eksplisit `new Date(dto.tanggalLahir)` sama seperti pola `students.service.ts`). Verifikasi: submit minimal (NIY+Nama saja) berhasil, submit lengkap semua field berhasil + dicek `SELECT * FROM teachers` via MySQL MCP cocok persis, data persist saat pindah tab, footer sticky. **Ada 2 baris data uji coba tertinggal di tabel `teachers`** (niy `199001012020011099` dan `199912312099011000`) — MySQL MCP read-only jadi tidak bisa dihapus dari sini, dan UI Guru belum ada tombol hapus; user perlu hapus manual.
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T066-form-siswa-sheet-lengkap|T066 — UI: Form Siswa — Migrasi ke Sheet + Field Wali Murid & Status Lulus]] — `siswa-view.tsx` migrasi `Dialog`→`Sheet`, 4 tab (Data Pokok/Biodata/Kontak & Alamat/Data Wali). Logic kondisional 2-tingkat (`Status=Nonaktif` → tampilkan Alasan Nonaktif → `Alasan=Lulus` → tampilkan Tahun Lulus) diverifikasi Playwright: semua transisi (nonaktif→lulus→tahun muncul, lulus→mengundurkan_diri→tahun hilang lagi, balik ke aktif→semua hilang) sesuai spec persis. Backend: `CreateStudentDto` diperluas 6 field baru (`status`, `alasanNonaktif`, `tahunLulus`, `noHpSiswa`, `noHpAyah`, `noHpIbu` — field T063 ternyata belum pernah di-wire ke DTO, sama seperti temuan T065 di Teacher) + `students.service.ts` create/update disesuaikan. Tidak ada form edit siswa terpisah (dicek di `siswa-detail-view.tsx`, cuma ada Dialog hapus-foto — benar tetap Dialog, ≤6 field). Verifikasi: submit minimal (NISN+Nama+Kelas) berhasil, submit lengkap semua field + `SELECT * FROM students` via MySQL MCP cocok persis, badge status pakai `StatusBadge` shared (ikon per varian, bukan hand-rolled). **2 baris data uji coba tertinggal** (nisn `2026999001`, `2026999002`) — sama seperti T065, tidak ada tombol hapus siswa di UI dan MySQL MCP read-only, perlu dihapus manual.

## 🏗️ Blok 12 — Kelengkapan Activity Log (BARU, 2026-07-22, BELUM DIKERJAKAN)
> Audit menemukan **14 dari 22 controller** dengan endpoint mutasi TIDAK memakai `@LogActivity` — termasuk aksi berdampak luas (aktivasi semester, CRUD guru, izin guru+upload bukti, perubahan jadwal blok). Infrastrukturnya sudah ada (`ActivityLogInterceptor` + decorator), ini murni menambahkan decorator yang hilang secara konsisten.

- [x] (**kode ada, belum commit — cek 4 sample controller, belum diverifikasi semua 14**) [[Projek/AbsenSI/06-Features/tasks/T067-lengkapi-activity-log-semua-controller|T067 — Lengkapi @LogActivity di Semua Controller Mutasi yang Belum Tercatat]]

## 🏗️ Blok 13 — TV Piket (BARU, 2026-07-22, spec DIFINALISASI hari ini — T068/T069 dieksekusi, T070 masih tersisa)
> `tv-piket.md` sebelumnya `planning-interview` dengan 5 open question ("menunggu sketsa") — **dituntaskan 2026-07-22** tanpa sketsa, berdasarkan referensi EzMart + komponen generik `DataTableCard`/`ActivityFeedCard` (T058/T059) yang sudah ada. Layout final: bar persentase 1 baris + 2 kolom tengah (siswa tidak hadir + guru belum mulai) + 1 baris bawah (guru izin). Auth pakai token TANPA expiry (beda dari TV Kepsek 30 hari) karena TV ini murni read-only — tapi WAJIB ada mekanisme revoke manual oleh `super_admin`. Urutan: T068 (auth/token) → T069 (API data, reuse Rekap Fase 1) → T070 (UI).

- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T068-schema-auth-tv-piket|T068 — Schema & Auth: Token TV Piket (Tanpa Expiry, Revocable)]] — `apps/api/src/tv-sessions/` (module/service/controller/dto) sudah ada
- [x] (**kode ada, belum commit**) [[Projek/AbsenSI/06-Features/tasks/T069-api-data-tv-piket|T069 — API: Data Agregat TV Piket (4 Widget)]] *(depend T068)* — `apps/api/src/tv-piket/` (module/service/controller) sudah ada
- [ ] [[Projek/AbsenSI/06-Features/tasks/T070-ui-halaman-tv-piket|T070 — UI: Halaman TV Piket]] ⚠️ **BELUM ADA** — hanya ditemukan `apps/web/src/app/api/tv-piket-proxy` (proxy API route), belum ada halaman `/tv-piket/[kampusId]` yang sesungguhnya *(depend T068 [selesai], T069 [selesai], DataTableCard/ActivityFeedCard T058/T059 [selesai])*

## 🏗️ Blok 14 — Plotting Siswa ke Kelas (BARU, 2026-07-22, semua BELUM DIKERJAKAN)
> Celah ditemukan: siswa cuma bisa di-assign ke kelas SEKALI saat form "Tambah Siswa", tidak ada mekanisme pindah kelas maupun penempatan massal. Spec lengkap: [[Projek/AbsenSI/06-Features/plotting-siswa-kelas|plotting-siswa-kelas.md]]. 3 mekanisme terpisah + 1 fitur terkait (Tandai Keluar): (1) Penempatan siswa baru via paste-NISN dengan validasi visual hijau/merah real-time, (2) Kenaikan kelas massal — **REVISI 2026-07-22: 1 halaman semua kelas sekaligus**, dropdown tujuan per baris, opsi "Lulus" khusus kelas XII, (3) Pindah kelas individual dari halaman detail siswa, (4) Tandai siswa Lulus/Mengundurkan Diri. **T075 (kolom tingkat) WAJIB sebelum T073** — dibutuhkan untuk logic "opsi Lulus cuma di kelas XII". Urutan: T071+T075 (schema, paralel) → T072 → T073 (revisi total, baca ulang task file-nya) → T074 independen kapan saja.

- [ ] [[Projek/AbsenSI/06-Features/tasks/T071-schema-tinggal-kelas|T071 — Schema: Field Tinggal Kelas + Kolom Jumlah Siswa]]
- [ ] [[Projek/AbsenSI/06-Features/tasks/T075-schema-tingkat-kelas|T075 — Schema: Kolom Tingkat Terpisah di Kelas]] *(baru, 2026-07-22 — WAJIB sebelum T073)*
- [ ] [[Projek/AbsenSI/06-Features/tasks/T072-plot-siswa-baru-ke-kelas|T072 — UI+API: Penempatan Siswa Baru ke Kelas (Paste-NISN)]] *(depend T071)*
- [ ] [[Projek/AbsenSI/06-Features/tasks/T073-kenaikan-kelas-massal|T073 — UI+API: Kenaikan Kelas Massal (Menu Tersendiri, Semua Kelas Sekaligus)]] ⚠️ *DIREVISI TOTAL 2026-07-22 — desain lama (1 kelas per proses) sudah usang, baca ulang task file. Depend T071, T072, T075*
- [ ] [[Projek/AbsenSI/06-Features/tasks/T074-pindah-kelas-individual-dan-tandai-keluar|T074 — UI: Pindah Kelas Individual + Tandai Siswa Keluar]] *(depend T063 [selesai] — independen, bisa paralel)*

## 🏗️ Blok 15 — Direktori Siswa Dashboard Piket (BARU, 2026-07-22, BELUM DIKERJAKAN)
> Celah ditemukan: dashboard piket cuma punya daftar kontekstual (tap hari ini, belum kembali, terkunci), tidak ada direktori siswa yang bisa dicari/difilter. Spec: [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]] bagian "6b" (baru). **REUSE halaman detail siswa existing** (foto+biodata+riwayat catatan sudah lengkap di `/admin/siswa/[id]`) — task ini murni buka akses role + 1 halaman list baru, BUKAN membangun halaman detail dari nol. **Pengecualian sadar**: scope LINTAS KAMPUS (beda dari semua fitur piket lain yang di-scope kampus).

- [ ] [[Projek/AbsenSI/06-Features/tasks/T076-direktori-siswa-dashboard-piket|T076 — UI+API: Direktori Siswa (Search & Filter) di Dashboard Piket]]

---

## 📌 Catatan Penting

1. **Sebelum mulai coding setiap task:** baca `Ref` yang direferensikan di task file, dan baca ulang bagian relevan di `dashboard-guru-jurnal.md` — spec ini masih berstatus `planning-interview`, bukan `final`, jadi ada baiknya cross-check tidak ada perubahan sejak breakdown ini dibuat.
2. **Geofence hard-block tanpa override (T043)** adalah keputusan sadar user meski berisiko UX (GPS gagal di gedung beton, dst) — JANGAN diam-diam menambahkan jalur override "untuk jaga-jaga", itu perubahan scope yang harus dikonfirmasi ulang ke user kalau ternyata dibutuhkan.
3. **Open questions yang SENGAJA tidak dijawab task ini** (jangan improvisasi sendiri, tunggu keputusan user): batas waktu edit jurnal setelah closed, radius geofence pasti per kampus (admin isi manual via UI kampus — bukan scope batch task ini, kemungkinan perlu task tambahan kecil untuk extend UI master-data kampus), field "Tujuan Pembelajaran" terhubung CP/TP formal atau tidak. **Role Wali Kelas SUDAH final** (lihat Blok 6), bukan open question lagi.
4. **TV Piket ([[Projek/AbsenSI/06-Features/tv-piket|tv-piket.md]])** bergantung ke data dari batch task ini (`teaching_sessions`, `teacher_permits`) — jangan mulai task TV Piket sebelum Blok 0-3 batch ini selesai dan datanya bisa diverifikasi nyata.
5. **T053 (menu Wali Kelas) HANYA tampilkan field `catatan` dari jurnal**, bukan materi/tujuan/tugas — kalau nanti diminta buka akses penuh, itu perubahan kecil (extend query + ubah tampilan), sudah didokumentasikan caranya di spec, JANGAN dikerjakan sekarang tanpa instruksi eksplisit.
16. **WAJIB `@LogActivity` di SETIAP controller/endpoint mutasi BARU** (POST/PATCH/PUT/DELETE yang mengubah data) — ini aturan permanen sejak audit T067 (2026-07-22), bukan cuma untuk task itu saja. Kalau menulis controller baru di task manapun (T038-T066 dan seterusnya), cek dulu apakah decorator ini sudah terpasang sebelum menandai task selesai. Lihat memory permanen `feedback_wajib_log_activity`.
6. **`teacher_permits.bukti_file_path` WAJIB diisi sejak awal (T038/T046)** — beda dari `tugas_file_path` yang nullable. Jangan tertukar keduanya: `bukti_file_path` = alasan izin (surat dokter/nota dinas/dst), `tugas_file_path` = materi pengganti untuk siswa.
7. **Semester (T054) adalah domain `super_admin`, bukan `admin_jurnal`** — meski konsumen utamanya (assign jadwal) ada di dashboard admin_jurnal. Jangan beri admin_jurnal wewenang create/edit/activate semester, hanya read + pilih dari dropdown. **Tapi `block_week_ranges` (isi rentang blok A/B) ADALAH wewenang `admin_jurnal`** — dua tabel beda pemilik, jangan disamakan.
8. **Validasi no-gap/no-overlap di T054 adalah bagian TERPENTING dari seluruh batch semester** — jangan pernah skip atau menyederhanakan validasi ini demi kecepatan implementasi. Kalau validasi ini bocor, `ScheduleResolverService` (T039) bisa menghasilkan data ambigu (>1 hasil untuk 1 tanggal) yang merembet ke seluruh sistem (sesi mengajar, jurnal, rekap).
9. **Audit kritik tambahan (2026-07-22) sudah ditutup di T054:** (a) race condition submit rentang overlap harus dicegah dengan DB transaction+lock, bukan cek-lalu-insert biasa; (b) `PATCH /semesters/:id/activate` WAJIB tolak (409) semester `mode: blok` yang masih ada lubang jadwal — mencegah insiden "semua guru terkunci besok"; (c) `block_week_ranges` tidak boleh diubah/dihapus untuk tanggal hari ini/masa lalu, hanya masa depan; (d) endpoint `upcoming-gaps` WAJIB dipakai sebagai banner peringatan proaktif di layout admin_jurnal (T050), bukan cuma info pasif yang harus dicari manual. Semua ini SUDAH masuk ke T054/T050 versi terkini — jangan implementasi versi yang lebih longgar dari ini.
10. **Trade-off yang disadari, JANGAN "diperbaiki" tanpa instruksi:** hard block lubang jadwal berlaku per-SEMESTER (semua kelas terkunci sekaligus), bukan per-kelas granular — meski ada kelas yang jadwalnya `setiap_minggu` dan sebenarnya tidak terdampak minggu A/B. Ini keputusan sadar demi kesederhanaan 1 sumber kebenaran, lihat "Trade-off yang Disadari" di `dashboard-guru-jurnal.md`.
11. **Sebelum sesi berikutnya mulai eksekusi apapun: pertimbangkan `git add`+`git commit` dulu untuk T038-T054/T056** — semuanya sudah ada di working tree tapi belum ter-commit sama sekali (dicek 2026-07-22). Ini bukan instruksi untuk auto-commit tanpa diminta (tetap ikuti aturan "jangan commit kecuali diminta user"), tapi ini risiko nyata yang perlu diingatkan ke user di awal sesi berikutnya.
12. **MCP tools tersedia untuk sesi berikutnya:** Playwright MCP (`mcp__playwright__*`) untuk screenshot/interaksi browser nyata, MySQL MCP read-only (`mcp__mysql-absensi__mysql_query`) untuk cek state data langsung, dan plugin Impeccable (`/impeccable audit`, `/impeccable critique`, dst) untuk deteksi anti-pola desain otomatis — pakai ketiganya terutama untuk T060.
13. **`DESIGN.md` di root repo dan `06-Features/design-system/*.md` di vault HARUS disinkronkan** — vault adalah sumber kebenaran, `DESIGN.md` di root adalah rangkuman untuk Impeccable. Kalau salah satu diedit, cek yang lain juga perlu update (lihat riwayat 2026-07-22: token status-shipped/processing ditambahkan ke KEDUANYA).
14. **Aturan Sheet+Tabs (T064) berlaku PERMANEN untuk SEMUA form ke depan, tidak cuma T065/T066** — kapan pun ada task baru yang butuh form input >6 field, WAJIB pakai pola Sheet+Tabs dari `03-components.md`, JANGAN buat `Dialog` baru untuk kasus itu. Form ≤6 field sederhana tetap boleh `Dialog` kecil seperti sebelumnya.
15. **Migrasi database lama (Blok 10) dan form modern (Blok 11) SALING TERKAIT tapi independen urutannya** — Blok 11 (T065/T066) depend ke Blok 10 (T061/T063) untuk field baru di backend, tapi Blok 10 tidak depend ke Blok 11. Bisa kerjakan Blok 10 duluan tanpa menunggu Blok 11, tapi TIDAK sebaliknya.

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]] — spec fitur lengkap
- [[Projek/AbsenSI/13-Backlog|13-Backlog]] — posisi fitur ini dalam roadmap Fase 2
- [[Projek/AbsenSI/TASKS-FASE-1|TASKS-FASE-1]] — pola/format breakdown task yang diikuti

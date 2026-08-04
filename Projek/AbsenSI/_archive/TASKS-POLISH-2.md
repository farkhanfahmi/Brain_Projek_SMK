---
tags: [absensi, tasks, polish, batch2]
updated: 2026-07-21
---

# Task Perbaikan Pasca-Fase-1 — Batch 2

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> 10 usulan perbaikan dari user (Fahmi), disampaikan 2026-07-20 setelah uji coba menyeluruh aplikasi (login role, kiosk fisik, dashboard piket). Dianalisis lewat riset kode langsung (bukan asumsi) sebelum ditulis jadi task — beberapa usulan ternyata sudah sebagian ada di kode (dicatat sebagai "sudah ada, cuma bug X" alih-alih "buat dari nol"), sebagian benar-benar butuh fitur baru total. Dikerjakan bertahap satu-per-satu, sama seperti pola [[Projek/AbsenSI/TASKS-POLISH-1|TASKS-POLISH-1]].

---

## 📊 Progress

| Kategori | Task | Selesai |
|---|---|---|
| Bug kiosk & UI | T029–T030 | 2/2 |
| Dashboard Piket — navigasi | T031 | 1/1 |
| Jadwal Piket (fitur baru) | T032 | 1/1 |
| Import Data (UX) | T033 | 1/1 |
| Activity Log (fitur baru) | T034 | 1/1 |
| Rekap PDF (sengaja dilewati) | T035 | 0/1 |
| Detail Siswa (foto + riwayat) | T036 | 1/1 |
| Lock Otomatis 2x Terlambat | T037 | 1/1 |
| **Total** | | **8/9** (T035/PDF sengaja dilewati) |

---

## ✅ Sudah Diklarifikasi Lewat Riset Kode (Sebelum Menulis Task)

Poin-poin usulan user berikut ternyata **sudah ada** sebagiannya di kode — dicatat di sini supaya tidak salah asumsi "belum ada sama sekali":

1. **UI scan kiosk (usulan #1)** — foto, nama, DAN jam **sudah** dirender di `apps/kiosk/src/components/feedback-screen.tsx` (lewat `KioskAvatar` + `response.name` + `response.time`). Warna beda untuk terlambat (merah `#E13B3B`) vs normal (hijau `#1E9E4C`) **juga sudah ada** — usulan #10 bagian "beda warna" sebenarnya sudah selesai, cuma perlu diverifikasi user melihatnya dengan benar. Root cause klaim "cuma nama" kemungkinan besar: (a) siswa dummy belum punya foto sama sekali → fallback avatar seharusnya tetap muncul tapi ada bug rendering (lihat T029), atau (b) foto ada tapi gagal fetch dan `onError` cuma `display:none` tanpa fallback apapun (bug nyata, lihat T029).
2. **Dashboard Piket — siswa belum-scan/izin/terlambat (usulan #2 bagian pertama)** — board utama **sudah** menampilkan semua status ini (`Belum Hadir`/`Hadir`/`Terlambat`/`Izin`/`Sakit`/`Terkunci` via `StatusBadge`). Yang **belum ada**: kolom "Keterangan" izin di tabel board (`piketBoard()` tidak mengembalikan `alasanDetail`).
3. **Scoping kampus piket (usulan #2 bagian keamanan)** — sudah diverifikasi **AMAN**, semua endpoint (`piketBoard`, `tidakTapPulangKemarin`, `findTerkunci`, dan endpoint permits terkait) sudah pakai `kampusId` dari JWT (`requireKampusId(user)`), bukan parameter klien. **Tidak ada celah keamanan** — bagian ini di luar T031, cukup diverifikasi ulang saat testing manual, bukan pekerjaan development.
4. **Import Data 3-jenis (usulan #6)** — backend (`POST /import/{students,teachers,cards}`) dan UI upload+laporan **sudah lengkap**, cuma lokasinya di menu terpisah `/import`, bukan terintegrasi ke masing-masing menu Siswa/Guru seperti diminta.

---

## ✅ T029 — Fix Bug Fallback Foto Kiosk (Bug Nyata, Bukan Fitur Baru) — SELESAI 2026-07-20

**Prioritas: tinggi** — ini kemungkinan penyebab utama keluhan user #1 ("cuma nama").

- [x] `apps/kiosk/src/components/kiosk-avatar.tsx` — saat ini `onError` cuma `e.currentTarget.style.display = "none"`, TIDAK ada fallback avatar generik yang muncul menggantikannya (beda dari kasus `foto === null` yang sudah benar render fallback). Perbaiki: `onError` harus switch ke render fallback yang sama seperti kasus `foto` null (state React, bukan manipulasi DOM langsung).
- [x] Verifikasi ulang dengan foto yang sengaja dibuat gagal load (misal file dihapus manual dari disk tapi kolom `foto` masih terisi) — pastikan fallback avatar muncul, bukan area kosong.
- [x] Assign foto dummy ke siswa/guru test (lewat menu `/foto`) untuk memverifikasi kasus foto BERHASIL load juga tampil benar di kiosk — user kemungkinan belum sempat assign foto sama sekali ke data dummy sebelum menyimpulkan "fitur belum ada".

**Implementasi:** Ganti `onError` di `KioskAvatar` dari manipulasi DOM langsung (`style.display = "none"`) jadi React state (`loadFailed`), dengan `useEffect` reset state tiap `foto`/`token` berganti (supaya tidak "macet" di mode fallback kalau tap siswa lain sesudahnya berhasil load). Diverifikasi lewat Playwright: (1) foto valid di disk → render foto asli dengan benar; (2) file foto sengaja dipindah dari disk (kolom DB tetap terisi, proxy return 404) → fallback avatar generik muncul, bukan area kosong. Root cause keluhan user #1 terkonfirmasi: bug ini yang membuat area foto kosong/hilang total saat fetch foto gagal, BUKAN fitur foto/jam yang belum ada (itu semua sudah benar di `feedback-screen.tsx`).

**Ref:** `apps/kiosk/src/components/kiosk-avatar.tsx`, `apps/kiosk/src/components/feedback-screen.tsx` (sudah benar, tidak perlu diubah).

---

## ✅ T030 — Kolom Keterangan Izin di Dashboard Piket — SELESAI 2026-07-20

- [x] `apps/api/src/attendance/attendance.service.ts` — method `piketBoard()`: tambah field keterangan (`alasanDetail` dari `Permit` terkait tanggal berjalan) ke return type `PiketBoardRow`. Perlu join/lookup ke tabel `permits` untuk siswa dengan `status = izin` atau `status = sakit` pada tanggal hari ini.
- [x] `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — tampilkan kolom "Keterangan" di tabel, isi dari field baru di atas, kosong/`-` kalau bukan status izin/sakit.

**Implementasi:** `piketBoard()` tambah `include: { permits: { where: { tanggal: today }, take: 1 } }` di query siswa (pakai relasi `Student.permits` yang sudah ada di schema, tidak perlu migrasi), map jadi field baru `alasanDetail` di `PiketBoardRow`. Tidak ada `attendanceRecordId` eksplisit yang menghubungkan Permit ke AttendanceRecord — join dilakukan lewat `studentId + tanggal` (kunci yang sama dipakai `createTidakMasuk()` untuk cegah duplikat). Kolom "Keterangan" di tabel cuma tampil isi kalau `status` hari ini `izin`/`sakit`, selain itu `-`. Diverifikasi lewat Playwright: login sebagai `piket_kampus1`, dashboard menampilkan keterangan izin siswa dengan benar dari data permit yang ada.

**Catatan:** ditemukan celah kecil (bukan scope T030, dicatat untuk referensi): `createTidakMasuk()` di `permits.service.ts` seharusnya sudah mencegah 2 permit untuk siswa+tanggal yang sama (guard by AttendanceRecord existing), tapi saat testing ternyata bisa ada 2 permit row untuk 1 siswa di tanggal sama kalau dibuat lewat 2 request terpisah yang keduanya lolos guard race-condition-like sebelum AttendanceRecord pertama ke-commit. Tidak berdampak fungsional untuk T030 (board tetap ambil salah satu lewat `take:1`), tapi kalau nanti butuh akurasi 100% urutan tampil, pertimbangkan `orderBy: { createdAt: "desc" }` atau constraint unique `(studentId, tanggal)` di level DB.

**Ref:** `apps/api/src/attendance/attendance.service.ts` (`piketBoard`, `PiketBoardRow`), `apps/api/prisma/schema.prisma` (model `Permit`), `apps/web/src/lib/core-types.ts` (`PiketBoardRow`).

---

## ✅ T031 — Restrukturisasi Navigasi Dashboard Piket + Badge Notifikasi — SELESAI 2026-07-20

**Menggabungkan usulan #3 dan #4.**

**Interpretasi final (dikonfirmasi user):** sidebar/tab bar tetap **2 menu** (Dashboard, Perizinan Keluar) — TIDAK diganti/ditambah menu baru. Di dalam halaman **Dashboard**, urutan section jadi: 4 section prioritas di ATAS (Belum Kembali, Tidak Tap Pulang Kemarin, Perlu Ditinjau, Siswa Terkunci — posisi yang dulu ditempati board utama), lalu board utama semua siswa jadi section ke-5 ("Board Semua Siswa") DI BAWAH 4 section itu — bukan dipindah ke halaman lain. Badge: **4 badge kecil terpisah** (bukan 1 gabungan) ditampilkan berdampingan di bawah tab "Dashboard", masing-masing untuk 1 section, hanya render kalau count > 0.

- [x] **Endpoint count ringan (baru)** — ditambah `GET /attendance/piket-notification-counts` (bukan modul piket terpisah, taruh di `AttendanceController` karena sudah jadi tempat data piket-board lain) return `{ belumKembali, tidakTapPulang, perluDitinjau, terkunci }`. Dibangun dari 3 method count baru: `PermitsService.countBelumKembali()`/`countPerluDitinjau()` (where-clause identik dengan versi list, ganti `findMany` → `count`), `AttendanceService.countTidakTapPulangKemarin()`, `StudentsService.countTerkunci()`. `AttendanceModule` sekarang import `CoreModule` + `PermitsModule` untuk akses 3 service itu (tidak ada circular import).
- [x] `apps/web/src/app/(piket)/piket-nav.tsx` — tambah 4 badge kecil terpisah (bukan gabungan) di bawah tab bar, dari hook baru `usePiketNotificationCounts()` (`apps/web/src/lib/use-piket-notification-counts.ts`) — polling ringan tiap 15 detik, BUKAN channel Socket.IO baru (tidak signifikan untuk kebutuhan ini).
- [x] Pindahkan 4 section ke bagian atas `piket-board-view.tsx`, board utama jadi section terakhir "Board Semua Siswa" dengan judul + deskripsi baru. Semua 4 section + board utama sekarang pakai `border border-border` eksplisit (sebelumnya cuma `shadow-card` tanpa border).

**Verifikasi:** Playwright — login `piket_kampus1`, screenshot Dashboard menunjukkan urutan section benar (4 section prioritas lalu Board Semua Siswa, semua dengan border terlihat). Dibuat 1 permit test "izin keluar" dengan jam kembali di masa lalu → badge "Belum Kembali ①" muncul benar di bawah tab Dashboard, section itu juga menampilkan baris data yang sesuai. Halaman "Perizinan Keluar" tetap berfungsi normal, tidak terpengaruh perubahan (tidak dapat badge karena bukan salah satu dari 4 section). Test permit dibersihkan setelah verifikasi.

**REVISI 2026-07-21 (setelah user lihat hasil di browser):** interpretasi awal "tab bar horizontal" ternyata salah paham — yang dimaksud user adalah **sidebar collapsible** (persis seperti `(admin)`), bukan tab bar di atas konten. Dan 4 section prioritas dimaksud tampil **berjajar horizontal sebagai kartu ringkas** (bukan list vertikal penuh tabel). Diperbaiki:
- `piket-nav.tsx` (tab bar horizontal lama) **dihapus total**, diganti `piket-sidebar.tsx` (baru) — sidebar collapsible dengan 2 item (Dashboard, Perizinan Keluar), reuse pola persis `components/shell/sidebar.tsx` (state `useSidebarState`, tombol "Ciutkan", localStorage persist).
- `piket-content.tsx` (baru) — wrapper konten yang menyesuaikan margin kiri sesuai state collapsed, reuse pola `(admin)/admin-content.tsx`. Banner read-only ADR-024 dipindah ke sini (sebelumnya di `layout.tsx` langsung).
- `layout.tsx` — restrukturisasi total mengikuti pola `(admin)/layout.tsx`: `PiketSidebar` + `PiketContent` alih-alih header+nav+main linear.
- `piket-board-view.tsx` — 4 section (`BelumKembaliSection`, dst) TIDAK lagi dirender langsung sebagai card penuh; sekarang dibungkus `Dialog` masing-masing, dipicu oleh 4 `SummaryCard` (baru) yang berjajar horizontal dalam grid (`grid-cols-2 sm:grid-cols-4`) — tiap card cuma judul + angka besar + badge merah kalau count > 0. Konten section (tabel + form aksi) tidak berubah sama sekali, cuma wrapper luarnya (`<div className="rounded-xl border...">` → `<DialogHeader>` polos, karena sekarang sudah dalam konteks `DialogContent`).

**Verifikasi ulang:** Playwright — sidebar tampil dengan 2 menu benar, bisa diciutkan jadi ikon saja (state localStorage persist, sama seperti admin), navigasi ke "Perizinan Keluar" via sidebar berfungsi. Klik kartu "Belum Kembali" membuka dialog detail dengan benar.

**REVISI KE-2 2026-07-21 (user minta lagi setelah lihat hasil revisi pertama):** popup/dialog untuk 5 kartu (4 section prioritas + "Board Semua Siswa") diganti jadi **expand di bawah grid**, bukan lagi dialog — pola "accordion" dengan cuma 1 section aktif dalam satu waktu, klik kartu yang sama sekali lagi untuk menutup. Perubahan:
- `piket-board-view.tsx` — `openSection` state ditambah varian `"board"`. Grid jadi 5 kartu (`sm:grid-cols-5`), termasuk kartu baru "Board Semua Siswa" (menampilkan `board.length`). Semua 4 `<Dialog>` untuk section prioritas **dihapus total** — konten section (`BelumKembaliSection`, dll, termasuk tabel "Board Semua Siswa" yang sekarang inline langsung di JSX bukan fungsi terpisah) dirender kondisional di dalam 1 `<div className="rounded-xl border...">` di bawah grid, berdasarkan `openSection` yang aktif.
- 4 fungsi section (`BelumKembaliSection`, `TidakTapPulangSection`, `PerluDitinjauSection`, `TerkunciSection`) — `<DialogHeader><DialogTitle>` diganti `<h2>` polos, karena sekarang dirender langsung (bukan lagi di dalam `DialogContent`). Form-form terpisah (`PermitForm`, `LockForm`, `UnlockForm`, `TidakTapPulangForm`) TETAP pakai `Dialog` asli — itu form input aksi, bukan "section", tidak berubah.
- `SummaryCard` — tambah prop `active`, styling border+background oranye kalau section itu yang sedang terbuka (`border-primary bg-primary-soft`).

**Verifikasi:** Playwright — 5 kartu tampil benar, klik "Board Semua Siswa" → tabel muncul di bawah grid (bukan popup) dengan card aktif ter-highlight. Klik ulang kartu yang sama → section tertutup (toggle berfungsi). Klik kartu lain saat 1 section terbuka → berpindah otomatis ke section baru, cuma 1 aktif dalam satu waktu (accordion, bukan expand-semua).

**Ref:** `apps/web/src/app/(piket)/piket-sidebar.tsx` (baru), `apps/web/src/app/(piket)/piket-content.tsx` (baru), `apps/web/src/app/(piket)/layout.tsx`, `apps/web/src/app/(piket)/piket/piket-board-view.tsx`, `apps/web/src/lib/use-piket-notification-counts.ts`, `apps/api/src/attendance/attendance.controller.ts` (`piketNotificationCounts`), `apps/api/src/permits/permits.service.ts`, `apps/api/src/core/students/students.service.ts`, `apps/api/src/attendance/attendance.module.ts`. File `piket-nav.tsx` dihapus (digantikan `piket-sidebar.tsx`).

---

## ✅ T032 — Jadwal Hari Piket (Fitur Baru, ADR-024) — SELESAI 2026-07-21

**Fitur baru total.** Keputusan implementasi yang dikonfirmasi user sebelum coding:
- **Basis angka hari:** independen, 1=Senin..6=Sabtu (BUKAN basis DAYOFWEEK `Schedule.hari` yang 1=Minggu..7=Sabtu) — cocok dengan `new Date().getDay()` untuk Senin-Sabtu (Minggu/0 sengaja tidak pernah cocok, grid memang cuma 6 kolom).
- **Timezone:** tidak perlu konversi baru — server sistem sudah `Asia/Jakarta` (dicek `date`/`Intl.DateTimeFormat().resolvedOptions().timeZone`), `new Date().getDay()` di Node sudah otomatis ikut timezone lokal proses, konsisten dengan pola `startOfDay()` yang sudah ada di `attendance.service.ts`.
- **Lokasi menu admin:** menu baru terpisah "Jadwal Piket" (`/jadwal-piket`, ikon `CalendarCheck`) di `primaryNav`, bukan sub-tab di Manajemen Akun.

- [x] Migrasi: tabel baru `piket_schedules` (`id`, `hari` Int 1-6, `userId` FK `users`, `createdById`, `createdAt`, `@@unique([hari, userId])` cegah duplikat assignment).
- [x] Backend: `GET/POST/DELETE /piket-schedules` (`PiketSchedulesController`, role `super_admin`), `GET /piket-schedules/me/today` (role `guru_piket`, dipanggil dari `(piket)/layout.tsx`) return `{ bertugasHariIni: boolean }`.
- [x] **Backend guard `PiketOnDutyGuard`** (`apps/api/src/piket-schedules/guards/piket-on-duty.guard.ts`) — cek `isBertugasHariIni()`, early-return `true` untuk role selain `guru_piket` (tidak mengganggu `super_admin`/`card_admin` yang berbagi endpoint sama, mis. `students.controller.ts` lock/unlock). Dipasang di semua endpoint tulis modul piket: `attendance.controller.ts` (`manualPulang`, `confirmIzinPulang`, `konfirmasiPulangRetroaktif`), `permits.controller.ts` (`create`, `confirmKembali`, `setPulang`, `tandaiIzinTidakKembali`), `students.controller.ts` (`lock`, `unlock`). Endpoint GET tetap tanpa guard ini — piket off-duty tetap bisa lihat data (read-only, bukan no-access).
- [x] Frontend admin: `apps/web/src/app/(admin)/jadwal-piket/` — grid 6 kolom Senin-Sabtu (gaya visual meniru `kalender-view.tsx`, komponen baru). Klik hari → dialog pilih guru piket (dari `/users` difilter role `guru_piket` di server component, backend belum punya filter query-param), assign/lepas. Nama ter-assign tampil langsung di kotak hari.
- [x] Frontend piket: `(piket)/layout.tsx` (server component) fetch `bertugasHariIni`, render banner merah read-only kalau `false`, teruskan lewat context baru `piket-duty-context.tsx` (`usePiketOnDuty()`) ke semua client component piket. Semua tombol aksi tulis di `piket-board-view.tsx` dan `izin-keluar-view.tsx` disable (`disabled={!onDuty}`) atau disembunyikan total (tombol Izin/Sakit di board utama) saat read-only — backend guard tetap jadi penegak utama, ini cuma UX supaya tidak ada klik yang gagal membingungkan.

**Bug ditemukan & diperbaiki saat verifikasi (di luar scope T032, tapi terekspos olehnya):** `apps/web/src/components/shell/sidebar.tsx` memakai `pathname.startsWith(item.href)` untuk highlight menu aktif — ini membuat menu "Jadwal" (`/jadwal`) ikut ter-highlight saat berada di `/jadwal-piket` karena keduanya berbagi prefix `/jadwal`. Diperbaiki jadi exact-match atau `startsWith(href + "/")`. Ini bug lama yang laten sampai ada 2 rute berbagi prefix persis seperti ini.

**Verifikasi:** Playwright — admin assign `piket_kampus1` ke hari ini (Selasa) → dashboard piket tampil normal tanpa banner, semua tombol aktif. Admin lepas assignment → login ulang sebagai `piket_kampus1` → banner merah muncul, tombol "Izin"/"Sakit" hilang dari board utama, tombol "Klarifikasi" disabled. Backend enforcement dites langsung via curl (bypass UI): `POST /attendance/manual-pulang` dan `POST /permits` keduanya menolak dengan 403 saat off-duty; `GET /attendance/piket-board` tetap 200 (read tidak diblokir); `PATCH /students/:id` oleh `super_admin` tetap 200 (guard tidak mengganggu role lain). Test data (assignment) dihapus setelah verifikasi.

**Ref:** [[Projek/AbsenSI/11-Decisions|ADR-024]], `apps/api/src/piket-schedules/` (module baru lengkap), `apps/web/src/app/(admin)/jadwal-piket/`, `apps/web/src/app/(piket)/layout.tsx`, `apps/web/src/app/(piket)/piket-duty-context.tsx` (baru), `apps/web/src/components/shell/sidebar.tsx` (fix prefix-match).

---

## ✅ T033 — Import Data Terintegrasi ke Menu Siswa & Guru — SELESAI 2026-07-21

- [x] `apps/web/src/app/(admin)/siswa/siswa-view.tsx` — tombol "Import Data" di sebelah "Tambah Siswa", klik → dialog berisi keterangan format kolom CSV + file picker + tombol upload, panggil `POST /import/students` lewat `/api/proxy-upload/import/students` (pola lama, reuse).
- [x] `apps/web/src/app/(admin)/guru/guru-view.tsx` — tombol serupa untuk `POST /import/teachers`.
- [x] Kartu: tombol "Import Data" ditambah di `/kartu` (bukan siswa/guru), untuk `POST /import/cards` (butuh nisn_nip + uid, 2 sumber sekaligus — tetap masuk akal di sini karena baris CSV-nya bukan tentang 1 entitas siswa/guru tunggal).
- [x] Menu `/import` lama: dihapus dari sidebar (`nav-items.ts`), route/halaman `/import` dibiarkan tetap ada sebagai fallback (bisa diakses langsung by URL, keputusan user).

**Implementasi:** Logic upload+laporan dari `import-view.tsx` lama diekstrak jadi 1 komponen reusable `apps/web/src/components/import-dialog.tsx` (`ImportDialog`) — menerima `endpoint`/`columns`/`example`/`onImported` sebagai props, dipakai identik di 3 tempat (siswa/guru/kartu) tanpa duplikasi logic. Callback `onImported` memanggil `router.refresh()` supaya tabel ter-update tanpa reload manual setelah import sukses.

**Bug ditemukan & diperbaiki saat verifikasi (bukan dari T033, tapi baru kelihatan sekarang karena 2 route baru — `/jadwal-piket` dan tombol dialog — muncul bersamaan):** tidak ada bug baru kali ini — sidebar prefix-match sudah diperbaiki di T032 dan tetap benar di sini (menu "Jadwal" dan item lain tidak saling tertimpa highlight).

**Verifikasi:** Playwright — tombol "Import Data" muncul di ketiga menu (Siswa/Guru/Kartu), dialog terbuka dengan instruksi kolom yang benar per jenis data. Upload CSV baris tidak lengkap → laporan gagal dengan alasan validasi yang jelas (perilaku benar, bukan bug). Upload CSV valid → siswa baru masuk ke DB (dikonfirmasi lewat Prisma) dan muncul di tabel setelah `router.refresh()` (butuh sedikit waktu, dikonfirmasi via fresh page load). Test data (1 siswa dummy) dihapus setelah verifikasi.

**Ref:** `apps/web/src/components/import-dialog.tsx` (baru, reusable), `apps/web/src/app/(admin)/siswa/siswa-view.tsx`, `apps/web/src/app/(admin)/guru/guru-view.tsx`, `apps/web/src/app/(admin)/kartu/kartu-view.tsx`, `apps/web/src/components/shell/nav-items.ts` (item "Import Data" lama dihapus), `apps/api/src/import/import.controller.ts` (tidak diubah).

---

## ✅ T034 — Activity Log: Catat Login + Halaman Admin "Log" — SELESAI 2026-07-21

**Keputusan dikonfirmasi user sebelum coding:** catat login gagal juga (nilai forensik brute-force lebih besar dari risiko volume). Cakupan log tambahan: kiosk (create/deactivate/rotate-token) dan foto (upload/delete) — piket-schedule TIDAK ditambahkan (di luar pilihan user, meski ternyata `piket_schedule.assign/unassign` sudah ter-log otomatis dari T032 karena sudah pakai `@LogActivity` sejak awal).

- [x] `apps/api/src/auth/auth.service.ts` — `login()` sekarang terima parameter `ipAddress`, catat `auth.login` (sukses) dan `auth.login_failed` (password salah, actorId = user yang ditemukan). Username tidak ditemukan sama sekali TIDAK dicatat (tidak ada actor valid untuk dirujuk).
- [x] Halaman baru `apps/web/src/app/(admin)/log/` — tabel dengan filter (dari/sampai tanggal, actor ID, action, target type) + pagination. `ActivityLogService.findAll()` diubah total: sebelumnya `findMany()` tanpa batas, sekarang terima `ListActivityLogDto` (filter + page/pageSize) dan return `{ items, total, page, pageSize }`.
- [x] Log tambahan: `kiosk.create`/`kiosk.deactivate`/`kiosk.rotate_token` (lewat `@LogActivity` decorator, `sensitiveFields: ["deviceToken"]` supaya token tidak pernah masuk snapshot). `photo.upload`/`photo.delete` (manual `ActivityLogService.record()` di `PhotosService` karena `uploadBulk`/`assign` bisa target siswa ATAU guru per-file, tidak cocok pola decorator standar 1-target-per-request).

**Implementasi:** Menu baru "Log Aktivitas" di `secondaryNav` (dekat Manajemen Akun), akses dibatasi backend (`@Roles(UserRole.super_admin)` di `ActivityLogController`, sudah ada sebelumnya) — tidak ada gating tambahan di frontend, konsisten dengan pola existing (role lain yang nyasar ke `/log` kena 401 dari `apiFetch`, ditangkap `error.tsx`).

**Verifikasi:** curl langsung ke tiap jenis aksi (login sukses/gagal, kiosk create/deactivate, foto upload/delete) — semua tercatat benar dengan `actorId`/`targetType`/`targetId` yang sesuai, `deviceToken` kiosk terbukti ter-redact dari snapshot. Playwright: halaman `/log` menampilkan 67 entri historis (termasuk aksi dari task-task sebelumnya di sesi ini — konfirmasi decorator `@LogActivity` sudah berjalan sejak awal untuk lock/unlock/permit/card/user/holiday/piket_schedule), filter dan pagination tampil benar. Unit test `auth.service.spec.ts` diupdate untuk signature `login()` baru (tambah `ActivityLogService` mock + parameter `ipAddress` di semua call site).

**Ref:** `apps/api/src/activity-log/` (`findAll()` diubah, `dto/list-activity-log.dto.ts` baru), `apps/api/src/auth/auth.service.ts`, `apps/api/src/kiosks/kiosks.controller.ts`, `apps/api/src/photos/photos.service.ts`, `apps/web/src/app/(admin)/log/` (baru), `apps/web/src/components/shell/nav-items.ts`.

---

## 🟢 T035 — Export Rekap Kehadiran ke PDF

**Belum ada library PDF terpasang di monorepo ini sama sekali** — perlu pilih & pasang library baru.

- [ ] **Diskusi format PDF** (ditunda user, wajib diselesaikan sebelum mulai coding, JANGAN asumsikan format) — kandidat: tabel rekap per siswa (nama, kelas, hadir/telat/izin/sakit/alfa) untuk rentang tanggal terpilih, mirip laporan bulanan resmi. Konfirmasi ulang dengan user di awal sesi task ini.
- [ ] **Pilih pendekatan generate PDF** — opsi: (a) render HTML lewat headless browser (Puppeteer/Playwright) di `apps/api`, konsisten dengan pola thermal-print yang sudah pernah dipakai (`print/struk-izin/route.ts` di `apps/web`, walau itu print browser bukan PDF generate — beda use-case tapi bisa jadi referensi styling), (b) library PDF generation murni (`pdfkit`, `@react-pdf/renderer`, dst, tanpa browser headless — lebih ringan tapi styling lebih terbatas). **Putuskan saat implementasi berdasarkan kompleksitas format yang disepakati.**
- [ ] Backend: endpoint baru `GET /attendance/report/pdf` (atau serupa) dengan query params sama seperti `report()` yang sudah ada, return `application/pdf` binary.
- [ ] Frontend: `apps/web/src/app/(admin)/rekap/rekap-view.tsx` — tombol "Download PDF" yang memicu download file (pola: `<a download>` ke URL proxy, atau fetch blob lalu `URL.createObjectURL` — pola serupa CSV download di `import-view.tsx` bisa jadi referensi, meski itu client-side generate bukan server-side).

**Ref:** `apps/api/src/attendance/attendance-report.service.ts`, `apps/web/src/app/(admin)/rekap/rekap-view.tsx`, `apps/web/src/app/(admin)/import/import-view.tsx` (pola download client-side sebagai referensi tidak langsung).

---

## ✅ T036 — Detail Siswa: Edit Foto Langsung + Riwayat Catatan — SELESAI 2026-07-21

**Keputusan dikonfirmasi user sebelum coding:** endpoint upload-by-id baru (bukan reuse upload-bulk) — lebih eksplisit, tidak bergantung penamaan file NISN. Riwayat catatan tanpa batas tanggal (semua riwayat, bukan cuma tahun ajaran aktif).

- [x] **Edit foto langsung** — endpoint baru `POST /photos/students/:id` dan `POST /photos/teachers/:id` (`PhotosService.uploadOne()`) upload 1 file langsung by ID, terpisah dari `uploadBulk()`/`assign()` yang match-by-filename. Di `siswa-detail-view.tsx`: overlay hover pada avatar fallback (foto null) sekarang punya tombol "Upload Foto" yang buka file picker langsung, hasil upload update state tanpa reload.
- [x] **Riwayat Catatan** — section baru di antara data diri dan "Riwayat Kartu", method baru `AttendanceReportService.riwayatCatatan(studentId)`, endpoint `GET /attendance/students/:id/riwayat-catatan`:
  - **Terlambat**: tanggal + jam tap. Tidak ada kolom petugas (tap otomatis kiosk) — kolom "Petugas" tampil "-" untuk baris ini.
  - **Izin/sakit**: tanggal, `alasanDetail`, nama petugas dari `Permit.approvedBy.username`.
  - **Alfa**: dihitung live (reuse `resolveHariWajib()` yang sudah ada di `AttendanceReportService`, method itu di-publicize dari `private` supaya bisa dipanggil lintas method dalam service yang sama).

**Keterbatasan diketahui (didokumentasikan di kode, bukan bug):** `resolveHariWajib()` cuma hitung hari wajib dari `academic_years` yang `isActive: true` — kalau siswa punya riwayat dari tahun ajaran yang SUDAH TIDAK aktif lagi, alfa dari periode itu tidak terhitung. Sengaja tidak diubah supaya tidak diam-diam mengubah perilaku `report()` Rekap yang sudah ada dan dipakai method yang sama.

**Verifikasi:** curl endpoint riwayat-catatan langsung — data izin+alfa historis tampil benar dengan urutan tanggal terbaru dulu. Playwright: upload foto dari halaman detail (overlay pada avatar fallback) langsung tampil tanpa reload; hapus foto (fitur lama) tetap berfungsi normal berdampingan dengan fitur upload baru — tidak ada regresi. Test foto dan state dibersihkan (foto kembali null di DB+disk).

**Ref:** `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx`, `apps/api/src/photos/photos.service.ts` (`uploadOne`, baru), `apps/api/src/photos/photos.controller.ts`, `apps/api/src/attendance/attendance-report.service.ts` (`riwayatCatatan`, `resolveHariWajib` di-publicize), `apps/api/src/attendance/attendance.controller.ts`, `apps/web/src/lib/core-types.ts` (`RiwayatCatatanEntry`).

---

## ✅ T037 — Lock Otomatis Setelah 2x Terlambat (ADR-025) — SELESAI 2026-07-21

**Dikerjakan terakhir sesuai rencana** (setelah T029-T036 stabil). Keputusan dikonfirmasi user sebelum coding: payload tetap `result: accepted` + flag `justLocked: true` (bukan varian enum baru), `lateStrikeResetAt` null = hitung dari seluruh riwayat (bukan dibatasi tahun ajaran).

- [x] Migrasi: `Student.lateStrikeResetAt DateTime?` (default null).
- [x] `AttendanceService.tap()` — method baru `applyLateStrikeLock(studentId, effectiveTime)`, dipanggil HANYA saat `status === terlambat` DAN `card.studentId` ada (guru tidak pernah kena — dicek eksplisit). Hitung count `attendance_records` berstatus terlambat sejak `lateStrikeResetAt` (atau semua riwayat kalau null); genap 2 → update `Student` (`lockedAt`, `lockedReason: "Terlambat 2 kali — hubungi orang tua"`, `lockedById: null`). `lockedById` sudah nullable dari awal di schema, tidak perlu migrasi tambahan untuk itu — lock sistem ini TIDAK lewat `StudentsService.lock()` (method itu untuk piket, butuh `kampusId` scoping + `lockedById` non-null yang tidak cocok untuk lock otomatis), ditulis langsung sebagai Prisma update di dalam service yang sama.
- [x] `TapResultPayload`/`TapResponse` (`@absensi/types`) — field baru `justLocked?: boolean`, message otomatis di-override jadi "Sudah terlambat 2 kali, silahkan hadirkan orang tua" saat true.
- [x] `apps/kiosk/src/components/feedback-screen.tsx` — varian baru warna merah gelap (`#7A1F1F`, beda dari merah terlambat `#E13B3B` dan merah rejected `bg-danger-text`), ikon `Lock`, judul "Kartu Terkunci".
- [x] `StudentsService.unlock()` — deteksi lock-otomatis via `lockedById === null` (manual lock SELALU py piket dengan id non-null; ini satu-satunya kasus null), set `lateStrikeResetAt = now()` di baris update yang sama saat itu terjadi.
- [x] Dashboard Piket: siswa terkunci otomatis muncul normal di "Siswa Terkunci" existing, `lockedReason` "Terlambat 2 kali — hubungi orang tua" tampil jelas beda dari alasan manual.

**BUG DITEMUKAN & DIPERBAIKI saat verifikasi ketat** (bukan cuma verifikasi kosmetik — proses ini benar-benar menangkap kesalahan nyata): perbandingan awal `tanggal: { gt: student.lateStrikeResetAt }` salah kategori — `tanggal` tersimpan sebagai tanggal murni (00:00 UTC) sedangkan `lateStrikeResetAt` adalah timestamp lengkap (jam unlock persis). Akibatnya, tap terlambat pada HARI YANG SAMA dengan unlock (tapi lebih pagi dari jam unlock persis) salah dihitung sebagai "sebelum reset" dan tidak ikut terhitung strike baru — bug ini akan membuat siswa butuh 3 keterlambatan (bukan 2) untuk terkunci lagi pada hari unlock-nya. Diperbaiki: bandingkan terhadap `startOfDay(lateStrikeResetAt)` dengan `gte`, bukan timestamp presisinya dengan `gt`.

**Verifikasi end-to-end (semua lolos, dijalankan ketat sesuai kesepakatan sebelum ditandai selesai):**
1. Tap siswa lain (tidak terlibat) — tidak terpengaruh sama sekali.
2. Siswa dengan 1x terlambat — tidak terkunci, bisa tap lagi normal.
3. Siswa dengan 2x terlambat (tap ke-2 real, strike ke-1 di-seed) — terkunci otomatis, payload `justLocked: true` + pesan yang benar.
4. Tap saat sudah terkunci — ditolak `rejected_locked` seperti lock manual.
5. Unlock oleh piket (lewat guard `PiketOnDutyGuard` dari T032 — piket harus bertugas hari itu) — `lateStrikeResetAt` terisi benar.
6. Cycle ke-2 setelah unlock: strike 1 baru tidak langsung lock (benar) — **ini yang awalnya menangkap bug di atas**, setelah fix: strike 2 di cycle baru berhasil lock lagi, membuktikan reset+fix bekerja.
7. Tap GURU dengan riwayat terlambat serupa — sama sekali tidak terpengaruh (Teacher model memang tidak punya field lock).
8. Verifikasi visual: kiosk (varian tampilan khusus, warna+ikon+pesan benar) dan dashboard piket (section Siswa Terkunci menampilkan data dengan benar).

Semua kiosk test (id 13, 14) dan data test (attendance_records, tap_events, piket_schedule sementara) dibersihkan setelah verifikasi selesai.

**Ref:** [[Projek/AbsenSI/11-Decisions|ADR-025]], `apps/api/src/attendance/attendance.service.ts` (`tap()`, `applyLateStrikeLock()`), `apps/api/src/core/students/students.service.ts` (`unlock()`), `apps/kiosk/src/components/feedback-screen.tsx`, `packages/types/src/index.ts` (`TapResponse.justLocked`).

---

## Catatan Umum

- Baca ulang bagian kode terkait sebelum tiap task — deskripsi di sini berdasarkan riset kode per 2026-07-20, tapi mungkin sudah ada perubahan kecil dari task-task sebelumnya di batch ini.
- T037 (lock otomatis) HARUS dikerjakan setelah task lain selesai — ini menyentuh fungsi paling sentral (`tap()`) yang dipakai semua alur tap siswa/guru, risiko regresi tertinggi kalau dikerjakan sambil ada perubahan lain yang belum stabil.
- Server AbsenSI berjalan di timezone `Asia/Jakarta` (dikonfirmasi saat T032) — `new Date()` di Node otomatis ikut timezone lokal proses tanpa konversi manual. Semua task berikutnya yang butuh "hari ini"/timezone (mis. T036 riwayat catatan, T037 lock otomatis) cukup ikuti pola `startOfDay()` yang sudah ada di `attendance.service.ts`, jangan tambah konversi timezone baru.
- T035 (PDF) — format belum diputuskan, WAJIB didiskusikan dulu di awal task ini sebelum menulis kode apapun.
- Setiap task selesai: build (`./scripts/build.sh`) + test (`./scripts/test.sh`) hijau, verifikasi Playwright untuk perubahan UI, restart dev server lewat `./scripts/dev-start.sh` setelah build (hindari `.next` cache corruption), cleanup data test, update checklist & catatan implementasi di file ini.

# T226 — API+Web: Wali Kelas — Dashboard Monitoring Kehadiran Harian (Real-Time)

## Depends on
**WAJIB SETELAH T224a-T224d selesai** (halaman Wali Kelas dengan tab Daftar Siswa/Rekap Detail sudah ada — task ini menambah 1 elemen lagi ke halaman yang sama). Independen dari rangkaian jadwal T203-T215.

## Konteks — Permintaan User (2026-08-20)

Selain rekap historis (T224c/d), wali kelas butuh cara CEPAT memonitor kehadiran kelasnya **hari ini** — tanpa perlu buka tab Rekap dan set filter tanggal tiap kali. Ini elemen BARU, murni live/snapshot-harian, terpisah dari tab-tab historis yang sudah ada.

## Pola Existing yang Direplikasi (dikonfirmasi via riset 2026-08-20 — JANGAN desain ulang dari nol)

Proyek ini SUDAH punya 3 pola matang yang persis cocok untuk kebutuhan ini, scope-nya kampus/global — task ini mereplikasi pola yang SAMA, hanya scope-nya diperkecil ke kelas wali:

1. **Kategori status live per siswa** — `AttendanceService.resolveKategoriLive()` (`apps/api/src/attendance/attendance.service.ts`, private method dalam `piketBoard()`) — logic penentuan hadir/terlambat/izin/sakit/dispen/alfa/belum_hadir sudah ADA dan teruji, HANYA perlu dipanggil dengan filter tambahan `kelasId` (saat ini scope-nya kampus, semua kelas).
2. **Kartu ringkasan klik-untuk-expand** — `SummaryCard` di `apps/web/src/app/(piket)/piket/piket-board-view.tsx:563-592` — pola visual: angka besar (`text-display`), warna merah otomatis kalau count > 0 (`text-danger-text`), klik untuk expand daftar nama di bawahnya (bukan modal).
3. **Progress bar proporsional** — `apps/web/src/app/tv-piket/[kampusId]/components/bar-persentase.tsx` — 3 segmen `flexGrow = persen`, warna token `success`/`status-shipped`(amber)/`danger`.
4. **Update real-time** — Socket.IO room `attendance:kampus:${kampusId}` (`attendance.gateway.ts`) sudah broadcast event `attendance:kampus:update` tiap ada tap/perubahan status — task ini SUBSCRIBE ke room yang SAMA (bukan bikin room baru), filter di CLIENT hanya siswa yang `kelasId` cocok kelas wali (KONSISTEN cara `piket-board-view.tsx` konsumsi event yang sama untuk kampus, hanya beda level filter).
5. **Badge lonceng notifikasi role-gated** — `PiketNotificationBell` (`apps/web/src/components/shell/top-bar.tsx:140-206`) + context provider `PiketNotificationsProvider` yang `null` di luar route group `(piket)` (`piket-notifications-context.tsx`) — pola "icon hanya muncul untuk role tertentu tanpa prop manual" SUDAH ADA, direplikasi untuk wali kelas.

## Keputusan Dikonfirmasi User (2026-08-20)

1. **Update real-time via Socket.IO** — konsisten pola Papan Piket, BUKAN fetch-sekali+refresh-manual.
2. **Badge notifikasi di icon top bar TERPISAH**, khusus wali kelas (bukan cuma badge angka di sidebar) — replikasi pola `PiketNotificationBell`, muncul HANYA untuk user dengan `kelasIdWali` terisi.

## Spec Detail

### 1. Backend — endpoint snapshot status harian (scoped ke kelas wali)

- `GET /journal/kelas-wali-status-hari-ini` (KONSISTEN pola `journal-kelas-wali.controller.ts` — `kelasId` selalu dari JWT).
- Method BARU di `AttendanceService` (atau extract logic `resolveKategoriLive()`+query terkait jadi lebih reusable kalau saat ini terlalu menyatu dengan `piketBoard()` — VERIFIKASI SAAT IMPLEMENTASI seberapa mudah reuse tanpa duplikasi, REKOMENDASI: extract query+kategorisasi jadi method `resolveStatusHarianSiswa(where: Prisma.StudentWhereInput)` yang generik terima where clause, dipanggil `piketBoard()` dengan `{kelas: {kampusId}}` dan endpoint baru ini dengan `{kelasId}` — SATU sumber logic, DUA scope pemanggil).
- Response: daftar siswa dengan `kategoriLive` (sama enum dengan Papan Piket), plus agregat count per kategori untuk kartu ringkasan.
- **Siswa nonaktif** — TIDAK disertakan (konsisten fix T220, ini snapshot kehadiran real hari ini, bukan daftar histori seperti T224a).

### 2. Frontend — kartu ringkasan + progress bar, di tab/section baru "Hari Ini"

- Tab/section baru di halaman Wali Kelas (`(guru)/guru/wali-kelas/`), REPLIKASI visual `SummaryCard` (Papan Piket) untuk kategori: Hadir, Terlambat, Izin+Sakit+Dispen, Alfa, Belum Tap — angka merah otomatis kalau Alfa/Belum Tap > 0.
- Progress bar proporsional (REPLIKASI `bar-persentase.tsx`) di atas kartu — ringkasan visual cepat "seberapa penuh kelas hari ini".
- Klik kartu → expand daftar nama siswa kategori itu (KONSISTEN pola expand-inline Papan Piket, BUKAN modal/dialog terpisah).

### 3. Frontend — koneksi Socket.IO real-time

- Subscribe ke room `attendance:kampus:${kampusId}` YANG SUDAH ADA (kampus milik kelas wali — resolve `kampusId` dari `kelasIdWali` → `Kelas.kampusId`) — REUSE hook `useAttendanceSocket()` existing kalau strukturnya cukup generik, atau bikin varian tipis yang filter event HANYA untuk siswa `kelasId` cocok kelas wali (KONSISTEN pola `piket-board-view.tsx` konsumsi event yang sama).
- Event `attendance:kampus:update` masuk → patch state lokal siswa terkait (BUKAN re-fetch seluruh endpoint tiap event, KONSISTEN optimasi existing Papan Piket).

### 4. Backend+Web — Badge lonceng notifikasi wali kelas

- Context provider baru `WaliKelasNotificationsContext` (REPLIKASI STRUKTUR `piket-notifications-context.tsx` — join Socket.IO room yang sama, gabung event real-time + polling fallback, reset tengah malam otomatis) — return `null` di luar kondisi `user.kelasIdWali` terisi (KONSISTEN pola gerbang visibility existing, BUKAN prop role manual).
- Isi notifikasi: jumlah siswa kategori "alfa" ATAU "belum_hadir" (setelah lewat jam masuk) hari ini di kelas wali — REUSE endpoint/logic poin 1, filter kategori yang "perlu perhatian" saja (bukan hadir/terlambat yang normal).
- Render: `WaliKelasNotificationBell` baru di `top-bar.tsx`, REPLIKASI styling badge (`animate-alert-pulse`, angka merah, "9+" kalau lebih dari 9) — DIPASANG TERPISAH dari `PiketNotificationBell` (2 lonceng berbeda bisa tampil bersamaan kalau user kebetulan py akses ganda role, meski jarang terjadi — TIDAK PERLU digabung jadi 1 komponen).

## Edge Cases

- **Wali kelas belum ada siswa tap sama sekali pagi ini** (masih pagi, jam masuk belum lewat) — kategori "Belum Tap" wajar tinggi, JANGAN salah kategorikan sebagai "Alfa" sebelum `deadline` (jam masuk) terlewati — REUSE logic `resolveKategoriLive()` yang SUDAH benar menangani ini.
- **Hari libur/bukan hari wajib** — tampilkan state khusus ("Hari ini bukan hari sekolah" / sejenis), BUKAN kartu kosong tanpa penjelasan — REUSE `resolveHariWajib()` yang sudah jadi sumber kebenaran di banyak tempat lain.
- **Socket.IO terputus** (koneksi jaringan wali kelas tidak stabil) — fallback polling KONSISTEN pola `piket-notifications-context.tsx` (polling 90 detik sebagai jaring pengaman), jangan biarkan data "membeku" tanpa indikasi kalau koneksi live terputus.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (extract/reuse `resolveKategoriLive()`), `apps/api/src/journal/journal-kelas-wali.controller.ts` (endpoint baru), `apps/web/src/app/(guru)/guru/wali-kelas/` (tab/section baru "Hari Ini"), `apps/web/src/components/shell/top-bar.tsx` (lonceng baru).
- **Buat:** `apps/web/src/app/(guru)/wali-kelas-notifications-context.tsx` (REPLIKASI `piket-notifications-context.tsx`).
- **Jangan sentuh:** `piketBoard()`/`PiketNotificationBell` existing (tetap seperti sekarang, task ini paralel bukan pengganti), Socket.IO gateway (`attendance.gateway.ts`) — REUSE room yang sudah ada, TIDAK bikin room baru.

## Acceptance Criteria
- [x] Dashboard "Hari Ini" tampil kartu ringkasan (Hadir/Terlambat/Izin+Sakit+Dispen/Alfa/Belum Tap) + progress bar, scoped kelas wali.
- [x] Update real-time — siswa tap kartu, status di dashboard berubah otomatis TANPA reload halaman.
- [x] Klik kartu kategori → expand daftar nama siswa kategori itu.
- [x] Icon lonceng notifikasi baru di top bar — HANYA muncul untuk user `kelasIdWali` terisi, badge jumlah "perlu perhatian" (alfa saja — lihat deviasi di bawah).
- [x] Siswa nonaktif TIDAK muncul di dashboard hari ini (konsisten T220).
- [x] Hari libur/bukan hari wajib — state khusus jelas, bukan kartu kosong membingungkan.
- [x] Build + type-check hijau, jest baru untuk endpoint snapshot (kategori benar per skenario, siswa nonaktif exclude, hari libur).

## Validasi Claudian
- [x] Konfirmasi `resolveKategoriLive()` di-REUSE (via extract method generik atau reuse langsung), BUKAN diduplikasi/ditulis ulang logic kategorisasi yang sama.
- [x] Konfirmasi Socket.IO REUSE room `attendance:kampus:${kampusId}` yang sudah ada, TIDAK bikin room/gateway baru untuk kebutuhan yang sama.
- [x] Konfirmasi lonceng wali kelas TERPISAH dari lonceng piket (2 komponen independen), BUKAN digabung jadi 1 komponen dengan banyak percabangan role.

## Implementasi (Selesai 2026-08-20)

**Backend:**
- `AttendanceService.piketBoard(kampusId)` di-refactor: sekarang delegasi penuh ke method privat baru `resolveStatusHarianSiswa(where: Prisma.StudentWhereInput)` — generik terima where clause Prisma, SATU sumber kebenaran query+kategorisasi `kategoriLive` untuk 2 caller: `piketBoard()` (scope kampus, `where: {kelas:{kampusId}, status:"aktif", pklRecords:{none:{endedAt:null}}}`) dan method baru `statusHarianKelas(kelasId)` (scope kelas, where sama tapi `kelasId` langsung). Sesuai rekomendasi spec — tidak ada duplikasi logic kategorisasi.
- `AttendanceService` ditambahkan ke `exports` di `attendance.module.ts` supaya bisa di-inject ke `JournalKelasWaliController`.
- Endpoint baru `GET /journal/kelas-wali-status-hari-ini` di `journal-kelas-wali.controller.ts` — `kelasId` selalu dari `user.kelasIdWali` (JWT, KONSISTEN pola existing). Response: `{kampusId, hariIniWajib, siswa: PiketBoardRow[]}`.
- **Gap ditemukan saat implementasi**: JWT wali kelas TIDAK punya `kampusId` milik kelasnya (field `user.kampusId` di JWT itu punya arti lain — kampus assignment piket, tidak terkait). Fix: endpoint resolve `Kelas.kampusId` via `prisma.kelas.findUnique({where:{id:kelasId}, select:{kampusId:true}})`, dikembalikan di response supaya FE bisa join room socket yang benar.
- Jest baru: `attendance.service.spec.ts` (where-clause kampus vs kelas, reuse kategoriLive) + `journal-kelas-wali.controller.spec.ts` (`statusHariIni` — happy path, hari libur, kelas not found 403, guru bukan wali 403). Semua lulus, 0 regresi ke `piketBoard()` existing (43/43 test lama tetap hijau).

**Frontend:**
- Tab baru "Hari Ini" (`hari-ini-tab.tsx`) di halaman Wali Kelas, jadi tab default (paling kiri). REPLIKASI visual (bukan import) `SummaryCard` (Papan Piket) + `BarPersentase` (TV Piket) sebagai komponen lokal — keduanya kecil dan coupled ke modul asalnya masing-masing.
- REUSE `useAttendanceSocket()` apa adanya (hook sudah generik), subscribe room `attendance:kampus:${kampusId}` yang sudah ada, patch state lokal per event (bukan re-fetch).
- Context provider baru `wali-kelas-notifications-context.tsx`, REPLIKASI struktur `piket-notifications-context.tsx` — 1 koneksi socket per Provider, event real-time + polling fallback 90 detik, tanpa timer reset tengah malam terpisah (endpoint snapshot sendiri sudah scoped "hari ini" server-side, jadi re-fetch = otomatis benar).
- Provider dipasang di `(guru)/layout.tsx`, mount HANYA kalau `user.kelasIdWali` terisi.
- **Lonceng dipasang di 2 tempat** — codebase ini punya 2 file top-bar terpisah untuk breakpoint berbeda: `components/shell/top-bar.tsx` (desktop ≥768px, shared dengan role lain) dan `(guru)/components/top-bar.tsx` (mobile <768px, khusus route group guru). Dikonfirmasi user (AskUserQuestion, proyek ini mobile-first) — lonceng WAJIB di keduanya, bukan cuma desktop. Kedua `WaliKelasNotificationBell` identik secara visual/behavior, ditulis terpisah per file (bukan diekstrak jadi 1 shared component — masing-masing top-bar sudah punya struktur/dependency sendiri, replikasi kecil lebih murah daripada abstraksi baru).

**Deviasi dari spec:**
- Spec poin 4 menyebut "alfa ATAU belum_hadir (setelah lewat jam masuk)" untuk badge notifikasi. Diimplementasikan HANYA "alfa" — karena `resolveKategoriLive()` SUDAH otomatis mengubah `belum_hadir` → `alfa` begitu deadline (jam masuk) terlewati (lihat logic existing di method itu). Jadi "belum_hadir setelah lewat jam masuk" secara logic TIDAK PERNAH terjadi sebagai state terpisah — sudah tercakup dalam kategori "alfa". Filter tunggal `kategoriLive === "alfa"` sudah benar dan tidak kehilangan skenario apapun.

**Verifikasi:** `tsc --noEmit` (web) bersih, `nest build` bersih, `next build` sukses (route `/guru/wali-kelas` 7.13 kB), dev server web restart bersih (curl `/login` 200), full jest suite backend hijau (lihat hasil run terakhir).

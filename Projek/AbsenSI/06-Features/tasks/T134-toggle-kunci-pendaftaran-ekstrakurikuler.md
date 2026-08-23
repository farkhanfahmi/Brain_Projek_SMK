# T134 — Schema+API+Web: Toggle Kunci Pendaftaran Ekstrakurikuler (Cegah Pindah Ekstra via Publik)

## Depends on
Tidak ada dependency teknis. Model config baru mengikuti pola singleton yang sudah ada (`ScheduleConfig`/`AttendanceLockConfig`), tidak menyentuh model lain kecuali `EkstraPublikService.submit()`.

## Objective
Super admin bisa mengunci pendaftaran ekstrakurikuler via toggle — begitu AKTIF, siswa yang **sudah pernah** terdaftar di ekstra manapun **tidak bisa lagi mengganti/pindah** lewat halaman publik (`/daftar-ekstra`). Siswa yang **belum pernah** terdaftar di ekstra manapun **tetap bisa** mendaftar pertama kali seperti biasa, terlepas status toggle.

## Context
- **App:** `apps/api` (model config baru + validasi di `submit()`) + `apps/web` (halaman toggle admin)
- **Riset 2026-08-07 (Explore agent, baca kode langsung)** — temuan penting yang membentuk desain task ini:
  - `EkstraPendaftaran.studentId` (`schema.prisma:939-940`) **`@unique`** — 1 siswa cuma boleh punya 1 baris pendaftaran SEUMUR HIDUP di seluruh sistem (bukan per-ekstra). Ini constraint level DATABASE, bukan cuma aplikasi.
  - `EkstraPublikService.submit()` (`apps/api/src/ekstra-publik/ekstra-publik.service.ts:71-101`) — endpoint publik TANPA GUARD (`POST /ekstra-publik/pendaftaran`, siapa saja bisa akses tanpa login) memakai **`upsert`** keyed `studentId` — submit ulang dari siswa yang SAMA otomatis **menimpa** `ekstrakurikulerId` lama tanpa penghalang apa pun. Ini **BUKAN bug**, ada komentar eksplisit di kode (baris ±66-70) menjelaskan ini SENGAJA (mencegah duplikat baris/spam) — TAPI efek sampingnya adalah pindah-ekstra tanpa batas selalu diizinkan, yang sekarang ingin dikunci lewat toggle.
  - `pindahkanEkstra()` (`ekstra-publik.service.ts:186-209`) — jalur ADMIN untuk memindahkan siswa secara manual, DIPAKAI dari halaman admin `ekstra-monitoring` (dengan guard `@Roles(super_admin, card_admin)`) — pola `upsert` yang SAMA, TAPI ini BUKAN jalur publik.
  - **Tidak ada model config ekstrakurikuler sama sekali** — perlu dibuat baru, REUSE pola persis `ScheduleConfig`/`AttendanceLockConfig` (`schema.prisma:320-345`): singleton, **"enforce di service layer, BUKAN constraint DB"** (sesuai komentar existing di kedua model itu), field `updatedById`/`updatedAt` untuk audit siapa terakhir mengubah.
  - Halaman admin yang sudah ada: `(admin)/ekstra-kurikuler/` (CRUD master data ekstrakurikuler, guard `super_admin`+`card_admin`) — **lokasi paling natural** untuk toggle GLOBAL ini (beda dari `ekstra-monitoring` yang scope-nya per-siswa).

## Keputusan Final (dikonfirmasi user 2026-08-07)

1. **Cakupan**: GLOBAL — 1 toggle untuk SEMUA ekstrakurikuler sekaligus, BUKAN per-ekstrakurikuler.
2. **Yang dikunci**: HANYA jalur PUBLIK/self-service (`POST /ekstra-publik/pendaftaran`, dipakai halaman `/daftar-ekstra`). Siswa yang **sudah** punya `EkstraPendaftaran` (row `studentId` sudah ada) **DITOLAK** kalau coba submit ulang dengan `ekstrakurikulerId` BERBEDA saat toggle aktif. Siswa yang **belum pernah** terdaftar (row belum ada) **TETAP BISA** daftar pertama kali, tanpa terpengaruh status toggle.
3. **Jalur admin TIDAK terkunci** — `pindahkanEkstra()` (dipanggil dari halaman admin `ekstra-monitoring`) TETAP berfungsi normal meski toggle aktif. Toggle ini murni membatasi self-service siswa, BUKAN wewenang admin.
4. **Submit ulang dengan `ekstrakurikulerId` yang SAMA** (siswa klik submit lagi tanpa ganti pilihan) — TIDAK dianggap "pindah", boleh tetap lolos (tidak ada perubahan data sesungguhnya, tidak melanggar niat toggle ini) — putuskan saat implementasi apakah perlu dibedakan eksplisit atau cukup `upsert` idempotent yang sama hasilnya kalau `ekstrakurikulerId` tidak berubah.

## Spec Detail

### Schema (Prisma)
Model baru `EkstraRegistrationConfig` (nama sementara, sesuaikan konvensi kalau ada pola penamaan lain yang lebih pas saat implementasi):
```prisma
model EkstraRegistrationConfig {
  id             Int      @id @default(1)
  lockPindahEkstra Boolean @default(false)
  updatedById    Int?
  updatedBy      User?    @relation(fields: [updatedById], references: [id])
  updatedAt      DateTime @updatedAt

  @@map("ekstra_registration_config")
}
```
- Pola singleton SAMA seperti `ScheduleConfig`/`AttendanceLockConfig` — `SINGLETON_ID = 1`, `findFirst()`/`upsert()` di service, enforce 1-baris di level SERVICE (bukan DB constraint, konsisten pola existing).
- Tambah back-relation di `User` model.
- Migration baru.

### Backend
- Modul baru KECIL (atau tambahkan ke modul `ekstra-publik` yang sudah ada kalau dirasa lebih related — putuskan saat implementasi) — service dengan method `isLockPindahEkstra()` dan `setLockPindahEkstra(value, updatedById)`, pola sama persis `AttendanceLockConfigService`.
- `GET /ekstra-registration-config` (atau nama serupa) — bisa diakses semua role terautentikasi (untuk kebutuhan render di admin) ATAU minimal `super_admin`+`card_admin` (konsisten guard halaman `ekstra-kurikuler` yang sudah ada) — putuskan scope akses baca saat implementasi.
- `PATCH /ekstra-registration-config` — `@Roles(UserRole.super_admin)` SAJA (konsisten pola `AttendanceLockConfig` yang juga super_admin-only untuk PATCH, meski GET lebih longgar).
- `@LogActivity` wajib di endpoint PATCH.
- **`EkstraPublikService.submit()`** (`ekstra-publik.service.ts:71-101`) — modifikasi:
  1. Cek dulu apakah `studentId` SUDAH punya `EkstraPendaftaran` (row existing).
  2. Kalau BELUM ada row → lanjut seperti biasa (upsert insert baru), TIDAK terpengaruh toggle.
  3. Kalau SUDAH ada row DAN `ekstrakurikulerId` yang dikirim SAMA dengan yang lama → lanjut seperti biasa (bukan "pindah", tidak ada perubahan berarti).
  4. Kalau SUDAH ada row DAN `ekstrakurikulerId` BERBEDA dari yang lama → cek `isLockPindahEkstra()` — kalau `true`, TOLAK (`ConflictException`/`ForbiddenException` dengan pesan jelas ke siswa, misal "Pendaftaran ekstrakurikuler sedang dikunci, hubungi admin untuk pindah ekstra"); kalau `false`, lanjut seperti biasa (upsert menimpa).
- **`pindahkanEkstra()`** (`ekstra-publik.service.ts:186-209`) — **TIDAK DIUBAH SAMA SEKALI**, tetap berfungsi normal terlepas status toggle (jalur admin, bukan publik).

### Frontend
- `apps/web/src/app/(admin)/ekstra-kurikuler/ekstra-kurikuler-view.tsx` (atau `page.tsx` terkait) — tambah toggle switch (REUSE komponen toggle yang sudah ada, cek `pengaturan-absensi-view.tsx` untuk pola custom toggle switch yang sudah dibuat sebelumnya — termasuk fix bug posisi thumb yang pernah terjadi, JANGAN ulangi bug yang sama) — label jelas: "Kunci Pendaftaran Ekstrakurikuler" + deskripsi singkat: "Siswa yang sudah terdaftar tidak bisa pindah ekstra sendiri. Siswa baru tetap bisa mendaftar."
- Halaman publik `/daftar-ekstra` — kalau submit ditolak karena toggle aktif, tampilkan pesan error yang jelas ke siswa (bukan error generik), sesuai pesan yang dikembalikan backend.

## Edge Cases
- Siswa yang row `EkstraPendaftaran`-nya SUDAH DIHAPUS admin (lewat `batalkanPendaftaran()`, kalau ada method itu — cek dulu) → dianggap "belum pernah terdaftar" lagi, boleh daftar baru meski toggle aktif (konsisten prinsip: toggle cuma soal PINDAH dari state existing, bukan soal siapa yang boleh daftar).
- Toggle diaktifkan SAAT ADA siswa sedang di tengah proses submit (race condition kecil, tidak kritis) — cukup validasi di service seperti biasa, tidak perlu penanganan khusus.

## Files
- **Buat:** migration Prisma baru, modul/service config baru (atau tambahan ke `ekstra-publik` existing).
- **Modifikasi:** `apps/api/prisma/schema.prisma` (model baru + relasi `User`), `apps/api/src/ekstra-publik/ekstra-publik.service.ts` (`submit()`), `apps/web/src/app/(admin)/ekstra-kurikuler/ekstra-kurikuler-view.tsx` (toggle UI), `apps/web/src/app/daftar-ekstra/` (penanganan pesan error kalau ditolak).
- **Jangan sentuh:** `pindahkanEkstra()` (jalur admin, tidak boleh terpengaruh toggle sama sekali), `EkstraPendaftaran.studentId @unique` constraint (tidak perlu diubah, sudah benar untuk kebutuhan ini).

## Acceptance Criteria
- [x] Toggle "Kunci Pendaftaran Ekstrakurikuler" muncul di halaman admin `ekstra-kurikuler`, hanya `super_admin` yang bisa mengubahnya.
- [x] Toggle AKTIF: siswa yang SUDAH terdaftar di ekstra A, coba daftar ke ekstra B via `/daftar-ekstra` → DITOLAK dengan pesan jelas.
- [x] Toggle AKTIF: siswa yang BELUM PERNAH terdaftar di ekstra manapun → TETAP BISA daftar pertama kali seperti biasa.
- [x] Toggle AKTIF: submit ulang dengan ekstra yang SAMA (bukan pindah) → tetap berhasil, tidak ditolak.
- [x] Toggle AKTIF: admin lewat halaman Monitoring TETAP BISA memindahkan siswa manual (`pindahkanEkstra()` tidak terpengaruh).
- [x] Toggle NONAKTIF: semua perilaku kembali seperti sekarang (siswa bebas pindah ekstra sendiri), regresi nol.
- [x] `@LogActivity` terpasang di endpoint PATCH config.
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] Cek apakah ada method `batalkanPendaftaran()`/serupa (disinggung di monitoring service per riset) — pastikan siswa yang dibatalkan pendaftarannya oleh admin bisa daftar ulang normal meski toggle aktif (dianggap "belum terdaftar" lagi).
- [x] Pastikan `pindahkanEkstra()` di `ekstra-publik.service.ts` BENAR-BENAR tidak tersentuh validasi baru — cek ulang tidak ada shared helper function yang tidak sengaja ikut menerapkan validasi toggle ke situ juga.
- [x] Reuse pola toggle switch UI yang sudah ada (`pengaturan-absensi-view.tsx`) — termasuk perhatikan bug posisi thumb yang pernah terjadi di komponen serupa sebelumnya, pastikan tidak terulang.

## Status Eksekusi (2026-08-07)

**Selesai, diverifikasi live end-to-end.**

### Backend
- Model `EkstraRegistrationConfig` (singleton, pola sama `AttendanceLockConfig`) + migration `20260807155917_t134_ekstra_registration_config`.
- `EkstraRegistrationConfigModule` baru (`get()`, `isLockPindahEkstra()`, `update()` dengan `@LogActivity` + `activityLog.record()`).
- `GET /ekstra-registration-config` — semua role terautentikasi. `PATCH` — `@Roles(super_admin)` saja.
- `EkstraPublikService.submit()` dimodifikasi: cek `existing.ekstrakurikulerId !== dto.ekstrakurikulerId` → kalau beda (pindah) DAN `isLockPindahEkstra()` true → `ConflictException` (409) pesan "Pendaftaran ekstrakurikuler sedang dikunci, hubungi admin untuk pindah ekstra". `pindahkanEkstra()` TIDAK disentuh sama sekali.
- `batalkanPendaftaran()` (existing) menghapus baris `EkstraPendaftaran` sepenuhnya → siswa otomatis kembali dianggap "belum terdaftar" oleh cek `submit()` yang baru, tanpa perlu logic tambahan.
- Halaman publik `/daftar-ekstra` sudah menampilkan `body.message` dari response non-OK apa pun (`daftar-ekstra-form.tsx`) — pesan error toggle otomatis tampil tanpa perlu perubahan frontend di halaman itu.

### Frontend
- `LockPindahEkstraToggle` (client component baru di `ekstra-kurikuler-view.tsx`) — toggle switch di-copy dari pola `pengaturan-absensi-view.tsx`, posisi thumb dicek benar (tidak mengulang bug lama).

## Task Susulan — Tombol "Pindahkan Ekstra" di Halaman Admin (2026-08-08)

**Status: kode ditulis, BELUM diverifikasi penuh, BELUM di-commit.**

### Temuan
Saat cek T134 selesai, user bertanya "fitur admin bisa mengisi/memindah ekstra belum terbuat?" — ternyata benar: halaman `(admin)/ekstra-monitoring/ekstra-monitoring-view.tsx` cuma punya tombol **Batalkan pendaftaran** (hapus baris, siswa harus daftar ulang sendiri via `/daftar-ekstra`). Endpoint backend `PATCH /ekstra-monitoring/siswa/:studentId/pindahkan` (`pindahkanEkstra()`) **sudah ada sejak 2026-07-27** dan dipakai sebagai bukti "jalur admin tidak terkunci toggle" di verifikasi T134 — tapi **tidak pernah ada tombol UI yang memanggilnya**.

Ditemukan juga komentar kode (sekarang sudah diperbarui) yang menyatakan ini **keputusan sengaja tanggal 2026-07-28**: "admin TIDAK BOLEH memilihkan ekstra baru untuk siswa" — bukan bug, melainkan keputusan lama yang eksplisit melarang fitur ini. **User mengonfirmasi keputusan itu sudah berubah** (2026-08-08) — admin sekarang BOLEH langsung pindahkan siswa ke ekstra lain dalam 1 aksi, tanpa lewat "batalkan dulu → siswa daftar ulang sendiri".

### Perubahan (belum di-commit)
- `apps/web/src/app/(admin)/ekstra-monitoring/ekstra-monitoring-view.tsx`:
  - Tombol baru "Pindahkan ekstra" (ikon `ArrowRightLeft`) di sebelah tombol Batalkan, hanya untuk siswa status "Sudah Mengisi" (konsisten dengan constraint `pindahkanEkstra()` yang butuh baris `EkstraPendaftaran` sudah ada).
  - Dialog baru: pilih ekstra tujuan dari `Select` (daftar ekstra dikurangi ekstra saat ini), tombol "Pindahkan" memanggil `PATCH /ekstra-monitoring/siswa/:studentId/pindahkan`.
  - Komentar lama & teks dialog "Batalkan Pendaftaran" yang menyebut "admin tidak akan memilihkan ekstra baru" dihapus/diperbarui karena sudah tidak akurat.
- Tidak ada perubahan backend — endpoint `pindahkanEkstra()` + `@LogActivity` sudah lengkap sejak sebelumnya.

### Yang belum selesai
- [ ] Verifikasi live end-to-end (Playwright sempat jalan sampai buka dialog + pilih ekstra tujuan, tapi terhenti sebelum klik tombol "Pindahkan" konfirmasi — user memilih tes manual sendiri, hasil belum dilaporkan balik ke sesi ini).
- [ ] `tsc --noEmit` untuk `apps/web` sudah hijau (dicek sebelum terhenti), tapi belum re-cek final kalau ada perubahan lanjutan.
- [ ] Update Acceptance Criteria/Status Eksekusi final setelah verifikasi selesai.
- [ ] Commit + deploy production — **JANGAN dulu**, sesuai instruksi user.

### Verifikasi live (dev, port 3100/3101)
- Curl matrix lengkap dgn JWT super_admin: (1) siswa belum pernah daftar + lock=true → berhasil; (2) submit ulang ekstra sama + lock=true → berhasil; (3) pindah ekstra beda + lock=true → 409 ditolak; (4) admin `pindahkanEkstra` + lock=true → tetap berhasil (200); (5) toggle off → pindah via publik berhasil lagi.
- Playwright browser: login `adminSU`, toggle di halaman `ekstra-kurikuler` — render benar, klik toggle on/off berfungsi, reload halaman → state persisten dari DB.
- Data uji (`ekstra_pendaftaran` row test) dibersihkan setelah verifikasi; config disisakan di default `lockPindahEkstra: false`.
- Tidak ada test suite existing untuk modul `ekstra-publik` (konsisten dengan modul itu sebelumnya) — verifikasi mengandalkan curl + browser live, bukan unit test baru.

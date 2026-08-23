# T111 — API+Web: Menu "Riwayat Aktivitas Saya" di Profil Piket (di atas Logout)

## Depends on
Tidak ada dependency teknis. Reuse `ActivityLog` yang sudah ada (insert-only, sudah mencatat `actorId` untuk aksi izin/lock/unlock/cetak-surat-T107) — tidak ada model/tabel baru.

## Objective
Piket bisa melihat riwayat SEMUA aksinya sendiri di aplikasi ini (dari awal pakai sampai sekarang) lewat menu baru "Riwayat Aktivitas Saya" di dropdown/halaman profil, ditempatkan di atas tombol Logout.

## Context
- **App:** `apps/api` (perluas akses + filter endpoint existing) + `apps/web` (menu baru di profil)
- **Riset 2026-08-05 (Explore agent, baca kode langsung):**
  - `ActivityLog` (`apps/api/prisma/schema.prisma:589-605`) SUDAH generic, insert-only, mencatat `actorId`, `action`, `targetType`, `targetId`, `snapshotBefore`/`snapshotAfter`, `ipAddress`, `createdAt` — lintas modul, dipasang via `@LogActivity` di banyak controller (izin, lock/unlock siswa, surat T107, dll).
  - Endpoint `GET /activity-log` (`apps/api/src/activity-log/activity-log.controller.ts:11`) **SAAT INI `@Roles(UserRole.super_admin)` SAJA** — piket tidak bisa akses sama sekali, bahkan untuk data miliknya sendiri.
  - Filter yang SUDAH ADA di `ListActivityLogDto` (`apps/api/src/activity-log/dto/list-activity-log.dto.ts`): `actorId`, `action`, `targetType`, `from`, `to`, `page`, `pageSize` — filter by `actorId` SUDAH ADA, jadi query "riwayat saya" secara teknis tinggal `actorId = user.sub` dari JWT, tidak perlu field DTO baru.
  - Frontend konsumsi SAAT INI cuma `apps/web/src/app/(admin)/log/page.tsx` + `log-view.tsx` (admin only, generic, semua actor).

## Keputusan Desain (untuk didiskusikan/dikonfirmasi saat eksekusi kalau ada yang ambigu)

1. **Akses**: piket HANYA boleh lihat aktivitasnya SENDIRI (`actorId = diri sendiri`), TIDAK boleh lihat aktivitas piket lain — beda dari halaman admin `/log` yang bisa lihat semua actor. Ini BUKAN memberi role `guru_piket` akses penuh ke `GET /activity-log` apa adanya (itu akan membocorkan aktivitas piket lain) — perlu salah satu:
   - (a) endpoint BARU khusus `GET /activity-log/me` yang otomatis filter `actorId` dari JWT, tidak menerima parameter `actorId` dari client sama sekali (mencegah piket iseng lihat punya piket lain lewat query param manipulasi), ATAU
   - (b) endpoint existing `GET /activity-log` dibuka untuk `guru_piket` TAPI backend memaksa override `actorId` ke `user.sub` sendiri kalau role bukan `super_admin` (abaikan `actorId` dari query kalau bukan admin).
   - **Rekomendasi: opsi (a)**, lebih eksplisit dan tidak mengandalkan logic percabangan role di 1 endpoint yang sama — tapi putuskan saat implementasi, keduanya valid secara fungsional.
2. **Cakupan action yang ditampilkan**: SEMUA action yang tercatat untuk piket itu (izin/lock/unlock/cetak-surat/dll) — tidak perlu filter khusus, tampilkan apa adanya secara kronologis (terbaru dulu).
3. **Format tampilan**: per entry tampilkan minimal: tanggal+jam, jenis aksi (`action`, mungkin perlu mapping ke label Indonesia yang lebih ramah daripada string mentah seperti `"permit.create"` → "Membuat Izin Keluar" — cek apakah sudah ada mapping label seperti ini di halaman admin `/log`, reuse kalau ada), target (nama siswa terkait kalau relevan, dari `snapshotAfter`/`targetId` — mungkin perlu join tambahan atau cukup tampilkan apa adanya dari snapshot JSON tanpa join baru, putuskan tingkat kedalaman saat implementasi mengingat scope task ini sederhana).

## Spec Detail

### Backend
- Endpoint baru `GET /activity-log/me` (`apps/api/src/activity-log/activity-log.controller.ts`) — `@Roles(UserRole.guru_piket)` (tambahkan role lain juga kalau masuk akal semua role punya menu "riwayat aktivitas saya" di masa depan, tapi untuk scope task ini fokus piket dulu sesuai permintaan). Service method reuse `ActivityLogService` yang sudah ada, paksa `actorId = req.user.sub`, terima `page`/`pageSize` saja dari query (bukan `actorId`/`action`/`targetType` bebas dari client).
- Pastikan pagination wajar (default pageSize kecil, misal 20-50) karena riwayat "dari awal pakai aplikasi" bisa panjang.

### Frontend
- Cari komponen dropdown/menu profil yang sudah ada (kemungkinan di `apps/web/src/components/shell/top-bar.tsx`, tombol avatar baris ±63-78, kalau ada dropdown menu yang expand saat diklik — cek dulu apakah tombol ini sudah punya dropdown atau cuma tombol Logout langsung; kalau belum ada dropdown sama sekali, task ini juga perlu menambah struktur dropdown dasar sebelum bisa menyisipkan menu baru di atas Logout).
- Tambah item menu "Riwayat Aktivitas Saya" di atas Logout, klik → halaman/modal baru menampilkan list dari `GET /activity-log/me` (pagination, terbaru dulu).
- Halaman/komponen baru: `apps/web/src/app/(piket)/riwayat-aktivitas/` (atau lokasi lain yang lebih cocok kalau ternyata menu profil ini shared lintas role — cek dulu apakah top-bar/profil dropdown dipakai bersama admin/guru juga, kalau iya pertimbangkan bikin ini generic per-role bukan piket-only, TAPI untuk scope task ini fokus dulu piket sesuai permintaan user, role lain bisa menyusul task terpisah).

## Files
- **Buat:** endpoint `GET /activity-log/me` (modifikasi controller+service existing, bukan modul baru), halaman/komponen riwayat aktivitas piket baru di frontend.
- **Modifikasi:** `apps/api/src/activity-log/activity-log.controller.ts`, `apps/web/src/components/shell/top-bar.tsx` (atau lokasi dropdown profil yang sesuai).
- **Jangan sentuh:** `ActivityLog` model/schema (tidak perlu field baru), halaman admin `/log` yang sudah ada (tetap seperti sekarang, tidak boleh regresi).

## Acceptance Criteria
- [x] Menu "Riwayat Aktivitas Saya" muncul di dropdown/area profil piket, di atas Logout. `top-bar.tsx` — `RiwayatAktivitasMenuItem`, hanya render untuk `guru_piket` (gerbang via `usePiketNotifications() !== null`, pola sama `PiketNotificationBell`).
- [x] Klik menu menampilkan daftar SEMUA aksi yang pernah dilakukan piket yang login, terbaru dulu, dengan pagination. Halaman baru `apps/web/src/app/(piket)/piket/riwayat-aktivitas/`.
- [x] Piket TIDAK BISA melihat aktivitas piket lain lewat endpoint ini. **Diverifikasi live 2x lipat lebih kuat dari target**: `?actorId=1` bukan cuma diabaikan tapi DITOLAK 400 (`"property actorId should not exist"`, `whitelist`+`forbidNonWhitelisted` pada `ListMyActivityLogDto` yang sengaja tidak punya field `actorId` sama sekali).
- [x] Halaman admin `/log` tetap berfungsi seperti sebelumnya, tidak ada regresi. **Diverifikasi live**: `GET /activity-log` via token super_admin tetap 200 normal.
- [x] Build + type-check `apps/api` dan `apps/web` hijau. `tsc --noEmit` bersih kedua app, `nest build`+`next build` sukses (`/piket/riwayat-aktivitas` compile 1.59 kB), jest 183/183 tetap lulus.

## Validasi Claudian
- [x] **Opsi (a) dipilih** (endpoint baru `GET /activity-log/me`, sesuai rekomendasi spec) — `actorId` dipaksa dari JWT di controller, DTO terpisah (`ListMyActivityLogDto`) tanpa field `actorId` sama sekali supaya client tidak punya jalur apa pun untuk override.
- [x] Struktur dropdown profil dicek langsung: `top-bar.tsx` SUDAH punya dropdown (bukan tombol Logout langsung), dan SHARED lintas semua role group (`(piket)`, `(admin)`, `(guru)`, `(admin-jurnal)`, `(pembina-ekstra)`) via `TopBarWithTitle`. Menu baru digate piket-only pakai context existing (`usePiketNotifications`), TIDAK perlu prop role baru yang di-thread manual — konsisten dengan pola T108.

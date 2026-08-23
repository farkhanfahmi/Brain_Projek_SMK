# T153 — Fix: Auto-Lock Siswa Terlambat Tidak Tercatat ke Activity Log

## Depends on
Tidak ada dependency teknis. Independen, prioritas SEDANG-TINGGI (gap audit trail nyata, ditemukan saat investigasi insiden lock massal 2026-08-10).

## Objective
Setiap kali `applyLateStrikeLock()` mengunci siswa secara otomatis (2x terlambat), aksi itu **WAJIB** tercatat ke `ActivityLog` — TANPA GAGAL DIAM-DIAM. Saat ini kode SUDAH BERMAKSUD mencatatnya (pakai akun "sistem"), tapi dikonfirmasi via data production: banyak siswa dengan pola `lockedReason` auto-lock (`lockedById: null`) **TIDAK PUNYA** entri `student.lock` di `ActivityLog` sama sekali.

## Context — Bug Dikonfirmasi (Riset 2026-08-10)

Saat investigasi laporan user "siswa masih banyak terkunci walau toggle lock otomatis sudah dimatikan" — ditemukan bahwa toggle-nya sendiri SUDAH BENAR (tidak ada bug di situ, mematikan toggle memang hanya mencegah lock BARU, siswa yang sudah terlanjur terkunci sebelumnya memang harus di-unlock manual — ini expected behavior, sudah dijelaskan di UI). TAPI dalam proses investigasi, ditemukan bug TERPISAH:

`AttendanceService.applyLateStrikeLock()` (`apps/api/src/attendance/attendance.service.ts:272-340`) — setelah mengunci siswa (`student.update`, baris ~301-308), method ini memanggil `this.activityLog.record()` (baris ~310-324) dengan `actorId` dari `activityLog.getSystemActorId()` (akun "sistem" khusus, dibuat T112) untuk mencatat aksi ini ke audit trail.

**Masalahnya**: `student.update` (mengunci) dan `activityLog.record()` (mencatat) adalah **2 OPERASI TERPISAH, TIDAK dibungkus dalam 1 transaksi (`$transaction`)**. Dikonfirmasi lewat data production: ditemukan siswa dengan `lockedReason` pola auto-lock jelas (`lockedById: null`, teks menyebut "2x terlambat") — TAPI `ActivityLog` untuk siswa itu **HANYA** punya entri `student.unlock` (manual, belakangan), **TIDAK PERNAH ADA** entri `student.lock` sebelumnya. Ini membuktikan: siswa BERHASIL dikunci (`student.update` sukses), tapi pencatatan log-nya GAGAL SECARA DIAM-DIAM (exception di `activityLog.record()` atau `getSystemActorId()` tidak ter-handle, request tap tetap dianggap sukses karena update kunci sudah commit duluan).

**Dampak nyata**: admin/piket TIDAK BISA mengetahui kapan dan kenapa seorang siswa di-lock otomatis oleh sistem hanya dari halaman Log Aktivitas — harus menebak dari kondisi lain (`Student.lockedAt` ada isinya, `lockedById: null` sebagai penanda tidak langsung). Ini pola yang SAMA dengan aturan permanen proyek "setiap operasi mutasi wajib tercatat" (memory `feedback_wajib_log_activity`), tapi di sini gap-nya BUKAN lupa pasang decorator — kodenya SUDAH ADA niat mencatat, cuma implementasinya rapuh (non-atomik).

## Spec Detail

### 1. Bungkus `student.update` (lock) + `activityLog.record()` dalam 1 operasi atomik

Di `applyLateStrikeLock()` — ganti 2 panggilan terpisah (`prisma.student.update` lalu `activityLog.record()`) menjadi SATU `$transaction`:
```ts
await this.prisma.$transaction(async (tx) => {
  await tx.student.update({
    where: { id: studentId },
    data: { lockedAt: effectiveTime, lockedReason, lockedById: null },
  });
  // activityLog.record() HARUS menerima parameter `tx` opsional untuk ikut transaksi ini —
  // CEK apakah ActivityLogService.record() SUDAH mendukung ini (kemungkinan besar tidak,
  // karena biasanya dipakai lewat interceptor HTTP di luar transaksi bisnis) — kalau belum,
  // method itu PERLU diperluas terima Prisma client/transaction opsional sebagai parameter.
});
```
- **PENTING**: cek dulu signature `ActivityLogService.record()` (`apps/api/src/activity-log/activity-log.service.ts`) — apakah SUDAH bisa menerima `tx: Prisma.TransactionClient` sebagai parameter opsional (untuk ikut transaksi caller), atau method ini SELALU pakai `this.prisma` (koneksi global) secara hardcoded. Kalau hardcoded — method ini PERLU diperluas (parameter opsional `tx?: Prisma.TransactionClient`, default ke `this.prisma` kalau tidak diberikan) supaya BISA dipanggil di dalam `$transaction` caller lain — JANGAN duplikasi logic `record()` di tempat baru, perluas method yang sudah ada.
- Kalau `getSystemActorId()` (dipanggil SEBELUM masuk transaksi, karena ini murni lookup read-only cached, tidak perlu ikut transaksi write) THROW exception (misal akun "sistem" entah kenapa tidak ditemukan) — SELURUH proses lock (termasuk `student.update`) HARUS GAGAL BERSAMA (rollback), BUKAN siswa terlanjur terkunci tanpa log. Ini prinsip inti perbaikan: **atomicity** — kedua efek terjadi bersama, atau tidak terjadi sama sekali.

### 2. Pastikan kegagalan transaksi ini TIDAK menggagalkan seluruh proses `tap()`

- `applyLateStrikeLock()` dipanggil dari `tap()` SETELAH `AttendanceRecord` untuk tap itu sendiri sudah berhasil dibuat/diupdate (siswa tetap tercatat hadir/terlambat, TERLEPAS dari apakah lock berhasil atau tidak) — task ini TIDAK BOLEH mengubah urutan ini. Kalau transaksi lock+log GAGAL (kasus langka), `tap()` yang memanggilnya HARUS tetap mengembalikan response sukses tap SEPERTI BIASA ke kiosk (siswa tetap tercatat masuk/terlambat) — cukup lock-nya yang tidak terjadi kali ini (siswa TIDAK terkunci, TIDAK ada log — kegagalan yang AMAN, bukan kegagalan yang MERUSAK data tap).
- Bungkus pemanggilan `applyLateStrikeLock()` di `tap()` dengan try/catch — kalau gagal, LOG ERROR (`Logger.error`, supaya admin/developer bisa lihat di server log kalau ini terjadi sering) TAPI JANGAN lempar ulang exception ke caller `tap()` — proses tap utama harus tetap sukses.

## Edge Cases
- Kegagalan transaksi ini (jarang, tapi mungkin — misal DB connection pool penuh sesaat) — siswa yang SEHARUSNYA kena lock strike ke-2 hari ini TIDAK terkunci saat itu. Ini BUKAN masalah permanen — tap BERIKUTNYA kalau siswa itu terlambat LAGI (strike ke-3, dst, tergantung logic T154 kalau sudah dikerjakan) akan mencoba lagi, kemungkinan besar berhasil di percobaan berikutnya. TIDAK PERLU mekanisme retry otomatis khusus — cukup pastikan tidak ada state "setengah jalan" yang merusak data (itulah tujuan atomicity di poin 1).
- Data LAMA yang SUDAH terlanjur "auto-locked tanpa log" (sebelum fix ini) — task ini TIDAK PERLU backfill log historis (tidak mungkin direkonstruksi akurat, kapan tepatnya lock itu terjadi mungkin masih ada di `Student.lockedAt`, tapi actor/detail lain sudah hilang) — cukup pastikan KE DEPAN tidak terulang.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`applyLateStrikeLock()`, bungkus transaksi, try/catch di pemanggil `tap()`), `apps/api/src/activity-log/activity-log.service.ts` (`record()`, perluas terima `tx` opsional KALAU belum mendukung).
- **Jangan sentuh:** logic toggle `AttendanceLockConfig`/`isLateLockAutoEnabled()` (sudah benar, tidak ada bug di situ), logic hitung "2x terlambat" itu sendiri (itu scope T154, task terpisah).

## Acceptance Criteria
- [x] `applyLateStrikeLock()` — lock siswa dan pencatatan log terjadi ATOMIK (1 transaksi) — kedua-duanya berhasil, atau kedua-duanya gagal (rollback), TIDAK PERNAH salah satu saja.
- [x] Kegagalan proses lock+log TIDAK menggagalkan response `tap()` ke kiosk — siswa tetap tercatat hadir/terlambat normal.
- [x] Test manual: simulasikan `getSystemActorId()` gagal (misal akun "sistem" sementara dihapus/dinonaktifkan saat testing) — verifikasi siswa TIDAK terkunci (bukan terkunci-tanpa-log), dan tap tetap sukses direspons ke kiosk.
- [x] Test manual: siswa mencapai strike ke-2 dalam kondisi normal — verifikasi BAIK `Student.lockedAt` BAIK entri `ActivityLog` (`student.lock`, actor "sistem") muncul BERSAMAAN, tidak ada yang tertinggal.
- [x] Build + type-check `apps/api` hijau. Test suite existing lulus 100%.

## Validasi Claudian
- [x] **JANGAN** ubah logic toggle atau logic hitung "2x terlambat" — task ini MURNI soal atomicity lock+log, scope sempit dan terisolasi.
- [x] Perluasan `ActivityLogService.record()` (kalau diperlukan) — pastikan TIDAK merusak SEMUA pemanggil lain yang sudah ada (parameter `tx` HARUS opsional dengan default aman, backward compatible).
- [x] JANGAN backfill log historis untuk kasus yang sudah terlanjur terjadi sebelum fix ini — di luar scope, data historis itu sudah tidak bisa direkonstruksi akurat.

## Status Eksekusi (2026-08-14)

**Selesai.** Perubahan minimal, scope sempit sesuai spec.

- `apps/api/src/activity-log/activity-log.service.ts` — `record()` gained optional 2nd parameter `tx?: Prisma.TransactionClient` (default `this.prisma`), backward compatible — 27 caller lain di codebase TIDAK diubah, semua tetap pakai signature 1-argumen.
- `apps/api/src/attendance/attendance.service.ts` — `applyLateStrikeLock()`: `getSystemActorId()` dipanggil SEBELUM `$transaction` (murni lookup, exception di sini membatalkan seluruh proses SEBELUM ada mutasi apapun); `student.update` + `activityLog.record(..., tx)` dibungkus `this.prisma.$transaction()`. Pemanggil di `tap()` dibungkus try/catch + `Logger.error` (logger baru ditambahkan ke class, belum ada sebelumnya) — kegagalan tidak dilempar ulang ke caller.

**Verifikasi live (dev DB port 3307, API dev port 3101, production tidak disentuh):**
1. Skenario normal — siswa strike ke-2 (tap real via `POST /attendance/tap` dengan kiosk device token asli, `client_timestamp` dipaksa lewat jam masuk sekolah) → response `justLocked: true`, `Student.lockedAt` terisi DAN `ActivityLog` (`student.lock`, actor "sistem") muncul BERSAMAAN dalam 1 verifikasi SELECT.
2. Skenario gagal — username akun "sistem" di-rename sementara (memaksa `getSystemActorId()` throw `InternalServerErrorException`), API di-restart dulu supaya in-memory cache actor id kosong → tap strike-2 tetap `result: accepted, status: terlambat` (kiosk tidak melihat kegagalan), TAPI `Student.lockedAt` tetap NULL, TIDAK ADA entri `ActivityLog` baru — server log mencatat `[AttendanceService] ERROR Gagal menerapkan lock otomatis 2x-terlambat untuk studentId=2`. Username "sistem" dikembalikan setelah verifikasi.
3. Data uji (attendance_records, activity_log, lock state siswa test, schedule jam_sekolah sementara, semester sementara — dibutuhkan karena dev DB kebetulan dalam state stray tanpa semester aktif/jadwal jam_sekolah sama sekali saat sesi ini dimulai, tidak terkait T153) dan `kiosks.allowed_ip` (di-override sementara ke 127.0.0.1 untuk request dari localhost) — semua dikembalikan/dihapus, dikonfirmasi via re-query.
4. `tsc --noEmit` bersih, `jest` — 22 suite / 273 test lulus 100%.

**Catatan di luar scope T153 (ditemukan saat setup verifikasi, dilaporkan bukan diperbaiki):** dev DB sempat punya 0 baris `semesters` (academic year aktif tanpa semester) dan 0 baris `schedules type=jam_sekolah` — kemungkinan sisa testing task lain yang belum di-restore. Tidak diperbaiki permanen di sini (di luar scope), hanya dibuat sementara lalu dihapus lagi untuk kebutuhan verifikasi tap.

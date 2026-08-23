# T237 — API+Web: Guru Wajib Isi Jurnal Sebelum Tap Pulang + Lock Otomatis Jika Tidak Tap Pulang

## Depends on
Tidak ada dependency teknis. Independen, murni modul `attendance`/`teaching-sessions`/`users`.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-21)

**Lock mekanisme HANYA ada untuk `Student`** (ADR-017: `lockedAt`/`lockedReason`/`lockedById`/`unlockedAt`/`unlockedById`/`unlockNote`, plus `lateStrikeResetAt` ADR-025 untuk lock otomatis 2x-terlambat) — **model `User`/`Teacher` TIDAK PUNYA field lock apa pun** saat ini. Task ini membangun mekanisme SERUPA untuk `User` (akun guru), BUKAN reuse field `Student` (beda model).

**Tap pulang guru SAAT INI tidak ada validasi jurnal sama sekali** — `AttendanceService.tap()` cabang tap ke-2/pulang (`attendance.service.ts:324-330`) berlaku SAMA untuk siswa maupun guru, HANYA update `waktuPulang`+`pulangVia`, TIDAK ADA query ke `TeachingSession`/`JournalEntry` sebelum diizinkan.

**Sudah ada dari diskusi sebelumnya (dicatat, bukan bagian task ini)**: guru bisa MULAI mengajar (`startSession`) setelah tap gerbang masuk — `TeachingSessionsService.startSession()` (`teaching-sessions.service.ts:355-458`) sudah cek `AttendanceRecord` guru hari itu ADA sebelum izinkan mulai sesi. **Task ini MENAMBAH aturan baru**: guru TIDAK BISA tap pulang sebelum SEMUA sesi mengajarnya hari itu sudah diisi jurnal.

**`autoCloseDueSessions`** (`teaching-sessions.service.ts:467-499`, cron BullMQ 5 menitan) — HANYA menyentuh `TeachingSession.status/closedAt`, TIDAK ADA logic terkait lock guru atau deteksi "belum tap pulang" saat ini.

**Jam pelajaran terakhir guru per hari** — TIDAK ADA method siap pakai. `getSesiUntukTanggal(teacherId, tanggal)` (`teaching-sessions.service.ts:325-347`) return semua `TeachingSession` guru hari itu dengan jam sudah di-resolve, sorted ascending — elemen terakhir array = jam terakhir hari itu (SETELAH cron generate sesi hari itu jalan).

## Keputusan Dikonfirmasi User (2026-08-21)

1. **Guru TIDAK BISA tap pulang** sebelum mengisi jurnal untuk SEMUA sesi mengajarnya hari itu (asumsi: sesi yang SUDAH DIMULAI — sesi yang tidak pernah di-start karena guru memang tidak mengajar hari itu di jam itu TIDAK dihitung sebagai kewajiban).
2. **Guru yang TIDAK tap pulang sama sekali di hari itu** — **DIKUNCI TOTAL di hari berikutnya** (blokir SEMUA akses dashboard, KONSISTEN pola lock siswa ADR-017 — bukan cuma blokir mulai sesi jurnal, akun benar-benar tidak bisa dipakai sampai dibuka admin).
3. **Buka kunci**: guru konfirmasi ke admin (di luar sistem, misal lisan/WA — TIDAK PERLU fitur "request unlock" formal di sistem, DI LUAR SCOPE task ini kecuali diminta terpisah), admin buka kunci via UI, DENGAN OPSI input jam pulang manual.
4. **Kalau admin TIDAK input jam pulang saat buka kunci** — jam pulang OTOMATIS = jam akhir jadwal pelajaran guru itu hari itu (hari yang terkunci, BUKAN hari buka kunci) **+ 5 menit**.

## Spec Detail

### 1. Schema — tambah field lock ke `User`

```prisma
// TAMBAH ke model User, REPLIKASI pola Student ADR-017 tapi field TERPISAH (model beda):
lockedAt      DateTime? @map("locked_at")
lockedReason  String?   @map("locked_reason")
lockedById    Int?      @map("locked_by")
unlockedAt    DateTime? @map("unlocked_at")
unlockedById  Int?      @map("unlocked_by")
unlockNote    String?   @map("unlock_note")

lockedBy   User? @relation("UserLockedBy", fields: [lockedById], references: [id])
unlockedBy User? @relation("UserUnlockedBy", fields: [unlockedById], references: [id])
```
- **VERIFIKASI SAAT IMPLEMENTASI**: self-relation `User → User` butuh nama relasi eksplisit (`@relation("UserLockedBy", ...)`) supaya tidak bentrok dengan relasi User lain yang sudah ada — CEK skema `User` existing untuk pola self-relation kalau sudah ada di tempat lain.
- Migration ADDITIF murni (`ADD COLUMN` nullable), TIDAK ADA DROP — aman sesuai protokol CLAUDE.md migration non-destruktif.

### 2. Backend — validasi tap pulang guru: WAJIB semua jurnal terisi

`AttendanceService.tap()` — cabang tap pulang (`attendance.service.ts:324-330`), TAMBAH validasi KHUSUS untuk `card.teacherId` (guru, BUKAN siswa — siswa TIDAK terkena aturan ini):
```ts
if (card.teacherId) {
  const sesiHariIni = await this.teachingSessions.getSesiUntukTanggal(card.teacherId, today);
  const sesiBelumIsiJurnal = sesiHariIni.filter(
    (s) => s.startedAt !== null && !s.journalEntry // sesi yang SUDAH dimulai TAPI belum ada JournalEntry
  );
  if (sesiBelumIsiJurnal.length > 0) {
    await this.logTapEvent(dto, kioskId, TapResult.rejected_jurnal_belum_lengkap, card.id, period);
    return {
      result: TapResult.rejected_jurnal_belum_lengkap,
      message: `Anda belum mengisi jurnal untuk ${sesiBelumIsiJurnal.length} sesi mengajar hari ini (${sesiBelumIsiJurnal.map(s => s.mapel.nama).join(", ")}) — isi jurnal dulu sebelum tap pulang.`,
    };
  }
}
```
- **TAMBAH varian baru `TapResult`**: `rejected_jurnal_belum_lengkap` — KONSISTEN pola `rejected_wrong_kiosk_type`/`rejected_wrong_kampus` (T216) yang sudah ada.
- **VERIFIKASI SAAT IMPLEMENTASI**: definisi "belum isi jurnal" — apakah cukup `JournalEntry` ADA (row exists), atau perlu cek field TERTENTU di `JournalEntry` terisi (misal `materi`/`capaianPembelajaran` tidak kosong) — REKOMENDASI: cukup ROW EXISTS (guru sudah submit form jurnal apa adanya), JANGAN validasi kelengkapan field internal jurnal di titik INI (itu urusan form jurnal sendiri, bukan gerbang tap pulang).
- Pesan error WAJIB actionable (sebut nama mapel/jumlah sesi yang belum diisi, KONSISTEN aturan CLAUDE.md pesan error).

### 3. Backend — job deteksi & lock otomatis guru yang tidak tap pulang

- Scheduled job BARU (BullMQ, REPLIKASI pola `auto-close.processor.ts` existing) — jalan **SETELAH tengah malam** (misal 00:30, supaya hari sebelumnya sudah pasti berakhir) — cek SEMUA guru yang:
  1. Punya `TeachingSession` dengan `startedAt !== null` KEMARIN (artinya guru benar-benar mengajar hari itu).
  2. `AttendanceRecord` KEMARIN untuk guru itu — `waktuPulang === null` (tidak pernah tap pulang sama sekali).
- Guru yang match KEDUA kondisi — `User.update({ lockedAt: now, lockedReason: "Tidak tap pulang pada [tanggal]", lockedById: null })` (`lockedById` NULL karena ini otomatis sistem, KONSISTEN pola deteksi `actorId === systemActorId` yang sudah ada di riwayat catatan siswa untuk membedakan lock manual vs otomatis).
- **VERIFIKASI SAAT IMPLEMENTASI**: guru yang SUDAH terkunci sebelumnya (belum sempat dibuka admin) — JANGAN re-lock/timpa `lockedAt` kalau sudah terkunci (idempotent, cek `lockedAt === null` dulu sebelum lock lagi).
- Log aktivitas — `ActivityLogService.record()` manual per guru terkunci (KONSISTEN pola bulk, actorId sistem).

### 4. Backend — guard blokir akses guru terkunci

- REPLIKASI pola guard lock siswa (cari implementasi existing untuk `Student.lockedAt` cek di endpoint mana — KEMUNGKINAN di level middleware/guard auth) — TAMBAH pengecekan SERUPA untuk `User.lockedAt` di titik otentikasi/otorisasi utama (JWT validation atau guard terpisah) — guru dengan `lockedAt !== null` DITOLAK akses SEMUA endpoint dashboard (kecuali endpoint auth/logout itu sendiri, supaya guru tidak "terjebak" tanpa bisa logout).
- Frontend — halaman "Akun Terkunci" (REPLIKASI pola halaman blocking untuk siswa terkunci kalau ada, atau buat serupa) — pesan jelas: "Akun Anda terkunci karena tidak tap pulang pada [tanggal]. Hubungi admin untuk membuka kunci."

### 5. Backend+Web — endpoint buka kunci guru (admin)

- `PATCH /users/:id/unlock` (atau pola serupa endpoint unlock siswa yang sudah ada, REPLIKASI) — body opsional `{ waktuPulang?: string }`.
- **Kalau `waktuPulang` DIKIRIM** — update `AttendanceRecord` guru untuk TANGGAL YANG TERKUNCI (bukan hari ini) dengan `waktuPulang` sesuai input admin.
- **Kalau `waktuPulang` TIDAK DIKIRIM** — HITUNG OTOMATIS: jam akhir jadwal pelajaran guru itu PADA TANGGAL YANG TERKUNCI + 5 menit. Method BARU (belum ada, perlu dibangun): resolve `JadwalSlot` guru itu di hari tsb (`hari` dari tanggal terkunci), ambil `jamKe` TERTINGGI (`orderBy: {jamKe: "desc"}, take: 1`), resolve wall-clock `jamSelesai` via `AlokasiWaktuSlot` (REPLIKASI pola `resolveJamSesi` di `TeachingSessionsService`, method BARU untuk PER-GURU bukan per-sesi tunggal — VERIFIKASI SAAT IMPLEMENTASI apakah bisa reuse sebagian logic `resolveJamSesi` atau perlu ditulis terpisah).
- `User.update({ lockedAt: null, unlockedAt: now, unlockedById: adminId, unlockNote })`.
- `@Roles(UserRole.super_admin)` (buka kunci akun = aksi sensitif).

### 6. Frontend — halaman admin buka kunci guru

- Halaman/section baru (REKOMENDASI: `(admin)/akun/akun-view.tsx` tambah aksi "Buka Kunci" untuk guru berstatus terkunci, KONSISTEN pola tombol Reset Password/Set Password yang sudah ada di halaman itu) — Dialog input opsional jam pulang, tombol submit.

## Edge Cases

- **Guru tidak mengajar sama sekali hari itu** (tidak ada `TeachingSession` yang `startedAt !== null`) — TIDAK terkena lock otomatis (kondisi poin 3.1 tidak terpenuhi), tap pulang TETAP normal (tidak ada jurnal yang perlu diisi).
- **Guru mengajar tapi TIDAK PERNAH tap masuk gerbang sama sekali** (kasus lain, di luar cakupan lock ini — kalau tidak ada `AttendanceRecord` sama sekali, tidak ada `waktuPulang` untuk dicek NULL vs terisi) — VERIFIKASI SAAT IMPLEMENTASI apakah kasus ini perlu ditangani terpisah atau otomatis aman (kemungkinan aman, karena tanpa `AttendanceRecord` job deteksi poin 3 tidak akan match kondisi manapun).
- **Guru punya beberapa `JadwalSlot` di hari yang sama TAPI Opsi Jadwal berbeda (blok, minggu A/B)** — resolusi "jam akhir jadwal" (poin 5) HARUS pertimbangkan SEMUA opsi jadwal aktif hari itu, bukan cuma 1, KONSISTEN cara `TeachingSession` di-generate (multi opsi jadwal aktif sekaligus dimungkinkan).
- **Admin buka kunci TAPI guru itu TERNYATA tidak mengajar hari itu** (data janggal, harusnya tidak match kondisi lock — tapi kalau terjadi) — method resolusi jam otomatis (poin 5) HARUS punya fallback jelas kalau tidak ada `JadwalSlot` ditemukan (misal pesan error, JANGAN crash/null).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (field lock `User`), `apps/api/src/attendance/attendance.service.ts` (validasi tap pulang guru), `apps/api/src/users/users.service.ts`+`users.controller.ts` (endpoint unlock), guard auth (cek `lockedAt`), `apps/web/src/app/(admin)/akun/akun-view.tsx` (UI buka kunci).
- **Buat:** scheduled job baru (BullMQ, deteksi+lock otomatis), method resolusi "jam akhir jadwal guru per tanggal" (baru, tidak ada sebelumnya), halaman "Akun Terkunci" guru (kalau belum ada versi generic yang bisa direuse).

## Acceptance Criteria
- [ ] Guru dengan sesi mengajar yang sudah dimulai TAPI jurnal belum semua terisi — tap pulang DITOLAK, pesan actionable sebut mapel yang belum diisi.
- [ ] Guru yang sudah isi SEMUA jurnal sesi hari itu — tap pulang berhasil normal.
- [ ] Guru yang tidak tap pulang sama sekali (padahal mengajar hari itu) — TERKUNCI otomatis di hari berikutnya (via job), TIDAK bisa akses dashboard.
- [ ] Guru tidak mengajar hari itu — TIDAK terkena lock otomatis.
- [ ] Admin buka kunci TANPA input jam pulang — `waktuPulang` otomatis = jam akhir jadwal guru hari terkunci + 5 menit.
- [ ] Admin buka kunci DENGAN input jam pulang manual — dipakai apa adanya.
- [ ] Job deteksi TIDAK re-lock guru yang sudah terkunci (idempotent).
- [ ] Build + type-check hijau, jest baru: validasi tap pulang (jurnal belum lengkap ditolak, lengkap diizinkan), job lock (mengajar+tidak pulang→terkunci, tidak mengajar→aman), unlock (dengan+tanpa jam manual).

## Validasi Claudian
- [ ] Konfirmasi lock guru TERPISAH total dari lock siswa (model beda, field beda) — TIDAK ada campur logic antara `Student.lockedAt` dan `User.lockedAt`.
- [ ] Konfirmasi guard akses guru terkunci TETAP izinkan logout (guru tidak "terjebak" tanpa jalan keluar).
- [ ] Konfirmasi resolusi jam otomatis (poin 5) pakai tanggal YANG TERKUNCI (bukan hari ini/hari buka kunci) untuk cari jadwal guru.
- [ ] Konfirmasi `rejected_jurnal_belum_lengkap` dicatat di `TapEvent` (insert-only, konsisten aturan proyek).

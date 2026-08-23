# T216 — API+Web: Validasi Kampus Saat Tap Absen (Toggle Global On/Off)

## Depends on
Tidak ada dependency ke rangkaian T203-T215 (modul jadwal). Independen, murni modul `attendance`.

## Konteks — Bug Ditemukan (2026-08-17)

Siswa Kampus 1 bisa tap absen sukses di kiosk Kampus 2 — **bukan regresi, validasi ini memang belum pernah ditulis**. `AttendanceService.tap()` (`apps/api/src/attendance/attendance.service.ts:113-332`) sudah punya kedua data yang dibutuhkan di scope yang sama (`card.student.kelas.kampusId` via include baris 132-135, dan `kioskKampusId` dari parameter method, diisi `KioskGuard` dari `req.kiosk.kampusId`) — tapi tidak pernah dibandingkan. Satu-satunya pemakaian `kioskKampusId` dalam `tap()` adalah baris 324, untuk scoping broadcast realtime dashboard piket ("5 terbaru"), BUKAN validasi/gate.

Validasi yang SUDAH ADA di `tap()` sebagai pola pembanding: cek **tipe** kiosk vs tipe kartu (ADR-022, baris 203-212) — `rejected_wrong_kiosk_type` kalau kiosk siswa dipakai tap kartu guru atau sebaliknya. Task ini menambah pemeriksaan setara untuk **kampus**, bukan tipe.

Pola cross-check kampus SUDAH ADA di file yang sama untuk endpoint lain (`ensureYesterdayRecordInKampus`, `confirmIzinPulang`, `findTodayRecordInKampus` — pola `student.kelas?.kampusId !== kampusId`), hanya belum direplikasi ke `tap()`.

## Keputusan Dikonfirmasi User (2026-08-17)

1. **Toggle GLOBAL** (1 untuk semua kampus) — ikut pola `AttendanceLockConfig` (singleton), BUKAN per-kampus seperti `TvPiketDisplayConfig`.
2. **Default toggle: NONAKTIF** — perilaku saat ini (longgar, tap lintas kampus diizinkan) TETAP JALAN sampai admin aktifkan manual. TIDAK ADA perubahan perilaku mendadak saat fitur ini di-deploy.
3. **Pesan error saat ditolak WAJIB sebut nama kampus asli siswa** — format: `"Kartu ini terdaftar di [Nama Kampus Siswa], tidak bisa tap di gerbang [Nama Kampus Kiosk]"` — BUKAN pesan generik seperti `rejected_wrong_kiosk_type` yang cuma bilang "Kartu ini bukan untuk gerbang ini" (sesuai CLAUDE.md "pesan error sesuai kondisi, bukan generik", dan permintaan eksplisit user rangkaian T203-T215 soal katalog pesan error actionable — pola sama diterapkan di sini).

## Spec Detail

### 1. Schema — config singleton baru

```prisma
// Toggle global: kalau aktif, siswa HANYA bisa tap di kiosk kampus yang sama dengan
// kampus kelasnya. Default nonaktif — perilaku longgar saat ini tetap jalan sampai
// admin aktifkan manual (T216, 2026-08-17).
model KampusTapConfig {
  id                Int      @id @default(1)
  kampusMatchEnabled Boolean @default(false) @map("kampus_match_enabled")
  updatedById       Int      @map("updated_by")
  updatedAt         DateTime @updatedAt @map("updated_at")

  updatedBy User @relation(fields: [updatedById], references: [id])

  @@map("kampus_tap_config")
}
```

- REPLIKASI PERSIS pola `AttendanceLockConfig` (`schema.prisma:453-462`) — singleton `id @default(1)`, 1 boolean, `updatedById`+`updatedAt`.
- Migration: seed 1 baris default (`kampusMatchEnabled: false`) — KONSISTEN cara `AttendanceLockConfig` di-seed (cek `seed.ts` existing untuk pola upsert-nya).

### 2. Backend — modul config baru

REPLIKASI STRUKTUR PERSIS `apps/api/src/attendance-lock-config/` (4 file: service, controller, module, dto) — buat modul baru `apps/api/src/kampus-tap-config/`:

- `kampus-tap-config.service.ts` — `get()`, `isKampusMatchEnabled()` (dipakai `AttendanceService`, tanpa log — KONSISTEN pola `isLateLockAutoEnabled()`), `update()` (upsert + `ActivityLogService.record()` dengan `snapshotBefore`/`snapshotAfter`, KONSISTEN aturan CLAUDE.md `@LogActivity`/log manual untuk mutasi).
- `kampus-tap-config.controller.ts` — `GET /kampus-tap-config` (di balik `JwtAuthGuard` saja, semua role bisa baca), `PATCH /kampus-tap-config` (`@Roles(UserRole.super_admin)` + `RolesGuard`, KONSISTEN `AttendanceLockConfig` yang restrict PATCH ke super_admin).
- `dto/update-kampus-tap-config.dto.ts` — `{ kampusMatchEnabled: boolean }`.
- `kampus-tap-config.module.ts` — export service untuk dipakai `AttendanceModule`.

### 3. Backend — validasi di `tap()`

- `AttendanceModule` import `KampusTapConfigModule`, inject `KampusTapConfigService` ke `AttendanceService`.
- Di `AttendanceService.tap()`, tambah blok validasi **SETELAH** cek `rejected_wrong_kiosk_type` (baris ~212, sebelum debounce baris 214) — HANYA untuk kartu SISWA (`card.studentId`), guru TIDAK di-gate kampus (konsisten catatan existing di baris 318-323: "Guru tidak terikat 1 kampus secara data, bisa mengajar lintas kampus"):

```ts
if (card.studentId) {
  const kampusMatchEnabled = await this.kampusTapConfigService.isKampusMatchEnabled();
  const studentKampusId = card.student?.kelas?.kampusId;
  if (kampusMatchEnabled && studentKampusId && studentKampusId !== kioskKampusId) {
    await this.logTapEvent(dto, kioskId, TapResult.rejected_wrong_kampus, card.id, period);
    const [studentKampus, kioskKampus] = await Promise.all([
      this.prisma.kampus.findUnique({ where: { id: studentKampusId } }),
      this.prisma.kampus.findUnique({ where: { id: kioskKampusId } }),
    ]);
    return {
      result: TapResult.rejected_wrong_kampus,
      message: `Kartu ini terdaftar di ${studentKampus?.nama ?? "kampus lain"}, tidak bisa tap di gerbang ${kioskKampus?.nama ?? "ini"}`,
    };
  }
}
```

- **Siswa TANPA kelas (`studentKampusId` null/undefined, T072 — siswa baru belum di-plot)** — TIDAK DITOLAK (tidak ada kampus untuk dibandingkan, biarkan lolos ke validasi berikutnya) — KONSISTEN pola fallback existing baris 318-323 yang juga tidak block untuk kasus ini.
- **PERTIMBANGKAN** query `Kampus.findUnique` dobel (studentKampus+kioskKampus) tiap tap ditolak — kalau mau optimal, bisa `include: { kampus: true }` sekali di awal `tap()` untuk siswa (REKOMENDASI: cek apakah query siswa di awal method sudah include relasi ini, kalau belum tambah di situ saja daripada query terpisah di blok validasi).

### 4. Enum `TapResult` — varian baru

```prisma
enum TapResult {
  accepted
  rejected_inactive
  rejected_locked
  rejected_unknown
  rejected_duplicate
  rejected_wrong_kiosk_type
  rejected_wrong_kampus   // BARU — T216
}
```

- Migration ALTER TYPE / enum MySQL — cek cara migration Prisma menangani enum existing di proyek ini (biasanya aman, additif, tidak breaking).

### 5. Frontend — toggle baru di halaman Pengaturan Absensi

- `apps/web/src/app/(admin)/pengaturan-absensi/pengaturan-absensi-view.tsx` — TAMBAH section BARU ke-3, SEJAJAR section "Lock Otomatis 2x Terlambat" (baris 39-83) dan "Tanggal Sistem Mulai Live" (baris 90-159) — REPLIKASI PERSIS styling card (`rounded-xl bg-surface p-6 shadow-elevated`) dan toggle switch (`role="switch"`, `aria-checked`, translate-x animasi, `bg-primary`/`bg-surface-subtle`) dari section pertama.
- Judul section: **"Wajib Kampus Sama Saat Tap Absen"** (atau serupa, PUTUSKAN teks final saat implementasi) — deskripsi singkat di bawah toggle jelaskan konsekuensi: "Jika aktif, siswa hanya bisa tap di kiosk kampus yang sama dengan kelasnya. Siswa yang tap di kampus lain akan ditolak."
- `page.tsx` — tambah fetch paralel ke-3 (`GET /kampus-tap-config`), pass sebagai prop baru.
- `PATCH /kampus-tap-config` on toggle click — KONSISTEN pola `handleToggle` section pertama.

### 6. Kiosk — pesan penolakan tampil ke layar

- Cek `apps/kiosk/src/components/feedback-screen.tsx` (SUDAH ADA, dari status git ada modifikasi terbaru) — pastikan `message` dari response `rejected_wrong_kampus` DITAMPILKAN APA ADANYA ke layar kiosk (KONSISTEN aturan CLAUDE.md "Pesan dari backend diteruskan APA ADANYA ke UI, bukan di-generic-kan di frontend") — cek apakah ada mapping hardcoded per-`TapResult` di kiosk yang perlu ditambah varian baru, atau sudah generik pakai field `message` dari response langsung.

## Edge Cases

- **Guru tap** — TIDAK PERNAH di-gate kampus (guru tidak terikat 1 kampus di data, `Teacher` model tidak punya field kampus sama sekali) — validasi HANYA berlaku untuk `card.studentId`.
- **Siswa belum punya kelas** (`kelasId` null, siswa baru T072) — TIDAK DITOLAK, lolos (tidak ada kampus untuk dibandingkan).
- **Toggle dinonaktifkan lagi setelah sempat aktif** — TapEvent lama dengan `rejected_wrong_kampus` TETAP tersimpan sebagai histori (insert-only, tidak ada endpoint hapus/ubah `tap_events` — aturan mutlak proyek).
- **Kiosk itu sendiri tidak berubah kampus** — `kioskKampusId` selalu dari `Kiosk.kampusId` (data kiosk, bukan input kiosk) jadi tidak ada celah spoofing dari sisi request body.

## Files
- **Buat:** `apps/api/src/kampus-tap-config/` (4 file baru, REPLIKASI struktur `attendance-lock-config/`), migration Prisma baru (model `KampusTapConfig` + enum `TapResult` +1 varian).
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (blok validasi baru di `tap()`), `apps/api/src/attendance/attendance.module.ts` (import `KampusTapConfigModule`), `apps/web/.../pengaturan-absensi/pengaturan-absensi-view.tsx`+`page.tsx` (section toggle baru), cek `apps/kiosk/src/components/feedback-screen.tsx` (pastikan pesan custom tampil).

## Acceptance Criteria
- [x] Toggle default NONAKTIF pasca-migration — tap lintas kampus TETAP diizinkan sampai admin aktifkan manual.
- [x] Toggle AKTIF — siswa Kampus 1 tap di kiosk Kampus 2 DITOLAK, pesan sebut nama kampus asli siswa + nama kampus kiosk.
- [x] Toggle AKTIF — siswa tap di kiosk kampus SENDIRI tetap sukses seperti biasa.
- [x] Guru tap — TIDAK PERNAH ditolak karena kampus, toggle aktif/nonaktif tidak berpengaruh ke guru.
- [x] Siswa tanpa kelas (belum di-plot) — tetap bisa tap, tidak ditolak karena kampus.
- [x] `TapEvent` untuk tap yang ditolak kampus tercatat dengan `result: rejected_wrong_kampus` (insert-only, konsisten aturan).
- [x] Halaman Pengaturan Absensi — toggle baru tampil, styling konsisten 2 section existing, `PATCH` bekerja + `ActivityLogService` mencatat perubahan.
- [x] Kiosk — pesan penolakan custom (sebut nama kampus) tampil apa adanya di layar, bukan pesan generik.
- [x] Build + type-check hijau, jest baru untuk skenario: toggle nonaktif (lolos), toggle aktif+kampus beda (ditolak+pesan benar), toggle aktif+kampus sama (lolos), guru (selalu lolos), siswa tanpa kelas (lolos).

## Validasi Claudian
- [x] Konfirmasi `@LogActivity`/`ActivityLogService.record()` terpasang di endpoint `PATCH /kampus-tap-config` — manual `record()` di `KampusTapConfigService.update()` (pola singleton config, sama seperti `AttendanceLockConfigService`).
- [x] Konfirmasi pesan error TIDAK generik — sebut nama kampus asli via query `Kampus.nama`, bukan ID mentah atau teks generik.
- [x] Konfirmasi validasi HANYA jalan untuk siswa, guru tidak pernah ter-gate kampus (sesuai keputusan eksplisit user + catatan existing di kode soal guru lintas kampus) — dicek eksplisit via `if (card.studentId)` membungkus SELURUH blok, `kampusTapConfig.isKampusMatchEnabled()` bahkan tidak dipanggil untuk guru (diverifikasi test).

## Implementasi (2026-08-17)

**Schema**: `KampusTapConfig` (singleton `@default(autoincrement())`, bukan `@default(1)` seperti draft di spec — disesuaikan supaya konsisten dengan pola nyata `AttendanceLockConfig` yang pakai `findFirst()`/`upsert({where:{id:SINGLETON_ID}})`, bukan `findUnique({id:1})` langsung). Enum `TapResult` +1 varian `rejected_wrong_kampus`. Migration `20260817133557_t216_kampus_tap_config` di dev DB (port 3307) — TIDAK disentuh production.

**Backend**: modul `apps/api/src/kampus-tap-config/` REPLIKASI PERSIS struktur `attendance-lock-config/` (service dengan `get()`/`isKampusMatchEnabled()`/`update()`, controller `GET` semua role + `PATCH` super_admin-only, module, DTO). `AttendanceService.tap()` — blok validasi baru SETELAH cek `rejected_wrong_kiosk_type`, SEBELUM debounce: `card.studentId` saja yang di-gate, `studentKampusId` null (siswa belum di-plot) lolos, kampus cocok lolos, kampus beda → `logTapEvent` + reject dengan pesan sebut `Kampus.nama` kedua sisi (2 query `findUnique` paralel, cuma jalan saat DITOLAK — bukan tiap tap).

**Shared types**: `TapResult.REJECTED_WRONG_KAMPUS` ditambah ke `packages/types/src/index.ts` — perlu rebuild `@absensi/types` (`tsup`) supaya kiosk/web tsc lihat member baru (resolusi lewat `dist/index.d.ts`, bukan source langsung).

**Kiosk — poin kritis**: `REJECTION_MESSAGE` map di `tap-messages.ts` diubah dari `Record<...>` PENUH jadi `Partial<Record<...>>` — `REJECTED_WRONG_KAMPUS` SENGAJA TIDAK dipetakan di situ. Kalau dipetakan dengan teks generik, itu akan MENIMPA `response.message` dinamis dari backend (pola existing `response.result in REJECTION_MESSAGE ? generic : response.message` di `feedback-screen.tsx` MEMPRIORITASKAN map statis kalau key ada) — persis anti-pattern yang task ini coba hindari. Dengan `Partial`, key yang tidak ada di map otomatis fallback ke `response.message` apa adanya (pesan sebut kampus asli). Tidak ada perubahan lain di `feedback-screen.tsx` — 2 lokasi existing yang pakai pola ini otomatis benar tanpa modifikasi.

**Frontend**: section ke-3 "Wajib Kampus Sama Saat Tap Absen" di `pengaturan-absensi-view.tsx`, styling+toggle REPLIKASI PERSIS section pertama. `page.tsx` tambah fetch paralel ke-3.

**Verifikasi**: 9 test baru (5 skenario `tap()` T216 di `attendance.service.spec.ts` — toggle nonaktif lolos, toggle aktif+beda kampus ditolak+pesan benar, toggle aktif+sama kampus lolos, guru selalu lolos+`isKampusMatchEnabled` tidak dipanggil, siswa tanpa kelas lolos; 4 test `KampusTapConfigService` baru). 575/575 backend test lulus (naik dari 565). tsc bersih 3 app (api/web/kiosk), `nest build` bersih, `next build` bersih 2 app (web+kiosk). Dev server web direstart bersih pasca build. Live browser verify TIDAK dilakukan (kredensial login belum tersedia sesi ini), curl `/pengaturan-absensi` konfirmasi 307 (bukan 500).

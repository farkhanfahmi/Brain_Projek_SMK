# T040 — Job Harian: Generate `teaching_sessions`

## Depends on
T039 (schedule resolver)

## Objective
Buat cron job yang berjalan tiap awal hari (sebelum jam sekolah mulai) untuk generate baris `teaching_sessions` bagi SEMUA slot jadwal yang aktif hari itu — supaya sesi "belum dimulai guru" tetap terdeteksi tanpa insert on-demand saat guru buka dashboard.

## Context
- **App:** `apps/api`
- **Tables:** `teaching_sessions`, `schedules`
- **Queue:** BullMQ (pola sama seperti job existing Fase 1, lihat `Projek/AbsenSI/02-Tech-Stack.md` bagian Queue)
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — catatan di bagian skema `teaching_sessions`: "baris dibuat oleh job harian... bukan dibuat on-demand saat guru klik"

## Spec Detail

### Job: `generate-teaching-sessions` (BullMQ, scheduled/repeatable job)

**Jadwal eksekusi:** setiap hari jam `00:05` (dini hari, sebelum sekolah mulai) — gunakan BullMQ repeatable job dengan cron pattern `5 0 * * *`

**Logic:**
1. Ambil tanggal hari ini (server time)
2. Kalau hari ini adalah hari libur (cek `school_holidays`) atau Minggu → **skip, tidak generate apapun**, log info saja
3. Kalau bukan libur:
   - Ambil SEMUA `schedules` dengan `type = jam_mengajar` (jadwal mengajar guru, bukan `jam_sekolah`/`jadwal_khusus`)
   - Untuk tiap schedule, cek apakah aktif hari ini via `ScheduleResolverService`:
     - Hari (`DAYOFWEEK`) cocok dengan `tanggal` hari ini
     - Kalau mode blok: `minggu` cocok minggu aktif hari ini ATAU `setiap_minggu`
   - Untuk tiap schedule yang lolos filter → `upsert` baris `teaching_sessions` dengan `schedule_id`, `tanggal = hari ini`, `teacher_id`, `kelas_id`, `mapel_id` (disalin dari schedule), `status: open`, `started_at: null`
   - **Idempotent**: kalau job dijalankan ulang (retry/manual trigger) untuk tanggal yang sama, tidak boleh bikin duplikat — pakai `upsert` berbasis unique constraint `(schedule_id, tanggal)` dari T038, bukan `create`

### Endpoint manual trigger (untuk testing & recovery kalau job gagal jalan)

**POST `/teaching-sessions/generate-today`** — role `admin_jurnal` atau `super_admin`
- Jalankan logic yang sama seperti job, tapi sinkron (bukan queue) untuk tanggal hari ini
- Response: `{ generated: number, skipped_reason: string | null }`
- Berguna kalau job cron gagal jalan semalam (mati listrik server, dst) dan admin perlu generate manual pagi harinya sebelum guru mulai login

## JANGAN
- ❌ JANGAN generate sesi untuk hari libur/Minggu — cek `school_holidays` dan `DAYOFWEEK` dulu sebelum generate apapun (konsisten dengan logic "hari wajib" di modul Rekap Fase 1)
- ❌ JANGAN generate sesi untuk `schedules` yang `tanggal_berlaku_mulai`/`tanggal_berlaku_selesai`-nya tidak mencakup hari ini (kalau ada jadwal khusus dengan rentang tanggal terbatas)
- ❌ JANGAN pakai `create` biasa — WAJIB `upsert` supaya idempotent terhadap retry
- ❌ JANGAN generate sesi untuk `type` selain `jam_mengajar` — `jam_sekolah` dan `jadwal_khusus` bukan scope `teaching_sessions`

## Files
- **Buat:** `apps/api/src/teaching-sessions/teaching-sessions.module.ts`
- **Buat:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts`
- **Buat:** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` (endpoint manual trigger)
- **Buat:** `apps/api/src/teaching-sessions/generate-sessions.processor.ts` (BullMQ processor)
- **Modifikasi:** `apps/api/src/app.module.ts` — register queue baru

## Acceptance Criteria
- [ ] Trigger manual di hari kerja normal (bukan libur) → `teaching_sessions` terisi sesuai jumlah `schedules` aktif hari itu (cocok hari + minggu blok kalau relevan)
- [ ] Trigger manual di hari libur (ada di `school_holidays`) → 0 sesi digenerate, response `skipped_reason` terisi jelas
- [ ] Trigger manual 2x berturut-turut di tanggal yang sama → tidak ada duplikat, jumlah baris tetap sama setelah trigger kedua
- [ ] Job cron terjadwal jalan otomatis (verifikasi via log BullMQ, tidak perlu tunggu real 00:05 — bisa test dengan trigger manual method yang sama)
- [ ] Endpoint manual trigger dari role selain `admin_jurnal`/`super_admin` return 403

## Handoff ke T041–T044
Task-task dashboard guru (T041 dst) akan query `teaching_sessions` yang SUDAH ADA (hasil generate job ini) — mereka tidak boleh membuat baris baru, hanya update (`started_at`, `closed_at`, `status`) baris yang sudah digenerate.

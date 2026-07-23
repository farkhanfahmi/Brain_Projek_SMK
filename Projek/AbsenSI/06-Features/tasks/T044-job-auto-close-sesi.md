# T044 — Job Auto-Close Sesi & Flag Follow-Up Izin

## Depends on
T043 (start-session harus ada dulu supaya ada sesi `open` untuk di-close)

## Objective
Buat job berkala yang: (1) menutup otomatis `teaching_sessions` yang jam selesainya sudah lewat, (2) menandai `teacher_permits` yang belum diisi tugasnya sebagai `follow_up_needed` setelah jam sesi lewat.

## Context
- **App:** `apps/api`
- **Tables:** `teaching_sessions`, `teacher_permits`
- **Queue:** BullMQ
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — "Sesi auto-close otomatis saat jam selesai jadwal tercapai" & bagian "Izin Guru Tidak Mengajar" poin 5

## Spec Detail

### Job: `auto-close-sessions` (BullMQ repeatable, jalan tiap 5 menit)

**Logic bagian 1 — auto-close sesi:**
- Query semua `teaching_sessions` dengan `status = open` dan `closed_at IS NULL` dimana `tanggal = hari ini` dan jam selesai jadwalnya (`schedules.jam_selesai` terkait) sudah lewat dari waktu sekarang
- Untuk tiap sesi yang lolos filter:
  - Update `closed_at = now()`, `status = closed`
  - **Tidak peduli apakah `started_at` terisi atau tidak** — sesi yang guru tidak pernah mulai sama sekali TETAP di-close otomatis (supaya tidak menggantung selamanya sebagai "open", meski secara substansi ini "guru tidak masuk kelas" — itu tercermin dari `started_at IS NULL` saat data ini dibaca nanti oleh TV Piket/rekap, task ini TIDAK perlu membuat status "bolos mengajar" eksplisit, cukup biarkan `started_at IS NULL` + `closed_at IS NOT NULL` sebagai sinyal yang bisa diinterpretasi consumer lain)

**Logic bagian 2 — flag follow-up izin:**
- Query semua `teacher_permits` dengan `tanggal = hari ini`, `submitted_at IS NULL`, `follow_up_needed = false`
- Untuk permit dengan `session_id` terisi (izin spesifik 1 sesi): kalau jam selesai sesi terkait sudah lewat → set `follow_up_needed = true`
- Untuk permit dengan `session_id = null` (izin seharian penuh): kalau SEMUA sesi guru itu hari ini sudah lewat jam selesainya (atau sudah lewat jam pulang sekolah, gunakan `jam_sekolah` schedule sebagai acuan akhir hari) → set `follow_up_needed = true`

### Endpoint manual trigger (untuk testing & recovery, sama pola seperti T040)

**POST `/teaching-sessions/run-auto-close`** — role `admin_jurnal`/`super_admin`, jalankan kedua logic di atas secara sinkron, return jumlah yang diproses

## JANGAN
- ❌ JANGAN hapus atau ubah `journal_entries`/`class_attendance_marks` yang sudah diisi guru saat auto-close — job ini HANYA mengubah `teaching_sessions.status`/`closed_at` dan `teacher_permits.follow_up_needed`, tidak menyentuh data jurnal
- ❌ JANGAN auto-close sesi yang `tanggal` bukan hari ini — job ini hanya untuk sesi hari berjalan, sesi masa lalu seharusnya sudah closed dari run sebelumnya (kalau ada yang terlewat karena job pernah mati, itu dianggap data historis yang tidak perlu dikoreksi retroaktif, bukan scope task ini)
- ❌ JANGAN buat auto-close menghapus kemampuan guru edit jurnal setelahnya — batas waktu edit jurnal adalah keputusan terpisah (masih open question di spec), task ini murni menutup STATUS sesi, bukan mengunci field jurnal

## Files
- **Buat:** `apps/api/src/teaching-sessions/auto-close.processor.ts` (BullMQ processor)
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts` — tambah method `autoCloseDueSessions()` dan `flagPermitsNeedingFollowUp()`
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` — tambah `POST /teaching-sessions/run-auto-close`

## Acceptance Criteria
- [ ] Sesi dengan `jam_selesai` sudah lewat & `started_at` terisi → `closed_at` terisi, `status = closed`
- [ ] Sesi dengan `jam_selesai` sudah lewat & `started_at` MASIH `null` (guru tidak pernah mulai) → tetap di-close (`closed_at` terisi), tidak dibiarkan menggantung
- [ ] Sesi yang `jam_selesai`-nya BELUM lewat → tidak disentuh job ini
- [ ] Permit izin 1 sesi spesifik, jam sesi belum lewat → `follow_up_needed` tetap `false`
- [ ] Permit izin 1 sesi spesifik, jam sesi sudah lewat, `submitted_at` masih null → `follow_up_needed` jadi `true`
- [ ] Permit izin 1 sesi spesifik yang SUDAH `submitted_at` terisi → `follow_up_needed` tidak pernah di-set true meski jam sudah lewat
- [ ] Trigger manual dari role selain admin_jurnal/super_admin → 403

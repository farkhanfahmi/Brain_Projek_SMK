# Task-CORE-012: Backend — Perluas autoCloseDueSessions() untuk Auto-Lock Kartu Siswa

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` §4. Dieksekusi oleh Claude Code.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Menyentuh job cron kritikal existing (`autoCloseDueSessions`, dipanggil job terjadwal + endpoint recovery manual) — perlu ketelitian tidak memperlambat/merusak job existing, plus REPLIKASI pola lock yang sudah ada (`lockGuruTanpaTapPulang`).

## 2. Konteks & Tujuan Utama

Baca `06-Features/desain-redesign-presensi-izin-keluar.md` §4. **Keputusan final user**: auto-lock kartu siswa dieksekusi SAAT SESI DITUTUP (bukan instant saat guru klik Alfa), dan diperluas dari `autoCloseDueSessions()` existing (`apps/api/src/teaching-sessions/teaching-sessions.service.ts`) — BUKAN job/cron terpisah.

**Depends on:** task-CORE-010 (field `keterangan` harus ada), task-CORE-011 (guru sudah bisa isi keterangan Alfa).

## 3. Langkah Eksekusi Detail

1. Di `TeachingSessionsService.autoCloseDueSessions()` — SETELAH loop `for (const session of due)` yang men-set `closedAt`/`status: closed` (baris ~652-657 existing), TAMBAHKAN langkah baru: untuk setiap sesi yang BARU di-close, cek `ClassAttendanceMark` dengan `status: tidak_ada_di_kelas` untuk sesi itu.

2. Untuk tiap mark Alfa yang ditemukan — cek apakah `Student` itu **tap gerbang** hari itu (`AttendanceRecord` where `studentId` + `tanggal` sesi) — kalau YA (kontradiksi: tap masuk tapi ditandai tidak ada di kelas), jalankan:
   - **Kunci kartu siswa**: `Student.update({ lockedAt: now, lockedReason: <keterangan dari ClassAttendanceMark>, lockedById: null })` — `lockedById: null` menandakan lock OTOMATIS sistem (REPLIKASI PERSIS pola `lockGuruTanpaTapPulang()` yang SUDAH ADA, baris ~771-821, gunakan sebagai referensi struktur kode).
   - **Idempotent** — kalau `Student.lockedAt` SUDAH terisi (mis. dari sebab lain), JANGAN timpa (SAMA pola `lockGuruTanpaTapPulang`, cek `lockedAt: null` di query kondisi).
   - **Catat ke `activityLog`** — action baru mis. `"student.lock_auto_alfa_kelas"`, `targetType: "student"`, snapshot before/after — REPLIKASI pola `activityLog.record()` di `lockGuruTanpaTapPulang()`.

3. **Tambahkan referensi ke sesi/mapel** di `lockedReason` atau field terpisah — pesan lock HARUS jelas menyebutkan konteks (mis. `"Tidak ada di kelas — <mapel>: <keterangan guru>"`) supaya piket tahu detail tanpa buka riwayat terpisah, KONSISTEN §4 desain ("supaya piket tahu alasan tanpa buka riwayat terpisah").

4. **Verifikasi tidak memperlambat job existing secara signifikan** — job ini dipanggil tiap 5 menit (cron) DAN via endpoint recovery manual (`POST /teaching-sessions/run-auto-close`) — pastikan query tambahan (cek marks Alfa + tap gerbang) di-scope HANYA ke sesi yang BARU di-close batch ini (`due` array yang sudah ada), BUKAN full-table-scan semua sesi historis.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts` — `autoCloseDueSessions()`

**Dilarang dilakukan:**
- Jangan buat job/cron terpisah — KEPUTUSAN EKSPLISIT user, perluas yang existing.
- Jangan timpa `lockedAt` kalau siswa SUDAH terkunci sebab lain — idempotent, sama pola existing.
- Jangan lock siswa yang Alfa TAPI TIDAK tap gerbang (itu Alfa "wajar" — siswa memang tidak masuk sekolah sama sekali, bukan kontradiksi) — HANYA lock untuk kasus tap-gerbang-tapi-alfa-di-kelas.

**Skenario kegagalan yang WAJIB ditangani:**
- Sesi di-close TAPI tidak ada mark Alfa sama sekali → tidak ada aksi tambahan, job selesai normal seperti sebelumnya.
- Multiple sesi di-close dalam 1 batch job (banyak sesi lewat jam bersamaan) → semua diproses, tidak ada yang terlewat.
- `Student` yang di-lock ternyata SUDAH di-lock manual sebelumnya (`lockedById` terisi non-null) → SKIP, jangan timpa reason yang sudah ada.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Sesi yang closed dengan mark Alfa (siswa tap gerbang tapi ditandai tidak ada di kelas) → kartu siswa terkunci otomatis
- [ ] `lockedReason` berisi keterangan guru + konteks mapel
- [ ] `lockedById: null` (menandakan lock sistem, konsisten pola existing)
- [ ] Idempotent — tidak menimpa lock existing
- [ ] Tercatat di `activityLog`
- [ ] Job existing (`autoCloseDueSessions`) tidak melambat signifikan / tidak regresi behavior lama

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 100 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: task-CORE-010, task-CORE-011 WAJIB selesai dulu

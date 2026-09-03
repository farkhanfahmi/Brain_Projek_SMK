# Task-CORE-023: Riwayat "Tidak Absen Pulang" + Auto-Lock 3x + Riwayat Catatan Siswa

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> **Depends on task-CORE-022** — task ini mendefinisikan model data + strike counter yang DIPANGGIL oleh job cron yang dibuat task-CORE-022. Kerjakan SETELAH atau BERSAMAAN task-CORE-022 (koordinasi wajib, jangan kerjakan terisolasi).

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium-high
**Alasan pemilihan:** Model data baru + migration + logic strike counter (REPLIKASI pola `lateStrikeResetAt`/`applyLateStrikeLock()` existing tapi domain berbeda) + integrasi ke Riwayat Catatan siswa (komponen shared) — butuh ketelitian supaya tidak duplikasi/bentrok dengan mekanisme lock 2x-terlambat yang sudah ada.

## 2. Konteks & Tujuan Utama

Melanjutkan task-CORE-022: saat job cron auto-carry berjalan (baris yang tidak diklarifikasi piket sampai ganti hari), baris itu **hilang dari daftar aktif "Tidak Absen Pulang"** TAPI harus **tercatat permanen** sebagai kejadian di riwayat siswa, DAN terakumulasi jadi strike counter — **3x kejadian → siswa otomatis terkunci** dengan alasan spesifik `"3X Tidak Absen Pulang"`.

**Keputusan yang sudah disepakati user (2026-09-03):**
1. Kejadian "tidak absen pulang" yang tidak diklarifikasi masuk ke **Riwayat Catatan siswa** (komponen `riwayat-catatan-table.tsx`, SAMA seperti terlambat/izin/lock — accessible dari halaman detail siswa admin & wali kelas).
2. **3x kejadian → auto-lock**, `lockedReason`: **"3X Tidak Absen Pulang"** (format spesifik, bukan generik) — REPLIKASI pola `applyLateStrikeLock()` (`attendance.service.ts:427-520`) yang sudah ada untuk lock 2x-terlambat, TAPI domain/counter TERPISAH (Student hanya punya 1 `lockedAt`, jadi kalau siswa SUDAH terkunci sebab lain, lock baru ini TIDAK menimpa — sama prinsip `if (!student || student.lockedAt !== null) return false`).
3. **"Data akan hilang ketika sudah 3x dan dibukakan kunci"** — user klarifikasi: maksudnya adalah **list yang tampil di halaman Wali Kelas > Grafik** (lihat task-WEB-027) — begitu siswa di-unlock oleh piket, akumulasi kejadian "menuju 3x" itu hilang dari list AKTIF wali kelas (strike counter reset ke 0). **Riwayat Catatan siswa TETAP permanen** (tidak pernah dihapus, untuk audit historis) — reset HANYA memengaruhi PERHITUNGAN strike berikutnya, bukan riwayat lama.

**Depends on:** task-CORE-022 (job cron trigger + fondasi query kandidat baris expired).

## 3. Langkah Eksekusi Detail

### A. Skema Data Baru

1. **Model baru** `TidakAbsenPulangCatatan` (atau nama serupa, VERIFIKASI SAAT IMPLEMENTASI konvensi penamaan proyek — cek pola nama model lain yang mirip) di `schema.prisma`:
   ```prisma
   model TidakAbsenPulangCatatan {
     id                Int      @id @default(autoincrement())
     studentId         Int      @map("student_id")
     attendanceRecordId Int     @unique @map("attendance_record_id") // 1:1 dgn AttendanceRecord asal
     tanggal           DateTime @db.Date
     createdAt         DateTime @default(now()) @map("created_at")
     // Reset tracking — REPLIKASI pola Student.lateStrikeResetAt (T154): kejadian SEBELUM
     // resetAt siswa tidak dihitung ke strike counter aktif, TAPI baris TETAP ada permanen.
     student           Student  @relation(fields: [studentId], references: [id])
     attendanceRecord  AttendanceRecord @relation(fields: [attendanceRecordId], references: [id])
     @@index([studentId])
     @@map("tidak_absen_pulang_catatan")
   }
   ```
   **VERIFIKASI SAAT IMPLEMENTASI**: apakah field reset tracking taruh di model baru ini (kolom `dihitungSetelah`-style per baris) ATAU reuse pola `Student.lateStrikeResetAt` (tambah field BARU `tidakAbsenPulangResetAt` ke model `Student`, TERPISAH dari `lateStrikeResetAt` yang sudah dipakai domain terlambat — JANGAN campur 2 counter berbeda ke 1 field yang sama). REKOMENDASI: tambah field baru ke `Student` (`tidakAbsenPulangResetAt DateTime?`) — REPLIKASI PERSIS pola existing, konsisten dengan cara proyek ini menangani strike-per-domain.

2. **Migration** — additif murni (`CREATE TABLE` + `ADD COLUMN` di `Student`). Baca `migration.sql` untuk konfirmasi sebelum apply, sesuai aturan CLAUDE.md.

### B. Service — Catat Kejadian + Hitung Strike + Auto-Lock

3. **Method baru** `AttendanceService.catatTidakAbsenPulangExpired(attendanceRecordId)` (dipanggil dari job cron task-CORE-022, per baris yang expired):
   - `create()` 1 baris `TidakAbsenPulangCatatan` (idempotent — cek `attendanceRecordId` belum ada dulu, constraint `@unique` di schema jadi pengaman kedua).
   - Hitung strike aktif: `count()` baris `TidakAbsenPulangCatatan` untuk `studentId` itu, `tanggal >= student.tidakAbsenPulangResetAt` (kalau null, hitung dari awal/semua) — REPLIKASI PERSIS pola `applyLateStrikeLock()` (baris 438-465) untuk konsistensi logic reset-window.
   - Kalau strike count `>= 3` DAN `student.lockedAt === null` (belum terkunci sebab lain) → **lock otomatis**, `lockedReason: "3X Tidak Absen Pulang"`, `lockedById: null` (lock sistem), REPLIKASI PERSIS struktur `$transaction` (student.update + activityLog.record dalam 1 transaksi) dari `applyLateStrikeLock()` baris 482-504, TERMASUK broadcast Socket.IO `attendanceGateway.broadcastStudentLock()` (baris 508-517).
   - `activityLog.record()` — action baru `"student.lock_tidak_absen_pulang"` (BEDA dari `"student.lock"` yang dipakai lock-terlambat, supaya bisa dibedakan di audit trail/Riwayat Catatan).

4. **Reset saat unlock** — cari method `unlock()` di `StudentsService` (`students.service.ts:570`) — VERIFIKASI SAAT IMPLEMENTASI apakah perlu percabangan berdasar `lockedReason` (mis. kalau `lockedReason` mengandung "3X Tidak Absen Pulang" → set `tidakAbsenPulangResetAt = new Date()` saat unlock, REPLIKASI pola persis bagaimana `lateStrikeResetAt` di-set saat unlock lock-terlambat — cari kode itu dulu sebagai referensi, KEMUNGKINAN ada di `unlock()` yang sama dengan percabangan serupa untuk `lateStrikeResetAt`).

### C. Riwayat Catatan Siswa — Tambah Jenis Baru

5. **`RiwayatCatatanEntry` type** (`apps/web/src/lib/core-types.ts:415-437`) — tambah varian baru:
   ```ts
   | { jenis: "tidak_absen_pulang"; tanggal: string }
   ```
6. **Backend endpoint** yang men-generate `RiwayatCatatanEntry[]` (cari controller/service yang assemble data ini untuk `GET /attendance/students/:id/riwayat-catatan`, VERIFIKASI SAAT IMPLEMENTASI lokasi persis — kemungkinan di `attendance.service.ts` atau `students.service.ts`) — tambahkan query `TidakAbsenPulangCatatan` untuk siswa itu, map ke bentuk `RiwayatCatatanEntry` baru, gabung ke array yang sudah ada (union semua jenis).
7. **`riwayat-catatan-table.tsx`** — tambah label (`RIWAYAT_LABEL`), badge class (`RIWAYAT_BADGE_CLASS` — REKOMENDASI warna beda dari "terkunci"/"terlambat" existing, mis. `bg-status-processing-bg`/token yang belum dipakai kombinasi ini, VERIFIKASI SAAT IMPLEMENTASI token yang tersedia), ikon (`RIWAYAT_ICON`, mis. `LogOut`/`AlertTriangle` — cek `lucide-react` yang belum dipakai di file itu) untuk jenis `tidak_absen_pulang` baru. Kolom "Detail" → tampilkan teks statis sederhana (mis. "Tidak tap pulang, tidak diklarifikasi piket") atau kosong "-" (VERIFIKASI SAAT IMPLEMENTASI apakah perlu detail tambahan).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/prisma/schema.prisma` — model `TidakAbsenPulangCatatan` baru + field `Student.tidakAbsenPulangResetAt`
- **Buat:** migration baru
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` — method `catatTidakAbsenPulangExpired()`
- **Modifikasi:** `apps/api/src/core/students/students.service.ts` — `unlock()` percabangan reset
- **Modifikasi:** endpoint riwayat-catatan (lokasi VERIFIKASI SAAT IMPLEMENTASI)
- **Modifikasi:** `apps/web/src/lib/core-types.ts` — `RiwayatCatatanEntry`
- **Modifikasi:** `apps/web/src/components/riwayat-catatan-table.tsx` — jenis baru
- **Jangan sentuh:** `applyLateStrikeLock()`/`lateStrikeResetAt` (domain TERPISAH, hanya dijadikan REFERENSI pola, TIDAK digabung/dicampur dengan counter tidak-absen-pulang).

**Dilarang dilakukan:**
- Jangan campur counter "3x terlambat" dengan "3x tidak absen pulang" — dua domain lock terpisah, `Student` cuma punya 1 `lockedAt` (siapa pun yang lock duluan menang, yang kedua di-skip via `lockedAt !== null` check) tapi COUNTER-nya harus independen (field terpisah, JANGAN reuse `lateStrikeResetAt` untuk domain ini).
- Jangan hapus baris `TidakAbsenPulangCatatan` lama saat unlock — riwayat permanen, HANYA counter/window yang di-reset.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: siswa sudah terkunci sebab LAIN (mis. 2x terlambat) SAAT strike tidak-absen-pulang mencapai 3x → TIDAK dikunci ulang/ditimpa (`lockedAt !== null` check mencegah), TAPI baris `TidakAbsenPulangCatatan` ke-3 TETAP tercatat (riwayat tetap lengkap, cuma efek lock-nya yang di-skip).
- Kondisi: siswa di-unlock, lalu beberapa hari kemudian kena strike tidak-absen-pulang lagi (hitungan mulai dari 0 lagi pasca-reset) — pastikan window `tidakAbsenPulangResetAt` benar-benar exclude kejadian SEBELUM reset dari perhitungan strike aktif (REPLIKASI logic `resetDateFloor` di `applyLateStrikeLock()` baris 454-461).
- Kondisi: job cron task-CORE-022 memanggil `catatTidakAbsenPulangExpired()` 2x untuk `attendanceRecordId` yang sama (retry/race) → idempotent via constraint `@unique` pada `attendanceRecordId`, catch P2002 dan skip diam-diam (bukan error, ini expected retry-safe behavior).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Kejadian "tidak absen pulang" yang tidak diklarifikasi piket tercatat permanen ke tabel baru `TidakAbsenPulangCatatan`.
- [ ] Kejadian ini muncul di Riwayat Catatan siswa (admin & wali kelas, via komponen shared).
- [ ] Siswa dengan 3x kejadian (dalam window aktif) otomatis terkunci, `lockedReason: "3X Tidak Absen Pulang"`.
- [ ] Siswa yang sudah terkunci sebab lain TIDAK ditimpa lock baru, tapi kejadian tetap tercatat.
- [ ] Unlock oleh piket me-reset window strike (kejadian lama tetap di riwayat, hitungan berikutnya mulai dari 0).
- [ ] Test unit baru untuk `catatTidakAbsenPulangExpired()` (idempotent, strike<3 no-lock, strike=3 lock, sudah-terkunci-sebab-lain skip-lock, reset-window).
- [ ] Full test suite lulus tanpa regresi, typecheck bersih.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (konsisten pola `applyLateStrikeLock`)
- [ ] Dependency: task-CORE-022 WAJIB selesai/dikoordinasikan dulu (job cron trigger)

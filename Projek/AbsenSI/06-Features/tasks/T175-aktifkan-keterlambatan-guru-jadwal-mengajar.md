# T175 — API: Aktifkan Logic Keterlambatan Guru Berdasarkan Jadwal Mengajar

## Depends on
Tidak ada dependency teknis (jadwal mengajar `Schedule`/`TeachingSession` SUDAH ADA). **WAJIB dikerjakan SEBELUM T176** (Rekap Kehadiran Guru) — tanpa task ini, kolom "Terlambat" di rekap guru akan selalu 0 untuk semua guru, membuat fitur itu tampak rusak sejak awal.

## Objective
Guru yang tap gerbang SETELAH jam mulai jadwal mengajar pertamanya hari itu (+ toleransi) → status `terlambat` (bukan selalu `hadir` seperti sekarang). Guru tanpa jadwal mengajar hari itu TETAP `hadir` (tidak ada patokan jam untuk dibandingkan).

## Context — Temuan Riset (2026-08-14)

`determineStatus()` di `apps/api/src/attendance/attendance.service.ts:769-792` punya percabangan eksplisit:
- Kalau `card.studentId` truthy → resolve jadwal via `schedulesService.resolveJamMasuk(kelasId, dayOfWeek)`, bandingkan `scannedAt` vs deadline (jam masuk + toleransi) → `terlambat`/`hadir`.
- Kalau kartu GURU (`card.studentId` falsy) → **langsung `return AttendanceStatus.hadir` tanpa pengecekan apa pun** (baris 791). Komentar developer eksplisit (baris 787-790): "jadwal mengajar guru belum diinput user, jadi SEMENTARA guru tidak pernah dianggap terlambat... JANGAN pakai jam_sekolah sebagai patokan pengganti."

Prasyarat komentar itu ("jadwal mengajar guru belum diinput") **SUDAH TERPENUHI** — `Schedule`/`TeachingSession` sudah berfungsi penuh (dipakai jurnal mengajar, dst). User eksplisit konfirmasi (2026-08-14): aktifkan logic ini sekarang.

**PERINGATAN EKSPLISIT dari komentar lama**: "JANGAN pakai jam_sekolah sebagai patokan pengganti" — artinya JANGAN reuse `Schedule type: jam_sekolah` (jam masuk sekolah 3-lapis siswa, T145) sebagai jam patokan guru. Patokan HARUS jadwal MENGAJAR guru itu sendiri (`Schedule type: jam_mengajar` milik `teacherId` itu, ATAU `TeachingSession` miliknya hari itu).

## Spec Detail

### 1. Backend — logic baru di `determineStatus()`

- Ganti cabang `else` (guru) di `attendance.service.ts:786-791` — SEBELUM langsung `return hadir`, cari jadwal mengajar PERTAMA milik `teacherId` itu di hari (`dayOfWeek`) tap ini terjadi:
  - Query `Schedule` (type `jam_mengajar`) milik `teacherId` = kartu ini, `hari` = hari tap ini, `orderBy: jamMulai asc`, ambil yang PERTAMA (kalau ada beberapa jadwal hari itu, jam paling pagi jadi patokan telat).
  - **VERIFIKASI dulu**: apakah resolusi jam mulai jadwal mengajar guru butuh lookup ke `JamPelajaranOption` aktif (T158, KALAU sudah dikerjakan duluan dan Opsi A dipilih — `Schedule` simpan `jamKeAwal` bukan `jamMulai` string) ATAU masih `Schedule.jamMulai` string langsung (kalau T158 belum/Opsi B) — SESUAIKAN dengan kondisi kode SAAT task ini dieksekusi, JANGAN asumsi salah satu tanpa cek dulu.
  - Kalau TIDAK ADA jadwal mengajar guru itu di hari tersebut → `return AttendanceStatus.hadir` (PERILAKU LAMA dipertahankan untuk kasus ini — guru tanpa jadwal hari itu tidak punya patokan telat, WAJAR dianggap hadir biasa, BUKAN alfa/masalah).
  - Kalau ADA jadwal → bandingkan `scannedAt` vs (jam mulai jadwal + toleransi). **Toleransi guru dari mana?** — VERIFIKASI apakah ada toleransi keterlambatan KHUSUS guru (beda dari toleransi siswa `(admin-jurnal)/admin-jurnal/toleransi/`) — KEMUNGKINAN BESAR belum ada, REKOMENDASI reuse config toleransi yang sama (1 nilai global untuk semua, siswa maupun guru) KECUALI ditemukan sudah ada pemisahan eksplisit di kode. TANYAKAN ke user kalau ragu, JANGAN putuskan sepihak nilai toleransi guru berbeda dari siswa tanpa konfirmasi.
  - Lewat deadline → `terlambat`; sebelum/pas → `hadir`.

### 2. Lock 2x-terlambat (T153/T154) — TETAP TIDAK BERLAKU untuk guru

- `applyLateStrikeLock` (dipicu di `tap()` sekitar baris 251) SAAT INI eksplisit dibatasi `status === terlambat && card.studentId` — task ini **TIDAK MENGUBAH** batasan ini. Guru yang sekarang BISA berstatus `terlambat` TETAP TIDAK memicu lock otomatis (fitur lock itu murni siswa, di luar scope task ini, TIDAK ADA instruksi user untuk memperluasnya ke guru).

### 3. Method lain yang punya logic serupa — VERIFIKASI konsistensi

- Riset menyebut ada method lain (`myHistory` sekitar baris 424) dengan komentar "PERSIS SAMA dengan determineStatus()" — BACA method itu, PASTIKAN kalau memang duplikasi logic untuk guru, method itu JUGA diperbarui konsisten (supaya riwayat guru sendiri `/guru/riwayat` dan rekap admin tidak menunjukkan status berbeda untuk kejadian tap yang sama).

## Edge Cases
- Guru dengan BEBERAPA jadwal mengajar di hari yang sama (misal 3 kelas berbeda) — patokan telat pakai jadwal PALING PAGI (jam pertama hari itu), BUKAN rata-rata/jadwal terakhir.
- Guru piket yang punya jadwal piket (`PiketSchedule`, modul terpisah) TAPI TIDAK ADA jadwal mengajar (`Schedule` type jam_mengajar) hari itu — DI LUAR SCOPE task ini (task ini murni soal jadwal MENGAJAR), guru piket murni tanpa jadwal mengajar tetap `hadir` tanpa patokan telat (PERILAKU SAMA seperti guru lain tanpa jadwal).
- Perubahan ini RETROAKTIF secara logic (kode baru berlaku untuk tap BARU sejak deploy) — data `AttendanceRecord` LAMA (guru yang sudah tercatat `hadir` padahal sebenarnya telat menurut logic baru) TIDAK di-backfill/diubah — task ini HANYA mengubah perilaku KE DEPAN, TIDAK mengoreksi histori.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`determineStatus()`, dan `myHistory`/method serupa kalau terkonfirmasi duplikasi logic).
- **Jangan sentuh:** logic keterlambatan siswa (cabang `if(studentId)`, TIDAK diubah), `applyLateStrikeLock` (batasan `card.studentId` TETAP, TIDAK diperluas ke guru).

## Acceptance Criteria
- [ ] Guru dengan jadwal mengajar hari itu, tap SETELAH jam mulai+toleransi → status `terlambat`.
- [ ] Guru dengan jadwal mengajar hari itu, tap SEBELUM/PAS jam mulai → status `hadir`.
- [ ] Guru TANPA jadwal mengajar hari itu → tetap `hadir` (perilaku lama dipertahankan untuk kasus ini).
- [ ] Guru dengan beberapa jadwal hari itu → patokan jadwal PALING PAGI.
- [ ] Lock 2x-terlambat TIDAK terpicu untuk guru (verified, batasan `studentId` masih ada).
- [ ] Method lain dengan logic duplikat (`myHistory`) konsisten dengan `determineStatus()`.
- [ ] Build + type-check hijau, jest existing tetap pass, jest baru untuk skenario guru terlambat/tidak ditambahkan.

## Validasi Claudian
- [ ] Konfirmasi sumber toleransi keterlambatan guru (reuse config siswa existing, ATAU ditemukan sudah ada pemisahan — laporkan yang mana, jangan putuskan sepihak kalau ambigu).
- [ ] Konfirmasi resolusi jam mulai jadwal mengajar SESUAI kondisi T158 saat task ini dieksekusi (jamKe vs jamMulai string langsung — cek dulu, jangan asumsi).
- [ ] Konfirmasi perubahan ini TIDAK retroaktif ke data lama (hanya tap baru sejak deploy).

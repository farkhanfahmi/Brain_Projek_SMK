# Task-CORE-024 / KIOSK-002: Validasi Tap Pulang Sesuai Jadwal Kelas (Recreate T101, Fondasi Sudah Siap)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — recreate T101 (lama, BLOCKED sejak 2026-07-30) setelah audit kode 2026-09-03 membuktikan fondasi jadwal riil (`JadwalSlot`/`OpsiJadwal`/`resolveJamSesi()`) SUDAH ADA dan SUDAH dipakai fitur lain (`PermitsService.syncPermitToAttendance()`, task-CORE-015). **Verifikasi dilakukan MURNI via analisa alur kode (bukan query data production)** — keputusan eksplisit user 2026-09-03: "kalau alur kode sesuai maka data pastinya juga akan mengikuti". T101 lama (`06-Features/tasks/T101-validasi-jam-pulang-jadwal-kelas.md`) TIDAK dihapus (arsip histori), task ini MENGGANTIKANNYA.
> **[REVISI 2026-09-03, sesi lanjutan]** Draft AWAL task ini (field `Permit.untukPulangAwal` + endpoint pre-otorisasi tap) SUDAH DIHAPUS TOTAL — digantikan mekanisme LEBIH SEDERHANA di **task-CORE-025** (piket set `waktuPulang` langsung saat buat surat izin "Tidak Kembali", siswa TIDAK PERNAH tap untuk kasus itu). Task ini SEKARANG MURNI validasi tap tanpa pengecualian/bypass apa pun.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium-high
**Alasan pemilihan:** Mengubah alur `tap()` INTI yang dipakai SETIAP siswa SETIAP hari (risiko tinggi kalau salah) — TAPI scope sudah lebih sempit dari draft awal (turun dari `high`) setelah bagian pre-otorisasi dipindah ke task-CORE-025: task ini murni validasi tap + pesan kiosk baru, tidak ada endpoint/UI piket baru lagi.

## 2. Konteks & Tujuan Utama

**Objective**: Siswa TIDAK BOLEH tap pulang sebelum jam pelajaran TERAKHIR kelasnya selesai hari itu. Tap sebelum waktunya **DITOLAK** dengan pesan jelas ("Belum waktunya pulang! Pulang pukul HH:mm"), BUKAN diterima tanpa syarat seperti sekarang (`tap()`, `attendance.service.ts:390-394` — tap ke-2 mengisi `waktuPulang` TANPA pengecekan waktu apa pun, dikonfirmasi baca kode langsung).

**Kenapa BISA dieksekusi sekarang (fondasi yang tidak ada saat T101 ditulis 2026-07-30)**:
- `JadwalSlot`+`OpsiJadwal` (T204+) — assignment jadwal riil per Kelas+Mapel+Guru+JamKe, MENGGANTIKAN `Schedule type=jam_mengajar` lama yang cuma 6 baris dummy saat T101 ditulis.
- `TeachingSessionsService.resolveJamSesi()` (T209, dibuat PUBLIC di task-CORE-015) — resolve `jamMulai`/`jamSelesai` dari `JadwalSlot` + `AlokasiWaktuSlot`, SUDAH TERBUKTI DIPAKAI PRODUCTION-READY oleh `PermitsService.syncPermitToAttendance()` (task-CORE-015, sudah selesai+dites 9 skenario) untuk kasus SERUPA (cari sesi yang overlap rentang waktu izin).
- **Metodologi verifikasi task ini**: KARENA `resolveJamSesi()` SUDAH dipakai fitur lain yang sudah production-ready untuk tujuan resolusi jam yang SAMA PERSIS dibutuhkan task ini, alur kode TERBUKTI benar secara struktural — TIDAK PERLU query data production terpisah (kalau `JadwalSlot` kosong/tidak lengkap untuk kelas tertentu, method ini AKAN return `null` secara graceful, ditangani via fallback di langkah 2, BUKAN crash) — inilah dasar keputusan user "kalau alur kode sesuai, data akan mengikuti".

**Kenapa TIDAK ADA LAGI mekanisme bypass/pre-otorisasi di task ini** — dengan task-CORE-025 (mekanisme "Tidak Kembali" tanpa tap), SEMUA siswa yang izinnya sudah diproses piket (baik Sub-alur A "akan kembali" MAUPUN mekanisme baru "tidak kembali") **TIDAK PERNAH tap gerbang** untuk skenario itu (`waktuPulang` diisi piket langsung, bukan dari tap). Validasi di task ini jadi MURNI untuk menolak tap tanpa izin apa pun yang terjadi sebelum jadwal selesai (siswa nekat pulang sendiri tanpa lapor piket) — tidak perlu tahu-menahu soal Permit sama sekali.

**Depends on:** Tidak ada dependency teknis — TAPI REKOMENDASI kerjakan SETELAH task-CORE-025 (urutan logis: mekanisme "tidak kembali tanpa tap" dulu, supaya saat task ini dites, sudah ada jalur legal yang tidak butuh tap — mengurangi tumpang tindih testing). **VERIFIKASI SAAT IMPLEMENTASI WAJIB** (bagian 4) sebagai pengganti verifikasi data production yang biasanya dilakukan Hermes via SSH.

## 3. Langkah Eksekusi Detail

### A. Backend — Method Resolve Jam Pulang Kelas + Validasi di `tap()`

1. **Method baru** `resolveJamPulangKelas(kelasId, tanggal): Promise<string | null>` (lokasi: `AttendanceService`, dekat `determineStatus()` — REKOMENDASI, logic-nya serupa "cari 1 kelas 1 tanggal 1 jam kunci", VERIFIKASI SAAT IMPLEMENTASI kalau ada lokasi lebih konsisten):
   - Cari SEMUA `JadwalSlot` untuk `kelasId` + `hari` (dayOfWeek dari `tanggal`, SAMA konversi `now.getDay() + 1` yang sudah dipakai di `piketBoard()` baris 731) — **VERIFIKASI SAAT IMPLEMENTASI WAJIB**: query ini perlu tahu `OpsiJadwal` mana yang AKTIF untuk tanggal itu (REPLIKASI PERSIS logic resolusi `OpsiJadwal isActive` + mode blok/normal yang SUDAH ADA di `TeachingSessionsService` — dikutip di komentar `resolveJamSesi()` baris 152-160, method itu SENDIRI TIDAK melakukan filter isActive, method BARU ini WAJIB replikasi logic pemilihan OpsiJadwal aktif dari method LAIN di `TeachingSessionsService` yang sudah punya logic itu — BACA method itu dulu, JANGAN tulis ulang dari nol tanpa referensi).
   - Dari kandidat `JadwalSlot` yang valid, ambil yang `jamKe`/`jamKeAkhirRentang` **TERBESAR** (representasi "mapel terakhir").
   - Resolve jam selesai slot itu via `TeachingSessionsService.resolveJamSesi()` (REUSE, jangan tulis ulang resolusi `AlokasiWaktuSlot`).
   - **Fallback**: kalau TIDAK ADA `JadwalSlot` sama sekali untuk kelas+hari itu → fallback ke `SchedulesService.resolveJamMasuk()` (`jamSelesai`-nya, field SUDAH ADA di return type `JamMasukResolved`).
   - Kalau KEDUANYA null → return `null` (caller di `tap()` WAJIB treat sebagai "tidak ada jadwal untuk kelas ini" → **PERILAKU FALLBACK AMAN: LOLOSKAN tap tanpa validasi** — SAMA PERSIS filosofi existing `determineStatus()` baris 1281 `if (!jamMasuk) return AttendanceStatus.hadir` — konsisten "tidak ada jadwal = tidak ada aturan yang bisa dilanggar", BUKAN block semua tap untuk kelas itu).

2. **Ubah `tap()`** (`attendance.service.ts`, cabang `else` baris 367-396, tap KE-2/PULANG untuk **SISWA SAJA** — `card.studentId`, guru **TIDAK** kena validasi ini, KONSISTEN prinsip existing "guru tidak terikat jam pelajaran siswa"):
   - SEBELUM melakukan `attendanceRecord.update()` (baris 390), tambahkan blok baru:
     a. Cek siswa PKL aktif (`pklRecords: { some: { endedAt: null } }`) → SKIP validasi total, lanjut `update()` normal (siswa PKL tidak terikat jadwal kelas asal, sudah exclude dari board juga — konsisten pola `piketBoard()` baris 664).
     b. Kalau siswa tidak punya `kelasId` (belum di-plot) → SKIP validasi, lanjut normal (KONSISTEN pola `determineStatus()` baris 1278).
     c. Panggil `resolveJamPulangKelas(card.student!.kelasId, today)`.
     d. Kalau jam pulang kelas RESOLVED dan `effectiveTime < jamPulangResolved` (belum waktunya) → **TOLAK** tap: `logTapEvent(..., TapResult.rejected_belum_waktunya_pulang, ...)`, return `{ result: TapResult.rejected_belum_waktunya_pulang, message: `Belum waktunya pulang! Pulang pukul ${jamPulangResolved}` }` — **JANGAN** panggil `attendanceRecord.update()` sama sekali (tap ditolak, `waktuPulang` TIDAK berubah).
     e. Kalau `effectiveTime >= jamPulangResolved` ATAU `resolveJamPulangKelas()` return `null` → lanjut seperti sekarang (`update()` normal).

3. **`TapResult` enum** (`schema.prisma:988-997`) — tambah value baru: `rejected_belum_waktunya_pulang` — migration additif (MENAMBAH value enum di MySQL, VERIFIKASI SAAT IMPLEMENTASI ini tidak dianggap destruktif oleh Prisma — biasanya aman untuk MENAMBAH value, BUKAN redefinisi/hapus value existing).

### B. Kiosk — Pesan Baru

4. **`apps/kiosk/src/components/feedback-screen.tsx`** — varian baru untuk `rejected_belum_waktunya_pulang` — REPLIKASI struktur varian `rejected_*` existing (styling/animasi konsisten), tampilkan `message` dari response API (sudah berisi jam pulang yang benar, format "Belum waktunya pulang! Pulang pukul HH:mm").

5. **`apps/kiosk/src/lib/tap-client.ts`** — VERIFIKASI SAAT IMPLEMENTASI apakah `rejected_belum_waktunya_pulang` perlu masuk `PERMANENT_FAILURE_RESULTS` (gagal permanen, jangan retry offline-sync) — REKOMENDASI: JANGAN masuk situ, hasil ini bisa berubah (siswa boleh tap lagi SETELAH jam pelajaran selesai) — SAMA pola `rejected_locked`/`rejected_inactive` yang sengaja dibiarkan "pending" untuk diretry (komentar baris 149-150 existing).

## 4. Batasan & Penanganan Kasus Khusus

**⚠️ PENGGANTI VERIFIKASI DATA PRODUCTION (keputusan eksplisit user 2026-09-03)**: task ini TIDAK diverifikasi lewat query SSH ke database production — verifikasi dilakukan MURNI via analisa alur kode: `resolveJamSesi()` SUDAH dipakai production-ready oleh task-CORE-015 (9 test lulus, sudah di-deploy), method itu sendiri SUDAH menangani kasus `JadwalSlot`/`OpsiJadwal` tidak ditemukan secara graceful (`return null`). Fallback di langkah 2e ("resolve null = loloskan tap tanpa validasi") memastikan KALAU pun ada kelas dengan data jadwal kosong/tidak lengkap di production, task ini TIDAK memblokir siswa kelas itu secara keliru. **Claude Code TIDAK PERLU dan TIDAK BISA verifikasi data production sendiri** (lihat `06-Features/akses-data-production.md`).

**Files:**
- **Modifikasi:** `apps/api/prisma/schema.prisma` — `TapResult` enum
- **Buat:** migration baru
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` — `tap()`, method baru `resolveJamPulangKelas()`
- **Modifikasi:** `apps/kiosk/src/components/feedback-screen.tsx` — varian pesan baru
- **Modifikasi:** `apps/kiosk/src/lib/tap-client.ts` — VERIFIKASI retry policy (langkah 5)
- **Jangan sentuh:** `PermitsService` (task ini TIDAK butuh tahu-menahu soal Permit sama sekali — beda dari draft awal), `confirmIzinPulang()` (Sub-alur B lama TETAP ada sebagai fallback manual, TIDAK dihapus/disentuh).

**Dilarang dilakukan:**
- Jangan terapkan validasi jam-pulang ini ke tap GURU — guru tidak terikat jadwal pelajaran siswa (`card.teacherId`, cabang existing berbeda, JANGAN disentuh).
- Jangan block tap sama sekali untuk kelas tanpa data jadwal — WAJIB fail-safe "loloskan" (lihat catatan verifikasi di atas).
- Jangan tambahkan logic apa pun yang membaca tabel `Permit` di dalam `tap()` — task ini SENGAJA dipisah dari domain izin (task-CORE-025 menangani semua kasus "boleh pulang lebih awal" TANPA melibatkan tap sama sekali).

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: siswa TANPA izin apa pun tap gerbang sebelum jam pelajaran terakhir selesai → tap **DITOLAK**, pesan jelas jam yang benar.
- Kondisi: siswa dengan izin YANG SUDAH DIPROSES PIKET (Sub-alur A atau mekanisme "Tidak Kembali" task-CORE-025) → siswa-siswa ini **TIDAK PERNAH tap** untuk skenario itu (`waktuPulang` sudah diisi piket duluan) — TIDAK ADA interaksi dengan validasi task ini sama sekali, dijamin oleh desain task-CORE-025 (bukan sesuatu yang perlu ditangani KHUSUS di sini).
- Kondisi: siswa PKL aktif tap (jarang) → SKIP validasi total.
- Kondisi: hari Sabtu (hadir dicatat normal, tidak masuk alfa) → validasi jam-pulang TETAP berlaku SAMA seperti hari biasa (aturan proyek: Sabtu cuma dikecualikan dari perhitungan ALFA, bukan dari SELURUH aturan kehadiran) — pastikan `resolveJamPulangKelas()` resolve dengan benar untuk `hari=7` (Sabtu) kalau ada jadwalnya, atau fallback aman kalau tidak ada.
- Kondisi: mode blok (`OpsiJadwal.mode='blok'`) — jadwal berbeda per minggu generate (`OpsiJadwalMingguGenerate`) → `resolveJamPulangKelas()` WAJIB replikasi logic resolusi mode blok yang SAMA dengan method existing di `TeachingSessionsService` (lihat langkah 1, WAJIB baca method itu dulu sebelum implementasi, JANGAN asumsikan mode normal saja).
- Kondisi: hari libur (`SchoolHoliday`) → `JadwalSlot` tidak akan match (jadwal hanya berlaku hari sekolah), fallback tetap fail-safe (loloskan tap, tidak crash).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Tap pulang siswa TANPA izin apa pun, SEBELUM jam pelajaran terakhir kelasnya selesai → DITOLAK, pesan "Belum waktunya pulang! Pulang pukul HH:mm" tampil di kiosk.
- [ ] Tap pulang siswa SETELAH jam pelajaran selesai → DITERIMA normal (perilaku sekarang tidak berubah).
- [ ] Kelas tanpa `JadwalSlot`/`Schedule jam_sekolah` sama sekali → tap TETAP diterima (fail-safe, tidak memblokir).
- [ ] Siswa PKL aktif — validasi jam-pulang di-skip.
- [ ] Siswa tanpa `kelasId` — validasi jam-pulang di-skip.
- [ ] Guru TIDAK terdampak validasi ini sama sekali.
- [ ] Kiosk tampilkan varian pesan baru untuk `rejected_belum_waktunya_pulang`.
- [ ] Test unit baru mencakup SEMUA skenario kegagalan di bagian 4.
- [ ] Full test suite lulus tanpa regresi, typecheck bersih di api+kiosk.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 200 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada — KONFIRMASI ULANG saat implementasi: `resolveJamSesi()`/`OpsiJadwal` aktif-selection logic dibaca dan direplikasi BENAR (bukan diasumsikan), ini persyaratan paling kritikal task ini
- [ ] Dependency: tidak ada, TAPI risiko TINGGI (mengubah alur tap inti) — REKOMENDASI KUAT pengujian menyeluruh manual sebelum dianggap benar-benar selesai, bukan cuma unit test lulus. REKOMENDASI kerjakan setelah task-CORE-025.

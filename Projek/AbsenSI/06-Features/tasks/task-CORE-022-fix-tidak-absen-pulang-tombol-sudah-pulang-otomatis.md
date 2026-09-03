# Task-CORE-022 / WEB-026: Fix Bug "Tidak Absen Pulang" Muncul Lagi + Tombol "Sudah Pulang" (Jam Otomatis dari Jadwal)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> Bagian dari perombakan besar "Tidak Absen Pulang" — 3 task saling terkait: task ini (fix bug + tombol), task-CORE-023 (auto-lock 3x + riwayat), task-WEB-027 (card Wali Kelas). Dipecah karena masing-masing punya area/effort berbeda, TAPI **task-CORE-023 depends on task ini** (butuh mekanisme "hilang dari daftar" yang benar dulu sebelum bisa hitung strike akurat).

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Bug fix jelas + fitur baru (resolve jam dari jadwal riil, REUSE `resolveJamSesi()` yang sudah ada) — bukan riset baru, tapi menyentuh query inti dashboard piket yang dipakai tiap hari, perlu ketelitian test.

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) + laporan user menemukan **bug nyata**: klik "Sudah Pulang Semua" di card "Tidak Absen Pulang" — setelah halaman di-refresh, baris yang sudah diklarifikasi **muncul lagi**.

**Root cause terverifikasi** (baca kode langsung):
- `konfirmasiPulangRetroaktifBulk()` (`attendance.service.ts:898-936`) SELALU mengirim `waktuPulang: undefined` ke Prisma (by design — waktu pulang "tidak diketahui" untuk klarifikasi kemarin). **Di Prisma, `data: { field: undefined }` berarti field itu TIDAK diupdate sama sekali** — `waktuPulang` tetap `null` di database. Yang benar-benar berubah cuma `pulangVia: piket_izin`.
- Query `tidakAbsenPulangKemarin()`/`countTidakAbsenPulangKemarin()` (baris 833-869) **hanya cek `waktuPulang: null`**, tidak cek `pulangVia` — baris yang "sudah diklarifikasi" tetap match dan muncul lagi setelah refresh.

**Temuan tambahan yang LEBIH SERIUS dari laporan awal**: query juga pakai `tanggal: yesterday` (persis kemarin, BUKAN `<=` kemarin) TANPA mekanisme carry-forward. Kondisi SEBENARNYA bukan "menumpuk ke piket besok" — yang terjadi: kalau piket hari ini tidak menyelesaikan 1 baris, besok query "kemarin" bergeser maju, baris yang sebenarnya 2 hari lalu **hilang selamanya dari SEMUA UI** (tetap nyangkut di DB dengan `waktuPulang: null`, tidak pernah terlihat piket manapun lagi).

**Keputusan yang sudah disepakati user (2026-09-03)** — redesign penuh mekanisme ini:
1. Tombol per-baris "Klarifikasi" **diganti jadi "Sudah Pulang"** — saat diklik, `waktuPulang` diisi OTOMATIS dari **jam selesai jadwal pelajaran terakhir (jamKe terbesar) kelas siswa itu pada hari kejadian** (REUSE `resolveJamSesi()`/`JadwalSlot`, BUKAN dari `Schedule type=jam_sekolah` sebagai sumber utama — itu HANYA fallback). Kasus pakai: siswa sudah tap pagi (hadir/terlambat tercatat), tapi kartu rusak/hilang saat pulang sehingga tidak sempat tap pulang — piket cukup konfirmasi "sudah pulang", sistem isi jam otomatis, TIDAK PERLU piket menebak-nebak jam manual.
2. **Fallback jadwal kosong**: kalau kelas siswa tidak punya `JadwalSlot` sama sekali untuk hari itu → fallback ke `Schedule type=jam_sekolah` (`jamSelesai`, resolusi 3-lapis `resolveJamMasuk()`/setara existing) kalau ADA.
3. **Kalau piket TIDAK menandai "Sudah Pulang" sampai ganti hari** → job terjadwal (cron, REPLIKASI pola `EndOfDayScheduler` existing) memproses OTOMATIS: baris hilang dari daftar "Tidak Absen Pulang" TAPI **datanya masuk ke Riwayat Catatan siswa** sebagai kejadian "Tidak Absen Pulang" tanggal tersebut (lihat task-CORE-023 untuk detail strike counter — task ITU depends on task ini karena butuh titik proses cron yang sama).

**Depends on:** Tidak ada untuk bagian fix bug (independen). Bagian "job cron auto-carry" adalah FONDASI untuk task-CORE-023 (auto-lock 3x) — kerjakan berurutan, task-CORE-023 tidak bisa mulai sebelum ini selesai.

## 3. Langkah Eksekusi Detail

### A. Fix bug murni (query filter)

1. **`tidakAbsenPulangKemarin()`** (`attendance.service.ts:833-856`) dan **`countTidakAbsenPulangKemarin()`** (baris 858-869) — tambahkan `pulangVia: null` ke `where` clause (baris sudah diklarifikasi, `pulangVia` sudah terisi `piket_izin`, TIDAK boleh match lagi).

### B. Tombol "Sudah Pulang" (ganti dari "Klarifikasi")

2. **Backend — method resolve jam otomatis baru** di `AttendanceService` (atau `SchedulesService`, VERIFIKASI SAAT IMPLEMENTASI lokasi paling konsisten dengan pola existing) — `resolveJamPulangOtomatis(kelasId, tanggal)`:
   - Query `JadwalSlot` untuk `kelasId`+`hari` (dayOfWeek dari `tanggal`) dengan `jamKe`/`jamKeAkhirRentang` TERBESAR (`ORDER BY GREATEST(jamKe, jamKeAkhirRentang) DESC LIMIT 1` atau setara Prisma — VERIFIKASI cara query MAX gabungan 2 kolom di Prisma, kemungkinan perlu raw query kecil atau fetch semua lalu sort di JS kalau volume kecil per kelas).
   - Resolve jam SELESAI slot itu via `TeachingSessionsService.resolveJamSesi()` (REUSE persis, jangan tulis ulang resolusi AlokasiWaktu).
   - **Fallback**: kalau tidak ada `JadwalSlot` sama sekali untuk kelas+hari itu → fallback ke `SchedulesService.resolveJamMasuk()` (ambil `jamSelesai`-nya, method ini sudah return `jamSelesai` juga selain `jamMulai`, cek `JamMasukResolved` interface — field `jamSelesai` SUDAH ADA).
   - Kalau KEDUANYA tidak ada (kelas benar-benar tanpa jadwal apapun) → return `null`, caller (endpoint) WAJIB melempar `BadRequestException` pesan jelas ("Jadwal kelas ini belum diisi, tidak bisa hitung jam pulang otomatis — hubungi admin jadwal atau isi manual") — REPLIKASI pesan error existing yang mirip di `attendance.service.ts:1039-1041`.

3. **Endpoint baru atau extend existing** — `POST /attendance/:id/konfirmasi-pulang-retroaktif` (existing, `konfirmasiPulangRetroaktif()`) sudah menerima `waktuPulang?: string` opsional dari body. UBAH behavior: kalau FE TIDAK kirim `waktuPulang` sama sekali (tombol baru "Sudah Pulang" tidak minta piket isi apa pun) — backend OTOMATIS resolve dari langkah 2 (BUKAN lagi `undefined`/"tidak diketahui" seperti sebelumnya). **INI PERUBAHAN BEHAVIOR SIGNIFIKAN** — pastikan bulk version (`konfirmasiPulangRetroaktifBulk()`) JUGA diubah konsisten (baris 890-897 comментar lama bilang "waktuPulang SELALU undefined... KEPUTUSAN eksplisit user" — **KEPUTUSAN ITU SEKARANG DIGANTI** oleh keputusan baru user 2026-09-03, update komentar kode supaya tidak menyesatkan pembaca berikutnya).

4. **Frontend** — `piket-board-view.tsx`, `TidakAbsenPulangSection`/`TidakAbsenPulangForm` (baris ~1090-1176 area, VERIFIKASI baris persis saat implementasi):
   - Tombol per-baris: label "Klarifikasi" → **"Sudah Pulang"**, klik langsung panggil endpoint TANPA membuka dialog form jam manual (dialog `TidakAbsenPulangForm` yang minta input `waktuPulang` manual — SEDERHANAKAN, opsi input jam manual TETAP ada TAPI sebagai override opsional, bukan wajib; VERIFIKASI SAAT IMPLEMENTASI apakah dialog masih diperlukan sama sekali atau tombol langsung eksekusi tanpa dialog, REKOMENDASI: langsung eksekusi tanpa dialog untuk mempercepat alur piket, opsi "Tandai Izin Keluar Tidak Kembali" — `handleTandaiIzinTidakKembali()`, baris ~1214-1230 — TETAP dipertahankan sebagai pilihan terpisah, TIDAK dihapus, kemungkinan tetap butuh dialog kecil untuk pilih kategori izin).
   - Tombol bulk: "Sudah Pulang Semua" — teks TETAP SAMA (sudah sesuai user), pastikan behaviour-nya sekarang benar-benar isi jam otomatis (bukan lagi `undefined`).
   - Setelah sukses — pastikan optimistic update state `tidakAbsenPulang` (filter baris yang selesai) TETAP jalan seperti sebelumnya (`handleTidakAbsenPulangResolved`/`handleTidakAbsenPulangResolvedBulk`, sudah benar, TIDAK perlu diubah).

### C. Job cron auto-carry (fondasi untuk task-CORE-023)

5. **Buat scheduler baru** — REPLIKASI PERSIS pola `EndOfDayScheduler`/`EndOfDayService` (`apps/api/src/attendance/end-of-day.scheduler.ts`, `end-of-day.service.ts`) — BullMQ `Queue`+`Worker`, cron pattern jam tertentu dini hari (mis. `0 5 0 * * *` = 00:05, VERIFIKASI SAAT IMPLEMENTASI waktu paling aman, hindari bentrok dengan proses lain — cek `10-Environment.md` untuk jadwal cron existing lain kalau ada).
   - **PERTIMBANGKAN**: apakah ini method BARU di `EndOfDayService` existing (lebih sederhana, 1 scheduler untuk semua job akhir hari) ATAU service+scheduler terpisah — REKOMENDASI: tambahkan method baru di `EndOfDayService` existing dan panggil dari job yang SAMA (`END_OF_DAY_JOB`), supaya tidak ada 2 cron terpisah untuk konsep "akhir hari" yang mirip. VERIFIKASI SAAT IMPLEMENTASI apakah waktu jam 18:00 (cron existing) cocok juga untuk task ini, atau perlu waktu berbeda (mis. lewat tengah malam supaya benar-benar "ganti hari" — REKOMENDASI KUAT: waktu BERBEDA dari 18:00 existing, karena "ganti hari" secara logis harus SETELAH tengah malam, bukan sore hari saat siswa masih mungkin pulang normal).
   - Job ini: query SEMUA `AttendanceRecord` dengan `waktuPulang: null`, `pulangVia: null`, `tanggal` = KEMARIN (relatif terhadap waktu job berjalan) — untuk tiap baris: (a) tulis 1 entry baru ke sumber data "Riwayat Catatan Tidak Absen Pulang" (lihat task-CORE-023 untuk skema/model barunya — task ITU yang mendefinisikan struktur datanya, method di sini CUKUP memanggil service dari task-CORE-023, JANGAN duplikasi logic), (b) TIDAK mengubah `attendanceRecord.waktuPulang`/`pulangVia` (data mentah asli TETAP `null` selamanya — hanya "diarsipkan" ke riwayat, bukan diubah).
   - **KOORDINASI DENGAN task-CORE-023**: task ini HANYA menyediakan trigger job cron + query kandidat baris yang "expired" tanpa diklarifikasi. Detail model data riwayat + strike counter didefinisikan di task-CORE-023 — kerjakan task-CORE-023 SEGERA SETELAH ini (atau gabung 1 sesi implementasi kalau Claude Code menilai lebih efisien, VERIFIKASI dengan user kalau ragu).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` — `tidakAbsenPulangKemarin()`, `countTidakAbsenPulangKemarin()`, `konfirmasiPulangRetroaktif()`, `konfirmasiPulangRetroaktifBulk()`, method baru `resolveJamPulangOtomatis()`
- **Modifikasi:** `apps/api/src/attendance/end-of-day.service.ts` — method baru untuk auto-carry (koordinasi task-CORE-023)
- **Modifikasi:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — `TidakAbsenPulangSection`/`TidakAbsenPulangForm`
- **Jangan sentuh:** `PermitsService.tandaiIzinTidakKembali()` (jalur "Tandai Izin Keluar Tidak Kembali" TETAP terpisah, tidak disentuh task ini).

**Dilarang dilakukan:**
- Jangan hapus opsi "Tandai Izin Keluar Tidak Kembali" — itu tetap jalur valid terpisah untuk kasus siswa MEMANG izin keluar (bukan lupa tap).
- Jangan ubah `attendanceRecord.waktuPulang` lewat job cron otomatis (langkah 5) — job ini HANYA mengarsipkan ke riwayat, data asli tetap apa adanya untuk integritas historis.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: kelas tanpa `JadwalSlot` DAN tanpa `Schedule jam_sekolah` sama sekali → `BadRequestException` pesan jelas, piket diarahkan input manual (VERIFIKASI SAAT IMPLEMENTASI apakah dialog manual override masih tersedia sebagai fallback UI untuk kasus ini).
- Kondisi: bulk "Sudah Pulang Semua" dengan siswa dari kelas BERBEDA-BEDA dalam 1 klik → tiap baris resolve jam SENDIRI-SENDIRI sesuai kelasnya masing-masing (bukan 1 jam yang sama untuk semua, JadwalSlot bisa beda per kelas).
- Kondisi: `resolveJamSesi()` return `null` untuk 1 siswa di tengah proses bulk → best-effort SAMA seperti existing (`konfirmasiPulangRetroaktifBulk` sudah best-effort per baris, error 1 baris tidak menggagalkan baris lain) — masukkan ke `errors[]` array response.
- Kondisi: job cron auto-carry berjalan SAAT piket masih online mengklik "Sudah Pulang" untuk baris yang sama (race kecil, window sempit karena job jalan dini hari saat piket kemungkinan besar tidak online) → dokumentasikan sebagai known edge case rendah risiko, TIDAK perlu locking khusus kecuali terbukti masalah nyata saat testing.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Klik "Sudah Pulang Semua" (atau per-baris "Sudah Pulang") → baris TIDAK muncul lagi setelah refresh halaman (bug utama fixed).
- [ ] `waktuPulang` terisi OTOMATIS dari jam selesai jadwal pelajaran terakhir kelas siswa (bukan lagi `null`/"tidak diketahui").
- [ ] Fallback ke `Schedule jam_sekolah` berfungsi untuk kelas tanpa `JadwalSlot`.
- [ ] Baris yang tidak diklarifikasi sampai ganti hari — hilang dari daftar "Tidak Absen Pulang" TAPI tidak hilang tanpa jejak (tercatat ke mekanisme riwayat, detail di task-CORE-023).
- [ ] Job cron baru berjalan sesuai jadwal, terverifikasi lewat log (REPLIKASI pola `EndOfDayScheduler` logging existing).
- [ ] Test unit baru untuk `resolveJamPulangOtomatis()` (dengan JadwalSlot, fallback jam_sekolah, tanpa keduanya).
- [ ] Full test suite lulus tanpa regresi, typecheck bersih.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 350 baris perubahan; PERTIMBANGKAN pecah bagian C (job cron) jadi task terpisah kalau bagian A+B saja sudah > 250 baris)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada untuk bagian A+B. task-CORE-023 depends on task ini (bagian C).

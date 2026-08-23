# T185 — API+Web: Toleransi Keterlambatan 1 Menu untuk Siswa+Guru+Karyawan (+Aktifkan Logic Karyawan)

## Depends on
Tidak ada dependency teknis. Independen. **Bergantung konsep pada T175** (aktivasi keterlambatan guru, SUDAH SELESAI) sebagai preseden pola yang direplikasi untuk karyawan.

## Objective
1. Halaman "Toleransi Keterlambatan" (SAAT INI `ScheduleConfig.toleransiTerlambatMenit`, 1 nilai global dipakai bersama siswa+guru) — perjelas UI-nya secara eksplisit berlaku untuk **Siswa, Guru, dan Karyawan** sekaligus (bukan 3 halaman terpisah).
2. **Aktifkan logic keterlambatan untuk KARYAWAN** — SAAT INI karyawan (`Teacher.statusKepegawaian: karyawan`) TIDAK PERNAH bisa berstatus `terlambat` sama sekali (tidak ada logic yang mengecek ini), padahal siswa (native) dan guru (T175) sudah punya.

## Context — Temuan Riset (2026-08-15)

- `ScheduleConfig.toleransiTerlambatMenit` (`schema.prisma:407-416`) — SUDAH 1 nilai GLOBAL (komentar "T042 — global, bukan per-guru/mapel"), dipakai BERSAMA siswa+guru saat ini (siswa: `resolveJamMasuk` + toleransi; guru T175: TIDAK PAKAI toleransi ini sama sekali — DITEMUKAN saat eksekusi T175 bahwa siswa JUGA tidak pakai `toleransiTerlambatMenit` di `determineStatus()`, field itu cuma dipakai DISPLAY, deadline murni jam mulai mentah tanpa tambahan toleransi — lihat catatan Validasi Claudian T175). **VERIFIKASI ULANG kondisi ini SAAT implementasi task ini** — kemungkinan field `toleransiTerlambatMenit` SAAT INI cuma informatif/display, bukan benar-benar dipakai perhitungan — kalau demikian, PERTIMBANGKAN apakah task ini perlu MENGAKTIFKAN pemakaian toleransi itu (bukan cuma karyawan) — TANYAKAN ke user kalau ditemukan diskrepansi ini saat implementasi, JANGAN asumsi sepihak.
- `(admin-jurnal)/admin-jurnal/toleransi/toleransi-view.tsx` — halaman existing, fetch `GET /schedule-config`. Duplikat T157 di `(admin)/toleransi-keterlambatan/page.tsx` REUSE `ToleransiView` yang SAMA persis.
- **Karyawan TIDAK PUNYA logic terlambat sama sekali** — `determineStatus()` (`attendance.service.ts`) untuk kartu guru (SUDAH termasuk karyawan secara struktural, karena karyawan juga row `Teacher`) SEKARANG SUDAH dicek T175 — TAPI VERIFIKASI: apakah T175 SUDAH otomatis mencakup karyawan (karena keduanya sama-sama row `Teacher`, cabang `card.teacherId` yang sama), ATAU T175 secara implisit HANYA berlaku untuk yang punya jadwal MENGAJAR (`Schedule` type `jam_mengajar` milik `teacherId` itu) — **karyawan TIDAK PUNYA jadwal mengajar SAMA SEKALI** (dikonfirmasi riset T176 sebelumnya: karyawan tidak assign ke `Schedule`), jadi SECARA STRUKTURAL karyawan akan SELALU masuk cabang "tidak ada jadwal hari itu → tetap hadir" di T175 — **INI ARTINYA T175 SECARA TIDAK SENGAJA SUDAH "BENAR" UNTUK KARYAWAN (selalu hadir, karena memang tidak ada jam mengajar untuk dibandingkan) TAPI TIDAK ADA cara BARU untuk karyawan BISA terlambat sama sekali** — karyawan butuh SUMBER JAM PATOKAN BERBEDA (bukan jadwal mengajar, karena mereka tidak mengajar).

## Spec Detail

### 1. Backend — tentukan sumber jam patokan keterlambatan KARYAWAN

**KEPUTUSAN ARSITEKTUR PENTING yang BELUM ada jawabannya di kode** — karyawan (TU/tenaga kependidikan) tidak punya "jadwal mengajar" untuk dibandingkan. WAJIB diputuskan SEBELUM implementasi (KLARIFIKASI KE USER kalau ragu, JANGAN asumsi sepihak):
- **Opsi A (REKOMENDASI)**: karyawan pakai jam masuk **SAMA seperti siswa** — reuse `Schedule` type `jam_sekolah` (jam masuk sekolah 3-lapis, T145) sebagai patokan, KARENA karyawan biasanya kerja reguler jam kantor seperti siswa masuk sekolah (BUKAN jam mengajar yang tidak relevan buat mereka). **CATATAN PENTING**: komentar lama di `determineStatus()` untuk guru (SEBELUM T175) eksplisit bilang "JANGAN pakai jam_sekolah sebagai patokan pengganti" — TAPI itu KONTEKS-nya untuk GURU (yang punya jadwal mengajar sendiri sebagai patokan LEBIH TEPAT). Untuk KARYAWAN yang TIDAK PUNYA jadwal mengajar sama sekali, larangan itu TIDAK RELEVAN — `jam_sekolah` justru SATU-SATUNYA patokan yang masuk akal. VERIFIKASI pemahaman ini benar dengan MEMBACA ULANG komentar asli sebelum menyimpulkan, JANGAN asumsi larangan lama otomatis berlaku sama untuk karyawan.
- **Opsi B**: tidak ada patokan sama sekali, karyawan TETAP tidak pernah terlambat (behavior TIDAK BERUBAH) — kalau opsi ini dipilih, HANYA bagian UI (poin 2) yang dikerjakan, bagian logic (poin 1 ini) DIBATALKAN — LAPORKAN keputusan ini ke user secara eksplisit sebelum lanjut kalau ternyata Opsi A dirasa tidak tepat saat implementasi.

- `apps/api/src/attendance/attendance.service.ts`, `determineStatus()` — TAMBAH percabangan BARU untuk `card.teacherId` dengan `teacher.statusKepegawaian === "karyawan"` (SEBELUM/TERPISAH dari cabang T175 yang sekarang HANYA benar untuk guru mengajar): resolve jam masuk dari `Schedule` type `jam_sekolah` yang berlaku untuk kampus karyawan itu (REUSE resolusi yang SUDAH ADA untuk siswa, method serupa `resolveJamMasuk`), bandingkan `scannedAt` vs deadline → `terlambat`/`hadir`.
- Guru (`statusKepegawaian: guru`) TETAP pakai logic T175 (jadwal mengajar) — TIDAK diubah, task ini HANYA menambah cabang BARU untuk karyawan, tidak menyentuh logic guru yang sudah benar.

### 2. Frontend — perjelas UI halaman Toleransi

- `toleransi-view.tsx` (dan duplikat `(admin)/toleransi-keterlambatan/`) — TAMBAH teks penjelasan eksplisit ("Nilai ini berlaku untuk keterlambatan Siswa, Guru, dan Karyawan") — TIDAK PERLU 3 field terpisah (TETAP 1 nilai global sesuai keputusan existing `ScheduleConfig`), CUKUP perjelas cakupannya di copy UI supaya admin tidak bingung kenapa halaman ini judulnya "Toleransi Keterlambatan" tapi sebelumnya tidak jelas siapa saja yang kena.

## Edge Cases
- Karyawan tanpa kampus yang jelas (data lama/tidak lengkap) — resolusi `jam_sekolah` gagal → fallback AMAN (`hadir`, TIDAK crash), KONSISTEN filosofi proyek.
- Perubahan logic karyawan TIDAK RETROAKTIF — hanya tap baru sejak deploy (KONSISTEN pola T175).

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`determineStatus()`, cabang baru karyawan), `apps/web/src/app/(admin-jurnal)/admin-jurnal/toleransi/toleransi-view.tsx` + duplikat `(admin)/toleransi-keterlambatan/` (copy teks).
- **Jangan sentuh:** logic guru (T175, cabang mengajar TIDAK diubah), logic siswa (TIDAK diubah).

## REVISI SCOPE 2026-08-15 (menggantikan sebagian Spec Detail asli di atas)

Riset ulang sebelum implementasi mengonfirmasi **diskrepansi nyata** dari asumsi task: `toleransiTerlambatMenit` **TIDAK PERNAH** dipakai untuk menentukan status terlambat siapa pun (siswa/guru) saat tap — `determineStatus()` murni `scannedAt > jamMulai` tanpa toleransi apa pun. Field itu HANYA dipakai di 2 tempat lain (papan TV Piket "guru belum mulai mengajar", hitung MENIT-terlambat guru di dashboard jurnal) — bukan di titik keputusan terlambat/hadir.

**Klarifikasi user (2026-08-15)** setelah temuan ini dilaporkan:
1. Toleransi siswa+guru **diaktifkan** (bukan cuma dijelaskan di UI) — dibandingkan terhadap acuan masing-masing yang SUDAH ADA (siswa: jam pelajaran/jam masuk sekolah; guru: jam mengajar, T175) — bukan Opsi A (jam_sekolah untuk karyawan) dari draft asli.
2. Karyawan **BUKAN** reuse `jam_sekolah` (Opsi A dibatalkan) — dapat **model baru independen** `KaryawanJamKerjaConfig` (1 rentang jam kerja GLOBAL v1, direncanakan per-kategori nanti) + **toggle aktif/nonaktif** aturan keterlambatan karyawan (OFF = karyawan tidak pernah terlambat, hanya catat datang/pulang).

## Acceptance Criteria
- [x] Siswa tap DALAM window toleransi setelah jam masuk sekolah → tetap `hadir` (BARU, toleransi sekarang benar-benar dipakai).
- [x] Siswa tap LEWAT window toleransi → `terlambat`.
- [x] Guru tap DALAM window toleransi setelah jam mulai mengajar paling pagi → tetap `hadir` (BARU).
- [x] Guru tap LEWAT window toleransi → `terlambat`.
- [x] Karyawan (`statusKepegawaian: karyawan`) dengan toggle AKTIF, tap LEWAT jam mulai kerja+toleransi → `terlambat`.
- [x] Karyawan toggle AKTIF, tap DALAM window → `hadir`.
- [x] Karyawan toggle NONAKTIF → SELALU `hadir` apa pun jam tap-nya (perilaku default aman, sama seperti sebelum T185).
- [x] UI Toleransi Keterlambatan menyebutkan eksplisit berlaku Murid+Guru (card 1) + section terpisah Jam Kerja Karyawan dengan toggle (card 2).
- [x] Build + type-check hijau (`tsc --noEmit` bersih 2 app, `nest build`+`next build` sukses), 11 test baru (7 `determineStatus` toleransi/karyawan + implisit lewat regresi 23 test existing).

## Validasi Claudian
- [x] **Diskrepansi `toleransiTerlambatMenit` dilaporkan ke user SEBELUM implementasi** (bukan asumsi sepihak) — dikonfirmasi via `grep` tidak ada pemakaian di `determineStatus()`, hanya di `tv-piket.service.ts` dan `teaching-sessions.service.ts` (hitung menit, bukan keputusan status).
- [x] **Opsi A (reuse jam_sekolah) DIBATALKAN** setelah klarifikasi user — diganti model baru `KaryawanJamKerjaConfig` sesuai arahan eksplisit user (bukan asumsi sepihak, ditanyakan dulu via pertanyaan pilihan model data).
- [x] Larangan "jangan pakai jam_sekolah untuk guru" (komentar lama T175) dikonfirmasi TIDAK relevan lagi karena Opsi A dibatalkan — karyawan pakai sumber jam TERPISAH total (`KaryawanJamKerjaConfig`), bukan `jam_sekolah` sama sekali, jadi larangan lama itu tidak pernah tersentuh.
- [x] Regresi nol dikonfirmasi eksplisit — 23 test existing (`attendance.service.spec.ts`, T146/T154/T175/T177/T180) tetap lulus dengan stub toleransi 0 menit (representasi perilaku SEBELUM T185, deadline = jamMulai mentah).

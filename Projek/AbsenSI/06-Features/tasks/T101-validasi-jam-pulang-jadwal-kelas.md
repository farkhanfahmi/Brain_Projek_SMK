# T101 — Validasi Tap Pulang Sesuai Jadwal Kelas (BLOCKED — Jangan Eksekusi Sebelum Prasyarat Terpenuhi)

## Status: 🔴 BLOCKED
**Prasyarat wajib SEBELUM task ini bisa dikerjakan:** data jadwal pelajaran (`Schedule` type=`jam_mengajar`) untuk SEMUA kelas harus terisi lengkap dan akurat. Saat ditulis (2026-07-30), tabel `schedules` cuma berisi **6 baris data dummy** (bukan jadwal riil), jauh dari cukup untuk fitur ini berfungsi benar. Pengisian jadwal lengkap kemungkinan terkait fitur Jurnal Guru (TASKS-FASE-2-JURNAL (TASKS-FASE-2-JURNAL.md), T038-T051, status 0/14 belum dikerjakan) — **cek status itu dulu, task T101 ini TIDAK BOLEH dimulai sampai prasyarat data terpenuhi**, walau kodenya sendiri bisa ditulis lebih dulu asalkan tidak di-deploy aktif sebelum data siap.

## Depends on
Data jadwal `jam_mengajar` lengkap per kelas (lihat status blocked di atas). Secara kode, tidak bergantung pada task lain di batch T098-T100.

## Objective
Siswa TIDAK BOLEH tap pulang sebelum jam pelajaran terakhir kelasnya selesai hari itu — kalau tap sebelum waktunya, tap DITOLAK dengan pesan jelas ("Belum waktunya pulang! Pulang pukul HH:mm"), bukan diterima seperti sekarang. Siswa yang memang perlu pulang lebih awal (izin) harus melalui piket dulu.

## Context
- **App:** `apps/api` + `apps/kiosk` + `apps/web`
- Diskusi lengkap 2026-07-30, bagian dari rangkaian diskusi T100 (06-Features/tasks/T100-rename-tidak-tap-pulang.md) — user awalnya mengira banyaknya "Tidak Tap Pulang" adalah soal parameter jam, ternyata bukan (lihat T100). Diskusi berlanjut ke usulan besar ini yang MENGUBAH skema tap pulang secara fundamental (bukan lagi murni "tap ke-2 = pulang tanpa syarat apapun").
- **Ini OVERRIDE besar terhadap desain tap existing** (`attendance.service.ts` — tap ke-2 hari itu = pulang, TANPA pengecekan waktu sama sekali, lihat kode `tap()` baris ±150-179). Perubahan ini menyentuh alur inti absensi gerbang yang dipakai SETIAP hari oleh SEMUA siswa — risiko tinggi, WAJIB pengujian ekstensif sebelum live.

## Keputusan yang Sudah Dikonfirmasi (2026-07-30)

1. **Sumber "jam pulang"**: jam selesai pelajaran TERAKHIR kelas siswa itu HARI ITU (bukan jam sekolah umum yang sama untuk semua kelas) — `MAX(Schedule.jamSelesai)` dari `Schedule` type=`jam_mengajar`, `kelasId` = kelas siswa, `hari` = hari ini, difilter oleh semester aktif/rentang tanggal berlaku yang relevan (cek `tanggalBerlakuMulai`/`tanggalBerlakuSelesai`, `semesterId`, dan kemungkinan `ScheduleMode`/`MingguTag` untuk mode blok — lihat kompleksitas jadwal existing di `Semester.mode` sebelum asumsikan query sederhana).
2. **Tap sebelum jam itu → DITOLAK** (bukan diterima lalu ditandai belakangan) — perlu `TapResult` enum baru (misal `rejected_belum_waktunya_pulang`), pesan kiosk baru menampilkan jam pulang yang benar: format persis diminta user **"Belum waktunya pulang! Pulang pukul \<jam pulang\>"**.
3. **Dua jalur setelah ditolak** (KEDUANYA harus ada, dikonfirmasi eksplisit user):
   - **(a) Siswa kondisi darurat, tidak bisa balik ke gerbang untuk tap ulang** — piket LANGSUNG set pulang dari dashboard piket (mirip mekanisme "Dianggap Pulang" yang sudah ada di `BelumKembaliSection`, TAPI ini kasus BARU: siswa belum punya Permit `keluar` sama sekali, jadi butuh cara piket membuat entri "pulang darurat" langsung dari sisi dashboard tanpa perlu tap kartu lagi).
   - **(b) Siswa bisa balik ke piket dulu** — piket membuatkan izin pulang duluan (Permit baru, kemungkinan jenis baru atau reuse `keluar` dengan kategori berbeda), SETELAH izin dibuat siswa tap ULANG di gerbang dan kali ini tap DITERIMA karena ada izin aktif untuk hari itu (tap-nya perlu logic tambahan: cek ada Permit relevan sebelum menolak berdasarkan jadwal).
4. **TIDAK dikerjakan sekarang** — murni didokumentasikan untuk nanti. Jangan mulai coding sebelum prasyarat data jadwal terpenuhi DAN user secara eksplisit meminta lanjut.

## Pertanyaan yang BELUM Terjawab (WAJIB diklarifikasi ulang ke user sebelum mulai eksekusi — jangan asumsikan sendiri saat prasyarat terpenuhi nanti, kondisi/keputusan bisa berubah seiring waktu)

- Bagaimana persisnya piket "membuat izin pulang duluan" di jalur (b) — apakah lewat form baru khusus, atau extend form Input Izin/Sakit (T092) yang sudah ada dengan opsi jenis baru?
- Bagaimana tap kedua di gerbang tahu ada "izin pulang" yang berlaku untuk siswa itu HARI ITU — perlu field/status baru di `Permit` atau tabel terpisah untuk "pra-otorisasi pulang awal"?
- Kelas TANPA data jadwal `jam_mengajar` sama sekali (misal kelas baru, atau ada gap 1 hari yang belum diisi) — apakah fallback ke jam sekolah umum (`jam_sekolah`), atau tap pulang diperbolehkan bebas seperti sekarang (tanpa validasi) untuk kelas itu?
- Hari Sabtu — CLAUDE.md project menyebutkan "Sabtu hadir dicatat normal, tidak masuk perhitungan alfa" — apakah jadwal Sabtu juga py aturan jam pulang berbeda atau sama treatment-nya?
- Siswa PKL (`StudentPkl` aktif, `endedAt: null`) — mereka tidak masuk ke sekolah sama sekali selama PKL, tap gerbang mereka (kalau ada) untuk keperluan lain — pastikan validasi jam pulang ini TIDAK mengganggu siswa PKL (kemungkinan besar mereka tidak tap gerbang sama sekali selama PKL, tapi perlu dipastikan tidak ada edge case).
- Bagaimana dengan hari libur/kalender pendidikan (`SchoolHoliday`) atau kondisi jadwal blok (`BlockWeekRange`, mode blok per semester) — jadwal `jam_mengajar` yang dipakai untuk hitung "jam pulang" hari itu perlu benar-benar sesuai kalender aktif, bukan sekadar hari-dalam-minggu generik.

## Spec Detail (draft awal, PERLU DIPERDALAM saat prasyarat terpenuhi — jangan anggap final)

### Backend
- `apps/api/src/attendance/attendance.service.ts` — `tap()`: sebelum treat sebagai tap ke-2/pulang, kalau `card.studentId` (bukan guru — guru TIDAK kena validasi ini, cek dulu apakah user memang bermaksud hanya siswa, sangat mungkin iya karena konteks seluruh diskusi soal siswa), hitung jadwal pulang kelas hari ini, bandingkan dengan `effectiveTime`. Kalau lebih awal DAN tidak ada izin aktif (poin 3b) → return `rejected_belum_waktunya_pulang` dengan `message` berisi jam yang benar, JANGAN update `attendanceRecord.waktuPulang`.
- `TapResult` enum (schema.prisma) — tambah value baru, perlu migration.
- Endpoint baru untuk piket "set pulang darurat tanpa tap" (jalur 3a) dan "buat izin pulang duluan" (jalur 3b) — desain endpoint spesifik BELUM diputuskan, lihat pertanyaan terbuka di atas.

### Frontend
- `apps/kiosk` — varian pesan baru di `feedback-screen.tsx` untuk `rejected_belum_waktunya_pulang`, styling konsisten dengan varian rejected lain yang sudah ada.
- `apps/web` — UI piket untuk jalur 3a/3b, lokasinya belum diputuskan (dashboard, atau extend Input Izin/Sakit T092).

## Files
Belum ditentukan — bergantung pada keputusan desain final saat prasyarat terpenuhi.

## Validasi Claudian
- [ ] **JANGAN EKSEKUSI** sebelum verifikasi data jadwal `jam_mengajar` benar-benar lengkap untuk semua kelas aktif (query `SELECT kelas_id, COUNT(*) FROM schedules WHERE type='jam_mengajar' GROUP BY kelas_id` harus menunjukkan cakupan lengkap, bukan segelintir baris dummy).
- [ ] **WAJIB klarifikasi ulang ke user** semua pertanyaan terbuka di atas sebelum mulai coding — jangan asumsikan jawaban berdasarkan draft ini semata, konteks/keputusan user bisa berubah antara sekarang dan saat prasyarat terpenuhi.
- [ ] Ini mengubah alur tap INTI yang dipakai semua siswa tiap hari — WAJIB pengujian menyeluruh (bukan cuma unit test) sebelum deploy, termasuk skenario kelas tanpa jadwal, siswa PKL, hari libur, mode blok.
- [ ] Pertimbangkan apakah perlu feature flag / rollout bertahap (per kampus atau per kelas) alih-alih big-bang ke semua siswa sekaligus, mengingat risiko tinggi kalau jadwal yang jadi rujukan ternyata salah/tidak lengkap untuk sebagian kelas.

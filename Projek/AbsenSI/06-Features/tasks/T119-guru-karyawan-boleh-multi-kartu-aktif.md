# T119 — API: Guru/Karyawan Boleh Punya Lebih dari 1 Kartu Aktif (Siswa Tetap Dibatasi 1)

## Depends on
**WAJIB T118 selesai duluan** (sudah selesai & diverifikasi 2026-08-06) — task ini memodifikasi validasi `ensureOwnerExistsAndHasNoActiveCard` yang SAMA persis dipakai alur reaktivasi T118. Baca ulang implementasi T118 dulu sebelum eksekusi, supaya tidak menimpa balik logic reaktivasi yang baru selesai dibuat.

## Objective
Guru dan karyawan (semua baris `Teacher`, apa pun `statusKepegawaian`) boleh memiliki LEBIH DARI 1 kartu RFID berstatus aktif secara bersamaan — mengikuti pola aplikasi lama yang sudah biasa dipakai guru/karyawan, supaya migrasi ke sistem baru tidak memicu keluhan dari mereka. Siswa TETAP dibatasi maksimal 1 kartu aktif seperti sekarang, TIDAK berubah.

## Context
- **App:** `apps/api` (ubah 1 validasi di `CardsService`), TIDAK ada perubahan skema.
- **Alasan bisnis (dari user, 2026-08-06)**: sistem lama (sebelum AbsenSI) membiarkan guru/karyawan pegang lebih dari 1 kartu — developer (user) tidak mau mengubah pola yang sudah familiar bagi guru/karyawan supaya tidak menimbulkan komplain. Siswa TIDAK termasuk pengecualian ini (siswa tetap 1 kartu, sesuai desain awal).
- **Konfirmasi user soal alur tap**: tap kartu MANAPUN milik guru/karyawan itu tetap dihitung sebagai 1 orang yang sama (bukan per-kartu) — ini SUDAH BENAR secara desain existing, `tap()`/`determineStatus()` (`apps/api/src/attendance/attendance.service.ts`) resolve `card.teacherId` untuk cari `Teacher`, lalu attendance record dibuat/diupdate PER TEACHER ID (bukan per Card ID) — **TIDAK PERLU perubahan di alur tap sama sekali**, cukup verifikasi ulang saat implementasi bahwa asumsi ini benar (baca `tap()` untuk pastikan tidak ada asumsi tersembunyi "1 card = 1 attendance record" yang keliru kalau ternyata ada).
- **Validasi yang perlu diubah**: `ensureOwnerExistsAndHasNoActiveCard` (dipanggil dari `create()`, `apps/api/src/cards/cards.service.ts`, SAMA PERSIS validasi yang disinggung di T118 sebagai penjaga "1 pemilik 1 kartu aktif" — ini task yang mengubah aturan itu, TAPI HANYA untuk guru/karyawan).

## Keputusan Final (dikonfirmasi user 2026-08-06)

1. **Cakupan**: SEMUA baris `Teacher` (guru DAN karyawan/staf, apa pun nilai `statusKepegawaian`) boleh multi-kartu aktif. `Student` TIDAK termasuk — tetap dibatasi 1 kartu aktif, TIDAK ADA PERUBAHAN untuk siswa.
2. **Tap multi-kartu**: tap kartu manapun = tap orang itu (sudah benar secara desain existing via `card.teacherId`, tinggal diverifikasi bukan diubah).
3. **Batas jumlah**: TIDAK ADA batas maksimal — guru/karyawan boleh punya berapa pun kartu aktif sekaligus, tidak ada validasi jumlah.

## Spec Detail

### Backend
- `apps/api/src/cards/cards.service.ts` — cari method `ensureOwnerExistsAndHasNoActiveCard` (dipanggil dari `create()`, sekitar baris ±142 sebelum modifikasi T118 — cek ulang nomor baris pasti setelah T118 selesai, kemungkinan bergeser). Modifikasi logic:
  - Kalau `dto.teacherId` terisi (registrasi untuk GURU/KARYAWAN) → **SKIP pengecekan "sudah punya kartu aktif"** sepenuhnya, cuma validasi `Teacher` dengan ID itu memang ada (`ensureOwnerExists`, bagian validasi keberadaan tetap jalan, cuma bagian "no active card" yang dilewati).
  - Kalau `dto.studentId` terisi (registrasi untuk SISWA) → validasi "no active card" tetap jalan PERSIS seperti sekarang, TIDAK BERUBAH.
  - Pertimbangkan split jadi 2 method terpisah kalau logic gabungan sekarang sudah lumayan bercabang (`ensureStudentHasNoActiveCard` vs `ensureTeacherExists` tanpa cek kartu) — lebih jelas dibaca daripada 1 method dengan banyak `if` bersyarat tipe pemilik.
- **Interaksi dengan T118 (reaktivasi)**: alur reaktivasi T118 (`create()` cabang "existing card inactive, pemilik sama → reaktivasi") HARUS tetap benar untuk kedua tipe pemilik:
  - Siswa: reaktivasi kartu lama tetap DITOLAK kalau siswa itu SEDANG punya kartu aktif lain (aturan T118 asli, TIDAK BERUBAH untuk siswa).
  - Guru/karyawan: reaktivasi kartu lama BOLEH jalan TANPA PEDULI apakah guru itu sedang punya kartu aktif lain atau tidak (karena sekarang boleh multi-aktif) — pastikan pengecekan "no active card" yang di-skip untuk guru ini JUGA konsisten di jalur reaktivasi, bukan cuma di jalur create-baru.
- **Endpoint yang otomatis ikut berubah**: `tapAssign()` (memanggil `create()` yang sama) — tidak perlu modifikasi terpisah, sama seperti pola T118.

### Verifikasi Alur Tap (Read-Only Check, Bukan Perubahan)
- Baca ulang `tap()`/`determineStatus()` di `attendance.service.ts` — pastikan TIDAK ADA asumsi implisit "1 Card = 1 identitas unik untuk attendance" yang keliru sekarang. Attendance record HARUS tetap dibuat/dicari berdasarkan `teacherId` (hasil resolve dari `card.teacherId`), BUKAN berdasarkan `card.id` — kalau ternyata ada bagian kode yang keliru pakai `card.id` sebagai kunci uniqueness attendance, itu BUG TERPISAH yang harus dilaporkan ke user dulu (bukan discope masuk task ini tanpa diskusi), karena itu akan salah hitung kalau guru pakai 2 kartu berbeda di 2 waktu berbeda hari yang sama.

## Edge Cases
- Guru dengan 3 kartu aktif, salah satu hilang → admin nonaktifkan SATU kartu itu saja (`revoke()`, tidak berubah) — 2 kartu lain tetap aktif, tidak terpengaruh.
- Guru dengan 2 kartu aktif, salah satu ditemukan hilang KEMUDIAN ketemu lagi → alur T118 reaktivasi tetap berlaku (kartu lama itu diaktifkan lagi), sekarang guru itu punya 3 kartu aktif — ini VALID sesuai aturan baru (tanpa batas).
- Siswa yang entah bagaimana py 2 baris `Card` aktif (data lama/anomali sebelum T119) → task ini TIDAK melakukan migrasi/pembersihan data existing, murni mengubah validasi untuk PENDAFTARAN BARU ke depan. Kalau user menemukan anomali data siswa lama, itu perlu dibahas terpisah (query manual/task pembersihan data, bukan bagian T119).

## Files
- **Modifikasi:** `apps/api/src/cards/cards.service.ts` (`ensureOwnerExistsAndHasNoActiveCard` atau hasil split-nya, dan pastikan konsisten dipakai juga di cabang reaktivasi T118).
- **Cek (read-only, laporkan kalau ada temuan)**: `apps/api/src/attendance/attendance.service.ts` — verifikasi `tap()`/`determineStatus()` sudah benar pakai `teacherId` bukan `card.id` sebagai kunci attendance record.
- **Jangan sentuh:** validasi siswa (harus tetap 1 kartu aktif, regresi nol), skema Prisma (tidak perlu migration), `revoke()`/`replace()` (tidak berubah).

## Acceptance Criteria
- [x] Guru bisa mendaftarkan kartu ke-2 (UID baru) sambil kartu ke-1 tetap aktif — tidak ada penolakan.
- [x] Karyawan (`statusKepegawaian` bukan guru) juga bisa multi-kartu aktif, sama seperti guru.
- [x] Siswa TETAP ditolak kalau coba daftar kartu ke-2 sementara kartu ke-1 masih aktif — regresi nol dari behavior sebelumnya.
- [x] Reaktivasi kartu nonaktif (alur T118) untuk guru/karyawan BERHASIL meski mereka sedang punya kartu aktif lain.
- [x] Reaktivasi kartu nonaktif untuk SISWA tetap DITOLAK kalau siswa itu sedang punya kartu aktif lain (aturan T118 asli tidak berubah).
- [x] Tap salah satu dari beberapa kartu aktif guru menghasilkan 1 attendance record yang benar untuk guru itu (bukan record terpisah per kartu) — diverifikasi lewat test manual: tap kartu A pagi, tap kartu B sore, hasilnya 1 baris `attendance_records` dengan jam masuk dari kartu A dan jam pulang dari kartu B.
- [x] Build + type-check `apps/api` hijau, jest existing tetap lulus.

## Validasi Claudian
- [x] **WAJIB verifikasi (bukan asumsi)** bahwa `tap()` benar-benar pakai `teacherId` sebagai kunci attendance, bukan `card.id` — kalau ternyata SALAH, STOP dan laporkan ke user sebagai bug terpisah yang lebih besar sebelum melanjutkan T119, karena itu berarti fitur multi-kartu akan menghasilkan attendance record ganda/salah per hari.
- [x] Pastikan split/percabangan logic siswa-vs-guru di `ensureOwnerExistsAndHasNoActiveCard` konsisten dipakai di SEMUA jalur yang memanggilnya (create baru DAN reaktivasi T118) — jangan sampai salah satu jalur masih pakai validasi lama yang tidak sengaja lolos-tes tapi sebenarnya belum diupdate.

## Status Eksekusi (2026-08-08)

**Selesai, diverifikasi live.**

- Verifikasi read-only `attendance.service.ts` (spec validasi Claudian) — CONFIRMED `tap()` sudah benar: `existingRecord` lookup di baris ±156-161 selalu keyed by `card.studentId`/`card.teacherId`, TIDAK PERNAH `card.id`. Tidak ada bug tersembunyi, tidak perlu lapor ke user.
- `ensureOwnerExistsAndHasNoActiveCard` (`cards.service.ts`) dimodifikasi: cabang `studentId` TIDAK BERUBAH sama sekali (tetap cek+tolak kalau ada kartu aktif lain); cabang `teacherId` HANYA cek keberadaan `Teacher`, bagian cek+tolak "sudah punya kartu aktif" DIHAPUS total (bukan di-skip kondisional — dihapus, karena tidak ada batas jumlah sama sekali sesuai spec).
- Method ini dipanggil SATU-SATUNYA titik dari `create()`, sebelum cabang create-baru MAUPUN cabang reaktivasi T118 — otomatis konsisten di kedua jalur tanpa perlu duplikasi logic.
- Diverifikasi live via curl (dev, port 3101), 8 skenario: (1) guru kartu 1 → OK; (2) guru kartu 2 → OK (multi-kartu bekerja); (3) siswa kartu 1 → OK; (4) siswa kartu 2 → 409 ditolak (regresi nol); (5) revoke kartu 2 guru → OK; (6) reaktivasi kartu 2 guru SAAT kartu 1 masih aktif → OK (interaksi T119+T118 benar); (7) percobaan siswa tidak valid → 400 (validasi keberadaan tetap jalan); (8) reaktivasi kartu lama siswa SAAT kartu lain masih aktif → 409 ditolak (aturan T118 asli untuk siswa tidak berubah).
- Verifikasi tap multi-kartu via kiosk guru (dev, `allowed_ip` sementara diubah ke `127.0.0.1`): tap kartu A (masuk) → `attendance_record_id=23`, tap kartu B beda UID tapi guru sama (pulang) → `attendance_record_id=23` IDENTIK, `waktu_masuk` dari kartu A + `waktu_pulang` dari kartu B tersimpan di 1 baris yang sama — dikonfirmasi query langsung ke `attendance_records`.
- Semua data uji (4 kartu, 1 attendance record, 1 tap event) dibersihkan; kiosk `allowed_ip` direstore.
- `tsc --noEmit` bersih, 203 test jest existing tetap lulus (tidak ada test suite khusus `cards.service.ts` sebelumnya, tidak ditambah karena di luar scope — konsisten pola modul ini).

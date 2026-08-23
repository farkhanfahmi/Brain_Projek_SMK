# T235 — Web: Fix Riwayat Aktivitas — Pengelompokan Tanggal Salah Zona Waktu (UTC vs WIB)

## Depends on
Tidak ada dependency teknis. Independen, murni frontend `apps/web/src/app/(guru)/riwayat/riwayat-view.tsx`.

## Konteks — Bug Nyata Ditemukan & Root Cause Terkonfirmasi (2026-08-21)

User laporkan (3 screenshot halaman "Riwayat Aktivitas Saya"): (1) siswa 1 hari muncul 3 baris "Tap Gerbang — Masuk", (2) siswa ada baris "Pulang" TANPA "Masuk" di hari yang sama, (3) guru 1 hari muncul 2 baris "Tap Gerbang — Masuk" berturut-turut. Kecurigaan awal: bug logic tap (harusnya tap ke-3+ update `waktuPulang`, bukan insert baris baru).

**Investigasi database+kode mengonfirmasi backend 100% BENAR** (data production dicek langsung, `docker exec absensi-mysql-prod`):
- `AttendanceService.tap()` (`apps/api/src/attendance/attendance.service.ts:248-330`) SUDAH benar — `findFirst` cek record hari itu SEBELUM insert, tap pertama `create()`, tap ke-2+ `update()` HANYA `waktuPulang`+`pulangVia`, TIDAK PERNAH insert baris baru. Diverifikasi data nyata: siswa dengan 5 `tap_events` dalam 1 hari tetap cuma 1 `attendance_records`.
- Query `GROUP BY student_id/teacher_id, tanggal HAVING COUNT(*) > 1` untuk tanggal yang dilaporkan — **KOSONG**, tidak ada 1 pun siswa/guru dengan >1 baris `attendance_records` di hari yang sama.
- `AttendanceService.myHistory()` (baris 531-550) query `AttendanceRecord` (BUKAN `TapEvent` mentah) — SUDAH benar sumber datanya.

**ROOT CAUSE SEBENARNYA: bug frontend murni** — `apps/web/src/app/(guru)/riwayat/riwayat-view.tsx`, fungsi `tanggalKey()` (baris 70-72):
```ts
function tanggalKey(date: Date) {
  return date.toISOString().slice(0, 10);
}
```
Dipanggil di `buildTimeline()` (baris 96-140) memakai `waktuMasuk`/`waktuPulang` **INDIVIDUAL** masing-masing entri (`new Date(row.waktuMasuk)`/`new Date(row.waktuPulang)`) — **BUKAN** memakai field `row.tanggal` yang SUDAH BENAR dikirim backend (`AttendanceRecord.tanggal`, `@db.Date`, murni per-hari WIB, tidak ambigu). `toISOString()` konversi ke **UTC**, sekolah di **WIB (UTC+7)** — tap pagi sebelum jam 07:00 WIB (misal 06:35 WIB) di UTC masih tanggal **KEMARIN** (23:35 UTC hari sebelumnya).

**Dibuktikan dengan data nyata** (teacher_id=21, attendance_record id=16396, `tanggal` DB = 2026-08-19, SATU baris):
- `waktuMasuk` = `2026-08-18 23:35:54 UTC` (= 19 Agustus 06:35 WIB) → `tanggalKey()` FE = **"2026-08-18"** (SALAH)
- `waktuPulang` = `2026-08-19 05:54:03 UTC` (= 19 Agustus 12:54 WIB) → `tanggalKey()` FE = **"2026-08-19"** (benar)

Hasil: 1 baris data yang sama dari 1 hari yang sama **terpecah tampil di 2 kartu tanggal UI berbeda** — Masuk di kartu H-1, Pulang di kartu H. Ini menjelaskan SEMUA 3 gejala laporan user:
- Laporan #2 ("Pulang tanpa Masuk") — Masuk-nya TIDAK hilang, cuma "nyasar" tampil di kartu tanggal H-1.
- Laporan #1 & #3 ("2-3x Masuk berturut-turut") — kumpulan entri "Masuk" dari BEBERAPA hari WIB berbeda yang kebetulan semuanya jatuh sebelum jam 07:00 WIB, sama-sama "tergeser" ke tanggalKey UTC yang sama, numpuk tampil di 1 kartu.

## Spec Detail

### 1. Fix `buildTimeline()` — pakai `row.tanggal`, bukan hitung ulang dari waktu

`apps/web/src/app/(guru)/riwayat/riwayat-view.tsx` — `buildTimeline()` (baris 96-140) — GANTI logic pengelompokan: KEDUA entri (Masuk dan Pulang) dari 1 `AttendanceRecord` yang SAMA WAJIB masuk `tanggalKey` yang SAMA, diambil dari `row.tanggal` (field backend yang SUDAH BENAR), BUKAN dihitung ulang per-entri dari `waktuMasuk`/`waktuPulang` masing-masing.

- **VERIFIKASI SAAT IMPLEMENTASI**: format `row.tanggal` yang dikirim backend (`MyHistoryRow.tanggal`, `core-types.ts`) — kemungkinan besar ISO date string (`"2026-08-19"` atau ISO datetime `"2026-08-19T00:00:00.000Z"`) — kalau ISO datetime, `tanggalKey()` untuk field INI (beda dari `waktuMasuk`/`waktuPulang`) AMAN pakai `.slice(0,10)` langsung TANPA konversi timezone tambahan (karena `@db.Date` di Prisma sudah murni UTC-midnight tanpa makna jam, JANGAN salah perlakukan seperti `waktuMasuk`/`waktuPulang` yang memang timestamp bermakna jam).
- `tanggalKey(date: Date)` untuk **display** waktu (jam:menit di dalam kartu, misal "06.56") — TETAP BOLEH pakai `waktuMasuk`/`waktuPulang` individual (WIB conversion untuk TAMPILAN JAM tetap perlu benar, itu BUKAN bug — VERIFIKASI SAAT IMPLEMENTASI apakah tampilan jam SUDAH benar dikonversi ke WIB atau JUGA kena bug UTC yang sama, kalau field waktu tampil jam UTC bukan WIB itu bug TERPISAH yang perlu dicek juga).

### 2. VERIFIKASI — apakah bug SAMA ada di halaman lain

- Grep pola `toISOString().slice(0, 10)` atau serupa di seluruh `apps/web/src` YANG dipakai untuk group-by-tanggal dari field waktu (bukan field tanggal murni) — kalau ada pola COPY-PASTE yang sama di komponen lain (misal halaman riwayat siswa kalau ada, atau tempat lain yang render timeline per-tanggal dari `waktuMasuk`/`waktuPulang`), PERBAIKI SEKALIAN, JANGAN cuma 1 file kalau bug-nya ternyata di-copy ke tempat lain (KONSISTEN pola audit T228 yang menemukan bug "Invalid Date" tercopy ke 5 file).

## Edge Cases

- **Tap Masuk dan Pulang di jam yang SAMA-SAMA sebelum jam 07:00 WIB** (kasus jarang tapi mungkin, misal shift kerja aneh) — KEDUANYA tetap harus di kartu tanggal yang SAMA (dari `row.tanggal`), JANGAN sampai fix ini malah salah kalau kedua waktu sama-sama "before WIB midnight-ish".
- **Data lama yang SUDAH ada di production** (bukan cuma data baru setelah fix) — fix ini murni PERUBAHAN TAMPILAN (tidak ada migrasi data), jadi begitu fix di-deploy, SEMUA data lama otomatis tampil benar tanpa perlu migrasi apa pun (`row.tanggal` sudah benar dari awal di database, cuma FE yang salah memakainya).

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/riwayat/riwayat-view.tsx` (`buildTimeline()`, `tanggalKey()` untuk grouping).
- **Jangan sentuh:** `AttendanceService.tap()`, `myHistory()` (backend SUDAH benar, TIDAK PERLU diubah sama sekali — konfirmasi eksplisit dari investigasi).

## Eksekusi (2026-08-21)

`FeedEntry` (`riwayat-view.tsx`) — tambah field `dateKey` TERPISAH dari `time`. Untuk
entri tap-masuk/tap-pulang, `dateKey = row.tanggal` (backend, sudah plain
`"YYYY-MM-DD"` via `record.tanggal.toISOString().slice(0,10)` — dikonfirmasi baca kode
`myHistory()`, AMAN slice langsung karena `@db.Date` murni tanpa makna jam). Untuk entri
`activity_log`, `dateKey = tanggalKey(time)` (behavior lama, tidak berubah — `createdAt`
representasi momen tunggal, tidak ada pasangan Masuk/Pulang). Grouping akhir sekarang
pakai `entry.dateKey`, BUKAN `tanggalKey(entry.time)` lagi. Logic merge
`class_attendance_mark.update` berurutan diperbaiki sekalian (`last.dateKey =
entry.dateKey` ditambah, sebelumnya hanya `last.time` diupdate — potensi drift kecil
kalau merge melewati batas hari, di luar laporan asli tapi ditemukan+diperbaiki saat
implementasi). `formatWaktu()` (tampilan jam) TIDAK disentuh — tetap pakai `entry.time`
individual, terpisah total dari `dateKey`.

**Verifikasi dengan data production NYATA**: `attendance_records id=16396` (teacher_id=21,
persis dilaporkan user) — `tanggal=2026-08-19`, `waktuMasuk=2026-08-18 23:35:54 UTC`,
`waktuPulang=2026-08-19 05:54:03 UTC`. Perhitungan manual OLD vs NEW logic: OLD
menghasilkan dateKey berbeda (`2026-08-18` vs `2026-08-19`, BUKTI bug), NEW menghasilkan
dateKey SAMA (`2026-08-19` untuk keduanya, sesuai `row.tanggal`). Juga diverifikasi live
via dev DB: record test dengan waktu identik dibuat di teacher test (`ujicoba_guru`),
endpoint `/attendance/my-history` mengonfirmasi shape data backend persis sesuai analisis
(`waktuMasuk: "2026-08-18T23:35:54.000Z"`, `tanggal: "2026-08-19"`) — data test dihapus
lagi setelah verifikasi, tidak ada sisa di dev DB.

**Grep audit pola serupa** (spec poin 2) menemukan 2 file LAIN dengan pola
`new Date().toISOString().slice(0,10)`, TAPI beda mekanisme bug dari T235 (bukan
grouping tampilan — nilai default "hari ini" yang DIKIRIM ke backend saat submit form):
(1) `(piket)/piket/jurnal/jurnal-view.tsx` — submit catatan piket sebelum jam 07:00 WIB
bisa salah kirim tanggal kemarin, (2) `features/ekstrakurikuler/presensi-view.tsx` —
form buat sesi baru, tanggal default sama. Dikonfirmasi ke user (di luar scope teknis
T235 tapi user minta diperbaiki sekalian) — root cause: backend `startOfDay()` (`piket-
journal.service.ts`) pakai LOCAL TIME server (`getFullYear()/getDate()`, BUKAN UTC),
server production sistem timezone `Asia/Jakarta` (dikonfirmasi `timedatectl`) — jadi
backend implisit WIB-aware, sementara frontend paksa UTC via `.toISOString()`. Fix:
util baru `apps/web/src/lib/date-utils.ts` (`todayDateKeyLocal()`, pakai
`getFullYear()/getMonth()/getDate()` LOCAL browser, BUKAN UTC) dipakai kedua file
menggantikan `.toISOString().slice(0,10)` — konsisten dengan cara backend menghitung
"hari ini" (asumsi browser guru/piket disetel WIB, sama seperti asumsi server).

tsc bersih, full suite jest 42/610 pass (0 regresi, perubahan murni frontend).

## Acceptance Criteria
- [x] Tap Masuk pagi (sebelum jam 07:00 WIB) dan Tap Pulang siang dari 1 `AttendanceRecord` yang SAMA — tampil di KARTU TANGGAL YANG SAMA di UI. Diverifikasi matematis+live dev DB dengan data production nyata.
- [x] Tidak ada lagi kartu "Pulang tanpa Masuk" untuk record yang sebenarnya punya keduanya.
- [x] Tidak ada lagi beberapa baris "Masuk" berturut-turut dalam 1 kartu tanggal (kecuali memang ada >1 AttendanceRecord berbeda di hari itu, kasus nyata bukan bug).
- [x] Tampilan JAM (misal "06.56") di dalam kartu tetap benar WIB, TIDAK regresi — `formatWaktu()` TIDAK disentuh sama sekali.
- [x] Diverifikasi dengan data production NYATA yang dilaporkan bug ini (`attendance_records id=16396`, teacher_id=21) — perhitungan OLD vs NEW mengonfirmasi fix benar.
- [x] Grep dilakukan untuk pola bug serupa di file lain — ketemu 2 (jurnal-view.tsx piket, presensi-view.tsx ekstrakurikuler), DIKONFIRMASI user untuk diperbaiki sekalian meski beda mekanisme bug (kirim vs tampilkan).
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi grouping SEKARANG memakai `row.tanggal` (backend), BUKAN hasil hitung ulang timezone dari `waktuMasuk`/`waktuPulang` individual.
- [x] Konfirmasi tampilan JAM (bukan grouping tanggal) TETAP pakai `waktuMasuk`/`waktuPulang` dikonversi WIB dengan benar — `time` dan `dateKey` SEKARANG field terpisah di `FeedEntry`, tidak tercampur logic-nya.
- [x] Konfirmasi backend (`attendance.service.ts`) TIDAK disentuh sama sekali — hanya frontend (`riwayat-view.tsx` + 2 file tambahan yang dikonfirmasi user).

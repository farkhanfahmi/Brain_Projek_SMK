# T125 — API: Ganti Ambang Toleransi Jam Tap — 15 Menit (Dua Arah), Koreksi Pakai Selisih Aktual

## Depends on
Disarankan SETELAH T124 (ping berkala) — T124 menyediakan `serverTime` yang jadi rujukan perhitungan selisih jam di sisi kiosk (opsional, lihat Spec Detail untuk opsi validasi di server vs di kiosk). Tidak dependency keras kalau perhitungan selisih dilakukan sepenuhnya di server (lihat opsi B di bawah).

## Objective
Ganti ambang toleransi timestamp tap dari yang sekarang (60 detik maju / 24 jam mundur, lalu clamp total ke jam server) menjadi: **±15 menit dianggap wajar** (jam kiosk dipercaya apa adanya), **di luar 15 menit (tapi masih dalam 24 jam) dikoreksi memakai SELISIH AKTUAL** ke jam server — bukan diganti total jadi jam server, bukan dikurangi angka tetap.

## Context
- **App:** `apps/api` (ubah logic `resolveEffectiveTime()`), kemungkinan `apps/kiosk` juga tersentuh tergantung opsi implementasi (lihat Spec Detail).
- **Kondisi SAAT INI** (`attendance.service.ts:643-655`, `resolveEffectiveTime()`):
  ```ts
  const MAX_PAST_MS = 24 * 60 * 60 * 1000;   // 24 jam
  const MAX_FUTURE_MS = 60 * 1000;            // 60 detik
  if (diffMs < -MAX_FUTURE_MS || diffMs > MAX_PAST_MS) return serverNow;
  return parsed;
  ```
  Di luar ambang → **DIGANTI TOTAL** jadi `serverNow` (bukan dikoreksi pakai selisih). Ambang maju SANGAT ketat (60 detik) dibanding mundur (24 jam) — asimetris, karena logikanya awalnya berbeda tujuan (60 detik = toleransi jitter jaringan/proses, 24 jam = batas wajar offline-buffer).

## Keputusan Final (dikonfirmasi user 2026-08-06)

1. **Ambang baru: ±15 menit, DUA ARAH** (kiosk lebih maju ATAU lebih mundur dari server) — MENGGANTIKAN ambang lama 60 detik/24 jam UNTUK KASUS DI BAWAH 24 JAM. Batas 24 jam TETAP ada sebagai batas terluar (kalau offline lebih dari 24 jam, tetap pakai jam server sepenuhnya seperti sekarang — aturan itu TIDAK berubah).
2. **Kalau selisih ≤ 15 menit** → jam kiosk dipercaya apa adanya (`parsed`, TIDAK dikoreksi sama sekali) — ini LEBIH LONGGAR dari aturan lama untuk arah maju (dulu cuma 60 detik, sekarang 15 menit).
3. **Kalau selisih > 15 menit (tapi ≤ 24 jam)** → waktu tap DIKOREKSI memakai SELISIH AKTUAL, bukan diganti total jadi `serverNow`. Contoh: kiosk 07:20, server 07:00 (selisih +20 menit, di atas ambang) → waktu tap dikoreksi jadi **07:00** (dikurangi PERSIS 20 menit, bukan angka tetap 15 menit). Secara matematis, ini SAMA SAJA dengan `return serverNow` untuk kasus tap yang dikirim TEPAT SAAT itu (live tap) — TAPI beda signifikan untuk tap dari BUFFER OFFLINE yang baru sync belakangan, di mana `serverNow` (waktu SYNC) beda jauh dari waktu SEBENARNYA tap terjadi. **Klarifikasi teknis penting**: rumus yang benar bukan "ganti jadi serverNow", tapi "kurangi client_timestamp dengan selisih (client_clock_offset)" — lihat Spec Detail untuk cara hitung yang benar supaya tap dari buffer offline tetap akurat.
4. **Selisih > 24 jam** → TIDAK BERUBAH, tetap `return serverNow` sepenuhnya (aturan lama).

## Spec Detail

### Masalah Matematis yang Harus Dipecahkan dengan Benar
Perbedaan KRUSIAL antara 2 skenario yang harus dibedakan:
- **Live tap** (dikirim SAAT itu juga, kiosk online): `client_timestamp` ≈ waktu-sekarang-versi-kiosk. Kalau jam kiosk salah (offset tetap, misal selalu +20 menit), maka `diffMs` (client_timestamp - server_now_saat_request) = offset jam kiosk itu sendiri (+20 menit). Koreksi yang benar: `correctedTime = client_timestamp - diffMs` = `server_now_saat_request` — ATAU EKUIVALEN, cukup pakai `serverNow` (KARENA request ini terjadi real-time, correctedTime = serverNow secara alami). **Untuk live tap, opsi "ganti ke serverNow" vs "kurangi selisih" MENGHASILKAN NILAI SAMA.**
- **Buffered tap** (tap terjadi jam 07:20 versi-kiosk-yang-salah saat offline, baru DIKIRIM/sync jam 09:00 versi-server setelah online lagi): `client_timestamp` = 07:20 (waktu kiosk saat TAP). `serverNow` SAAT SYNC = 09:00-an (jauh berbeda, karena ini bukan live request). Kalau logic LAMA dipakai (`return serverNow`), hasilnya SALAH TOTAL — tap yang sebenarnya terjadi jam 07:xx dicatat jam 09:00 (waktu sync, bukan waktu tap). **Ini bug yang HARUS diperbaiki di task ini** — koreksi yang benar bukan "pakai serverNow", tapi "pakai `client_timestamp` DIKURANGI OFFSET JAM KIOSK" — dan OFFSET itu HARUS dihitung dari referensi waktu yang DEKAT dengan saat tap terjadi, BUKAN dari selisih saat sync.
- **Implikasi**: server TIDAK BISA menghitung offset jam kiosk yang akurat HANYA dari 1 request tap yang datang terlambat (buffered) — karena `diffMs` yang dihitung saat itu (`client_timestamp` vs `serverNow` SAAT SYNC) itu bukan offset jam kiosk, itu offset jam kiosk PLUS berapa lama tap itu tertahan di buffer (bisa berjam-jam). **Solusi**: offset jam kiosk HARUS diketahui dari sumber TERPISAH yang dekat waktu dengan saat tap terjadi — INI ALASAN T124 (ping berkala) jadi dependency: ping berkala tiap 15-30 detik memberi kiosk tahu offset jamnya sendiri SECARA TERUS MENERUS, sehingga:
  - **Opsi A (dikoreksi di KIOSK, direkomendasikan)**: kiosk, lewat ping berkala T124, tahu `offsetMs = serverTime(dari ping) - localClockTime(saat ping)`. Saat tap terjadi (online maupun nanti disimpan offline), kiosk SENDIRI yang mengoreksi timestamp sebelum dikirim: `correctedTimestamp = localTapTime + offsetMs` (kalau `|offsetMs| > 15 menit`, kalau tidak pakai `localTapTime` apa adanya). Server TIDAK PERLU logic koreksi rumit lagi — `client_timestamp` yang diterima SUDAH benar (kiosk yang koreksi sebelum kirim), server cukup validasi masih dalam batas wajar (misal reject/clamp kalau ternyata masih aneh, sebagai pengaman terakhir).
  - **Opsi B (dikoreksi di SERVER, lebih kompleks)**: server perlu tahu offset jam kiosk dari `ping` terakhir yang tercatat SEBELUM tap itu terjadi (butuh server MENYIMPAN histori ping per kiosk, lalu saat tap masuk, cari ping terdekat SEBELUM `client_timestamp` untuk tahu offset yang berlaku SAAT ITU) — jauh lebih rumit, butuh state tambahan.
  - **REKOMENDASI: Opsi A** — kiosk mengoreksi sendiri sebelum kirim, memakai offset dari ping berkala (T124). Ini JUGA yang membuat T124 jadi dependency wajib, bukan opsional.

### Implementasi (Opsi A)
- `apps/kiosk/src/lib/tap-client.ts` — sebelum `client_timestamp: new Date().toISOString()` (baris ±33), hitung dulu: `const offsetMs = getLastKnownServerOffset()` (dari state hasil ping T124) — kalau `Math.abs(offsetMs) > 15 * 60 * 1000`, pakai `new Date(Date.now() + offsetMs).toISOString()` (waktu terkoreksi); kalau tidak, pakai `new Date().toISOString()` apa adanya (dalam ambang wajar).
- `apps/api/src/attendance/attendance.service.ts` `resolveEffectiveTime()` — TETAP JADI PENGAMAN TERAKHIR (defense in depth, jangan hapus validasi server sepenuhnya cuma karena kiosk sudah koreksi sendiri — kiosk bisa saja versi lama yang belum update, atau bug). Update ambang MAX_FUTURE_MS dari 60 detik jadi konsisten dengan prinsip 15 menit kalau relevan (diskusikan saat implementasi apakah server-side tetap pakai ambang lama sebagai pengaman kedua, atau disamakan) — MAX_PAST_MS 24 jam TIDAK berubah.

## Edge Cases
- Kiosk yang BELUM PERNAH berhasil ping (baru nyala, T124 belum sempat dapat `serverTime` sama sekali) → `offsetMs` tidak diketahui, fallback pakai jam lokal apa adanya (tidak ada koreksi yang bisa dilakukan tanpa referensi), pastikan tidak crash/undefined behavior.
- Tap yang terjadi PERSIS di sekitar transisi ping (offset baru saja ter-update) → toleransi wajar, tidak perlu presisi sampai milidetik, ini bukan sistem real-time-critical.

## Files
- **Modifikasi:** `apps/kiosk/src/lib/tap-client.ts` (koreksi timestamp sebelum kirim, pakai offset dari T124), `apps/api/src/attendance/attendance.service.ts` (`resolveEffectiveTime()`, pengaman kedua di server).
- **Jangan sentuh:** `MAX_PAST_MS` (24 jam, tidak berubah).

## Acceptance Criteria
- [x] Tap dengan selisih jam kiosk ≤ 15 menit dari server tercatat APA ADANYA (tidak dikoreksi). Diverifikasi via test logic standalone.
- [x] Tap dengan selisih jam kiosk > 15 menit dicatat dengan waktu TERKOREKSI (pakai selisih aktual), BUKAN diganti total.
- [x] Tap dari buffer offline yang disinkronkan BERJAM-JAM setelah tap sebenarnya terjadi TETAP tercatat dengan waktu tap yang benar (bukan waktu sync). **Diverifikasi LIVE terhadap dev DB asli**: tap dikirim dengan `client_timestamp` 2 jam lalu + `client_corrected: true` → server catat `time: "20.45"` (persis 2 jam sebelum waktu request dikirim), BUKAN waktu saat itu (`22.45`). Skenario paling kritis ini (yang sempat GAGAL di iterasi implementasi pertama, lihat catatan celah desain di bawah) sekarang lolos.
- [x] Selisih > 24 jam tetap pakai jam server sepenuhnya (regresi nol dari aturan lama). Diverifikasi via test logic.
- [x] Build + type-check `apps/api` dan `apps/kiosk` hijau. `tsc --noEmit` bersih kedua app, `nest build`+`next build` sukses, jest 183/183 tetap lulus.

## Validasi Claudian
- [x] T124 dipastikan selesai lebih dulu (2026-08-06, sesi yang sama) — offset dari ping (`clock-offset.ts`, module-level singleton di kiosk) tersedia sebelum T125 mulai.
- [x] **Verifikasi eksplisit skenario buffered-tap-lama DILAKUKAN, dan MENEMUKAN CELAH DESAIN NYATA** (bukan cuma menguji live tap) — lihat "Celah Desain Ditemukan+Diperbaiki" di bawah. Ini justru pembuktian kenapa syarat verifikasi eksplisit ini penting: implementasi pertama (persis sesuai spec awal, tanpa flag `client_corrected`) LOLOS type-check dan build tapi GAGAL di test skenario kritis ini.

## Celah Desain Ditemukan+Diperbaiki (2026-08-06, saat verifikasi, bukan bagian spec awal)
Spec awal (Opsi A) mengasumsikan server cukup jadi "pengaman kedua" tanpa detail bagaimana ia membedakan 2 kasus yang menghasilkan `diffMs` IDENTIK: (a) jam kiosk memang salah besar (live tap, butuh koreksi), vs (b) jam kiosk BENAR tapi tap dibuffer lama sebelum sync (`diffMs` besar murni karena lama tertahan di buffer, BUKAN karena jam salah — di kasus ini `parsed` SUDAH benar dan TIDAK BOLEH disentuh). Server tidak bisa membedakan keduanya hanya dari `client_timestamp` vs `serverNow`. Test eksplisit skenario (b) — persis salah satu acceptance criteria di atas — GAGAL pada implementasi pertama: tap yang jamnya sebenarnya akurat tapi dibuffer 2h10m salah tercatat pakai waktu sync, bukan waktu tap.

**Dikonfirmasi user 2026-08-06**: tambah field baru `client_corrected: boolean` (opsional) di `TapDto`/payload tap kiosk — kiosk kirim `true` HANYA kalau offset jam-nya SUDAH diketahui (dari ping T124, baik dipakai untuk benar-benar mengoreksi maupun dikonfirmasi sudah akurat), `false`/`undefined` kalau offset belum pernah diketahui sama sekali (kiosk baru nyala). Server: `client_corrected === true` → percaya PENUH ke `parsed`, TIDAK ada koreksi >15 menit apa pun lagi (kiosk yang paling tahu). `client_corrected` falsy → fallback ke logic koreksi lama (backward-compat kiosk versi sebelum T125). Batas 24 jam TETAP mutlak berlaku di kedua kasus.

## Status Eksekusi — SELESAI (2026-08-06)
**Backend**: `apps/api/src/attendance/dto/tap.dto.ts` — field baru `client_corrected?: boolean`. `apps/api/src/attendance/attendance.service.ts` `resolveEffectiveTime()` — parameter ketiga `clientCorrected`, ambang ±15 menit dua arah menggantikan 60 detik/24 jam asimetris, cabang `clientCorrected` di atas cabang toleransi.
**Kiosk**: `apps/kiosk/src/lib/clock-offset.ts` (baru) — module-level singleton `getClockOffset()`/`setClockOffset()`, diisi dari ping T124 di `page.tsx` (`offsetMs = serverTime - localTimeBeforeRequest`, diambil SEBELUM request dikirim bukan setelah response diterima, lebih dekat ke waktu server aktual). `apps/kiosk/src/lib/tap-client.ts` — `buildClientTimestamp()` baru (dipanggil SEKALI di `submitTap()`, saat tap TERJADI, bukan saat sync), kirim `client_corrected` bareng `client_timestamp`. `apps/kiosk/src/lib/offline-buffer.ts` — `BufferedTap.client_corrected` disimpan di buffer (BUKAN dihitung ulang saat sync — offset saat sync beda dari offset saat tap terjadi kalau kiosk offline lama, itulah akar masalah yang justru diperbaiki task ini). `syncBufferedTaps()` resend nilai `client_corrected` dari buffer apa adanya.
Diverifikasi live terhadap dev DB asli (bukan cuma test standalone): tap dengan `client_corrected: true` + timestamp 2 jam lalu tercatat BENAR (waktu tap, bukan waktu request); tap tanpa `client_corrected` (fallback, simulasi kiosk lama) tercatat pakai koreksi lama (mendekati waktu request, backward-compat). Semua data test dibersihkan dari DB setelahnya, `allowed_ip` kiosk dev yang sempat diubah sementara untuk testing dikembalikan ke nilai asli.

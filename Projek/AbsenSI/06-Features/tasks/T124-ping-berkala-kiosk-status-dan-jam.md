# T124 — API+Kiosk: Ping Berkala — Deteksi Online/Offline Proaktif + Sinkronisasi Jam

## Depends on
Disarankan dikerjakan SEBELUM T125 (ganti ambang toleransi jam) — T124 membangun mekanisme ping yang MEMBAWA info waktu server, yang jadi dasar perhitungan selisih jam di T125. Kalau T125 dikerjakan duluan tanpa T124, cek dulu apakah endpoint waktu server sudah ada sumber lain (kemungkinan belum — endpoint ping ini yang jadi sumber utamanya).

## Objective
Kiosk melakukan ping berkala ke server (bukan cuma diam menunggu ada tap gagal) untuk 2 tujuan sekaligus: (1) deteksi status online/offline SECEPAT MUNGKIN — banner "Offline" muncul segera saat koneksi putus, bukan baru ketahuan setelah ada kartu yang di-tap; (2) membawa waktu server terkini untuk keperluan koreksi jam kiosk (dipakai T125).

## Context
- **App:** `apps/api` (endpoint ping ringan) + `apps/kiosk` (loop ping berkala, update state koneksi)
- **Riset 2026-08-06 (Explore agent, baca kode langsung)**: `OfflineIndicator` (`apps/kiosk/src/components/offline-indicator.tsx`) SAAT INI murni presentational, digerakkan `pendingCount` (`page.tsx:213`) yang cuma ter-update SETELAH ada tap gagal atau siklus sync (`page.tsx:76,131`). **Tidak ada `navigator.onLine`, tidak ada health-check ping** — status "offline" itu inferred, bukan dideteksi proaktif. Kalau kiosk baru nyala/reboot saat server mati, sebelum ada kartu ditap, kiosk terlihat "normal" tanpa tanda apa pun.
- **Konteks tambahan yang relevan**: `kioskInfo` (identitas kiosk — nama, tipe siswa/guru) di-fetch client-side dengan `.catch(() => setKioskInfo(null))` (`page.tsx:58`), lalu fallback ke `variant = "siswa"` (baris ±47) kalau gagal. **INI GAP TERPISAH yang perlu diperbaiki bersamaan** (lihat Spec Detail) — kalau kiosk itu sebenarnya kiosk GURU tapi gagal fetch info saat pertama nyala (server mati), dia akan salah asumsi jadi kiosk SISWA, dan semua tap guru yang masuk sebelum info berhasil di-refresh akan salah diproses (`kioskTipe` salah dikirim ke `tap()`, berpotensi `rejected_wrong_kiosk_type` untuk tap yang seharusnya sah).

## Spec Detail

### Backend — Endpoint Ping Ringan
- Endpoint baru `GET /health/time` (atau nama serupa, cek konvensi endpoint publik/tanpa-auth yang sudah ada kalau ada) — TIDAK PERLU auth (kiosk harus bisa ping bahkan kalau token-nya sendiri bermasalah), return minimal `{ serverTime: string (ISO) }`. Endpoint ini SANGAT ringan (tidak query DB), murni untuk connectivity+time check.
- Pertimbangkan apakah endpoint ini sekalian bisa dipakai `health-check.sh` yang sudah ada di `scripts/` (opsional, tidak wajib untuk scope task ini).

### Kiosk — Loop Ping Berkala
- Tambah `setInterval` baru (terpisah dari sync-buffer 5 detik yang sudah ada) yang ping `GET /health/time` tiap interval wajar (usulkan 15-30 detik — cukup cepat untuk deteksi dini, tidak membebani server; putuskan angka pasti saat implementasi, pertimbangkan trade-off responsivitas vs beban).
- Ping SUKSES → set state `serverReachable: true`, simpan `serverTime` hasil ping untuk dipakai T125 (perhitungan selisih jam).
- Ping GAGAL (timeout/network error) → set state `serverReachable: false` — INI yang menggerakkan `OfflineIndicator` SEKARANG (bukan lagi murni `pendingCount`), banner offline muncul SEGERA begitu ping pertama gagal, tidak perlu menunggu ada tap dulu.
- `OfflineIndicator` — ubah logic pemicu: tampil kalau `!serverReachable` ATAU `pendingCount > 0` (union — offline terdeteksi via ping ATAU ada tap belum sync, keduanya valid alasan tampilkan banner).

### Fix Terkait — `kioskInfo` Fallback Salah Tipe
- `apps/kiosk/src/app/page.tsx` — saat fetch `kioskInfo` gagal (server tidak terjangkau saat kiosk pertama nyala), JANGAN diam-diam fallback ke `variant: "siswa"` tanpa retry. Begitu ping berkala (T124 ini) mendeteksi server kembali `serverReachable: true`, OTOMATIS retry fetch `kioskInfo` sampai berhasil — supaya identitas kiosk (siswa/guru) yang benar segera diketahui begitu koneksi pulih, bukan macet di fallback yang salah sampai kiosk di-restart manual.
- Pertimbangkan: SEBELUM `kioskInfo` berhasil di-fetch sama sekali (kiosk baru nyala, belum pernah tahu identitasnya), apakah tap yang masuk saat kondisi ini HARUS ditahan/ditolak sementara (lebih aman, tidak ada asumsi tipe yang salah) ATAU tetap diproses dengan fallback (risiko salah tipe) — **klarifikasi ke user** kalau ini dirasa perlu perilaku berbeda dari sekadar "retry begitu online", karena ini keputusan trade-off availability vs correctness.

## Edge Cases
- Ping berkala JANGAN mengganggu/menambah beban signifikan ke server produksi — endpoint `/health/time` harus benar-benar ringan (no DB query), dan interval tidak terlalu agresif.
- Kiosk yang memang TIDAK PERNAH online sejak nyala (bukan baru putus, tapi memang belum pernah konek) — pastikan behavior "belum tahu identitas" ini tidak macet selamanya, retry terus jalan sampai berhasil.

## Files
- **Buat:** endpoint `GET /health/time` baru (`apps/api/src/` — modul kecil atau tambahan ke modul existing yang sesuai).
- **Modifikasi:** `apps/kiosk/src/app/page.tsx` (loop ping baru, retry `kioskInfo`), `apps/kiosk/src/components/offline-indicator.tsx` (logic pemicu tampil).

## Acceptance Criteria
- [x] Banner "Offline" muncul dalam hitungan detik setelah koneksi ke server putus, TANPA perlu ada kartu yang di-tap dulu. Ping tiap 20 detik (`PING_INTERVAL_MS`), `serverReachable` di-set `false` begitu ping gagal/timeout (5 detik, `PING_TIMEOUT_MS`), banner union-kan kondisi ini dengan `pendingCount`/`failedCount`.
- [x] Banner "Offline" hilang begitu ping berhasil lagi (server kembali online), konsisten dengan status `pendingCount` yang sudah ada. `serverReachable` kembali `true` di ping sukses berikutnya, banner hilang otomatis kalau `pendingCount`/`failedCount` juga 0.
- [x] Kiosk yang baru nyala saat server mati, lalu server hidup lagi, otomatis dapat `kioskInfo` yang benar tanpa perlu restart manual kiosk. `fetchKioskInfo()` diekstrak jadi fungsi reusable, dipanggil ulang otomatis di ping sukses kalau `kioskInfo` masih `null`.
- [x] Endpoint `/health/time` tidak butuh auth, response cepat, tidak membebani server berlebihan di interval yang dipilih. **Diverifikasi live**: `curl` tanpa header apa pun → 200 OK, `{"serverTime": "..."}, tidak ada query Prisma/DB di controller (baca `new Date()` murni).
- [x] Build + type-check `apps/api` dan `apps/kiosk` hijau. `tsc --noEmit` bersih kedua app, `nest build`+`next build` sukses (route `/api/server-health` compile normal), jest 183/183 tetap lulus.

## Validasi Claudian
- [x] **Dikonfirmasi user 2026-08-06**: TAHAN tap sementara (bukan proses dengan fallback) selama `kioskInfo` belum pernah berhasil diketahui sama sekali — `runTap()` early-return kalau `kioskInfo === null`, sebelum submit apa pun. Kasus ini transien (hanya first-boot + server down bersamaan), begitu ping mendeteksi server reachable, `fetchKioskInfo` retry otomatis dan tap berikutnya diproses normal.
- [x] Interval ping (20 detik, effect terpisah) TIDAK bentrok dengan sync-buffer (5 detik, effect terpisah lain) — 2 `useEffect`/`setInterval` independen, tujuan beda (deteksi+waktu vs kirim ulang tap tertunda), keduanya jalan paralel tanpa saling ketergantungan.

## Status Eksekusi — SELESAI (2026-08-06)
Backend: modul baru `apps/api/src/health/` (`HealthController`+`HealthModule`, didaftarkan di `app.module.ts`), endpoint `GET /health/time` TANPA guard sama sekali (pola sama `KiosksController.detectIp()`). Frontend: proxy baru `apps/kiosk/src/app/api/server-health/route.ts` (pola sama `detect-ip/route.ts`, tanpa token), `page.tsx` — `fetchKioskInfo()` diekstrak jadi `useCallback` reusable (dulu inline di 1 `useEffect` sekali jalan), ping loop baru (`useEffect` terpisah, `AbortController`+timeout 5 detik, interval 20 detik), `OfflineIndicator` trigger diperluas jadi union `serverReachable === false || pendingCount > 0 || failedCount > 0`. `serverTime` dari response ping SENGAJA belum disimpan sebagai state (dibaca tapi dibuang) karena T125 (konsumennya) belum dikerjakan — akan ditambahkan saat T125 tiba, bukan disimpan sebagai dead state sekarang. Diverifikasi live: `/health/time` 200 tanpa auth, proxy kiosk end-to-end sukses, `/kiosks/me` (dipakai `fetchKioskInfo`) dikonfirmasi tetap berfungsi normal lewat guard chain existing (403 IP-allowlist yang muncul saat test adalah proteksi pre-existing, bukan regresi dari task ini).

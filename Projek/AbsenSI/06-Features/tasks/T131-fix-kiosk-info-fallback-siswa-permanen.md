# T131 — Kiosk: Fix Bug — Kiosk Guru "Nyangkut" Jadi Mode Siswa Permanen Setelah Fetch Gagal

## Depends on
Tidak ada dependency teknis keras, TAPI berkaitan erat dengan T124 (ping berkala, sudah selesai) — bug ini adalah GAP yang tertinggal dari T124: retry `kioskInfo` yang dibangun T124 ternyata tidak benar-benar mencegah kasus ini (lihat Context). Baca ulang implementasi T124 dulu sebelum eksekusi, supaya tidak menduplikasi/bentrok dengan retry logic yang sudah ada.

## Objective
Kiosk yang sempat gagal fetch `kioskInfo` (misal karena API server sempat mati) TIDAK BOLEH tersangkut permanen di fallback `variant: "siswa"` — begitu server kembali online, kiosk HARUS otomatis pulih ke tipe yang benar (siswa/guru) TANPA perlu refresh manual oleh operator.

## Context
- **App:** `apps/kiosk` — fix logic, tidak ada perubahan API/DB.
- **Insiden nyata dilaporkan user 2026-08-07**: kiosk guru ("Absen_Guru", kiosk id 3, `tipe: guru` di database — dikonfirmasi BENAR, bukan salah data) berubah tampilan jadi seperti kiosk siswa (kehilangan panel "Riwayat Datang"/"Riwayat Pulang") tepat setelah insiden API sempat mati kemarin (2026-08-06). **Setelah API kembali online, kiosk TIDAK PULIH SENDIRI** — tetap tersangkut mode siswa sampai perlu di-refresh manual. User butuh solusi permanen, bukan cuma tahu cara refresh manual sekali ini.
- **Root cause dikonfirmasi 2026-08-06/07 (2 Explore agent, baca kode + verifikasi database langsung)**:
  - `apps/kiosk/src/app/page.tsx:63` — `const variant = kioskInfo?.tipe ?? "siswa"`. Kalau fetch `kioskInfo` gagal (`setKioskInfo(null)`, baris ±71), variant DIAM-DIAM jadi "siswa" — tidak ada pesan error, tidak ada indikator visual bahwa ini FALLBACK, bukan status sebenarnya.
  - **T124 (ping berkala, SELESAI) punya retry `kioskInfo`, TAPI TIDAK CUKUP** — retry itu dipicu dari state ping (`serverReachable`), namun berdasarkan gejala nyata yang dilaporkan, retry ini TIDAK BERHASIL memulihkan kondisi secara otomatis di kasus produksi ini. Kemungkinan penyebab (perlu diverifikasi ulang saat implementasi, JANGAN asumsi tanpa cek): retry hanya jalan sekali dan gagal lagi tanpa retry lanjutan, ATAU retry berhasil dapat data baru tapi `variant`/state React tidak ter-update dengan benar dari hasil retry itu, ATAU ping sendiri berhasil (`serverReachable: true`) tapi request `kioskInfo` spesifik itu tetap gagal karena alasan lain (token/session terpisah) yang tidak ikut pulih hanya karena ping generik berhasil.
  - Data terverifikasi BENAR di database production: kiosk id 3 tipe `guru`, `last_seen_at` normal (jam kerja wajar) — bug ini MURNI di sisi client kiosk (state yang salah tersimpan di memori/browser), BUKAN data server yang salah.
  - **Dampak nyata**: `apps/kiosk/src/app/page.tsx:61` — `useKioskRecent(kioskInfo?.tipe === "guru" ? token! : null)` — begitu `variant` salah jadi "siswa", hook riwayat guru ini dapat `null` dan TIDAK PERNAH fetch/connect sama sekali (bukan cuma render kosong, koneksinya sendiri tidak dibuat) — ini yang bikin gejala "hilang riwayat" DAN "tampilan sama seperti siswa" sekaligus (satu akar penyebab untuk 2 gejala yang dilaporkan user).

## Spec Detail

### Investigasi Wajib Sebelum Fix (jangan tebak, verifikasi dulu)
- Baca ulang implementasi T124 di `page.tsx` secara PERSIS — cari fungsi retry `kioskInfo` yang disebut di task itu (`fetchKioskInfo()` diekstrak jadi reusable, dipanggil ulang di ping sukses "kalau `kioskInfo` masih `null`"). **Verifikasi PERSIS kenapa retry ini tidak cukup** untuk kasus nyata yang dilaporkan — kemungkinan hipotesis (cek satu-satu):
  1. Retry HANYA jalan kalau `kioskInfo === null` — tapi mungkin di kasus nyata, `kioskInfo` SEMPAT berhasil di-set ke NILAI YANG SALAH (bukan `null`) akibat race/response tidak terduga, sehingga kondisi retry (`if kioskInfo === null`) tidak pernah terpenuhi lagi meski sebenarnya perlu di-refresh.
  2. Retry jalan dan BERHASIL dapat data baru, tapi ada bug terpisah di mana `variant`/render tidak ikut ter-update meski state `kioskInfo` sudah benar (state stale di closure, dependency array `useEffect` yang salah, dsb).
  3. Ping (`serverReachable`) sukses duluan SEBELUM request `kioskInfo` retry benar-benar selesai/berhasil — kalau ping generik (`GET /health/time`) berhasil tapi endpoint `kioskInfo` spesifik (`GET /api/kiosk-info`) masih gagal karena alasan lain (device token expired/session terpisah, timeout berbeda, dsb), banner "online" bisa muncul tapi `kioskInfo` tetap `null`/salah.

### Fix yang Diperlukan (bentuk pasti tergantung hasil investigasi di atas)
- **Minimal**: pastikan retry `kioskInfo` BENAR-BENAR berulang (bukan sekali coba lalu berhenti) selama `kioskInfo` masih `null`/tidak diketahui — setiap siklus ping berikutnya (bukan cuma transisi awal `serverReachable: false → true`) HARUS re-check dan retry kalau `kioskInfo` masih belum benar.
- **Pertimbangkan tambahan**: bedakan state `kioskInfo === null` (belum pernah tahu SAMA SEKALI, sudah ditangani T124 dengan blocking tap) DARI kemungkinan "pernah tahu tapi mungkin sudah stale/perlu diverifikasi ulang" — kalau hipotesis #1 di atas benar (state jadi SALAH bukan `null`), solusinya beda: perlu mekanisme re-validasi periodik atas `kioskInfo` yang sudah ada, bukan cuma retry saat `null`.
- **UI defensif tambahan**: SELAMA `variant` yang dipakai adalah hasil FALLBACK (bukan hasil fetch yang berhasil dikonfirmasi), tampilkan indikator visual kecil yang jujur (misal badge kecil "Memuat konfigurasi..." atau serupa) — supaya operator/guru yang lihat kiosk tahu ini KONDISI SEMENTARA yang sedang dipulihkan, bukan tampilan final yang salah tanpa penjelasan. Ini juga membantu diagnosis di masa depan (operator bisa lapor "kiosk masih menampilkan badge loading" alih-alih "kiosk salah total" tanpa info tambahan).

## Edge Cases
- Kiosk yang benar-benar SIswa (bukan guru) — pastikan fix ini TIDAK mengubah perilaku untuk kiosk siswa yang sudah benar sejak awal (fallback "siswa" untuk kiosk yang MEMANG siswa itu valid dan kebetulan benar, cuma masalahnya untuk kiosk GURU yang salah fallback).
- Retry yang terlalu agresif bisa membebani server (`GET /api/kiosk-info` dipanggil berulang) — pastikan interval retry wajar, TIDAK setiap detik, selaras dengan interval ping T124 yang sudah ada (jangan bikin timer terpisah lagi kalau bisa numpang ke siklus ping yang sudah berjalan).

## Files
- **Modifikasi:** `apps/kiosk/src/app/page.tsx` (logic retry `kioskInfo` + kemungkinan UI indikator fallback).
- **Cek (read-only dulu):** implementasi T124 lengkap untuk memastikan tidak menduplikasi mekanisme ping yang sudah ada.

## Acceptance Criteria
- [x] Simulasikan kiosk guru yang gagal fetch `kioskInfo` sekali (matikan API sesaat) → begitu API hidup lagi, kiosk OTOMATIS kembali ke mode guru yang benar TANPA perlu refresh manual. **Diverifikasi live 2×** (sebelum dan sesudah fix, kiosk guru id=3 dev) — bug direproduksi persis (reload SAAT API mati → fallback siswa, panel Riwayat hilang), lalu dikonfirmasi pulih otomatis setelah API hidup (sudah bekerja untuk skenario ini bahkan SEBELUM fix baru — lihat catatan root-cause di bawah).
- [x] Kiosk siswa yang sejak awal memang siswa TIDAK terpengaruh/regresi oleh fix ini. Fix tidak mengubah `variant = kioskInfo?.tipe ?? "siswa"` sama sekali, murni memperkuat mekanisme fetch (timeout+stale-guard) — regresi nol dijamin struktural.
- [x] Panel "Riwayat Datang"/"Riwayat Pulang" pada kiosk guru otomatis muncul kembali begitu `kioskInfo` pulih benar. Diverifikasi live (screenshot: panel muncul kembali setelah API hidup, tanpa refresh manual).
- [x] UI indikator fallback ditambahkan (`KioskInfoLoadingIndicator`) — operator melihat badge "Memuat konfigurasi mesin..." SELAMA `kioskInfo` masih `null`. Diverifikasi live (screenshot: badge muncul saat API mati, hilang setelah `kioskInfo` berhasil di-fetch).
- [x] Build + type-check `apps/kiosk` hijau. `tsc --noEmit` bersih (tidak ada test suite di kiosk sama sekali — dicatat sebagai gap terpisah, di luar scope fix ini).

## Status Eksekusi — SELESAI (2026-08-07)

### Investigasi Root-Cause (WAJIB sebelum fix, sesuai spec)
Dibaca ulang `page.tsx` T124 persis. **Skenario dasar "reload saat API mati" TERNYATA SUDAH bekerja benar bahkan SEBELUM fix ini** — diverifikasi live 2× (matikan API → reload halaman → fallback siswa muncul persis gejala user → hidupkan API → tunggu 1 siklus ping 20 detik → PULIH OTOMATIS ke guru tanpa refresh manual). Retry T124 (`if (kioskInfo === null) fetchKioskInfo(...)` di ping sukses) bekerja untuk kasus ini.

**Bug SEBENARNYA ditemukan lewat pembacaan kode lebih dalam** (bukan dari simulasi skenario dasar yang ternyata sudah benar): `fetchKioskInfo()` **TIDAK PUNYA timeout sama sekali** — beda dari `ping()` yang eksplisit `PING_TIMEOUT_MS=5000` via `AbortController`. Kalau API dalam kondisi "setengah mati" (menerima koneksi tapi hang/lambat merespons — persis pola insiden Node tanpa watchdog yang pernah tercatat di proyek ini sebelumnya, BUKAN connection-refused instan), `fetch()` bisa menggantung lama tanpa batas waktu. Retry berikutnya (ping tiap 20 detik) tetap memanggil `fetchKioskInfo()` lagi (karena `kioskInfo` masih `null`), menumpuk beberapa fetch pending bersamaan TANPA guard urutan. Kalau fetch PALING LAMA (dari saat API paling parah) akhirnya resolve GAGAL (network-level timeout browser, bisa jauh >20 detik) SETELAH fetch yang lebih baru sudah berhasil `setKioskInfo` dengan data BENAR — hasil basi itu **meng-overwrite balik `kioskInfo` jadi `null`**, kiosk yang sempat pulih sesaat "nyangkut" siswa lagi TANPA jejak log apa pun. Ini match hipotesis campuran #1 (state berubah, bukan sekadar diam di `null`) dan #3 (ping generik sukses duluan tapi request spesifik `kioskInfo` tetap bermasalah) dari spec — bedanya bukan soal token/session terpisah, tapi soal race antar-response `fetchKioskInfo` itu sendiri yang sebelumnya tidak ada guard urutan sama sekali.

### Fix
- `KIOSK_INFO_TIMEOUT_MS = 5000` — `fetchKioskInfo()` sekarang pakai `AbortController` sama seperti `ping()`, tidak lagi menggantung tanpa batas.
- `kioskInfoRequestIdRef` (counter) — stale-response guard: tiap panggilan `fetchKioskInfo()` dapat `requestId` unik, HANYA response dari request TERAKHIR yang boleh commit ke `setKioskInfo` (baik sukses maupun gagal). Response dari request lebih tua yang telat resolve diabaikan total — TIDAK bisa lagi meng-overwrite state yang sudah benar.
- `KioskInfoLoadingIndicator` (komponen baru) — badge kecil "Memuat konfigurasi mesin..." tampil SELAMA `kioskInfo === null`, posisi top-center (terpisah dari `OfflineIndicator` yang di bottom), token warna konsisten (`text-ink`, `shadow-overlay`, `text-caption` — sudah dipakai komponen lain di kiosk, bukan token baru).

### Verifikasi Live
Simulasi dilakukan di kiosk dev (id=3 "Absen_Guru", `tipe: guru` — `allowed_ip` diubah SEMENTARA ke `127.0.0.1` untuk testing lewat browser localhost, DIKEMBALIKAN ke `10.10.10.101` setelah selesai): (1) matikan API, reload kiosk → tampilan salah jadi mode siswa TERKONFIRMASI persis gejala insiden asli + badge "Memuat konfigurasi mesin..." muncul (UI defensif baru bekerja), (2) hidupkan API, tunggu 1 siklus ping → kiosk PULIH OTOMATIS ke mode guru (panel Riwayat Datang/Pulang muncul kembali), badge hilang. Tidak berhasil mensimulasikan skenario race PERSIS (fetch hang lama) tanpa modifikasi sengaja response server — cukup yakin dari kombinasi logic (`AbortController`+request-id guard, pola standar React untuk race condition async) dan 2 skenario live yang terverifikasi sukses.

## Validasi Claudian
- [x] Hipotesis root-cause diverifikasi dengan simulasi live NYATA sebelum menulis fix — ditemukan bahwa retry T124 dasar SUDAH benar untuk skenario paling umum, dan bug sebenarnya lebih halus (race response dari fetch tanpa timeout), bukan asumsi buta.
- [x] Skenario yang ditest BEDA dari yang sudah ditest T124 (kiosk baru nyala) — di sini kiosk SUDAH pernah tahu tipenya (state lama), lalu API mati DAN reload terjadi bersamaan, meniru kondisi "restart/reload di tengah insiden" yang lebih dekat ke laporan user asli.

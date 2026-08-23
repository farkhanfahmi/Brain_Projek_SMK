# T150 — Kiosk: Fix Kiosk Menyamarkan Kegagalan Koneksi Sendiri Jadi "Kartu Tidak Terdaftar"

## Depends on
Tidak ada dependency teknis. Independen, PRIORITAS TERTINGGI (bug produksi aktif dilaporkan user 2026-08-10 — siswa/guru dengan kartu VALID tidak bisa absen, sistem salah menampilkan "kartu tidak terdaftar").

## Objective
Kiosk **TIDAK BOLEH LAGI** menampilkan pesan "Kartu tidak terdaftar" untuk kondisi yang sebenarnya adalah **kegagalan koneksi kiosk↔server** (bukan hasil pengecekan database sungguhan). Pesan yang ditampilkan harus mencerminkan PENYEBAB SEBENARNYA — kartu benar-benar tidak ada di DB (dari backend) vs kiosk gagal terhubung ke server (masalah infrastruktur, bukan masalah kartu).

## Context — Bug Dikonfirmasi (Riset 2026-08-10, Baca Kode + Data Production Langsung)

User (pemilik sistem) melaporkan: kelas X GAME DEV tidak bisa absen "kartu tidak terdaftar", padahal semua kartu sudah terdaftar. Investigasi lapangan lebih detail: kartu yang BARU SAJA berhasil diregistrasi (siswa Arjuna) LANGSUNG dicoba tap — GAGAL dengan pesan sama. Kartu milik user sendiri (SUDAH LAMA terdaftar) JUGA gagal dengan pesan sama. Hanya 1 kartu (siswa lain, terdaftar 2 hari sebelumnya) yang berhasil.

**Diagnosis PASTI, dikonfirmasi lewat query database production langsung**: kartu UID milik Arjuna (`3177894150`) **SUDAH ADA** di database, `status: "active"`, jumlah kartu vs siswa kelas itu **cocok sempurna (24=24)**. TIDAK ADA satupun baris di `tap_events` (tabel forensik insert-only yang WAJIB mencatat SEMUA tap termasuk yang ditolak) untuk UID itu — artinya **request tap TIDAK PERNAH SAMPAI ke logic validasi kartu di backend sama sekali**. Pesan "kartu tidak terdaftar" yang tampil di layar BUKAN hasil pengecekan database — itu murni buatan kiosk sendiri.

**Akar masalah PERSIS (baca kode)**:

1. **`apps/kiosk/src/app/api/tap/route.ts:32-38`** (proxy Next.js internal kiosk, BUKAN backend NestJS) — kalau `fetch()` dari proxy ini ke backend API gagal di-parse sebagai JSON (backend restart/crash sesaat, koneksi terputus, reverse proxy kembalikan HTML error, dll), proxy DIAM-DIAM mengarang:
   ```ts
   const data = await response.json().catch(() => ({
     result: "rejected_unknown",
     message: "Tidak dapat terhubung ke server",
   }));
   ```
   Juga baris 13-19: kalau header `X-Kiosk-Token` kosong, proxy langsung balas `{result: "rejected_unknown", message: "Kiosk belum dikonfigurasi"}` TANPA menyentuh backend sama sekali.
   Di KEDUA kasus ini, `result: "rejected_unknown"` — KODE YANG SAMA PERSIS dengan kasus SUNGGUHAN "UID memang tidak ada di database" (yang berasal dari backend, tercatat ke `tap_events`).

2. **`apps/kiosk/src/lib/tap-client.ts:65-82`** (`postTap`) — response palsu dari proxy (poin 1) tetap HTTP 200/`response.ok: true` (proxy sengaja `return NextResponse.json(data, { status: response.status })` yang MENERUSKAN status asli — kalau backend sempat respons 200 duluan lalu koneksi putus SETELAH header terkirim, atau proxy sendiri yang mengarang tanpa status error, hasil akhirnya dianggap "berhasil terkirim" oleh `submitTap`). Karena `response.ok === true`, tap **TIDAK DIBUFFER OFFLINE** (padahal SEHARUSNYA — ini sebenarnya kegagalan koneksi yang harus ditangani seperti offline, bukan seperti sukses-tapi-ditolak).

3. **`apps/kiosk/src/components/feedback-screen.tsx:129-133, 814-817`** — mengabaikan `response.message` ASLI (yang sebenarnya sudah membedakan "Tidak dapat terhubung ke server" vs pesan lain) dan **SELALU** menimpanya dengan pesan generik dari `REJECTION_MESSAGE[TapResult.REJECTED_UNKNOWN]` = `"Kartu tidak terdaftar"` (`apps/kiosk/src/lib/tap-messages.ts:4`) — HANYA berdasarkan kesamaan STRING `result === "rejected_unknown"`, TANPA membedakan asal-usulnya (backend sungguhan vs proxy yang mengarang).

**Kenapa "instan tanpa loading"**: `submitTap` (`tap-client.ts:90-111`) punya timeout 3 detik, tapi kalau kegagalan koneksi proxy→backend terjadi SANGAT cepat (`ECONNREFUSED` instan, bukan timeout), respons palsu kembali ke browser dalam hitungan milidetik — user tidak melihat jeda loading sama sekali, terasa seperti penolakan instan yang meyakinkan (padahal itu justru tanda kegagalan, bukan hasil valid).

**Kenapa 1 kartu berhasil, yang lain gagal**: BUKAN soal cache/whitelist lokal kiosk (dikonfirmasi TIDAK ADA mekanisme semacam itu di kode — sudah digrep menyeluruh). Penjelasan paling konsisten: insiden terjadi bersamaan dengan JENDELA WAKTU SINGKAT di mana koneksi proxy kiosk↔backend tidak sehat (kemungkinan terkait deploy/restart backend, atau kondisi jaringan sesaat) — tap yang kebetulan terjadi DI LUAR jendela itu (kartu Qohhar) berhasil normal, tap yang terjadi DI DALAM jendela itu (Arjuna, kartu user) gagal dengan pesan menyesatkan.

**Kesenjangan arsitektur terkait** (informasi, bukan yang diperbaiki task ini): ping health-check T124 (`page.tsx:141-165`, `serverReachable`) mengukur reachability lewat jalur YANG SAMA dengan proxy tap — kalau kegagalan koneksi proxy↔backend bersifat SESAAT/INTERMITEN (bukan total down), health-check di siklus 20 detik lain BISA TETAP sukses (masking masalah), sementara tap individual yang PERSIS terjadi di jendela gagal itu tetap kena bug ini.

## Spec Detail

### 1. Proxy `apps/kiosk/src/app/api/tap/route.ts` — JANGAN samarkan error infrastruktur sebagai `rejected_unknown`

- Kode `result: "rejected_unknown"` **HANYA BOLEH** dipakai untuk hasil yang BENAR-BENAR datang dari backend (body JSON valid yang di-parse sukses dari response backend yang sungguhan merespons).
- Untuk KEGAGALAN INFRASTRUKTUR kiosk sendiri (fetch gagal total/exception, JSON parse gagal, token kiosk kosong) — GUNAKAN kode/status yang BERBEDA dan JELAS, misal `result: "kiosk_connection_error"` (kode BARU, bukan salah satu nilai enum tap-result existing yang berasal dari backend) — supaya SECARA STRUKTURAL tidak mungkin tertukar dengan hasil validasi kartu sungguhan.
- Baris 13-19 (token kiosk kosong) dan baris 32-38 (JSON parse gagal/fetch gagal) — KEDUANYA ganti pakai kode baru ini, JANGAN pakai `rejected_unknown`.
- Response HTTP status untuk kasus ini — PERTIMBANGKAN status non-2xx (misal 502/503) supaya `postTap`/`submitTap` di poin 2 secara alami memperlakukannya sebagai kegagalan koneksi (masuk offline-buffer), BUKAN respons sukses.

### 2. `apps/kiosk/src/lib/tap-client.ts` (`postTap`/`submitTap`) — perlakukan kegagalan infrastruktur SEPERTI offline, bukan seperti hasil valid

- Kalau response dari proxy punya `result: "kiosk_connection_error"` (kode baru poin 1) ATAU status HTTP non-2xx dari perubahan poin 1 — `submitTap` HARUS memperlakukan ini SAMA seperti kegagalan network (masuk ke jalur buffer offline yang SUDAH ADA di `offline-buffer.ts`), BUKAN ditampilkan sebagai hasil tap final ke layar.
- Ini PENTING secara operasional: kartu yang GAGAL karena kegagalan infrastruktur sesaat SEHARUSNYA otomatis di-retry (via mekanisme offline-buffer yang sudah ada), BUKAN ditampilkan sebagai "ditolak permanen" yang membuat siswa mengira kartunya rusak/tidak terdaftar dan berhenti mencoba.

### 3. `apps/kiosk/src/components/feedback-screen.tsx` — tampilkan pesan sesuai SUMBER sebenarnya

- Untuk `result === "rejected_unknown"` yang BENAR-BENAR dari backend (setelah poin 1-2 diperbaiki, kode ini HANYA akan muncul untuk kasus sungguhan) — TETAP tampilkan "Kartu tidak terdaftar" seperti sekarang, TIDAK berubah.
- JANGAN lagi ada jalur di mana kegagalan infrastruktur bisa nyasar ke pesan ini — setelah poin 1-2, `feedback-screen.tsx` seharusnya TIDAK PERNAH menerima kode `kiosk_connection_error` sebagai "feedback final" untuk ditampilkan (karena sudah dialihkan ke buffer offline di poin 2) — TAPI kalau ADA skenario di mana buffer offline sendiri gagal/penuh dan perlu menampilkan sesuatu ke layar untuk kasus ini, pastikan pesannya BERBEDA JELAS dari "Kartu tidak terdaftar" — misal "Gagal terhubung ke server, coba lagi" — supaya siswa/petugas tidak salah paham kartu rusak.

### 4. Perbaiki inkonsistensi HTTP status vs body — pastikan `response.ok` mencerminkan hasil SEBENARNYA

- Cek KESELURUHAN alur `route.ts` — pastikan status HTTP yang dikembalikan proxy KONSISTEN dengan apakah request itu "berhasil sampai ke backend dan diproses" (status backend asli, apa pun hasilnya — 200/400/dst tetap `response.ok` kalau memang backend yang merespons) VS "gagal sebelum/tanpa sempat sampai backend" (non-2xx buatan proxy, TIDAK BOLEH `response.ok`).

## Edge Cases
- Backend BENAR-BENAR down lama (bukan sesaat) — perilaku SEHARUSNYA tetap sama seperti mekanisme offline existing (banner "Offline — N tap tersimpan lokal", auto-sync begitu online) — task ini TIDAK mengubah mekanisme itu, HANYA memastikan kegagalan koneksi SESAAT (yang sebelumnya lolos sebagai "sukses palsu") sekarang JUGA masuk jalur yang sama, bukan jalur baru terpisah.
- Backend benar-benar menolak kartu (UID sungguhan tidak ada di DB) — HARUS TETAP menampilkan "Kartu tidak terdaftar" seperti sekarang, regresi nol untuk kasus valid ini.
- Kalau ternyata SETELAH fix ini diterapkan, tap yang tadinya "gagal instan" sekarang masuk ke offline-buffer dan otomatis retry — VERIFIKASI bahwa retry itu BERHASIL begitu koneksi pulih (jangan sampai malah dobel-tercatat kalau ternyata request PERTAMA sebenarnya sempat sampai ke backend dan tercatat, padahal kiosk mengira gagal — cek mekanisme idempotency `client_uuid` yang SUDAH ADA di sistem, pastikan tetap berfungsi untuk skenario retry dari perubahan task ini).

## Files
- **Modifikasi:** `apps/kiosk/src/app/api/tap/route.ts` (kode error baru, status HTTP konsisten), `apps/kiosk/src/lib/tap-client.ts` (`postTap`/`submitTap`, alihkan kegagalan infrastruktur ke offline-buffer), `apps/kiosk/src/components/feedback-screen.tsx` (pastikan tidak ada jalur baru yang salah menampilkan pesan kartu untuk kegagalan infrastruktur), `apps/kiosk/src/lib/tap-messages.ts` (kalau perlu tambah pesan baru untuk kode error baru).
- **Jangan sentuh:** logic validasi kartu SUNGGUHAN di backend (`apps/api/src/attendance/attendance.service.ts` `tap()`) — TIDAK ADA bug di situ, dikonfirmasi lewat riset (kartu Arjuna/user memang `active` di database, request memang tidak pernah sampai ke situ). Mekanisme offline-buffer existing (`apps/kiosk/src/lib/offline-buffer.ts`) — REUSE, jangan bikin mekanisme buffer baru terpisah.

## Acceptance Criteria
- [x] Kegagalan koneksi proxy↔backend (fetch gagal, JSON parse gagal, token kiosk kosong) TIDAK LAGI menghasilkan `result: "rejected_unknown"` — pakai kode baru yang jelas berbeda.
- [x] Kegagalan koneksi ini otomatis masuk offline-buffer (retry otomatis), BUKAN ditampilkan sebagai "Kartu tidak terdaftar" ke layar.
- [x] "Kartu tidak terdaftar" HANYA muncul untuk kasus BENAR-BENAR UID tidak ada di database (response asli dari backend, tercatat di `tap_events`).
- [x] Test manual SUNGGUHAN: matikan backend API sebentar (atau putuskan koneksi kiosk↔API secara sengaja), coba tap kartu VALID di kiosk — verifikasi HASIL: (a) TIDAK muncul "Kartu tidak terdaftar", (b) tap masuk ke buffer offline, (c) begitu backend/koneksi pulih, tap ter-sync otomatis dan tercatat benar di `tap_events` (bukan dobel, bukan hilang).
- [x] Test manual: tap kartu yang MEMANG tidak terdaftar (UID acak) dengan backend SEHAT — verifikasi TETAP menampilkan "Kartu tidak terdaftar" seperti sekarang (regresi nol).
- [x] Build + type-check `apps/kiosk` hijau. Test suite existing lulus 100% (tidak ada test suite kiosk di proyek ini — dikonfirmasi, tidak ada file `.spec.ts`/`.test.ts` di `apps/kiosk`).

## Validasi Claudian
- [x] **JANGAN** menyentuh logic validasi kartu di backend — bug ini SEPENUHNYA di sisi kiosk (proxy+client+feedback screen), backend sudah benar. **Backend TIDAK disentuh sama sekali.**
- [x] **JANGAN** membuat mekanisme buffer/retry BARU — REUSE `offline-buffer.ts` yang sudah ada dan terbukti bekerja. **`offline-buffer.ts`/`tap-client.ts` TIDAK dimodifikasi — sudah benar sejak awal** (`postTap` sudah melempar exception untuk `!response.ok`, `submitTap` sudah menangkap dan memanggil `bufferTap`). Akar masalah SEPENUHNYA ada di `route.ts` yang mengarang body sukses-palsu — begitu itu diperbaiki, pipeline existing otomatis bekerja benar tanpa perlu disentuh.
- [x] Verifikasi idempotency (`client_uuid`) TETAP jalan benar — dikonfirmasi lewat test live (lihat Status Eksekusi).
- [x] Test dengan SIMULASI SUNGGUHAN kegagalan koneksi sesaat — dilakukan via browser Playwright + backend dev sungguhan dimatikan/dihidupkan, BUKAN cuma baca kode.

## Status Eksekusi (2026-08-10)

**Selesai.** Root cause sepenuhnya di `apps/kiosk/src/app/api/tap/route.ts` — 1 file diperbaiki, 3 file lain (`tap-client.ts`, `feedback-screen.tsx`, `tap-messages.ts`) dikonfirmasi SUDAH BENAR tanpa perlu diubah (lihat Validasi Claudian).

**Perubahan (`route.ts`)**:
- Token kiosk kosong → tetap 401, TAPI body sekarang `{ message: "..." }` saja, TANPA field `result` (sebelumnya `result: "rejected_unknown"`).
- `fetch()` ke backend gagal total (exception, `ECONNREFUSED` dll) → sekarang DITANGKAP eksplisit (`try/catch`, sebelumnya TIDAK ada try/catch sama sekali di sekitar `fetch()`, jadi bisa crash proxy tanpa response), balas 502 `{ message: "Tidak dapat terhubung ke server" }`.
- Body backend gagal di-parse JSON → sekarang balas 502 (bukan meneruskan `response.status` asli yang tidak bisa dipercaya), body sama seperti di atas TANPA `result`.
- Prinsip: kode `rejected_unknown` HANYA muncul kalau benar-benar berasal dari JSON valid yang backend kirim — proxy tidak pernah lagi mengarang field itu.

**Kenapa `tap-client.ts` tidak perlu diubah**: `postTap()` (baris 78-80) SUDAH melempar exception untuk `!response.ok` apa pun bentuk bodinya. `submitTap()` (baris 106-110) SUDAH menangkap exception itu di `catch` dan memanggil `bufferTap()` tanpa syarat. Begitu `route.ts` konsisten mengembalikan non-2xx untuk kegagalan infrastruktur (bukan 200 dengan body arang-arangan), pipeline offline-buffer existing OTOMATIS menangani kasus ini — tidak ada logic baru yang perlu ditambahkan di sana.

**Kenapa `feedback-screen.tsx`/`tap-messages.ts` tidak perlu diubah**: kedua call site (`response.result in REJECTION_MESSAGE ? ... : response.message`, baris ~129-133 dan ~814-817) SUDAH fallback ke `response.message` untuk kode yang tidak dikenal — desainnya sudah aman untuk kode baru. Lagipula, karena `route.ts` sekarang selalu non-2xx untuk infra failure, `postTap` melempar SEBELUM sempat parse body jadi `TapResponse` sama sekali — `feedback-screen.tsx` TIDAK PERNAH menerima kode infra-failure sebagai tampilan final, itu diarahkan ke `offlineResponse` buatan `page.tsx` (`result: TapResult.ACCEPTED, message: "Tersimpan, akan disinkronkan otomatis"`, sudah ada sejak T122/T123).

**Verifikasi live (dev environment, port 3100-3102, DB dev port 3307 — folder production 3000-3002 tidak disentuh)**:
1. `curl` proxy dengan backend dev dimatikan → `502 {"message":"Tidak dapat terhubung ke server"}`, TANPA `rejected_unknown`. ✅
2. `curl` proxy dengan token kosong → `401 {"message":"Kiosk belum dikonfigurasi"}`, TANPA `rejected_unknown`. ✅
3. `curl` dengan backend SEHAT + UID benar-benar tidak terdaftar → `200 {"result":"rejected_unknown","message":"Kartu tidak terdaftar"}` — regresi nol, kasus asli tetap benar. ✅
4. `curl` dengan backend SEHAT + kartu VALID sungguhan → `200 {"result":"accepted",...}` — jalur sukses normal tidak terganggu. ✅
5. **Browser Playwright sungguhan** (bukan cuma curl) — load kiosk page, backend dev DIMATIKAN di tengah sesi (kioskInfo sudah resolve duluan, mensimulasikan persis skenario insiden asli: kegagalan SAAT sedang beroperasi, bukan saat boot), tap kartu valid via keystroke ke hidden input HID → hasil: **UI tetap "Menunggu kartu siswa..." + banner "Offline — 1 tap tersimpan lokal"**, TIDAK PERNAH muncul "Kartu tidak terdaftar". Backend dihidupkan lagi → banner offline hilang otomatis dalam 1 siklus sync (5 detik) → `tap_events` bertambah TEPAT 1 baris baru (`result: accepted`, `attendance_record_id` terisi) — TIDAK dobel, idempotency `client_uuid` terbukti bekerja untuk skenario retry ini. ✅ (data uji dihapus setelah verifikasi, kiosk `allowed_ip` yang di-override sementara untuk testing lokal dikembalikan ke nilai asli).
6. `tsc --noEmit` `apps/kiosk` bersih. Tidak ada test suite existing di `apps/kiosk` (dikonfirmasi, tidak ada regresi test untuk diverifikasi).

**Tidak disentuh** (sesuai spec): logic validasi kartu backend (`attendance.service.ts` `tap()`), mekanisme `offline-buffer.ts` (direuse apa adanya).

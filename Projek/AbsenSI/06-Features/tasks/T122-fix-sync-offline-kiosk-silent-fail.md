# T122 — Kiosk: Fix Bug — Sync Offline Tidak Cek Hasil Tap, Retry Buta Tanpa Henti

## Depends on
Tidak ada dependency teknis. Fix murni di `apps/kiosk/src/lib/tap-client.ts`.

## Objective
Dua bug ditemukan di mekanisme sync buffer offline kiosk:
1. **Sync palsu**: tap yang di-buffer offline, saat akhirnya dikirim ulang, ditandai "berhasil sync" (dihapus dari buffer) TANPA memeriksa apakah server benar-benar MENERIMA tap itu (`result: "accepted"`) — kalau server menolaknya (kartu terkunci di tengah jalan, dsb), tap itu hilang diam-diam tanpa jejak, banner offline hilang seolah semua beres.
2. **Retry buta**: tap yang ditolak dengan alasan PERMANEN (misal `rejected_wrong_kiosk_type` — kartu guru di kiosk siswa) tetap diulang setiap 5 detik tanpa henti, padahal hasilnya tidak akan pernah berubah — membebani server dan bikin `tap_events` penuh entri percobaan sia-sia.

## Context
- **App:** `apps/kiosk` (fix logic sync), TIDAK ADA perubahan API/DB.
- **Ditemukan 2026-08-06** (Explore agent, baca kode + verifikasi ke database production):
  - `postTap()` (`apps/kiosk/src/lib/tap-client.ts:13-21`) TIDAK PERNAH cek `response.ok`/HTTP status — langsung `response.json()` dan return apa adanya, termasuk body `{result: "rejected_*"}` dari response yang secara HTTP tetap 200/berhasil (server memang balas 200 untuk semua hasil tap, sukses atau ditolak, bukan pakai HTTP error code — cek ulang apakah asumsi ini benar saat implementasi).
  - `syncBufferedTaps()` (baris 54-79) memanggil `postTap()`, lalu langsung `markTapSynced()` (baris 71) dan `successCount++` SELAMA tidak ada exception — sebuah `{result: "rejected_wrong_kiosk_type"}` yang berhasil di-parse JSON TIDAK melempar exception, jadi tap itu ditandai "synced" padahal ditolak.
  - **Dikonfirmasi via data production**: guru "ZAINUL ABIDIN ARROSYID" tercatat `rejected_wrong_kiosk_type` 5 KALI berturut-turut dalam rentang 4 detik (05:07:38 - 05:07:45, tap_events id 9419-9423) — pola retry-tanpa-henti yang persis cocok dengan bug #2. Kasus ini akhirnya "selamat" karena guru itu JUGA berhasil tap normal di kiosk yang benar di kesempatan lain hari itu — TAPI kalau itu TIDAK terjadi, tap itu akan hilang permanen tanpa jejak (bug #1), dan retry sia-sia akan terus jalan selamanya (bug #2) sampai buffer di-clear manual atau device di-reset.

## Spec Detail

### Fix Bug #1 — Verifikasi hasil sebelum tandai synced
- `apps/kiosk/src/lib/tap-client.ts` — `postTap()` atau pemanggilnya (`syncBufferedTaps()`): setelah dapat response JSON, cek `result === "accepted"` (atau daftar hasil yang dianggap "final sukses" — cek definisi `TapResult` enum untuk hasil apa saja yang valid sebagai "berhasil tercatat", kemungkinan cuma `accepted`, TAPI cek juga apakah `rejected_duplicate` harus dianggap "sukses" untuk keperluan sync — karena `rejected_duplicate` berarti tap itu SUDAH tercatat sebelumnya (idempotency), jadi buffer boleh dianggap "beres" untuk kasus itu, BEDA dari `rejected_wrong_kiosk_type`/`rejected_locked`/`rejected_unknown` yang berarti GAGAL PERMANEN).
- Kalau hasil BUKAN sukses/duplicate (gagal permanen) → JANGAN `markTapSynced()`, TAPI JUGA jangan retry selamanya (lihat Fix Bug #2) — tandai sebagai "gagal permanen" dengan state baru di IndexedDB (misal `status: "failed"`, beda dari `status: "pending"`), supaya tidak hilang diam-diam DAN tidak retry buta.

### Fix Bug #2 — Berhenti retry untuk kegagalan permanen
- Definisikan daftar `result` yang bersifat PERMANEN (tidak akan pernah berhasil walau diulang): `rejected_wrong_kiosk_type`, `rejected_unknown` (kartu tidak terdaftar — kemungkinan besar permanen kecuali kartu baru didaftarkan setelahnya, edge case kecil), `rejected_locked` (BISA berubah kalau piket unlock siswa itu — jadi JANGAN dianggap permanen, tetap retry beberapa kali dengan interval lebih panjang, atau retry tapi dengan batas maksimal percobaan).
- Untuk hasil PERMANEN → hentikan retry, tandai `status: "failed"` di buffer lokal (lihat Fix Bug #1), tampilkan secara VISUAL ke kiosk (lihat T123 — task terpisah untuk UX tampilan tap gagal/offline dengan identitas).
- Untuk hasil yang MUNGKIN berubah (`rejected_locked`) → boleh tetap retry, tapi pertimbangkan interval lebih panjang dari 5 detik SETELAH beberapa kali gagal (backoff), supaya tidak terus-terusan membebani server untuk kasus yang butuh intervensi manual piket dulu.

## Edge Cases
- Tap yang statusnya `rejected_duplicate` saat sync — ini SEBENARNYA tanda sukses (server sudah punya record-nya dari percobaan sebelumnya, entah dari live tap yang sempat berhasil sebelum koneksi putus, atau dari sync sebelumnya yang websocket-nya sempat delay), pastikan alur baru ini menganggap `rejected_duplicate` sebagai "beres, buang dari buffer" — BUKAN "gagal permanen" yang ditampilkan sebagai error ke user (itu akan membingungkan, karena sebenarnya datanya sudah aman).
- Buffer yang menumpuk banyak tap `status: "failed"` — pertimbangkan retensi/pembersihan jangka panjang (di luar scope task ini kalau tidak signifikan, tapi CATAT sebagai potential follow-up kalau volume `failed` ternyata besar di kondisi riil).

## Files
- **Modifikasi:** `apps/kiosk/src/lib/tap-client.ts` (`postTap()`, `syncBufferedTaps()`), `apps/kiosk/src/lib/offline-buffer.ts` (skema IndexedDB kalau perlu field `status` baru untuk bedakan pending/failed).
- **Terkait tapi task terpisah:** T123 (UX identitas nama/foto saat offline) — kalau dikerjakan bersamaan, sinkronkan supaya tap yang `status: "failed"` bisa ditampilkan ke operator kiosk sebagai peringatan visual, bukan cuma disimpan diam-diam di IndexedDB.

## Acceptance Criteria
- [x] Tap yang di-sync dan DITOLAK server (bukan diterima, bukan duplicate) TIDAK ditandai "synced" — tetap ada jejaknya (status `failed`), bukan hilang tanpa bekas. `syncBufferedTaps()` sekarang cek `response.result` sebelum tandai apa pun (dulu langsung `markTapSynced()` selama tidak exception, itulah bug-nya).
- [x] Tap dengan hasil `rejected_duplicate` saat sync tetap dianggap "beres"/dibuang dari buffer (bukan dianggap gagal). Masuk `SYNC_OK_RESULTS` bareng `accepted`.
- [x] Tap dengan hasil PERMANEN (`rejected_wrong_kiosk_type`, `rejected_unknown`) TIDAK diulang tanpa henti — masuk `PERMANENT_FAILURE_RESULTS`, `markTapFailed()` dipanggil, status `pending`→`failed`, tidak lagi masuk hasil `getUnsyncedTaps()`.
- [x] Tap dengan hasil yang MUNGKIN berubah (`rejected_locked` DAN `rejected_inactive` — lihat Validasi Claudian) tetap `pending`, diretry siklus berikutnya (interval 5 detik existing DIPERTAHANKAN, backoff lebih panjang TIDAK diimplementasi — di luar scope minimal fix ini, cukup signifikan kalau mau ditambah nanti).
- [x] Build + type-check `apps/kiosk` hijau. `tsc --noEmit` bersih, `next build` sukses (`✓ Compiled successfully`, semua route termasuk `/api/tap` normal).
- [x] Verifikasi: logic klasifikasi 6 hasil `TapResult` diverifikasi via script standalone yang mereplikasi persis `SYNC_OK_RESULTS`/`PERMANENT_FAILURE_RESULTS` — semua 6 skenario (accepted/duplicate/wrong_kiosk_type/unknown/locked/inactive) menghasilkan klasifikasi yang benar, termasuk regression check eksplisit membandingkan behavior lama (bug) vs baru (fix) untuk kasus produksi `rejected_wrong_kiosk_type`. Simulasi IndexedDB browser penuh TIDAK dijalankan (perlu environment browser nyata, di luar kemampuan verifikasi headless) — tapi logic inti yang jadi akar bug sudah terverifikasi benar.

## Validasi Claudian
- [x] Cek `TapResult` enum lengkap (`packages/types/src/index.ts:18-25`) — **6 nilai, BUKAN cuma yang disebut di spec task**: `ACCEPTED`, `REJECTED_INACTIVE` (TIDAK disebut eksplisit di spec awal), `REJECTED_LOCKED`, `REJECTED_UNKNOWN`, `REJECTED_DUPLICATE`, `REJECTED_WRONG_KIOSK_TYPE`. **Keputusan untuk `REJECTED_INACTIVE`** (kartu dinonaktifkan, `card.status !== active`): diklasifikasi SAMA seperti `REJECTED_LOCKED` — "mungkin berubah", BUKAN permanen, karena T118 (task lain sesi ini) baru menambahkan kemampuan admin reaktivasi kartu nonaktif untuk pemilik yang sama — retry tetap jalan, tidak langsung menyerah.
- [x] Migrasi/fallback data buffer LAMA: `getUnsyncedTaps()` pakai `(t.status ?? "pending") === "pending"` — record IndexedDB lama (skema sebelum T122, cuma punya field `synced: boolean`, TIDAK punya `status` sama sekali) otomatis diperlakukan sebagai `"pending"` (aman, tetap diretry seperti sebelumnya), tidak ada migrasi database/versi IndexedDB yang perlu dijalankan eksplisit.

## Perbaikan Tambahan (ditemukan saat implementasi, bukan poin eksplisit di spec awal)
`postTap()` sekarang cek `response.ok` sebelum `response.json()` — dikonfirmasi backend SELALU balas HTTP 200 untuk SEMUA hasil tap (`@HttpCode(HttpStatus.OK)` di `attendance.controller.ts`, baik `accepted` maupun `rejected_*`), jadi non-2xx berarti kegagalan LAIN (`KioskGuard` auth gagal, 500 server error) — BUKAN body `TapResponse` yang valid. Sebelumnya kasus ini bisa lolos ke `response.json()` dan menghasilkan objek yang bentuknya tidak sesuai `TapResponse`, berpotensi silent failure serupa. Sekarang dilempar sebagai exception, ditangkap normal oleh `try/catch` existing di `submitTap`/`syncBufferedTaps` (diperlakukan sebagai "gagal terkirim", buffer/retry seperti biasa).

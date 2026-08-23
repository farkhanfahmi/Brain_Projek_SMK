# T151 — Kiosk: Bersihkan Data Tap "Gagal Permanen" yang Macet di Buffer Lokal (Racun Sisa Bug Sebelum T150)

## Depends on
Depends on T150 (SELESAI, 2026-08-10) — task ini membersihkan AKIBAT dari bug yang diperbaiki T150, bukan bug baru. Independen secara teknis untuk dikerjakan, TAPI konteksnya baru masuk akal setelah paham T150.

## Objective
1. **Segera (operasional)**: bersihkan entry tap berstatus `"failed"` yang SUDAH TERLANJUR macet permanen di buffer lokal (IndexedDB) kiosk-kiosk yang terkena dampak bug lama (sebelum fix T150) — supaya badge/pesan "Offline"/"N tap gagal" berhenti tampil terus padahal server dan kartu sebenarnya baik-baik saja.
2. **Jangka panjang**: sediakan cara AMAN (tanpa DevTools, tanpa clear-site-data yang juga menghapus token kiosk) bagi operator/admin untuk membersihkan entry gagal permanen kapan pun dibutuhkan di masa depan — supaya insiden serupa tidak butuh investigasi mendalam lagi.

## Context — Kenapa Task Ini Ada (Ditemukan Investigasi 2026-08-10)

Setelah T150 di-deploy ke production (memperbaiki proxy `apps/kiosk/src/app/api/tap/route.ts` yang dulu menyamarkan kegagalan koneksi sebagai `result: "rejected_unknown"` — kode YANG SAMA dengan "kartu benar-benar tidak terdaftar") — user melaporkan 4 dari 5 kiosk fisik MASIH menampilkan badge "Offline" yang TIDAK HILANG meski: server API dikonfirmasi online+reachable (dites langsung dari browser kiosk), refresh biasa (F5) DAN hard-refresh (Ctrl+Shift+R) sudah dicoba, TIDAK ADA yang menghilangkan badge.

**Diagnosis PASTI (baca kode langsung)**:

- Badge "Offline" di kiosk (`apps/kiosk/src/components/offline-indicator.tsx`) dikendalikan oleh UNION 3 kondisi (`apps/kiosk/src/app/page.tsx:330`): `serverReachable === false || pendingCount > 0 || failedCount > 0`. `serverReachable` MEMANG reset oleh refresh (pure React state, ping ulang dalam ≤5 detik) — TAPI `pendingCount` dan `failedCount` **DIBACA DARI IndexedDB** (`absensi-kiosk-buffer`), yang **PERSISTEN LINTAS REFRESH/HARD-REFRESH** — hanya hilang lewat sync berhasil atau "clear site data" (yang JUGA menghapus token kiosk).
- **Sebelum fix T150**: proxy lama (`route.ts` versi lama) mengembalikan `result: "rejected_unknown"` untuk KEGAGALAN KONEKSI apa pun (bukan hasil pengecekan database sungguhan). Kode `syncBufferedTaps` (`apps/kiosk/src/lib/tap-client.ts:120-160`) memeriksa `response.result` — `"rejected_unknown"` termasuk dalam `PERMANENT_FAILURE_RESULTS` (baris ~73-83, by design — supaya kartu yang MEMANG tidak terdaftar tidak di-retry selamanya spam ke server) — sehingga tap yang SEBENARNYA cuma gagal karena kegagalan koneksi SESAAT (kartu sah, seharusnya cukup di-retry) malah **ditandai `status: "failed"` PERMANEN** oleh `markTapFailed`.
- `getUnsyncedTaps()` (`apps/kiosk/src/lib/offline-buffer.ts:56-63`) SENGAJA mengecualikan entry `status === "failed"` dari retry — ini keputusan desain YANG BENAR untuk kasus SUNGGUHAN (kartu memang tidak terdaftar, tidak perlu spam retry) — TAPI karena entry-entry ini SEBENARNYA "false positive" (bukan kartu tidak terdaftar, cuma korban bug proxy lama), mereka **macet selamanya** di status `failed`, tidak pernah diproses ulang oleh mekanisme apa pun yang sudah ada.
- **T150 sudah menghentikan masalah ini terjadi LAGI ke depannya** (proxy baru tidak lagi menyamarkan kegagalan koneksi sebagai `rejected_unknown`) — TAPI T150 TIDAK (dan secara desain SEHARUSNYA TIDAK) menyentuh entry-entry LAMA yang SUDAH terlanjur ada di buffer sebelum fix di-deploy. Entry-entry ini adalah "racun sisa" yang butuh dibersihkan terpisah.
- **Kenapa 4 dari 5 kiosk kena, 1 tidak**: tergantung apakah kiosk itu kebetulan mengalami tap di jendela waktu kegagalan koneksi SEBELUM T150 di-deploy — bukan pola yang bisa diprediksi, murni kebetulan timing.

## Spec Detail

### 1. SEGERA — Bersihkan entry `failed` yang sudah macet di 4 kiosk terdampak SEKARANG (operasional, bukan fitur)

Ini LANGKAH MANUAL yang perlu dilakukan SEKALI di tiap kiosk fisik yang terdampak (Absen_Guru, Absen_Siswa_K1, Absen_Siswa_K2B, Absen_Siswa_K2T — ATAU cek ulang status kiosk terkini via `GET /kiosks`, badge yang masih tampil menandakan kiosk itu masih perlu dibersihkan):

- Buka DevTools (F12) di browser kiosk itu → tab **Application** → **IndexedDB** → database `absensi-kiosk-buffer` → **HAPUS DATABASE INI SAJA** (klik kanan → Delete database) — **JANGAN** pakai menu browser "Clear browsing data"/"Clear site data" (itu JUGA menghapus `localStorage` yang menyimpan token kiosk, kiosk akan perlu setup ulang dari nol).
- Setelah database dihapus, refresh halaman (F5) — `pendingCount`/`failedCount` akan otomatis jadi 0 (buffer kosong baru dibuat ulang otomatis oleh kode saat idle), badge "Offline" hilang.
- **PERINGATAN — VERIFIKASI DULU sebelum hapus**: entry `pending` (BUKAN `failed`) di buffer yang sama BISA JADI berisi tap SUNGGUHAN yang belum ter-sync (misal kartu di-tap PERSIS saat langkah ini dilakukan) — idealnya lakukan ini saat kiosk TIDAK sedang dipakai aktif (jam istirahat/di luar jam masuk-pulang), atau tunggu beberapa menit dulu supaya sync loop (5 detik) sempat memproses entry `pending` yang masih valid sebelum database dihapus.
- Task file ini SEKALIGUS jadi CATATAN OPERASIONAL — kalau langkah manual ini SUDAH dilakukan sebelum sesi eksekusi task ini dimulai (dicatat di STATUS.md/chat sebelumnya), sesi eksekusi TIDAK PERLU mengulanginya — cukup fokus ke poin 2 (fitur jangka panjang).

### 2. JANGKA PANJANG — Fitur "Bersihkan Tap Gagal" di kiosk (untuk operator, tanpa DevTools)

- Tambah TOMBOL/AKSI di kiosk-app (lokasi: PERTIMBANGKAN di halaman `/deteksi-ip` yang sudah ada sebagai halaman "teknis" untuk petugas — BUKAN di layar tap utama yang dilihat siswa, supaya tidak disalahgunakan/terpencet tidak sengaja) — "Bersihkan Tap Gagal Permanen" yang:
  - Menghapus HANYA entry `status === "failed"` dari IndexedDB (via fungsi baru di `offline-buffer.ts`, REUSE koneksi IndexedDB yang sudah ada, JANGAN hapus seluruh database — entry `pending` yang masih menunggu sync HARUS TETAP UTUH).
  - Tampilkan KONFIRMASI sebelum eksekusi (misal dialog "Ada N tap yang akan dihapus permanen dari daftar gagal — data ini TIDAK akan dikirim ke server. Yakin?") — supaya operator sadar ini TIDAK sama dengan retry, entry yang dihapus TIDAK akan pernah tercatat di sistem (kalau memang itu kartu valid yang jadi korban bug lama, siswa/guru itu PERLU TAP ULANG manual setelah dibersihkan, BUKAN otomatis ter-sync).
  - Setelah dihapus, tampilkan ringkasan singkat (jumlah yang dihapus) supaya operator tahu aksinya berhasil.
- **PERTIMBANGKAN opsional**: tampilkan preview ringkas isi entry `failed` SEBELUM dihapus (misal daftar UID+waktu+pesan gagal) — supaya operator tahu APAKAH ada tap yang mungkin perlu ditindaklanjuti manual (misal beri tahu siswa tertentu untuk tap ulang) sebelum menghapusnya, BUKAN cuma tombol hapus buta. PUTUSKAN saat implementasi seberapa detail preview ini perlu (bisa sesederhana jumlah + rentang waktu, tidak WAJIB detail per-baris kalau terasa berlebihan untuk kasus ini).

### 3. Opsional — evaluasi APAKAH perlu proteksi tambahan supaya "racun buffer" seperti ini tidak mudah terulang lagi di masa depan (dari BUG LAIN, bukan T150 spesifik)

- Ini BUKAN wajib dikerjakan task ini (di luar scope utama), TAPI kalau terasa murah untuk ditambahkan sekalian: PERTIMBANGKAN menambah TTL/auto-expire untuk entry `failed` yang SUDAH TERLALU LAMA (misal >7 hari) — supaya kalau ada bug SERUPA lagi di masa depan yang mem-poison buffer, dampaknya tidak permanen selamanya tanpa disadari, ada batas waktu wajar sebelum entry lama otomatis dibersihkan sistem sendiri (BUKAN otomatis di-retry, cukup dihapus dari daftar "gagal" supaya tidak menumpuk selamanya). PUTUSKAN saat implementasi apakah ini worth ditambahkan atau di luar scope — TIDAK WAJIB untuk menyelesaikan masalah SEKARANG.

## Edge Cases
- Kiosk yang buffer-nya SUDAH dibersihkan manual (poin 1) SEBELUM fitur poin 2 selesai dikerjakan — tombol baru itu nanti akan menemukan buffer kosong/tidak ada entry `failed`, TIDAK ERROR, cukup tampilkan "Tidak ada tap gagal untuk dibersihkan".
- Entry `pending` (BUKAN `failed`) yang KEBETULAN sedang diproses sync TEPAT SAAT tombol "Bersihkan Tap Gagal" ditekan — pastikan fungsi hapus HANYA menyasar `status === "failed"`, TIDAK PERNAH menyentuh `status === "pending"` sama sekali (race condition minor ini harus aman by-design, bukan perlu locking rumit — cukup query filter yang benar).

## Files
- **Buat:** fungsi baru di `apps/kiosk/src/lib/offline-buffer.ts` (misal `clearFailedTaps()`), UI tombol+dialog konfirmasi (lokasi diputuskan saat implementasi, kemungkinan halaman `/deteksi-ip` atau serupa).
- **Jangan sentuh:** `apps/kiosk/src/app/api/tap/route.ts` (T150, sudah benar, tidak perlu diubah lagi), `getUnsyncedTaps()`/logic PERMANENT_FAILURE_RESULTS di `tap-client.ts` (desain "jangan retry kartu yang benar-benar tidak terdaftar" itu SENGAJA dan BENAR, TIDAK diubah — task ini cuma menambah cara MEMBERSIHKAN sisa entry lama, bukan mengubah logic kapan sesuatu ditandai gagal permanen).

## Acceptance Criteria
- [ ] **PENDING — operasional user** — Langkah manual pembersihan buffer (poin 1) BELUM dilakukan per konfirmasi user 2026-08-10 (ditanya eksplisit sebelum eksekusi). Instruksi langkah manual sudah diberikan ke user (hapus database `absensi-kiosk-buffer` via DevTools di 4 kiosk fisik). Cek ulang badge "Offline" di kiosk fisik + `GET /kiosks` setelah user konfirmasi selesai.
- [x] Tombol "Bersihkan Tap Gagal" tersedia di kiosk-app, HANYA menghapus entry `status === "failed"`, entry `pending` tidak tersentuh.
- [x] Dialog konfirmasi jelas sebelum eksekusi, ringkasan hasil setelah eksekusi.
- [x] Test manual: buat entry `failed` palsu (simulasi lewat browser DevTools/IndexedDB manual insert), verifikasi tombol berhasil membersihkannya TANPA menyentuh entry `pending` lain yang sengaja dibiarkan ada di buffer yang sama.
- [x] Build + type-check `apps/kiosk` hijau.

## Validasi Claudian
- [x] **JANGAN** mengubah logic `PERMANENT_FAILURE_RESULTS`/kapan sesuatu ditandai `failed` — TIDAK diubah, `tap-client.ts` sama sekali tidak disentuh task ini.
- [x] **JANGAN** taruh tombol "Bersihkan Tap Gagal" di layar tap utama yang dilihat siswa — ditaruh di `/deteksi-ip` (halaman teknis existing untuk petugas, T085), bukan layar tap.
- [x] Pastikan tombol ini TIDAK menghapus `localStorage` (token kiosk) — `clearFailedTaps()` HANYA operasi IndexedDB `db.delete("taps", client_uuid)` per-entry `status === "failed"`, tidak menyentuh localStorage sama sekali.
- [x] Verifikasi ke user: DITANYAKAN eksplisit sebelum eksekusi — user konfirmasi BELUM dilakukan, instruksi langkah manual sudah diberikan (lihat Acceptance Criteria poin 1, status PENDING sampai user konfirmasi selesai).

## Status Eksekusi (2026-08-10)

**Fitur (poin 2) selesai.** Poin 1 (pembersihan manual fisik) PENDING — instruksi sudah diberikan ke user, menunggu konfirmasi eksekusi di 4 kiosk fisik.

- **`apps/kiosk/src/lib/offline-buffer.ts`** — fungsi baru `clearFailedTaps()`: filter `status === "failed"`, hapus satu-satu via `db.delete()`, return jumlah yang dihapus. Tidak menyentuh entry `pending`/`synced` sama sekali (filter eksplisit sebelum delete).
- **`apps/kiosk/src/app/deteksi-ip/page.tsx`** — ditambah kartu baru: tampilkan jumlah tap gagal permanen (`getFailedTaps().length`, load saat mount), tombol "Bersihkan Tap Gagal" (disabled kalau 0), klik → dialog konfirmasi inline (bukan `window.confirm`, konsisten pola dialog kartu di halaman ini) dengan teks eksplisit "data ini TIDAK akan dikirim ke server... perlu tap ulang manual" → konfirmasi → `clearFailedTaps()` → ringkasan hasil ("Berhasil membersihkan N tap gagal permanen." / "Tidak ada tap gagal untuk dibersihkan.").
- **Verifikasi live** (browser real, dev kiosk port 3102): seed 2 entry `failed` + 1 entry `pending` langsung ke IndexedDB via DevTools console → reload → counter tampil "2" (benar, tidak ikut hitung `pending`) → klik tombol → dialog konfirmasi tampil teks lengkap → konfirmasi → counter jadi "0", tombol disabled, pesan "Berhasil membersihkan 2 tap gagal permanen." → inspeksi ulang IndexedDB: entry `pending-1` MASIH ADA utuh, kedua entry `failed` sudah tidak ada. ✅
- `tsc --noEmit` `apps/kiosk` bersih.
- Poin 3 (TTL/auto-expire opsional) — **tidak dikerjakan**, di luar scope wajib task ini, tidak ada permintaan eksplisit untuk menambahkannya sekarang.

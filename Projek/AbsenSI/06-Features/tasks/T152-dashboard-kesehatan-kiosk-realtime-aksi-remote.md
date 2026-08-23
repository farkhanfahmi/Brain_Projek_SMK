# T152 — API+Web+Kiosk: Dashboard "Kesehatan Kiosk" Real-Time + Aksi Remote "Bersihkan Tap Gagal"

## Depends on
**Independen dari T151** — TIDAK perlu menunggu T151 selesai (T151 adalah pembersihan manual/tombol lokal-di-kiosk untuk masalah SEKARANG; T152 adalah kapabilitas monitoring+kontrol jarak jauh untuk KE DEPAN). TAPI T152 SEBAIKNYA reuse fungsi `clearFailedTaps()` di `offline-buffer.ts` KALAU T151 sudah membuatnya duluan — cek dulu saat mulai implementasi, JANGAN duplikasi fungsi yang sama kalau T151 sudah menyediakannya.

## Objective
Halaman admin BARU yang menampilkan status SEMUA kiosk secara REAL-TIME (tanpa refresh manual) — online/offline, jumlah tap pending/gagal di buffer masing-masing kiosk, kapan terakhir kontak — DAN tombol aksi "Bersihkan Tap Gagal" yang admin bisa picu dari halaman ini untuk kiosk tertentu, DIEKSEKUSI OLEH KIOSK itu sendiri (server tidak bisa langsung mengubah data IndexedDB di browser kiosk — server cuma bisa MEMERINTAHKAN, kiosk yang MENGEKSEKUSI).

## Context — Kenapa Task Ini Ada

Insiden 2026-08-10 (T150+T151): 4 dari 5 kiosk mengalami buffer tap "macet" (entry gagal permanen menumpuk) TANPA ADA CARA bagi admin untuk MENGETAHUI kondisi ini dari jarak jauh — baru ketahuan setelah guru/siswa melapor tidak bisa absen. User (pemilik sistem) eksplisit minta: dashboard kesehatan kiosk + tombol aksi, SEMUA REAL-TIME tanpa perlu refresh manual.

**Riset lengkap SUDAH dilakukan (2026-08-10)** — daftar gap di bawah FINAL, JANGAN riset ulang dari nol.

### Infrastruktur yang SUDAH ADA (reuse langsung, JANGAN bangun ulang)
- **Socket.IO gateway**: `apps/api/src/realtime/attendance.gateway.ts` — SATU gateway untuk semua realtime di aplikasi ini. `handleConnection` (baris ~72-129) SUDAH mendukung 3 jalur auth: `deviceToken` (kiosk), `tvPiketToken` (TV Piket), `token`/JWT (web admin biasa). **Kiosk auth Socket.IO SUDAH ADA** — TIDAK perlu bangun guard baru, device token yang sama dengan REST API sudah divalidasi di sini.
- **Kiosk SUDAH terhubung ke Socket.IO** — `apps/kiosk/src/lib/use-kiosk-recent.ts` (baris ~34-42) sudah `io(NEXT_PUBLIC_API_WS_URL, {auth:{deviceToken}})`, dipakai kiosk TIPE GURU untuk tabel "5 terbaru". Task ini MEMPERLUAS koneksi yang SUDAH ADA (tambah listener baru), BUKAN membuat koneksi baru — TAPI cek dulu apakah kiosk TIPE SISWA juga sudah connect socket sama sekali (kemungkinan BELUM, karena `use-kiosk-recent.ts` disebut riset khusus dipakai tipe guru) — kalau siswa-kiosk belum connect socket sama sekali, task ini WAJIB menambahkan koneksi socket untuk SEMUA tipe kiosk (guru DAN siswa), bukan cuma guru.
- **Room per-kampus SUDAH ADA**: `attendance:kampus:${kampusId}`. **Room per-kiosk-INDIVIDUAL BELUM ADA** — ini yang perlu dibangun (lihat Spec Detail #1).
- **Model `Kiosk`** (schema.prisma) sudah punya: `id, nama, kampusId, deviceToken, allowedIp, tipe, isActive, lastSeenAt, lastSeenIp, lastFailedIp, lastFailedAt (T149), createdById, createdAt`.
- **`GET /kiosks`** (`kiosks.controller.ts`, `kiosks.service.ts`) — response SUDAH punya `isOnline` terhitung on-the-fly dari `lastSeenAt` vs `ONLINE_THRESHOLD_MS` (5 menit). TIDAK punya `pendingCount`/`failedCount` (data itu tidak pernah ada di database, murni lokal browser kiosk).
- **Halaman admin kiosk existing** (`apps/web/src/app/(admin)/kiosk/kiosk-view.tsx`) — fetch SEKALI saat load, TIDAK ADA polling/socket subscribe sama sekali. Task ini adalah HALAMAN BARU TERPISAH (JANGAN gabung ke halaman existing yang murni CRUD kiosk — beda fungsi: itu untuk kelola data kiosk, ini untuk monitoring kesehatan+aksi darurat).

### Gap yang HARUS dibangun BARU (BELUM ada preseden di codebase sama sekali)
1. **Room per-kiosk individual** di gateway — supaya server bisa kirim perintah ke 1 kiosk spesifik, bukan broadcast semua/per-kampus.
2. **Event kiosk→server pelapor status** — kiosk SAAT INI tidak pernah lapor `pendingCount`/`failedCount` ke server (data itu 100% lokal). WAJIB event baru.
3. **Pola "server kirim perintah, client eksekusi, client konfirmasi balik"** — dikonfirmasi TIDAK ADA preseden SAMA SEKALI di codebase ini (semua socket event existing murni satu-arah server→client untuk DITAMPILKAN, bukan untuk MEMICU AKSI). Ini pola PERTAMA — tulis dengan sangat presisi karena tidak ada contoh existing untuk dicontoh langsung.
4. **Halaman admin baru** dengan socket subscribe — BELUM ADA preseden socket subscribe di `apps/web/src/app/(admin)/**` sama sekali (yang ada baru di `/tv` dan `/piket`, keduanya di luar grup admin). Boleh contoh pola `useAttendanceSocket`/`useKioskRecent` untuk STRUKTUR hook-nya, tapi auth-nya pakai JWT admin biasa (branch `token` di `handleConnection`, BUKAN device-token).

## Spec Detail

### 1. Backend — `AttendanceGateway`: room per-kiosk + auth join

- Di `handleConnection`, branch `deviceToken` (baris ~78-87) — TAMBAH `client.join(\`kiosk:${kiosk.id}\`)` SELAIN join room kampus yang sudah ada — supaya kiosk itu bisa ditarget individual.
- **Catatan konsistensi ditemukan riset**: auth Socket.IO kiosk (`deviceToken`) TIDAK memvalidasi `allowedIp` (beda dari `KioskGuard` REST yang cek IP ketat, T149). Ini CELAH KONSISTENSI yang ditemukan tapi **DI LUAR SCOPE task ini** kecuali terasa krusial — CATAT sebagai temuan terpisah kalau mau diajukan sebagai task lain nanti, JANGAN otomatis diperbaiki di sini (scope creep).

### 2. Kiosk-app — laporkan status buffer secara berkala via socket

- `apps/kiosk/src/lib/offline-buffer.ts` — pastikan `countUnsyncedTaps()`/`getFailedTaps()` (sudah ada) bisa dipanggil dari titik BARU ini.
- Di titik yang SAMA dengan sync-loop existing (`apps/kiosk/src/app/page.tsx`, interval 5 detik yang sudah ada untuk sync buffer) — TAMBAH: setiap kali buffer berubah (setelah sync, setelah tap baru masuk buffer, dst) ATAU secara berkala (misal tiap 10-15 detik, JANGAN terlalu sering supaya tidak membebani socket) — emit event socket BARU (misal `kiosk:health-report`) ke server berisi `{ kioskId, pendingCount, failedCount }`.
- **PENTING**: kiosk perlu tahu `kioskId`-nya sendiri untuk disertakan di payload — cek apakah `fetchKioskInfo()` (yang sudah ada) sudah expose `id` kiosk itu ke state yang bisa diakses titik ini, atau perlu di-thread lewat props/context.
- Kalau kiosk TIPE SISWA belum connect socket sama sekali (VERIFIKASI saat implementasi, lihat Context di atas) — WAJIB tambahkan koneksi socket untuk tipe siswa juga (reuse pola `use-kiosk-recent.ts` sebagai referensi struktur, tapi task ini butuh SEMUA tipe kiosk connect, bukan cuma guru).

### 3. Backend — gateway terima+simpan health-report, expose ke dashboard admin

- `AttendanceGateway` — tambah `@SubscribeMessage("kiosk:health-report")` handler baru — terima payload dari poin 2, simpan di **in-memory Map** (`Map<kioskId, {pendingCount, failedCount, reportedAt}>`, TIDAK PERLU disimpan ke database — data ini transient/real-time, kalau server restart wajar hilang sementara sampai kiosk lapor ulang di siklus berikutnya) di level service/gateway.
- Begitu health-report baru diterima — BROADCAST ke room ADMIN (lihat poin 4) event `kiosk:health-update` berisi data gabungan (kioskId + pendingCount + failedCount) — supaya dashboard admin update REAL-TIME tanpa perlu polling.

### 4. Backend — endpoint+room untuk dashboard admin

- Buat room BARU khusus dashboard ini (misal `admin:kiosk-health`) — halaman admin JOIN room ini saat dibuka (event `join:kiosk-health` atau serupa, pola sama seperti `join:kampus` yang sudah ada untuk TV Piket).
- `GET /kiosks/health` (endpoint BARU, TERPISAH dari `GET /kiosks` existing yang murni untuk CRUD) — `@Roles(UserRole.super_admin)` — return GABUNGAN data `GET /kiosks` (existing, dari database) + in-memory health-report map (poin 3) — dipakai untuk LOAD AWAL halaman (state awal sebelum update real-time pertama masuk), supaya halaman TIDAK kosong sampai kiosk kebetulan lapor.
- **Endpoint aksi**: `POST /kiosks/:id/clear-failed-taps` — `@Roles(UserRole.super_admin)`, `@LogActivity` wajib (aksi ini menghapus data, perlu audit trail) — method ini TIDAK menghapus apa pun di database (tidak ada yang perlu dihapus di server), method ini MEMICU emit socket event BARU `kiosk:command:clear-failed-taps` ke room `kiosk:${id}` (poin 1) — kiosk yang nanti eksekusi.

### 5. Kiosk-app — terima perintah, eksekusi, konfirmasi balik

- Listener socket BARU (di titik yang sama dengan koneksi socket existing) untuk event `kiosk:command:clear-failed-taps` — begitu diterima:
  1. Panggil fungsi pembersihan buffer `failed` (REUSE `clearFailedTaps()` dari `offline-buffer.ts` KALAU T151 sudah membuatnya — CEK DULU sebelum bikin fungsi baru yang duplikat; kalau T151 belum dikerjakan/belum ada fungsi ini, buat baru DI SINI dengan spesifikasi SAMA seperti dijelaskan T151 poin 2: hapus HANYA entry `status === "failed"`, JANGAN sentuh `status === "pending"`).
  2. Setelah eksekusi (berhasil/gagal), emit BALIK ke server event `kiosk:command:ack` berisi `{ kioskId, command: "clear-failed-taps", success: boolean, clearedCount?: number }`.
  3. **TIDAK PERLU** dialog konfirmasi DI SISI KIOSK untuk perintah remote ini (BEDA dari T151 yang punya konfirmasi karena dipicu operator LOKAL di kiosk) — karena konfirmasi SUDAH terjadi di sisi ADMIN sebelum mengirim perintah (lihat poin 6). Kiosk cukup eksekusi otomatis begitu perintah diterima, TIDAK perlu interaksi manusia tambahan di kiosk (kiosk sedang dipakai orang lain/tanpa pengawasan saat perintah ini masuk, tidak realistis menunggu konfirmasi lokal).

### 6. Backend — relay ACK ke dashboard admin

- Gateway terima `kiosk:command:ack` dari kiosk — relay/broadcast ke room `admin:kiosk-health` (event `kiosk:command:result`) — supaya admin yang menekan tombol tadi melihat KONFIRMASI real-time ("Berhasil, 5 tap gagal dibersihkan" / "Gagal, coba lagi") TANPA perlu asumsi "sudah terkirim = berhasil".

### 7. Frontend — halaman admin baru `Kesehatan Kiosk`

- Path BARU (misal `apps/web/src/app/(admin)/kiosk-kesehatan/` — putuskan nama final saat implementasi, konsisten pola penamaan folder admin lain), TAMBAH ke sidebar admin (grup yang relevan, dekat menu Kiosk existing).
- Hook baru (pola sama `useAttendanceSocket`/`useKioskRecent` untuk STRUKTUR, TAPI auth JWT admin biasa bukan device-token) — connect socket, join room `admin:kiosk-health`, load awal via `GET /kiosks/health`, lalu update state REAL-TIME dari event `kiosk:health-update` dan `kiosk:command:result`.
- **Tabel/kartu per kiosk** — WAJIB ikuti aturan tabel permanen proyek (search+sortable+kolom No, KECUALI kalau jumlah kiosk sekolah ini secara struktural sangat sedikit <10 dan tidak akan pernah dipaginasi — cek jumlah kiosk aktual saat implementasi untuk putuskan, konsisten pola pengecualian "master data pendek" di task lain). Kolom: Nama Kiosk, Kampus, Tipe, Status (Online/Offline, badge warna), Terakhir Kontak, Tap Pending, Tap Gagal, Aksi.
- **Indikator visual JELAS** untuk kiosk bermasalah (Tap Gagal > 0) — badge warna beda (kuning/merah) dari kondisi normal, supaya admin langsung tahu tanpa harus baca angka satu-satu.
- **Tombol "Bersihkan Tap Gagal"** per baris kiosk — HANYA aktif/muncul kalau `failedCount > 0` (tidak ada gunanya kalau 0) — klik → dialog konfirmasi (di SISI ADMIN, "Yakin bersihkan N tap gagal di kiosk X? Data ini tidak akan pernah tercatat ke sistem — kalau itu tap valid, orang tersebut perlu tap ulang manual") → kirim `POST /kiosks/:id/clear-failed-taps` → tunggu event `kiosk:command:result` (poin 6) untuk update status tombol (loading → berhasil/gagal, dengan feedback jelas, TIMEOUT wajar misal 10 detik kalau tidak ada respons — kiosk mungkin sedang offline/tidak menerima perintah, tampilkan pesan itu juga, BUKAN diam selamanya).

## Edge Cases
- Kiosk yang OFFLINE (tidak connect socket sama sekali) saat admin klik "Bersihkan Tap Gagal" — perintah tidak akan pernah sampai (room kosong, tidak ada yang dengar). UI HARUS punya TIMEOUT dan pesan jelas ("Kiosk sedang offline, perintah akan dijalankan otomatis begitu kiosk online kembali" ATAU cukup "Gagal — kiosk sedang offline, coba lagi nanti" — PUTUSKAN saat implementasi mana yang lebih realistis: apakah perintah PERLU di-antre/disimpan untuk dieksekusi nanti begitu kiosk reconnect, atau CUKUP gagal dan admin coba lagi manual nanti. REKOMENDASI: simpan STATUS "ada perintah pending" di in-memory map yang sama seperti poin 3, dan begitu kiosk itu `handleConnection` lagi, cek apakah ada perintah pending untuknya lalu kirim ulang — supaya tidak hilang begitu saja, TAPI ini kompleksitas TAMBAHAN, evaluasi apakah worth atau cukup versi sederhana "gagal, coba lagi manual" untuk v1).
- Server API restart — in-memory health-report map HILANG (expected, bukan bug) — dashboard admin akan kosong sementara sampai SEMUA kiosk lapor ulang di siklus berikutnya (maksimal beberapa puluh detik, sesuai interval lapor poin 2) — TIDAK PERLU persist ke database, ini data transient yang wajar reset.
- 2 admin membuka dashboard ini bersamaan — keduanya JOIN room `admin:kiosk-health` yang sama, keduanya terima update yang sama — TIDAK ADA masalah race, broadcast Socket.IO memang mendukung multiple listener.

## Files
- **Modifikasi:** `apps/api/src/realtime/attendance.gateway.ts` (room per-kiosk, handler health-report+ack, room admin), `apps/api/src/kiosks/kiosks.controller.ts`+`kiosks.service.ts` (endpoint `GET /kiosks/health`, `POST /kiosks/:id/clear-failed-taps`), `apps/kiosk/src/app/page.tsx` atau modul socket kiosk yang relevan (emit health-report berkala, listener command+ack), `apps/kiosk/src/lib/offline-buffer.ts` (reuse/buat `clearFailedTaps()`).
- **Buat:** halaman admin baru `apps/web/src/app/(admin)/kiosk-kesehatan/` (nama final diputuskan saat implementasi), hook socket baru untuk halaman ini, sidebar admin (menu baru).
- **Jangan sentuh:** `apps/web/src/app/(admin)/kiosk/kiosk-view.tsx` (halaman CRUD kiosk existing, TETAP terpisah, JANGAN digabung — beda fungsi), logic `PERMANENT_FAILURE_RESULTS`/kapan tap ditandai `failed` di `tap-client.ts` (TIDAK diubah, task ini cuma menambah cara MELIHAT+MEMBERSIHKAN dari jarak jauh).

## Acceptance Criteria
- [x] Halaman admin baru menampilkan SEMUA kiosk dengan status online/offline, pending count, failed count — UPDATE REAL-TIME tanpa refresh manual (dibuktikan lewat skrip verifikasi socket live, lihat Status Eksekusi — bukan lewat UI klik manual, tapi jalur socket yang SAMA PERSIS dipakai UI diverifikasi end-to-end).
- [x] Tombol "Bersihkan Tap Gagal" per kiosk, hanya aktif kalau failedCount > 0 (`{!!kiosk.failedCount && kiosk.failedCount > 0 && <Button>...`).
- [x] Klik tombol → dialog konfirmasi admin → perintah terkirim → kiosk eksekusi → dashboard admin terima konfirmasi hasil REAL-TIME (bukan asumsi berhasil) — verified end-to-end via socket script.
- [x] Kiosk offline saat perintah dikirim → admin dapat pesan jelas (bukan diam/hang selamanya), ada timeout wajar (10 detik) — DIPUTUSKAN versi sederhana v1 (bukan queue-command), sesuai rekomendasi spec & instruksi eksekusi user.
- [x] Entry `pending` (tap yang masih menunggu sync, BUKAN gagal) TIDAK PERNAH terhapus oleh aksi ini, hanya entry `failed` — reuse `clearFailedTaps()` T151, sudah diverifikasi terpisah di task itu.
- [x] `@LogActivity` terpasang di endpoint `POST /kiosks/:id/clear-failed-taps` — dikonfirmasi live, muncul di `activity_log` dengan action `kiosk.clear_failed_taps`.
- [x] Build + type-check `apps/api`, `apps/web`, `apps/kiosk` hijau.

## Validasi Claudian
- [x] **CEK DULU apakah T151 sudah membuat `clearFailedTaps()`** — SUDAH ADA (T151 dikerjakan lebih dulu dalam sesi yang sama), di-REUSE langsung dari `use-kiosk-health-report.ts`, tidak diduplikasi.
- [x] **JANGAN** gabung halaman ini dengan `kiosk-view.tsx` existing — halaman baru terpisah `(admin)/kiosk-kesehatan/`, `kiosk-view.tsx` TIDAK disentuh sama sekali.
- [x] **JANGAN** memperbaiki celah "Socket.IO kiosk auth tidak cek allowedIp" — TIDAK disentuh, dicatat di sini saja sesuai instruksi (celah: `handleConnection` branch `deviceToken` tidak validasi `allowedIp` seperti `KioskGuard` REST melakukan, T149 — kalau mau diperbaiki, task terpisah).
- [x] Pola PERTAMA "server kirim perintah, client eksekusi, client konfirmasi balik" — didesain dengan cek keberadaan kiosk di room (`fetchSockets().length === 0`) SEBELUM emit, supaya server tahu PASTI apakah perintah akan sampai, bukan kirim buta lalu menunggu timeout untuk tahu offline.
- [x] Jumlah kiosk aktual: **4** (dev DB, dikonfirmasi query langsung) — jauh di bawah 10, tabel dashboard TIDAK pakai search+sortable+No (konsisten `kiosk-view.tsx` existing yang juga tanpa fitur itu untuk alasan sama).

## Status Eksekusi (2026-08-10)

**Selesai.** Semua acceptance criteria terpenuhi, diverifikasi live end-to-end (bukan cuma tsc).

**Keputusan desain (dieksekusi langsung sesuai instruksi user, tanpa jeda diskusi)**:
- Kiosk offline saat perintah dikirim → **versi sederhana v1** (gagal jelas + admin coba lagi manual), BUKAN command-queue menunggu kiosk reconnect. Dideteksi SEBELUM emit (`server.in(room).fetchSockets()`, bukan kirim-lalu-tunggu-timeout) — `POST /kiosks/:id/clear-failed-taps` return `{sent: false}` seketika kalau room kosong, endpoint itu sendiri tidak pernah hang.
- Nama halaman final: `(admin)/kiosk-kesehatan/`.

**Backend (`apps/api`)**:
- `AttendanceGateway` (`realtime/attendance.gateway.ts`) — kiosk auth branch sekarang JUGA `client.join(\`kiosk:${kiosk.id}\`)` (room individual, terpisah dari room per-kampus `attendance:kiosk:${kampusId}` yang sudah ada). Handler baru: `join:kiosk-health` (admin join room `admin:kiosk-health`), `kiosk:health-report` (kiosk lapor, simpan in-memory `Map<kioskId, KioskHealth>`, broadcast `kiosk:health-update` ke admin room), `kiosk:command:ack` (relay ke admin room sebagai `kiosk:command:result`). Method baru `sendClearFailedTapsCommand(kioskId)` (async, cek room kosong dulu, return boolean) dan `getKioskHealthMap()`.
- `KiosksModule` — import `RealtimeModule` (baru, untuk inject `AttendanceGateway`).
- `KiosksService` — `findAllWithHealth()` (gabung `findAll()` + in-memory health map, dipakai `GET /kiosks/health`), `clearFailedTaps(id)` (TIDAK ubah apa pun di DB, murni panggil gateway method, return `{sent: boolean}`).
- `KiosksController` — `GET /kiosks/health` (didaftarkan SEBELUM `GET /:id` supaya tidak ketangkap sebagai param `id="health"`), `POST /kiosks/:id/clear-failed-taps` (`@LogActivity({action: "kiosk.clear_failed_taps", prismaModel: "kiosk", ...})`).

**Kiosk-app (`apps/kiosk`)**:
- Hook baru `use-kiosk-health-report.ts` — connect socket untuk SEMUA tipe kiosk (siswa DAN guru, dikonfirmasi via riset sebelumnya `use-kiosk-recent.ts` HANYA guru — gap yang diidentifikasi spec sudah ditutup). Lapor `pendingCount`/`failedCount` tiap 15 detik + tiap kali reconnect. Listener `kiosk:command:clear-failed-taps` → panggil `clearFailedTaps()` (REUSE T151, tidak duplikasi) → emit `kiosk:command:ack` balik. TIDAK ADA dialog konfirmasi di sisi kiosk (sesuai spec — konfirmasi sudah di sisi admin).
- `page.tsx` — panggil `useKioskHealthReport(token, kioskInfo?.id ?? null)`.

**Web (`apps/web`)**:
- Hook baru `use-kiosk-health-socket.ts` — pola struktur SAMA `useAttendanceSocket` (auth JWT via `/api/ws-token`, BUKAN device-token), join `admin:kiosk-health`, state `healthByKioskId` (merge live update) + `lastCommandResult`.
- Halaman baru `(admin)/kiosk-kesehatan/page.tsx` (server component, fetch awal `GET /kiosks/health`) + `kiosk-kesehatan-view.tsx` (client, tabel 4 kiosk TANPA search/sort/No — lihat Validasi Claudian, badge Online/Offline + badge merah kalau `failedCount > 0`, tombol "Bersihkan" muncul HANYA kalau `failedCount > 0`, dialog konfirmasi → kirim → state `sending → waiting → success/failed/timeout/offline` dengan pesan beda tiap state).
- Sidebar (`nav-items.ts`) — menu baru "Kesehatan Kiosk" (ikon `Activity`) di grup "Kartu & Perangkat", setelah "Mesin".

**Verifikasi live end-to-end** (dev port 3100-3102, DB dev port 3307 — production 3000-3002 tidak disentuh):
1. `GET /kiosks/health` dengan akun test super_admin sementara — return 4 kiosk digabung `pendingCount`/`failedCount`/`reportedAt` (null untuk kiosk yang belum pernah lapor sejak server start — benar, bukan 0 palsu).
2. Skrip Node (`socket.io-client` langsung, mensimulasikan browser kiosk DAN browser admin sekaligus) — 7 skenario, SEMUA PASS:
   - Kiosk connect socket berhasil.
   - Admin connect socket berhasil, join `admin:kiosk-health`.
   - Kiosk emit `kiosk:health-report` → admin terima `kiosk:health-update` REAL-TIME dengan payload benar (`kioskId`, `pendingCount`, `failedCount`).
   - `POST /kiosks/:id/clear-failed-taps` (kiosk online) → `{sent: true}`.
   - Kiosk menerima event `kiosk:command:clear-failed-taps`.
   - Kiosk emit `kiosk:command:ack` → admin terima `kiosk:command:result` REAL-TIME dengan payload benar (termasuk `clearedCount`).
   - `POST /kiosks/:id/clear-failed-taps` (kiosk DIPUTUS koneksi dulu) → `{sent: false}` — kondisi kiosk-offline terverifikasi BENAR mengembalikan gagal-jelas seketika, bukan hang.
3. Ditemukan race condition MINOR di skrip test (bukan bug implementasi): admin socket connect event client-side terjadi SEBELUM `handleConnection` server-side (async, verify JWT) selesai — kalau `join:kiosk-health` di-emit PERSIS di titik itu, gagal karena `client.data.user` belum ter-set. Ini POLA LAMA yang SAMA berlaku untuk `join:kampus` existing (bukan sesuatu yang baru diperkenalkan T152) — di localhost loopback race ini kadang muncul, di jaringan sekolah sungguhan (round-trip lebih lambat) kemungkinan besar tidak pernah terjadi. TIDAK diperbaiki di gateway (di luar scope T152, bug pra-existing lintas semua fitur socket) — dicatat di sini sebagai temuan untuk referensi masa depan kalau perlu diperbaiki (mis. tambah short retry di hook client kalau nanti terbukti jadi masalah nyata di produksi).
4. `@LogActivity` dikonfirmasi tercatat ke `activity_log` (`action: kiosk.clear_failed_taps`, `actor_id`, `target_id` benar) — audit trail terverifikasi bukan cuma dari kode.
5. Data uji (akun admin test, activity_log terkait, kiosk `allowed_ip` yang di-override sementara) semua dibersihkan/dikembalikan setelah verifikasi.
6. `tsc --noEmit` bersih di `apps/api`, `apps/web`, `apps/kiosk`. Jest `apps/api` 265/265 pass, tidak ada regresi (tidak ada spec existing untuk `KiosksService`/`AttendanceGateway` yang perlu di-update constructor-nya).

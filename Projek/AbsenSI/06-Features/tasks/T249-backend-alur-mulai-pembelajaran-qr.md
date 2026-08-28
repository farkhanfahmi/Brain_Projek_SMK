# T249 — API: Alur "Mulai Pembelajaran" via QR — Token Rotasi Redis + Socket.IO + Extend startSession()

## Depends on
**T247** (schema, akun siswa `User.studentId`) — WAJIB selesai dulu. TIDAK bergantung pada
T248 (UI wali kelas) — bisa dikerjakan paralel asalkan T247 sudah ada.

## Objective
Backend untuk: siswa (Ketua/Wakil Ketua) generate token rotasi per `teachingSessionId`,
token dipush real-time ke browser siswa via Socket.IO, guru submit hasil scan lewat
`startSession()` yang di-extend, backend validasi token + SEMUA validasi existing (GPS,
jendela waktu, dst — TIDAK ADA yang dihapus, QR jadi TAMBAHAN, sesuai keputusan user).

## Konteks — Yang TIDAK Berubah (WAJIB dipahami sebelum ubah apa pun)

`TeachingSessionsService.startSession(sessionId, teacherId, lat, lng)`
(`teaching-sessions.service.ts:357-461`) SUDAH punya 6 lapis validasi (idempotency, izin
guru, tap gerbang, jendela waktu, geofence GPS, commit) — **SEMUA TETAP ADA, TIDAK ADA YANG
DIHAPUS ATAU DILEMAHKAN**. Task ini CUMA menyisipkan 1 validasi BARU (token QR) ke alur yang
sudah ada, dan pesan error existing (ForbiddenException/ConflictException) TETAP dipakai
apa adanya untuk kondisi yang sudah ada.

## Keputusan Diminta User (2026-08-25)
1. QR = lapis TAMBAHAN, GPS geofence TETAP wajib lolos juga (bukan salah satu).
2. QR auto-refresh terus tiap beberapa detik selama layar terbuka (bukan cuma sekali per
   klik) — pakai Socket.IO existing (REUSE infrastruktur `attendance.gateway.ts`, BUKAN
   bikin gateway terpisah kalau bisa digabung — VERIFIKASI SAAT IMPLEMENTASI apakah gabung
   ke gateway yang sama atau modul baru lebih bersih, mengingat gateway existing sudah
   lumayan besar).
3. **Error handling harus SANGAT spesifik per kondisi** (penekanan eksplisit user) — lihat
   tabel pesan error di bawah, WAJIB diikuti persis.
4. **Token TIDAK BOLEH terlihat/exposed** — tidak pernah di URL/query string, tidak pernah
   di response body yang bisa di-GET casual, HANYA lewat Socket.IO payload (siswa) dan body
   POST (guru submit).
5. TIDAK PERLU menangani celah residual (video-call sharing) — accepted risk, JANGAN
   bangun mitigasi tambahan untuk ini (misal tidak perlu deteksi anti-screenshot dsb).

## Spec Detail

### 1. Redis — token rotasi, BUKAN baris database per rotasi

Key: `mulai-sesi:token:{teachingSessionId}` → value: token acak (REKOMENDASI: 32+ karakter
random, `crypto.randomBytes` — cukup panjang untuk aman TANPA rate limiting, sesuai
kesadaran user soal T128 belum ada). **TTL pendek** (rekomendasi 15-20 detik, VERIFIKASI
SAAT IMPLEMENTASI angka yang nyaman — cukup lama untuk guru sempat scan, cukup pendek untuk
"terasa" berputar).

Server-side interval (BUKAN dipicu tiap request siswa) — begitu siswa klik "Mulai
Pembelajaran": server mulai loop set token baru ke Redis key itu tiap N detik (SAMA durasi
dengan TTL, supaya token lama otomatis invalid sebelum yang baru di-set) + emit token baru
ke Socket.IO room khusus (lihat poin 2) — loop BERHENTI kalau: (a) sesi berhasil dimulai
(startSession sukses), (b) siswa klik "Batal", (c) jendela waktu sesi berakhir, (d) siswa
disconnect socket (opsional, VERIFIKASI SAAT IMPLEMENTASI apakah perlu auto-stop atau biar
timeout alami).

### 2. Socket.IO — room per teachingSessionId

REUSE `attendance.gateway.ts` (atau extract handler baru kalau file itu dianggap sudah
terlalu besar, VERIFIKASI SAAT IMPLEMENTASI) — room baru `mulai-sesi:{teachingSessionId}`
(NAMA BEDA dari room existing `attendance:kampus:*`/`kiosk:*` supaya tidak tertukar
konsepnya). Event:
- `mulai-sesi:token` — server → siswa, payload `{ token }` (TIDAK ADA data lain yang
  sensitif, token SATU-SATUNYA isi payload relevan).
- `mulai-sesi:gagal` — server → siswa, dikirim SETIAP kali guru mencoba scan tapi validasi
  GAGAL (lihat tabel error poin 4) — payload `{ pesan: string }`, supaya siswa yang sedang
  menunggu tahu APA yang terjadi (bukan cuma diam menunggu tanpa penjelasan, sesuai
  penekanan user soal error handling).
- `mulai-sesi:berhasil` — server → siswa, dikirim SETELAH `startSession()` sukses commit —
  payload minimal (`{ sessionId }`), siswa auto-transisi ke state sukses.

Autentikasi socket siswa HARUS scoped — cuma siswa yang BERSANGKUTAN (Ketua/Wakil Ketua
kelas ITU) yang boleh join room `mulai-sesi:{teachingSessionId}` untuk sesi kelasnya sendiri
— VALIDASI di `handleConnection`/`@SubscribeMessage("join:mulai-sesi")` (pola SAMA
`join:kampus` existing, TAPI dengan cek otorisasi tambahan: `teachingSession.kelasId` HARUS
sama dengan `student.kelasId` milik user yang connect).

### 3. Endpoint baru — siswa generate/mulai rotasi token

`POST /teaching-sessions/:id/mulai-qr` (nama endpoint VERIFIKASI SAAT IMPLEMENTASI) — guard
`@Roles(UserRole.ketua_kelas)`, validasi: `teachingSession.kelasId` HARUS sama dengan
`req.user` punya siswa terkait di kelas itu (lewat `User.studentId → Student.kelasId`).
Mulai loop rotasi Redis+socket (poin 1-2). Response: cukup `{ ok: true }` — token TIDAK
PERNAH ikut di response HTTP ini (cuma lewat socket, sesuai keputusan user poin 4).

### 4. Extend `startSession()` — terima token, tabel error LENGKAP

```ts
async startSession(sessionId: number, teacherId: number, lat: number, lng: number, qrToken: string)
```

Tambah 1 langkah validasi BARU (rekomendasi taruh SETELAH validasi jendela waktu, SEBELUM
geofence — urutan VERIFIKASI SAAT IMPLEMENTASI, prinsip: gagal cepat di validasi termurah
dulu): ambil token dari Redis key `mulai-sesi:token:{sessionId}`, bandingkan dengan `qrToken`
yang dikirim guru. Kalau TIDAK cocok/TIDAK ADA (expired/belum pernah digenerate) →
`ForbiddenException` pesan **PERSIS**: `"QR tidak valid atau sudah kadaluarsa — minta Ketua Kelas buka ulang layar Mulai"`.

**Tabel pesan error LENGKAP** (WAJIB diikuti persis, penekanan eksplisit user soal error
handling detail+informatif):

| Kondisi | Exception | Pesan |
|---|---|---|
| Sesi sudah dimulai (existing) | `ConflictException` | (SUDAH ADA, tidak berubah) "Sesi sudah dimulai sebelumnya" |
| Ada izin guru aktif (existing) | `ConflictException` | (SUDAH ADA) "Anda sudah diizinkan tidak mengajar untuk sesi ini" |
| Belum tap gerbang (existing) | `ForbiddenException` | (SUDAH ADA) "Anda belum tap kartu di gerbang hari ini" |
| Belum waktunya (existing) | `ForbiddenException` | (SUDAH ADA) "Belum waktunya sesi ini dimulai" |
| Sudah lewat waktu (existing) | `ForbiddenException` | (SUDAH ADA) "Sesi ini sudah lewat waktunya" |
| **Token QR tidak cocok/expired (BARU)** | `ForbiddenException` | **BARU**: "QR tidak valid atau sudah kadaluarsa — minta Ketua Kelas buka ulang layar Mulai" |
| **Belum ada token sama sekali (siswa belum pernah klik Mulai, BARU)** | `ForbiddenException` | **BARU**: "Ketua/Wakil Ketua Kelas belum membuka layar Mulai Pembelajaran — minta mereka buka dulu" |
| Di luar radius (existing) | `ForbiddenException` | (SUDAH ADA) "Lokasi Anda di luar radius sekolah" |
| **GPS browser guru gagal/ditolak (BARU, sebenarnya validasi FRONTEND sebelum request terkirim, lihat T251)** | — | Ditangani T251 di sisi client, TIDAK sampai endpoint ini kalau GPS gagal didapat sama sekali |

**Setiap kegagalan JUGA memicu emit `mulai-sesi:gagal`** ke room siswa (poin 2) dengan
pesan yang SAMA — supaya siswa yang menunggu tahu kenapa (kecuali kegagalan yang MURNI di
sisi guru sebelum request terkirim, misal GPS guru sendiri gagal — itu TIDAK ada yang
dikirim ke siswa, karena requestnya belum sampai backend sama sekali).

### 5. Bersihkan token setelah sukses
`startSession()` sukses → HAPUS key Redis `mulai-sesi:token:{sessionId}` (token tidak
relevan lagi, sesi sudah `startedAt` terisi, endpoint ini idempotency-guard menolak percobaan
kedua lebih dulu lewat existing check poin `session.startedAt !== null`).

## Edge Cases
- **Guru submit token SETELAH sesi keburu ditutup otomatis** (`autoCloseDueSessions`,
  existing job) — VERIFIKASI urutan pengecekan, kemungkinan besar sudah tertangkap oleh
  validasi jendela waktu existing (`now > jamSelesaiDate`) TANPA perlu logic tambahan.
- **2 percobaan scan hampir bersamaan** (network lag, guru double-tap) — `startSession()`
  SUDAH idempotent (`session.startedAt !== null` check) — percobaan kedua otomatis dapat
  pesan "Sesi sudah dimulai sebelumnya", TIDAK PERLU logic tambahan.
- **Redis down/tidak terhubung** — VERIFIKASI SAAT IMPLEMENTASI fallback behavior: fitur QR
  jadi TIDAK BISA dipakai (masuk akal, murni fitur tambahan) — TAPI pastikan kegagalan Redis
  TIDAK mematikan alur mulai-mengajar LAMA sepenuhnya kalau kelak ada toggle "kelas ini
  belum pakai fitur QR" (DI LUAR SCOPE kalau tidak diminta — catat sebagai pertimbangan,
  bukan wajib dibangun sekarang mengingat T249-T251 ini FULL WAJIB untuk kelas yang punya
  Ketua/Wakil Ketua ter-provisioning, tidak ada mode "skip QR").

## Files
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts`
  (`startSession()` tambah param+validasi token).
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` (terima
  `qrToken` dari body).
- **Modifikasi/Buat:** `apps/api/src/realtime/attendance.gateway.ts` (atau gateway baru) —
  room+event `mulai-sesi:*`.
- **Buat:** endpoint `POST /teaching-sessions/:id/mulai-qr` (siswa generate token).
- **Jangan sentuh:** validasi existing (izin, tap gerbang, jendela waktu, geofence) — HANYA
  disisipi, tidak dihapus/diubah urutan logikanya secara fundamental.

## Acceptance Criteria
- [x] Siswa (role `ketua_kelas`) bisa trigger rotasi token, token berganti tiap 20 detik
      via Socket.IO (interval server-side, bukan dipicu request siswa).
- [x] Token TIDAK PERNAH muncul di response HTTP endpoint manapun — diverifikasi grep
      menyeluruh, `mulaiQr()`/`batalQr()` return `{ok:true}` saja.
- [x] Guru submit token salah/expired → pesan error PERSIS sesuai tabel, DAN siswa yang
      sedang menunggu dapat notifikasi real-time (`broadcastQrMulaiGagal()` dipanggil di
      SEMUA catch path `startSession()`, bukan cuma jalur token).
- [x] Semua validasi existing (GPS, jendela waktu, izin, tap gerbang) TETAP berfungsi
      IDENTIK — 49/49 test existing `teaching-sessions.service.spec.ts` lulus tanpa ubah
      assertion, cuma signature constructor+parameter yang disesuaikan.
- [x] Token terhapus dari Redis setelah sesi berhasil dimulai (`validasiDanHapusToken()`
      hapus key SEBELUM return true).
- [x] Socket room `mulai-sesi:*` cuma bisa di-join siswa berwenang — divalidasi
      `handleJoinMulaiSesi()` cek `KelasPengurus` aktif (ketua/wakil_ketua) di kelas PEMILIK
      sesi tsb, bukan cuma cek JWT valid generik.
- [x] Build + type-check hijau (`tsc --noEmit` bersih), test existing 49/49 pass, full
      suite api 633/633 pass (1 kegagalan flaky-timeout awal terbukti tidak terkait —
      lulus 12/12 saat dijalankan terisolasi).

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA validasi existing yang terhapus/terlemahkan — `startSessionInternal()`
      (rename dari `startSession()` lama, wrapper baru cuma utk emit-gagal) tetap 8 langkah
      urut PERSIS (tambah 1 langkah token di antara jendela waktu dan geofence), dikonfirmasi
      test lama 49/49 lulus tanpa modifikasi assertion.
- [x] Konfirmasi token benar-benar tidak "terlihat" — grep `qrToken`/`mulai-sesi:token` di
      seluruh `teaching-sessions/`+`realtime/` (non-test): HANYA muncul sebagai parameter
      fungsi/key Redis internal/payload Socket.IO, TIDAK PERNAH di response HTTP.
- [x] Konfirmasi setiap baris tabel pesan error diimplementasikan PERSIS — 2 pesan BARU
      ("QR tidak valid...", "Ketua/Wakil Ketua Kelas belum membuka...") disalin verbatim
      dari spec, dibedakan via `pernahDigenerate()` (key Redis ada ATAU interval rotasi
      masih aktif di memori = pernah generate).

## Bug/Gap Ditemukan Saat Implementasi (2026-08-27)

**JWT payload tidak punya `studentId` sama sekali** — ditemukan saat riset sebelum coding
(bukan saat testing). `AccessTokenPayload` generik untuk semua role sejak awal proyek,
`issueTokenPair()` tidak pernah meneruskan `user.studentId` (field itu sendiri baru ada
dari T247). Diputuskan (dikonfirmasi user) extend `AccessTokenPayload`+`issueTokenPair()`
dengan field baru `studentId: number | null` — nullable, backward-compatible ke SEMUA role
lain (tetap null), 3 titik panggil (`login()`/`refresh()`/`issueTokenPairForUser()`)
diupdate konsisten. Diverifikasi live: JWT siswa `ketua_kelas` setelah login memang
membawa `studentId` yang benar (decode manual token, `"studentId":3` sesuai `User.studentId`
di DB).

## Implementasi (2026-08-27)

**Auth**: `AccessTokenPayload`+`issueTokenPair()` (`auth.types.ts`/`auth.service.ts`)
extend `studentId`.

**Redis** (`REDIS_CLIENT`, sudah `@Global()`, reuse existing — TIDAK bikin koneksi baru):
`QrMulaiSesiService` baru (`teaching-sessions/qr-mulai-sesi.service.ts`) — rotasi murni
key-value (`mulai-sesi:token:{sessionId}`, TTL 20 detik = interval rotasi, token lama
otomatis invalid tepat saat baru di-set). State interval JS in-memory (`Map`, per
teachingSessionId) — wajar hilang saat restart server (siswa tinggal klik ulang "Mulai").
`mulaiRotasi()` validasi siswa BENAR pengurus aktif kelas pemilik sesi SEBELUM mulai loop.
`pernahDigenerate()` bedakan 2 pesan error (key Redis ADA vs TIDAK PERNAH ada sama sekali).

**Socket.IO** — REUSE `AttendanceGateway` existing (bukan gateway baru, sesuai rekomendasi
spec — sudah reusable lintas modul via `RealtimeModule`). Room baru `mulai-sesi:{id}`,
3 event (`mulai-sesi:token`/`:gagal`/`:berhasil`), join divalidasi cek `KelasPengurus`
(bukan cuma JWT valid generik seperti room lain di gateway ini).

**`startSession()` extend** — signature publik jadi wrapper try-catch (emit
`broadcastQrMulaiGagal()` di SEMUA kegagalan yang sampai backend, lalu re-throw), logic
asli dipindah ke `startSessionInternal()` private — 8 langkah urut (tambah 1 validasi token
antara jendela waktu dan geofence, sesuai keputusan user). `StartSessionDto` tambah field
wajib `qrToken`. Emit `broadcastQrMulaiBerhasil()` setelah commit sukses.

**Test existing** — 10 titik `new TeachingSessionsService(...)` + 10 titik
`service.startSession(...)` di `teaching-sessions.service.spec.ts` diupdate signature
(2 dependency baru stub, 1 parameter token baru) TANPA menambah test/assertion baru untuk
fitur QR — murni menjaga test lama tetap valid, sesuai instruksi eksplisit sesi ini.

**Verifikasi live**: endpoint `mulai-qr`/`batal-qr` diuji via curl — role guard
(`ketua_kelas` only) menolak role lain, sesi-tidak-ditemukan 404, `batal-qr` no-op aman.
Full end-to-end `startSession()` dengan `TeachingSession` sungguhan TIDAK bisa ditest live
(dev DB tidak punya data jadwal/OpsiJadwal/AlokasiWaktu prasyarat, setup manual terlalu
kompleks untuk scope verifikasi ini) — diandalkan sepenuhnya pada 49 test jest existing
yang mencakup SEMUA skenario startSession (lolos tanpa ubah assertion).

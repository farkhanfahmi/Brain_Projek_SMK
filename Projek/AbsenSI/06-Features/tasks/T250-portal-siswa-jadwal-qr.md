# T250 — Web: Portal Siswa (Ketua/Wakil Ketua) — Login + Jadwal Hari Ini + Layar QR

## Depends on
**T247** (akun `User.studentId`, role `ketua_kelas`) dan **T249** (backend token+socket) —
WAJIB keduanya selesai dulu.

## Objective
Portal baru, SANGAT SEMPIT scope-nya — login siswa (Ketua/Wakil Ketua) lihat jadwal
pelajaran hari itu (adaptif mode normal/blok, REUSE resolusi jadwal existing), klik "Mulai
Pembelajaran" pada 1 mapel → tampil QR yang auto-berputar sampai guru berhasil scan.

## Spec Detail — UI/UX (sudah disepakati dengan user 2026-08-25)

### 1. Routing — REUSE app+login existing, BUKAN app terpisah
Route group baru `apps/web/src/app/(siswa)/` (pola SAMA `(guru)`/`(piket)`/`(admin)` yang
sudah ada) — login page yang SUDAH ADA (`/login`) TIDAK PERLU halaman baru, cuma REDIRECT
setelah login sukses ditambah cabang untuk `role === "ketua_kelas"` → `/siswa` (VERIFIKASI
SAAT IMPLEMENTASI lokasi persis logic redirect-by-role saat ini, ikuti pola yang sudah ada
untuk role lain).

### 2. Halaman utama — "Jadwal Hari Ini"
List card per jam pelajaran hari itu — REUSE endpoint/logic resolusi jadwal yang SUDAH ADA
untuk guru (`getMyToday()` atau serupa di `teaching-sessions.service.ts`) TAPI di-scope ke
KELAS siswa ini (bukan ke guru) — kemungkinan perlu endpoint BARU sisi read (`GET
/teaching-sessions/kelas-hari-ini` atau serupa, scoped `req.user.studentId → student.kelasId`)
karena endpoint existing di-scope per TEACHER, bukan per KELAS — VERIFIKASI SAAT
IMPLEMENTASI apakah bisa reuse query yang sama dengan where-clause beda, atau perlu method
baru di service (REKOMENDASI: method baru yang REUSE resolusi jam/mapel yang sama, supaya
konsisten mode normal/blok tanpa duplikasi logic).

Tiap card: jam, mapel, nama guru, badge status (Belum Dimulai / Sedang Berlangsung /
Selesai / Terlewat — WARNA KONSISTEN token existing, lihat `STATUS_BADGE` di
`sesi-card.tsx` guru sebagai referensi palet).

Card dengan status "Belum Dimulai" DAN dalam jendela waktu → tombol **"Mulai Pembelajaran"**.

### 3. Layar QR (setelah klik Mulai)
- Transisi ke tampilan QR besar di tengah kartu/layar (VERIFIKASI SAAT IMPLEMENTASI modal
  vs full-card vs halaman terpisah — REKOMENDASI: ganti isi card itu sendiri jadi tampilan
  QR, TIDAK PERLU modal/navigasi halaman baru, supaya transisi terasa instan).
- QR di-generate dari `token` yang diterima lewat Socket.IO (subscribe room
  `mulai-sesi:{teachingSessionId}` begitu masuk state ini) — dependency baru: library
  generate QR dari string (rekomendasi `qrcode` npm package, ringan, tanpa canvas server-side
  diperlukan karena render di browser).
- Indikator visual "live" (ring berdenyut/animasi halus di sekeliling QR) — SEDERHANA,
  TIDAK PERLU animasi kompleks, cukup `animate-pulse` Tailwind atau setara.
- Teks instruksi: `"Tunjukkan ke [Nama Guru] untuk memulai"`.
- Tombol "Batal" kecil — klik → kirim signal stop rotasi ke backend (endpoint/socket event
  VERIFIKASI SAAT IMPLEMENTASI, minimal: berhenti subscribe + beri tahu backend hentikan
  loop token supaya tidak sia-sia jalan terus), kembali ke list jadwal.

### 4. Real-time feedback (WAJIB, penekanan user soal error informatif)
- Event `mulai-sesi:gagal` (dari T249) diterima → tampilkan pesan singkat DI LAYAR QR ITU
  JUGA (jangan cuma toast yang hilang cepat — siswa perlu tahu APA yang terjadi sambil QR
  tetap terbuka menunggu percobaan berikutnya), contoh: **"Guru mencoba mulai tapi lokasi
  GPS di luar radius sekolah — coba lagi setelah guru mendekat"**. Pesan ini AMBIL LANGSUNG
  dari payload socket (`pesan`), JANGAN di-generic-kan ulang di frontend (aturan wajib
  project: pesan backend diteruskan apa adanya).
- Event `mulai-sesi:berhasil` diterima → transisi ke state sukses singkat (`✓ Pelajaran
  Dimulai`), lalu otomatis kembali ke list dengan card itu ber-status "Sedang Berlangsung".
- **Jendela waktu sesi berakhir SELAGI layar QR terbuka** — auto-cancel, pesan jelas: "Waktu
  sesi ini sudah berakhir, tidak bisa dimulai lagi", kembali ke list (card sekarang
  "Terlewat").

### 5. Scope akses — SEMPIT, JANGAN lebih dari yang diminta
Portal ini **HANYA** halaman Jadwal Hari Ini + layar QR — TIDAK ADA menu nilai, absensi
pribadi, riwayat, atau apa pun yang tidak eksplisit diminta. `RolesGuard`/middleware pastikan
akun `ketua_kelas` TIDAK BISA akses route grup lain (`(admin)`/`(guru)`/`(piket)` dst) —
REPLIKASI pola proteksi role existing.

### 6. Mobile-first WAJIB
Ini portal yang HAMPIR PASTI diakses dari HP siswa (bukan laptop) — rancang mobile dulu
secara ketat: card jadwal full-width, QR besar dan jelas terlihat di layar kecil (minimal
ukuran yang gampang di-scan dari jarak wajar), tombol besar mudah di-tap.

## Edge Cases
- **Siswa buka portal ini tapi HARI ITU BUKAN hari sekolah** (libur/weekend) — pesan jelas
  "Hari ini bukan hari sekolah", REPLIKASI pola pesan serupa yang sudah ada di
  `hari-ini-tab.tsx` wali kelas.
- **Siswa belum tap kartu di kiosk hari itu** — REKOMENDASI tampilkan info di halaman jadwal
  ("Anda belum tap kartu hari ini") TAPI ini validasi milik GURU (guru yang butuh tap
  gerbang, bukan siswa) — VERIFIKASI apakah relevan ditampilkan di sisi siswa sama sekali,
  kemungkinan TIDAK relevan (siswa generate QR tidak butuh siswa itu sendiri sudah tap).
- **Akun ketua_kelas login tapi user itu SUDAH TIDAK jadi pengurus** (diganti wali kelas,
  akun harusnya nonaktif per T248) — kalau somehow masih bisa login (race condition), tolak
  akses jadwal dengan pesan jelas, JANGAN biarkan tampil data usang.
- **2 tab/device terbuka bersamaan pakai akun yang sama** (wajar kalau kredensial dioper
  antar pengurus sesuai keputusan user) — TIDAK PERLU dicegah, kedua tab akan terima event
  socket yang sama (broadcast ke room, bukan ke 1 koneksi spesifik).

## Files
- **Buat:** `apps/web/src/app/(siswa)/` — layout, halaman jadwal, komponen layar QR.
- **Buat/Modifikasi:** endpoint backend read "jadwal hari ini per kelas" (lihat poin 2).
- **Modifikasi:** logic redirect-by-role setelah login (tambah cabang `ketua_kelas`).
- **Modifikasi:** `apps/web/package.json` (tambah library generate QR, mis. `qrcode`).
- **Modifikasi:** middleware/guard proteksi role (pastikan `ketua_kelas` scoped ketat).

## Acceptance Criteria
- [x] Siswa dengan akun `ketua_kelas` bisa login, redirect ke portal ini (bukan dashboard
      role lain). **Catatan**: path final `/portal-siswa`, BUKAN `/siswa` seperti draf awal
      spec — `/siswa` sudah dipakai halaman Data Murid admin (Next.js menolak build karena
      2 route resolve ke path sama), dikonfirmasi user pilih `/portal-siswa`.
- [x] Jadwal hari ini tampil benar sesuai mode normal/blok kelasnya — REUSE `resolveJamSesi()`
      persis sama dengan `getMyToday()` guru (method baru `getKelasHariIni()` cuma beda
      where-clause scope kelas, bukan hitung ulang logic jam).
- [x] Klik "Mulai Pembelajaran" tampilkan QR yang berganti otomatis tanpa refresh manual
      (subscribe `mulai-sesi:token` per rotasi 20 detik, T249).
- [x] Kegagalan scan guru (kondisi apa pun dari tabel error T249) tampil pesan jelas
      REAL-TIME di layar siswa (event `mulai-sesi:gagal`, pesan backend diteruskan APA
      ADANYA, tidak di-generic-kan ulang).
- [x] Sukses scan → transisi otomatis ke status "Sedang Berlangsung" tanpa reload halaman
      (event `mulai-sesi:berhasil` → state sukses 1.5 detik → auto refresh list).
- [x] Akun `ketua_kelas` TIDAK BISA akses route/menu role lain — SEMUA layout route group
      lain (`(admin)/(guru)/(piket)/(admin-jurnal)/(pembina-ekstra)`) sudah self-guard
      redirect ke `/` untuk role tidak cocok (pola existing, tidak diubah); `(admin)`
      redirect `ketua_kelas` ke `/portal-siswa`; `(siswa)/layout.tsx` sendiri redirect ke
      `/` untuk role SELAIN `ketua_kelas` — scope tertutup rapat dari 2 arah.
- [x] Responsif penuh di layar HP — `max-w-lg` container, card full-width, QR 220px (cukup
      besar di-scan dari jarak wajar), tombol besar (`h-10`/`h-11`).
- [x] Build + type-check hijau (`tsc --noEmit` api+web bersih, `next build` web sukses).

## Validasi Claudian
- [x] Konfirmasi scope akses portal ini benar-benar sempit — grep menyeluruh: `(siswa)/`
      cuma berisi 1 layout + 1 halaman (`portal-siswa`) + 2 komponen client, TIDAK ADA
      link/navigasi ke halaman lain sama sekali (sesuai spec poin 5, tanpa sidebar).
- [x] Konfirmasi library QR — `qrcode.react` TERNYATA SUDAH ADA di `package.json` (terinstall
      sebelumnya, kemungkinan sisa eksplorasi task lain) — TIDAK perlu tambah dependency
      baru sama sekali. Dipakai varian `QRCodeSVG` (bukan Canvas, lebih ringan render-nya,
      tanpa OffscreenCanvas polyfill).
- [ ] Test manual end-to-end BERSAMA T251 (perlu 2 device/browser — siswa + guru) BELUM
      dilakukan — T251 (integrasi scanner QR sisi guru) belum dikerjakan di sesi ini, jadi
      sisi guru submit token belum ada UI-nya. Diverifikasi sejauh ini: build+type-check
      hijau, 49/49 test `teaching-sessions.service.spec.ts` tetap lulus (regresi nol), code
      review manual alur socket join/leave/token/gagal/berhasil. End-to-end penuh menunggu
      T251 selesai.

## Implementasi (2026-08-27)

**Backend**: `TeachingSessionsService.getKelasHariIni(kelasId)` baru — REUSE `resolveJamSesi()`
persis `getMyToday()`, TANPA konsep `sudahTapGerbang` (validasi tap gerbang milik guru,
bukan siswa) — `bisaMulai` diganti `bisaGenerateQr` (siswa generate QR, bukan startSession()
langsung). Endpoint baru `GET /teaching-sessions/kelas-hari-ini` (role `ketua_kelas`,
STATIC path — ditaruh SEBELUM route dinamis `:sessionId/*` di controller, replikasi
pelajaran route-ordering T248) — `kelasId` di-resolve dari `req.user.studentId` →
`Student.kelasId`, DENGAN validasi ulang siswa MASIH pengurus aktif (`KelasPengurus`
ketua/wakil_ketua) di controller — bukan cuma percaya role JWT, mengantisipasi edge case
race-condition akun belum sempat dinonaktifkan (spec poin edge case).

**Frontend**: route group baru `(siswa)/` — `layout.tsx` (self-guard role, top bar minimal
tanpa sidebar), `portal-siswa/page.tsx` (server component fetch jadwal), `jadwal-hari-ini-view.tsx`
(client, daftar card + state `qrSessionId` tunggal — jamin cuma 1 layar QR terbuka sekaligus),
`sesi-siswa-card.tsx` (client, transisi in-place list-card → QR-card TANPA modal/navigasi
halaman baru sesuai rekomendasi spec). Socket.IO — 1 koneksi PER CARD dibuat SAAT QR dibuka
(bukan 1 koneksi global dipegang halaman, REUSE pola `/api/ws-token` yang SAMA
`WaliKelasNotificationsProvider`), join room `mulai-sesi:{sessionId}`, disconnect otomatis
saat QR ditutup/unmount. Edge case "jendela waktu habis selagi QR terbuka" TIDAK ADA event
backend khusus (backend tidak "mendorong" murni-lewat-waktu) — dicek client-side interval
1 detik terhadap `jam_selesai`, auto-panggil `batal-qr` + pesan jelas + tutup QR.

**Bug/keputusan ditemukan saat implementasi**: draf awal path `/siswa` BENTROK dengan
halaman Data Murid admin existing (`(admin)/siswa/page.tsx`) — Next.js App Router menolak
build total ("cannot have two parallel pages that resolve to the same path"), BUKAN
peringatan runtime. Dikonfirmasi user pindah ke `/portal-siswa`.

**Verifikasi**: `tsc --noEmit` api+web bersih, `next build` web sukses (`/portal-siswa`
terdaftar, ukuran halaman wajar 2.22 kB), 49/49 test `teaching-sessions.service.spec.ts`
lulus tanpa regresi. Live browser E2E BELUM dilakukan (butuh T251 utk sisi guru submit
token — QR generate bisa diverifikasi visual tapi tidak ada yang men-scan-nya sampai T251
ada).

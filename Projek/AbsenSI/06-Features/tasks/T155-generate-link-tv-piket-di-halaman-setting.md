# T155 — Web+API: Tombol Generate Link TV Piket di Halaman Setting TV Piket

## Depends on
Tidak ada dependency teknis. Independen.

## Objective
Halaman admin `Setting TV Piket` (`(admin)/tv-piket-setting/`, hasil T143) mendapat kemampuan BARU: admin bisa **generate link akses TV Piket** untuk kampus yang sedang dipilih, TANPA perlu memanggil API manual (curl/Postman) seperti yang harus dilakukan sebelumnya.

## Context — Kenapa Task Ini Ada

User meminta fitur ini setelah sebelumnya HARUS dibantu generate 2 token TV Piket (Kampus 1 & 2) secara manual lewat `curl` langsung ke API — proses yang tidak bisa dilakukan admin sendiri lewat UI aplikasi. Riset 2026-08-11 mengonfirmasi:

- Model `TvSession` (`schema.prisma`) — token disimpan **PLAINTEXT** di database (`nanoid(32)`, tidak di-hash), TAPI **API SENGAJA menyembunyikannya** di endpoint list (`GET /tv-sessions` pakai `LIST_SELECT` yang exclude field `token`, dengan komentar eksplisit di kode: "JANGAN expose token mentah di list — cuma di response POST pertama kali"). Ini keputusan keamanan disengaja (reveal-once), BUKAN keterbatasan teknis — **JANGAN ubah desain ini** untuk "mempermudah" menampilkan token lama, itu akan melemahkan keamanan yang sudah sengaja dibangun.
- `POST /tv-sessions` (`apps/api/src/tv-sessions/tv-sessions.controller.ts`, body `{kampusId}`) — SATU-SATUNYA titik di mana token mentah pernah muncul di response API. Endpoint ini SUDAH ADA dan berfungsi, task ini MURNI membangun UI untuk memanggilnya, TIDAK mengubah backend endpoint ini.
- `POST /tv-sessions/:id/revoke` — SUDAH ADA, untuk mencabut token lama.
- **TIDAK ADA batasan 1 token per kampus** di level sistem — 1 kampus BISA punya banyak token aktif sekaligus (berguna kalau nanti ada >1 layar TV per kampus). Task ini TIDAK menambah pembatasan itu, TAPI UI harus membantu admin AWARE ada token aktif lain supaya tidak asal generate baru terus tanpa mencabut yang lama tanpa sadar.
- Halaman `tv-piket-setting-view.tsx` (T143) SUDAH punya dropdown pemilih kampus (flat, bukan tab) — fitur ini DITAMBAHKAN sebagai section BARU di halaman yang SAMA, mengikuti kampus yang sedang dipilih di dropdown existing, BUKAN halaman terpisah.

## Spec Detail

### 1. Backend — TIDAK ADA perubahan wajib ke endpoint existing

- `POST /tv-sessions`, `GET /tv-sessions`, `POST /tv-sessions/:id/revoke` — SEMUA SUDAH CUKUP, reuse apa adanya.
- **PERTIMBANGKAN opsional**: `GET /tv-sessions` saat ini TIDAK punya filter query param `kampusId` (list SEMUA kampus sekaligus) — kalau terasa lebih bersih untuk frontend, tambah `?kampusId=` opsional di endpoint ini (backward compatible, default tanpa filter = behavior lama) — TIDAK WAJIB, frontend BOLEH filter manual dari hasil list lengkap kalau lebih sederhana untuk scope kecil ini (jumlah kampus sedikit, tidak masalah performa).

### 2. Frontend — section baru di `tv-piket-setting-view.tsx`

Di BAWAH 5 toggle blok yang sudah ada (T143), untuk kampus yang SEDANG DIPILIH di dropdown — tambah card/section baru "Link Akses TV Piket":

- **Daftar token aktif untuk kampus ini** (dari `GET /tv-sessions`, filter client-side by `kampusId` kalau backend tidak diperluas di poin 1) — tampilkan `id`, `createdAt`, status aktif — TANPA token (memang tidak pernah dikirim API, sesuai desain). Untuk setiap token aktif, tombol **"Cabut"** (panggil `POST /tv-sessions/:id/revoke`, konfirmasi dialog dulu — ini aksi yang membuat TV yang sedang pakai token itu BERHENTI BERFUNGSI, beri peringatan jelas).
- **Tombol "Generate Link Baru"** — panggil `POST /tv-sessions` dengan `kampusId` yang sedang dipilih. Response berisi `token` mentah SEKALI — susun LINK LENGKAP: `${window.location.origin.replace(':3000', ':3000')}/tv-piket/${kampusId}?token=${token}` (PERHATIKAN: origin halaman ADMIN web dan origin yang harus diakses TV mungkin SAMA domain/port karena `apps/web` yang generate DAN yang menyajikan halaman `/tv-piket/[kampusId]` adalah APLIKASI YANG SAMA — verifikasi ini saat implementasi, cek `apps/web/src/app/tv-piket/[kampusId]/page.tsx` ada di app yang sama dengan halaman admin, BUKAN app terpisah seperti kiosk).
- **Tampilkan link itu dalam MODAL/DIALOG jelas** dengan:
  - Teks link lengkap dalam kotak yang BISA DI-COPY (tombol "Salin Link", pakai Clipboard API browser).
  - Peringatan tegas: **"Link ini HANYA ditampilkan SEKALI SAJA. Setelah dialog ini ditutup, link (dengan token ini) TIDAK BISA dilihat lagi — kalau hilang, harus generate token baru."** (konsisten sifat reveal-once dari backend).
  - Instruksi singkat cara pakai: "Buka link ini SEKALI di browser TV kampus ini, setelah itu tersimpan otomatis di perangkat tersebut — TV/browser bisa dijadikan homepage tanpa perlu link lagi untuk kunjungan berikutnya."
- Setelah dialog ditutup, refresh daftar token aktif (token baru otomatis muncul di daftar TANPA nilai token-nya, sesuai perilaku list yang sudah benar).

### 3. Edge case UX — kampus dengan BANYAK token aktif

- Kalau daftar token aktif untuk kampus yang dipilih SUDAH ADA 1+ (dari generate sebelumnya, termasuk yang mungkin masih dipakai TV fisik sekarang) — TAMPILKAN JELAS sebelum admin klik "Generate Link Baru", supaya admin SADAR ini akan MENAMBAH token baru (bukan mengganti), bukan otomatis mencabut yang lama. TIDAK PERLU blocking/konfirmasi tambahan sebelum generate (generate token baru itu sendiri aman/non-destruktif ke token lama), TAPI UI harus INFORMATIF (misal teks "Kampus ini sudah punya N link aktif" di atas tombol generate).

## Edge Cases
- Kampus BELUM PERNAH punya token sama sekali — daftar kosong, tampilkan state jelas ("Belum ada link aktif untuk kampus ini") + tombol Generate tetap tersedia.
- Admin generate token baru TAPI lupa menyalin sebelum menutup dialog (human error) — TIDAK ADA cara recovery selain generate token BARU LAGI (sesuai sifat reveal-once yang disengaja) — pastikan dialog TIDAK mudah ter-close tidak sengaja (misal klik di luar dialog TIDAK menutupnya, harus klik tombol "Selesai/Tutup" eksplisit setelah mengonfirmasi sudah menyalin).
- Revoke token yang SEDANG DIPAKAI TV aktif — TV itu akan berhenti berfungsi (kehilangan akses) begitu token di-revoke di server (halaman TV Piket kemungkinan akan menampilkan error auth di refresh berikutnya) — task ini TIDAK PERLU menangani notifikasi ke TV yang terdampak, cukup PERINGATAN JELAS di dialog konfirmasi revoke supaya admin sadar konsekuensinya SEBELUM klik.

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/tv-piket-setting/tv-piket-setting-view.tsx` (section baru + modal generate).
- **Opsional:** `apps/api/src/tv-sessions/tv-sessions.controller.ts`+`tv-sessions.service.ts` (filter `?kampusId=` di `GET /tv-sessions`, kalau dipilih saat implementasi).
- **Jangan sentuh:** `apps/api/src/tv-sessions/tv-sessions.service.ts` method `create()` (SUDAH BENAR mengembalikan token mentah sekali, JANGAN diubah), `LIST_SELECT` di `findAll()` (SENGAJA exclude token, JANGAN diubah untuk "mempermudah" — itu akan melemahkan keamanan by-design).

## Acceptance Criteria
- [x] Halaman Setting TV Piket, untuk kampus yang dipilih, menampilkan daftar token aktif (tanpa nilai token) + tombol Cabut per token.
- [x] Tombol "Generate Link Baru" berhasil membuat token baru dan menampilkan LINK LENGKAP siap-pakai dalam dialog, dengan tombol salin.
- [x] Link yang dihasilkan format benar: `{origin}/tv-piket/{kampusId}?token={token}`, dikonfirmasi BISA dibuka dan berfungsi (verifikasi manual: buka link itu di browser, halaman TV Piket kampus terkait tampil normal).
- [x] Dialog generate TIDAK mudah tertutup tidak sengaja, ada peringatan jelas "hanya sekali tampil".
- [x] Tombol Cabut token — konfirmasi dialog jelas sebelum eksekusi, token yang dicabut hilang dari daftar aktif setelahnya.
- [x] Build + type-check `apps/web` hijau (poin filter `?kampusId=` backend opsional TIDAK dikerjakan — filter client-side dipilih, sesuai rekomendasi spec untuk skala kecil).

## Validasi Claudian
- [x] **JANGAN** mengubah `LIST_SELECT`/desain reveal-once token — TIDAK disentuh, `tv-sessions.service.ts`/`tv-sessions.controller.ts` 0 diff, backend reuse 100% apa adanya.
- [x] **JANGAN** membatasi sistem ke 1 token per kampus — TIDAK dibatasi, UI menampilkan "Kampus ini sudah punya N link aktif" sebagai info, generate tetap selalu diizinkan.
- [x] Origin link — DIKONFIRMASI via riset kode: `(admin)/tv-piket-setting` dan `/tv-piket/[kampusId]` sama-sama di `apps/web` (bukan app terpisah seperti kiosk), `window.location.origin` valid dipakai apa adanya. Verified live: link yang di-generate dari halaman admin (`http://localhost:3100/...`) berhasil dibuka dan menampilkan data TV Piket normal.

## Status Eksekusi (2026-08-14)

**Selesai.** Backend 100% reuse (0 perubahan), frontend baru, verified live end-to-end.

**Frontend (`apps/web/src/app/(admin)/tv-piket-setting/tv-piket-setting-view.tsx`)**:
- Section baru "Link Akses TV Piket" di bawah 5 toggle blok existing (T143), mengikuti kampus yang dipilih di dropdown yang sama.
- Daftar token aktif — `GET /tv-sessions` di-fetch sekali, difilter client-side by `kampusId` (backend TIDAK diperluas, sesuai rekomendasi spec untuk skala kecil). Tiap baris: `id`, `createdAt` (format lokal id-ID), tombol "Cabut" (`confirm()` native browser, konsisten pola existing `kartu-view.tsx` — bukan dialog custom, sesuai konvensi proyek untuk aksi destruktif sederhana).
- Tombol "Generate Link Baru" — `POST /tv-sessions`, link disusun `${window.location.origin}/tv-piket/${kampusId}?token=${token}`.
- Dialog "Link Baru" (Radix `Dialog` langsung, BUKAN `confirm()` — butuh copy-to-clipboard + close-eksplisit): `onOpenChange={() => {}}` (no-op) + `onPointerDownOutside`/`onEscapeKeyDown` di-`preventDefault()` supaya TIDAK bisa ditutup via klik-luar/Escape; tombol X bawaan Radix disembunyikan via `className="[&>button]:hidden"` di `DialogContent` (CSS selector direct-child, tanpa perlu ubah `packages/ui`); tombol "Selesai, Sudah Disalin" DISABLED sampai admin klik "Salin Link" dulu (pengaman tambahan di luar spec literal, konsisten tujuan "pastikan sudah menyalin sebelum tertutup").
- Setelah dialog ditutup — `refreshSessions()` otomatis, token baru muncul di daftar tanpa nilai token.
- `apps/web/src/lib/core-types.ts` — `TvSessionListItem` (tanpa `token`) dan `TvSessionCreated` (dengan `token`, 1 response yang mengandungnya) baru, tipenya eksplisit terpisah untuk mencegah salah pakai shape yang salah di kode lain nanti.

**Verifikasi live** (dev DB port 3307, dev web port 3100, production tidak disentuh, akun `adminSU` password di-override sementara TestPass123 lalu DIKEMBALIKAN persis — dikonfirmasi via SELECT sebelum/sesudah, browser Playwright):
1. State awal "Belum ada link aktif untuk kampus ini" tampil benar untuk Kampus 1.
2. Klik "Generate Link Baru" → dialog muncul, link lengkap `http://localhost:3100/tv-piket/1?token=...` tampil.
3. Tekan Escape saat dialog terbuka → dialog TIDAK tertutup (dikonfirmasi snapshot tetap sama).
4. Klik "Salin Link" → tombol berubah jadi "Tersalin", tombol "Selesai, Sudah Disalin" berubah dari disabled ke aktif.
5. Buka link hasil generate di tab baru → halaman TV Piket Kampus 1 tampil NORMAL dengan data live (bar persentase, tabel siswa alfa) — BUKAN error auth, token valid diterima backend.
6. Klik "Selesai, Sudah Disalin" → dialog tertutup, daftar refresh otomatis menampilkan "Kampus ini sudah punya 1 link aktif" + "Link #1" (TANPA nilai token) + tombol Cabut.
7. Klik "Cabut" → dialog `confirm()` native muncul dengan pesan jelas + peringatan konsekuensi ("TV yang sedang memakai link ini akan BERHENTI BERFUNGSI") → diterima → daftar kembali ke "Belum ada link aktif untuk kampus ini".
8. `tsc --noEmit` bersih `apps/web` dan `apps/api` (tidak disentuh, dipastikan tetap hijau).

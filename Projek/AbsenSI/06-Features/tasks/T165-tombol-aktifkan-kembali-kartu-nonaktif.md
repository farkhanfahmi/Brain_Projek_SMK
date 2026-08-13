# T165 — API+Web: Tombol Aktifkan Kembali Kartu Nonaktif (Tanpa Perlu Hapus+Daftar Ulang)

## Depends on
Tidak ada dependency teknis. Independen dari T166 (rombakan besar halaman Kartu per-orang) — task ini bisa dan SEBAIKNYA dikerjakan LEBIH DULU sebagai perbaikan cepat, T166 boleh menyusul kapan pun.

## Objective
Admin bisa **mengaktifkan kembali** kartu yang statusnya `inactive` (pernah dicabut/diganti) langsung dari 1 klik tombol — TANPA perlu menghapus data kartu dan mendaftarkan ulang dari nol (workaround berisiko yang selama ini dipakai admin).

## Context — Insiden yang Memicu Task Ini (2026-08-13)

User (admin) melaporkan banyak siswa/guru dengan kartu "sudah terdaftar" tapi tap ditolak "kartu tidak terdaftar". **Investigasi mendalam dengan reproduksi LANGSUNG** (bukan dugaan) menemukan akar masalah SESUNGGUHNYA:

- Kasus guru MOHAMAD FARKHAN FAHMI ZUHRI — dicek `tap_events` PERSIS di detik kejadian (`2026-08-13 02:26:41` UTC): hasil **`rejected_inactive`** (BUKAN `rejected_unknown`/"tidak terdaftar" — pesan yang berbeda, mudah tertukar sekilas di layar kiosk). `card_id` di baris itu **BERHASIL MATCH** ke kartu di database (`id=2800`, UID `0333400879`) — TAPI kartu itu `status: inactive`, sudah dicabut (`revokedAt: 2026-08-12 02:23:54`) SAAT ITU JUGA admin mendaftarkan kartu PENGGANTI (UID `0793501459`) untuk guru yang sama.
- **Diagnosis pasti**: user (atau siswa/guru) menempelkan kartu FISIK LAMA yang sudah tidak aktif (tertukar dengan kartu baru, atau kartu lama tidak dibuang), sistem BENAR menolaknya sesuai desain. **Ini BUKAN bug validasi kartu** — sistem sudah bekerja sesuai rancangan.
- **Root cause sebenarnya kenapa ini jadi masalah OPERASIONAL**: TIDAK ADA cara mudah bagi admin untuk mengaktifkan kembali kartu lama itu KALAU TERNYATA memang kartu itu yang seharusnya dipakai (misal kartu "baru" yang didaftarkan robek/hilang, atau salah catat) — satu-satunya jalan SAAT INI adalah **hapus record kartu lalu daftar ulang dari nol**, yang BERISIKO (kehilangan histori `issuedAt` asli, dan berpotensi human-error di step manual).

**Riset kode mengonfirmasi**: mekanisme reaktivasi SEBENARNYA **SUDAH ADA secara implisit** di `CardsService.create()` (`apps/api/src/cards/cards.service.ts:154-217`) — kalau `POST /cards` dipanggil dengan UID yang SAMA PERSIS dengan kartu nonaktif MILIK OWNER YANG SAMA, sistem OTOMATIS mengaktifkannya kembali (`status: active, revokedAt: null, issuedAt: new Date()`) alih-alih menolak sebagai duplikat. **MASALAHNYA**: mekanisme ini HANYA bisa dipicu lewat jalur "tap kartu fisik itu lagi di halaman registrasi" — TIDAK ADA tombol eksplisit di UI admin untuk memicu ini tanpa perlu kartu fisiknya ada di tangan SAAT ITU.

## Spec Detail

### 1. Backend — endpoint baru `PATCH /cards/:id/activate`

- `apps/api/src/cards/cards.controller.ts` — endpoint baru, guard SAMA seperti `revoke()` (`@Roles(UserRole.super_admin, UserRole.card_admin)`, `@LogActivity`).
- `apps/api/src/cards/cards.service.ts` — method baru `activate(id: number, actorId: number)`:
  1. Ambil kartu by `id`, kalau tidak ada → `NotFoundException`.
  2. Kalau `status` SUDAH `active` → tolak dengan pesan jelas ("Kartu ini sudah aktif") — TIDAK PERLU proses lebih lanjut, hindari no-op membingungkan.
  3. **WAJIB panggil ULANG `ensureOwnerExistsAndHasNoActiveCard()`** (method privat yang SUDAH ADA, baris 281-311) — dengan `studentId`/`teacherId` dari kartu yang mau diaktifkan — supaya ATURAN T119 tetap tertegakkan: untuk SISWA, kalau siswa itu SUDAH punya kartu aktif LAIN saat ini → TOLAK aktivasi ini dengan `ConflictException` (pesan SAMA seperti yang sudah ada: "Siswa ini sudah punya kartu aktif (UID X) — gunakan aksi Ganti Kartu untuk mengganti"); untuk GURU/KARYAWAN → TIDAK ADA batasan (boleh aktifkan kartu ini meski sudah ada kartu lain yang aktif, SESUAI aturan multi-kartu T119, dikonfirmasi EKSPLISIT oleh user).
  4. Kalau lolos validasi — `update()`: `status: active, revokedAt: null, issuedAt: new Date()` — **REUSE PERSIS pola yang sudah ada** di `create()` baris 212-216 (PERTIMBANGKAN extract jadi 1 method privat `reactivateCard()` yang dipanggil BAIK dari `create()` (jalur implisit lewat tap-assign yang sudah ada) MAUPUN dari `activate()` (endpoint baru ini) — supaya kedua jalur TIDAK PERNAH bisa berbeda perilaku di masa depan).
  5. `@LogActivity` mencatat aksi ini (siapa mengaktifkan kartu siapa, kapan) — konsisten aturan permanen proyek.

### 2. Frontend — tombol "Aktifkan Kembali" untuk kartu nonaktif

- `apps/web/src/app/(admin)/kartu/kartu-view.tsx`, komponen `CardActions` (baris ~645-677) — SAAT INI `if (card.status !== "active") return null;` (kartu nonaktif TIDAK PUNYA aksi apa pun). UBAH jadi: kalau `status === "inactive"`, tampilkan tombol BARU "Aktifkan Kembali" (icon yang masuk akal, misal `RotateCcw`/`CheckCircle` — PUTUSKAN saat implementasi, konsisten style tombol lain di komponen ini) — panggil `PATCH /cards/:id/activate`.
- **Konfirmasi dialog SEBELUM eksekusi** — aksi ini punya konsekuensi nyata (kartu jadi bisa dipakai tap lagi), tampilkan dialog konfirmasi singkat sebelum submit (KONSISTEN pola dialog konfirmasi lain yang sudah ada di proyek, misal untuk revoke).
- **Tangani error dengan pesan jelas** — KALAU backend menolak (siswa sudah punya kartu aktif lain), tampilkan pesan error itu APA ADANYA ke admin (bukan generic "gagal"), supaya admin tahu PERSIS kenapa dan apa yang perlu dilakukan (misal nonaktifkan dulu kartu aktif yang lain).
- Setelah berhasil — refresh baris tabel itu (status berubah jadi `active`, `revokedAt` jadi kosong) — TIDAK PERLU reload seluruh halaman, cukup update state lokal (KONSISTEN pola optimistic-update yang sudah dipakai action lain di halaman ini kalau ada).

## Edge Cases
- Kartu yang mau diaktifkan itu UID-nya SUDAH DIPAKAI oleh baris `Card` LAIN yang statusnya aktif (skenario TIDAK MUNGKIN terjadi karena `Card.uid` adalah `@unique` di database — TIDAK PERLU penanganan khusus, constraint DB sudah menjamin ini tidak bisa terjadi).
- Admin mengaktifkan kartu SISWA yang TERNYATA (race condition kecil) siswa itu BARU SAJA dapat kartu aktif lain dari admin LAIN di detik yang hampir sama — `ensureOwnerExistsAndHasNoActiveCard()` (dipanggil ULANG tepat sebelum update, bukan di-cache dari request sebelumnya) akan MENANGKAP kondisi ini dan menolak dengan benar (karena selalu query FRESH ke database saat endpoint dipanggil) — TIDAK PERLU logic tambahan, method existing SUDAH aman untuk kasus ini.
- Kartu yang diaktifkan kembali — `issuedAt` di-RESET ke waktu SEKARANG (BUKAN mempertahankan `issuedAt` ASLI yang lama) — INI PERILAKU YANG SUDAH ADA di `create()` (baris 212-216), task ini MEMPERTAHANKAN perilaku itu APA ADANYA (konsisten dengan jalur reaktivasi implisit yang sudah ada), JANGAN diubah jadi "pertahankan issuedAt asli" tanpa diskusi eksplisit dengan user (ini keputusan desain existing, di luar scope task ini untuk diubah).

## Files
- **Modifikasi:** `apps/api/src/cards/cards.controller.ts` (endpoint baru), `apps/api/src/cards/cards.service.ts` (method `activate()`, PERTIMBANGKAN extract `reactivateCard()` shared), `apps/web/src/app/(admin)/kartu/kartu-view.tsx` (`CardActions`, tombol baru + dialog konfirmasi).
- **Jangan sentuh:** logic `create()`/`replace()`/`revoke()` yang SUDAH BENAR (TIDAK diubah perilakunya, cuma di-reuse), `ensureOwnerExistsAndHasNoActiveCard()` (dipanggil ulang APA ADANYA, tidak dimodifikasi).

## Acceptance Criteria
- [x] Kartu berstatus `inactive` di halaman `/kartu` (tab Siswa/Guru/Karyawan) menampilkan tombol "Aktifkan Kembali".
- [x] Klik tombol (dengan konfirmasi) → kartu itu jadi `status: active`, `revokedAt: null`, langsung bisa dipakai tap lagi TANPA perlu tap kartu fisik dulu di halaman registrasi — verified live.
- [x] SISWA yang SUDAH punya kartu aktif lain → aktivasi kartu KEDUA ditolak dengan pesan jelas menyebut kartu aktif yang sudah ada — verified live: `"Siswa ini sudah punya kartu aktif (UID 0794307971) — gunakan aksi \"Ganti Kartu\" untuk mengganti"`.
- [x] GURU/KARYAWAN — aktivasi kartu nonaktif SELALU berhasil meski sudah ada kartu lain yang aktif — verified live: guru dengan 1 kartu aktif + 1 kartu nonaktif, aktivasi kartu kedua BERHASIL (`status: active`).
- [x] `@LogActivity` mencatat aksi aktivasi — verified live, `activity_log` berisi `action: "card.activate"`, `target_type: "card"`, `target_id` benar.
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] **WAJIB** panggil `ensureOwnerExistsAndHasNoActiveCard()` — dipanggil ULANG (query fresh) di `activate()` sebelum reaktivasi, TIDAK dimodifikasi sama sekali (reuse apa adanya).
- [x] **JANGAN** ubah perilaku `issuedAt` di-reset saat reaktivasi — TIDAK diubah, `reactivateCard()` (helper baru, di-share `create()` DAN `activate()`) tetap `issuedAt: new Date()` persis seperti `create()` sebelumnya.
- [x] Test manual DUA skenario — DILAKUKAN via curl langsung ke dev API (bukan hanya baca kode): siswa ditolak (dikonfirmasi pesan persis), guru berhasil (dikonfirmasi response `status: active`).

## Status Eksekusi (2026-08-13)

**Selesai.** Backend + frontend + verifikasi live dengan skenario siswa DAN guru, semua hijau.

**Backend (`cards.service.ts`)**:
- Konstanta `CARD_WITH_OWNER_INCLUDE` diekstrak (sebelumnya di-duplikasi 2x identik di `findAll()` dan `create()`) — dipakai juga oleh `activate()`/`reactivateCard()` sekarang, 1 sumber shape response.
- `reactivateCard(cardId)` (private) — SATU implementasi update `{status: active, revokedAt: null, issuedAt: new Date()}`, dipanggil dari `create()` (jalur implisit T118, existing) DAN `activate()` (jalur eksplisit T165, baru) — kedua jalur TIDAK BISA lagi beda perilaku di masa depan.
- `activate(id)` (public, baru) — cek kartu ada (`NotFoundException`), cek belum aktif (`ConflictException` "Kartu ini sudah aktif" — no-op dicegah eksplisit), panggil ULANG `ensureOwnerExistsAndHasNoActiveCard(card.studentId, card.teacherId)` (query FRESH, TIDAK di-cache dari request sebelumnya — aman untuk race condition per edge case spec), lalu `reactivateCard()`.

**Backend (`cards.controller.ts`)**: `PATCH /cards/:id/activate` baru, guard/role level controller (sama seperti `revoke()`), `@LogActivity({action: "card.activate", targetType: "card", prismaModel: "card"})`.

**Frontend (`kartu-view.tsx`)**:
- `CardActions` — kartu `inactive` SEKARANG render tombol "Aktifkan Kembali" (icon `RotateCcw`, warna primary — beda dari tombol aktif Ganti/Nonaktifkan) alih-alih `return null` (dead-end lama).
- `handleActivate(card)` — pola SAMA PERSIS `handleRevoke()` existing (browser `confirm()` native, `apiClientFetch` PATCH, `router.refresh()` setelah sukses, `setError()` dengan pesan backend APA ADANYA kalau gagal — bukan generic "gagal").
- Wired ke ketiga tab (Siswa/Guru/Karyawan) — 3 titik `<CardActions>` semuanya diperbarui.

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh):
1. Siswa (id 1021) dengan 1 kartu aktif (`0794307971`) + kartu test kedua yang di-set `inactive` → `PATCH /cards/:id/activate` pada kartu kedua → `409 Conflict`, pesan PERSIS menyebut UID kartu aktif yang sudah ada.
2. Guru (id 9) dengan 1 kartu aktif (`1112223334`) + kartu test kedua `inactive` (`4443332221`) → `PATCH /cards/:id/activate` pada kartu kedua → `200`, response `status: "active"`, `revokedAt: null`, `issuedAt` reset ke waktu sekarang — BERHASIL meski sudah ada kartu aktif lain, sesuai aturan T119.
3. Re-aktivasi kartu yang SUDAH aktif (kartu guru dari langkah 2, dicoba lagi) → `409 Conflict`, "Kartu ini sudah aktif" — no-op dicegah.
4. `activity_log` dikonfirmasi berisi baris `action: "card.activate"` dengan `target_id` yang benar.
5. Semua data uji (3 kartu test, activity_log terkait, user test admin) dibersihkan setelah verifikasi.
6. `tsc --noEmit` bersih `apps/api` + `apps/web`. Jest `apps/api` 273/273 pass, tidak ada regresi.

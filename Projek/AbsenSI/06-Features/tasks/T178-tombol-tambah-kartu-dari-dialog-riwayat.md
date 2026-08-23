# T178 — API+Web: Tombol "Tambah Kartu" di Dialog Riwayat Kartu (Guru/Karyawan)

## Depends on
Tidak ada dependency teknis. Independen. Reuse penuh backend `CardsService.create()`/`tapAssign()` existing (T119 sudah mendukung guru multi-kartu tanpa perubahan).

## Objective
Di dialog "Riwayat Kartu" (T166, muncul saat nama Guru/Karyawan diklik di halaman `/kartu`) — tambahkan tombol **"Tambah Kartu"** supaya admin bisa langsung daftarkan kartu BARU untuk orang itu (guru/karyawan boleh multi-kartu aktif, T119) TANPA perlu keluar ke halaman `/kartu/tap-assign` terpisah.

## Context — Temuan Riset (2026-08-14)

- `RiwayatKartuDialog` (`apps/web/src/app/(admin)/kartu/kartu-view.tsx:607-729`) — SATU komponen shared untuk Siswa MAUPUN Guru/Karyawan (`target.type: "student" | "teacher"`), dipanggil dari 3 tab. State `cards: Card[]` dari `person.cards` (fetch `GET /students/:id`/`GET /teachers/:id`). Aksi per baris SUDAH ADA via `CardActions` (baris ~743-793): Ganti, Nonaktifkan, Aktifkan Kembali — **TIDAK ADA tombol "Tambah Kartu"** sama sekali di dialog ini.
- Form registrasi kartu SUDAH ADA tapi TERPISAH: `apps/web/src/app/(admin)/kartu/tap-assign-view.tsx` — alur: pilih dari daftar `GET /cards/unassigned-persons` → tap fisik/input UID manual → `POST /cards/tap-assign`.
- **MASALAH TEMUAN PENTING**: `unassignedPersons()` (`cards.service.ts:143-166`) query GURU dengan filter `cards: {none: {status: active}}` — SAMA seperti siswa, PADAHAL guru boleh multi-kartu aktif (T119). Akibatnya: **guru yang SUDAH punya 1 kartu aktif TIDAK PERNAH muncul** di halaman tap-assign untuk ditambahkan kartu KEDUA — bug tersembunyi yang jadi alasan KUAT task ini perlu jalur BARU (bukan reuse halaman tap-assign apa adanya untuk kasus guru).
- Backend `CardsService.create()` (`cards.service.ts:176-225`) via `ensureOwnerExistsAndHasNoActiveCard()` (baris ~424-447) — SUDAH BENAR untuk guru (hanya cek existensi Teacher, TIDAK cek kartu aktif, komentar eksplisit "guru/karyawan BOLEH multi-kartu aktif sekaligus"). **TIDAK PERLU perubahan backend `create()`/`ensureOwnerExistsAndHasNoActiveCard()`** — reuse 100% apa adanya.

## Spec Detail

### 1. Frontend — tombol "Tambah Kartu" di `RiwayatKartuDialog`

- `apps/web/src/app/(admin)/kartu/kartu-view.tsx`, `RiwayatKartuDialog` — tambah tombol "Tambah Kartu" di header/footer dialog (SELALU tampil untuk `target.type === "teacher"`; untuk `target.type === "student"` tombol ini TIDAK tampil kalau siswa itu SUDAH punya kartu aktif — siswa dibatasi 1 aktif, T119, TAPI TETAP tampilkan untuk siswa yang 0 kartu aktif sama sekali supaya bisa daftar kartu pertama dari dialog ini juga, konsisten kemudahan yang sama).
- Klik tombol → buka form kecil (REUSE pola visual `ReplaceCardForm` yang SUDAH ADA di komponen yang sama, baris ~795-860, sebagai referensi struktur — form baru ini SEDERHANA: 1 input UID (manual, ATAU opsional dukung tap-capture fisik seperti `tap-assign-view.tsx` kalau ingin konsisten UX, PUTUSKAN saat implementasi mana yang lebih praktis untuk dialog kecil ini — REKOMENDASI cukup input manual dulu, tap-capture fisik BOLEH ditambah kalau terasa perlu tapi TIDAK WAJIB v1).
- Submit → `POST /cards/tap-assign` (endpoint SUDAH ADA, TIDAK PERLU endpoint baru) dengan `{uid, personId: target.id, personType: target.type}` — REUSE PERSIS payload yang sama seperti `tap-assign-view.tsx` kirim.
- Sukses → refresh daftar `cards` di dialog (re-fetch `GET /students/:id`/`GET /teachers/:id`, KONSISTEN pola refresh yang SUDAH ADA di dialog ini setelah aksi lain seperti Ganti/Aktifkan Kembali — cek pola itu dan REUSE).
- Error (UID sudah dipakai kartu lain, dll) — tampilkan pesan APA ADANYA dari backend (KONSISTEN aturan proyek "pesan error sesuai kondisi").

### 2. Backend — TIDAK ADA perubahan wajib untuk task ini

- `POST /cards/tap-assign` SUDAH cukup dan benar untuk kasus guru (via `create()` yang sudah mendukung multi-kartu). TIDAK PERLU modifikasi.
- **CATATAN TERPISAH (bukan scope wajib task ini, tapi TEMUAN PENTING)**: bug `unassignedPersons()` yang mengecualikan guru ber-kartu-aktif dari daftar "Belum Punya Kartu" TETAP ADA setelah task ini selesai — task ini HANYA menambah JALUR BARU (dari dialog riwayat) yang tidak terpengaruh bug itu (karena tidak lewat `unassignedPersons()` sama sekali, target orang sudah diketahui dari dialog). Kalau ingin `unassignedPersons()` juga diperbaiki (misal filter beda untuk guru: selalu tampilkan semua guru aktif terlepas status kartu, karena "belum punya kartu" tidak relevan buat guru yang boleh nambah terus), itu DI LUAR SCOPE task ini — LAPORKAN temuan ini ke user sebagai potensi task terpisah, JANGAN diam-diam perbaiki tanpa konfirmasi (bisa jadi behavior itu MEMANG diinginkan untuk halaman tap-assign yang fokus "orang belum punya kartu sama sekali").

## Edge Cases
- Guru yang di-tap-assign dari dialog ini dengan UID yang SUDAH terpakai kartu AKTIF milik orang lain — backend `create()` SUDAH menolak ini (constraint `uid unique`), pesan error diteruskan apa adanya.
- Siswa yang sudah punya kartu aktif — tombol "Tambah Kartu" TIDAK tampil sama sekali (dicegah di UI), TAPI kalaupun request dipaksa lewat cara lain, backend `ensureOwnerExistsAndHasNoActiveCard()` untuk siswa TETAP menolak (defense-in-depth SUDAH ADA, tidak perlu ditambah).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kartu/kartu-view.tsx` (`RiwayatKartuDialog`, tombol+form baru).
- **Jangan sentuh:** `CardsService.create()`/`ensureOwnerExistsAndHasNoActiveCard()`/`tapAssign()` (reuse 100% apa adanya), `unassignedPersons()` (bug terpisah, DI LUAR SCOPE, laporkan saja).

## Acceptance Criteria
- [x] Dialog riwayat kartu Guru/Karyawan punya tombol "Tambah Kartu" yang SELALU tampil (di header dialog, sejajar judul).
- [x] Dialog riwayat kartu Siswa punya tombol "Tambah Kartu" HANYA kalau siswa itu 0 kartu aktif (`bisaTambahKartu = target.type === "teacher" || !cards.some(c => c.status === "active")`).
- [x] Submit UID baru memanggil `POST /cards/tap-assign` (payload persis sama `tap-assign-view.tsx`), daftar di dialog ter-refresh otomatis via `load()` + `onChanged()` (pola sama Ganti/Aktifkan Kembali).
- [x] Guru yang SUDAH punya kartu aktif tetap dapat tombol "Tambah Kartu" (selalu tampil untuk `type: teacher`) — jalur ini TIDAK lewat `unassignedPersons()` sama sekali (target sudah diketahui dari dialog), jadi tidak terkena bug filter itu.
- [x] Error (UID duplikat, dll) tampil apa adanya dari backend (`err.message`), bukan generic.
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `next build` sukses, `/kartu` naik 8.14kB→8.58kB, wajar untuk komponen baru).

## Validasi Claudian
- [x] **Konfirmasi TIDAK ADA perubahan backend** — `git status --porcelain apps/api/` dicek setelah implementasi, 0 file backend baru/berubah dari task ini (semua perubahan `apps/api` yang ada berasal dari task lain di sesi-sesi sebelumnya). `CardsService.create()`/`ensureOwnerExistsAndHasNoActiveCard()`/`tapAssign()` reuse 100% apa adanya.
- [x] **Temuan bug `unassignedPersons()` dilaporkan ke user** (lihat ringkasan chat) — guru ber-kartu-aktif tidak muncul di halaman `/kartu/tap-assign` karena filter `cards: {none: {status: active}}` berlaku sama untuk guru maupun siswa, padahal guru boleh multi-kartu (T119). TIDAK diperbaiki di task ini (di luar scope, task ini hanya membuka jalur baru dari dialog riwayat yang tidak lewat query itu).

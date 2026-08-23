# T183 — API: Tambah Unit Test `JamPelajaranService` (Gap Coverage T158)

## Depends on
**REKOMENDASI setelah T181** (kalau T181 mengubah `update()`, test baru sebaiknya ditulis terhadap versi yang SUDAH diperbaiki, supaya tidak perlu ditulis ulang). Independen dari T182 (beda service).

## Objective
Tambah unit test langsung untuk `JamPelajaranService` — SAAT INI **tidak ada file spec sama sekali** untuk service ini, semua skenario kritis dari spec T158 (constraint per-tingkat, validasi cakupan, dll) hanya diverifikasi TIDAK LANGSUNG lewat mock di consumer lain — regresi di service ini TIDAK akan tertangkap CI.

## Context — Temuan Audit (2026-08-14)

Audit kritis pasca-eksekusi T158+T159 menemukan: `find apps/api/src/jam-pelajaran -iname "*.spec.ts"` = **KOSONG**. Test yang ADA (`schedules.service.spec.ts`, `teaching-sessions.service.spec.ts`, `attendance.service.spec.ts`) hanya MOCK `JamPelajaranService` sebagai dependency — TIDAK menguji logic INTERNAL service itu sendiri.

Skenario kritis dari spec T158 yang **TIDAK ADA test langsung** untuk memverifikasinya:
1. 2 tingkat berbeda punya opsi AKTIF berbeda BERSAMAAN, tetap independen (constraint per-tingkat, bukan global).
2. `activate()` MENOLAK opsi yang cakupannya TIDAK mencakup tingkat yang mau diaktifkan.
3. `update()` MENOLAK ubah cakupan tingkat yang menghapus tingkat yang SEDANG AKTIF (lihat juga T181, race condition di area yang SAMA — test T181 dan T183 BOLEH overlap, JANGAN duplikasi kalau T181 sudah menulis test serupa, CEK dulu sebelum menulis test baru yang sama persis).
4. `delete()` MENOLAK hapus opsi yang MASIH aktif di tingkat manapun.
5. `getActiveForTingkat()`/`resolveJamSchedule()` — resolusi benar per tingkat kelas, TIDAK tercampur antar tingkat.

## Spec Detail

### 1. Buat file `apps/api/src/jam-pelajaran/jam-pelajaran.service.spec.ts`

Test unit MURNI untuk `JamPelajaranService` (mock `PrismaService`, KONSISTEN pola test service lain di proyek ini — cek 1-2 spec file existing seperti `teacher-permits.service.spec.ts` untuk pola mocking yang sudah established, REUSE gaya yang sama, JANGAN buat pola baru).

Skenario WAJIB dicakup:
- **`create()`**: opsi baru dengan `semuaTingkat: true` — tidak buat baris `JamPelajaranOptionTingkat`. Opsi baru dengan `semuaTingkat: false` + array tingkat — baris `JamPelajaranOptionTingkat` dibuat sesuai array. Slot per hari dibuat sesuai payload (termasuk baris istirahat `jamKe: null`).
- **`activate()`**: 
  - Opsi `semuaTingkat: true` — BERHASIL diaktifkan untuk tingkat apa pun (X/XI/XII).
  - Opsi `semuaTingkat: false` dengan cakupan `[XII]` — BERHASIL diaktifkan untuk XII, **DITOLAK** untuk X atau XI (`ConflictException`/error jelas).
  - Aktivasi tingkat X ke opsi A, LALU aktivasi tingkat XII ke opsi B — KEDUANYA aktif BERSAMAAN, saling independen (query `getActiveForTingkat("X")` return opsi A, `getActiveForTingkat("XII")` return opsi B, TIDAK saling menimpa).
  - Re-aktivasi tingkat X ke opsi BARU (opsi C) SETELAH sebelumnya aktif opsi A — opsi A otomatis TIDAK LAGI aktif untuk X (upsert menggantikan, BUKAN menambah baris kedua).
- **`update()`**: ubah cakupan tingkat opsi yang SEDANG AKTIF di tingkat yang mau dihapus dari cakupan — **DITOLAK**. Ubah cakupan opsi yang TIDAK aktif di tingkat manapun — BERHASIL bebas.
- **`delete()`**: opsi yang AKTIF di salah satu tingkat — **DITOLAK**. Opsi yang TIDAK aktif di tingkat manapun — BERHASIL dihapus (termasuk cascade slot+tingkatScopes).
- **`getActiveForTingkat()`**: tidak ada aktivasi untuk tingkat itu — return `null`/undefined (fallback aman, TIDAK throw).

### 2. VERIFIKASI overlap dengan test T181 (kalau sudah dikerjakan)

- Kalau T181 SUDAH menambahkan test concurrent untuk `update()`/`activate()` — task ini TIDAK PERLU menduplikasi test race condition itu, CUKUP referensi/pastikan skenario non-race (sekuensial biasa) di atas tetap tercakup di sini secara terpisah dari test race T181.

## Edge Cases
- Tidak ada edge case tambahan di luar skenario Spec Detail di atas — task ini MURNI test coverage, TIDAK mengubah logic produksi (KECUALI kalau test MENEMUKAN bug baru saat ditulis — kalau terjadi, LAPORKAN ke user sebagai temuan baru, JANGAN diam-diam perbaiki tanpa konfirmasi di luar scope T181/T182 yang sudah diketahui).

## Files
- **Buat:** `apps/api/src/jam-pelajaran/jam-pelajaran.service.spec.ts`.
- **Jangan sentuh:** `jam-pelajaran.service.ts` (task ini MURNI test, implementasi TIDAK diubah — perbaikan logic ada di T181/T182 terpisah).

## Acceptance Criteria
- [x] File spec sudah ada (dibuat T181, `jam-pelajaran.service.spec.ts`) — DITAMBAHKAN 13 test baru (bukan file baru terpisah, konsisten 1 spec file per service) mencakup SEMUA skenario Spec Detail poin 1 yang BELUM tercakup T181: `create()` (4 test), `activate()` multi-tingkat/re-aktivasi (3 test), `delete()` (2 test), `getActiveForTingkat()`/`resolveJamSchedule()` (4 test).
- [x] Test constraint per-tingkat independen (2 tingkat, 2 opsi berbeda, keduanya aktif bersamaan) — PASS (`aktivasi tingkat X ke opsi A DAN tingkat XII ke opsi B — keduanya aktif bersamaan, saling independen`).
- [x] Test `activate()` menolak opsi tak-mencakup-tingkat — SUDAH ADA dari T181 (`opsi tidak mencakup tingkat yang diminta → 400`), tidak diduplikasi.
- [x] Test `update()` menolak ubah cakupan yang aktif — SUDAH ADA dari T181 (3 test regresi + 1 test bug), tidak diduplikasi.
- [x] Test `delete()` menolak hapus opsi aktif — PASS (2 test baru: ditolak jika aktif, berhasil jika tidak aktif).
- [x] Build + type-check hijau, SEMUA jest existing tetap pass — regresi nol. Full suite naik dari 348 → **361/361** (13 test baru T183, 0 gagal).

## Validasi Claudian
- [x] **Dicek dulu isi `jam-pelajaran.service.spec.ts` (sudah ada dari T181) sebelum menulis** — dikonfirmasi T181 sudah mencakup: race condition `update()`+`activate()` (7 test: 3 regresi normal, 1 reproduksi bug, 1 verifikasi urutan row-lock, 2 test `activate()` row-lock+validasi cakupan dasar). T183 HANYA menambah skenario yang BELUM ada: `create()`, `activate()` multi-tingkat independen + re-aktivasi menggantikan (bukan validasi cakupan dasar yang sudah dicek T181), `delete()`, `getActiveForTingkat()`/`resolveJamSchedule()`. **Tidak ada duplikasi test** — dikonfirmasi via `grep -n '  it('` sebelum menulis.
- [x] **Tidak ditemukan bug produksi baru** — semua 13 test baru PASS pada percobaan pertama tanpa perlu mengubah `jam-pelajaran.service.ts` (dikonfirmasi via `git status --porcelain` — 0 file produksi tersentuh, HANYA spec file yang berubah). Task murni test coverage, sesuai scope.

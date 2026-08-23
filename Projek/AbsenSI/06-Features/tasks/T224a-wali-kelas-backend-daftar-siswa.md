# T224a — API: Wali Kelas — Endpoint Daftar Siswa (Scoped)

## Depends on
Tidak ada dependency teknis ke task lain. Independen, murni backend 1 endpoint baru. **Bagian 1 dari 4** rangkaian pemecahan T224 asli (T224a/b/c backend independen satu sama lain, semuanya WAJIB selesai sebelum T224d frontend dikerjakan).

## Konteks — Fitur Wali Kelas SUDAH SEBAGIAN ADA (dikonfirmasi via riset 2026-08-19)

**JANGAN bangun dari nol** — halaman `(guru)/guru/wali-kelas/` SUDAH ADA dan berfungsi (3 tab read-only: Ringkasan Kehadiran, Rekap Per Mapel, Catatan Guru Mapel — komentar eksplisit "T053 — TIDAK ADA aksi tulis apapun"). `User.kelasIdWali` (nullable, FK ke `Kelas`) — wali kelas BUKAN role terpisah, flag pada `User` role `guru` biasa (KONSISTEN pola `guru_piket.kampus_id`).

Task ini **HANYA** membuat 1 endpoint backend baru: Daftar Siswa scoped ke kelas yang diwalikan. TIDAK menyentuh frontend sama sekali (itu scope T224d).

## Keputusan Dikonfirmasi User (2026-08-19)

Daftar siswa tampil **SEMUA siswa kelas TERMASUK yang nonaktif** — beda dari rekap kehadiran (T224c) yang justru HARUS exclude siswa nonaktif. Ini 2 perilaku berbeda BY DESIGN, bukan inkonsistensi.

## Spec Detail

- Endpoint baru: `GET /journal/kelas-wali-siswa` (KONSISTEN pola `journal-kelas-wali.controller.ts` existing — base path `/journal`, `kelasId` SELALU derive dari `user.kelasIdWali` di JWT, TIDAK PERNAH dari query parameter bebas).
- Service: tambah method baru di `JournalKelasWaliService` (atau nama service yang menaungi `journal-kelas-wali.controller.ts` saat ini — VERIFIKASI nama file service pendamping controller ini) — query `Student.findMany({ where: { kelasId: user.kelasIdWali } })`.
- **TIDAK filter status** — siswa nonaktif TETAP tampil di daftar ini. Response tambahkan field `status` (sudah ada di model `Student`) supaya FE (T224d) bisa render badge "Nonaktif" untuk siswa yang bukan `aktif`.
- `@Roles(UserRole.guru)` + validasi tambahan: `user.kelasIdWali` HARUS terisi (kalau `null`, lempar 403 dengan pesan jelas — "Anda bukan wali kelas manapun, menu ini tidak tersedia").
- Field response: `id`, `nisn`, `nama`, `status`, `jenisKelamin` (opsional, untuk avatar/ikon FE) — TIDAK PERLU field biodata lengkap di sini (itu scope T224b, endpoint terpisah saat diklik).

## Edge Cases

- **Wali kelas dengan kelas kosong** (0 siswa) — return array kosong, bukan error.
- **Guru biasa (bukan wali kelas manapun)** akses endpoint ini langsung via API (bypass UI) — 403 ditolak, pesan actionable.

## Files
- **Modifikasi:** `apps/api/src/journal/journal-kelas-wali.controller.ts` (endpoint baru), service pendamping (method baru).
- **Jangan sentuh:** `StudentsService`/`students.controller.ts` (endpoint admin existing, TIDAK diubah), frontend apa pun (scope T224d).

## Acceptance Criteria
- [x] `GET /journal/kelas-wali-siswa` return semua siswa kelas wali (termasuk nonaktif), field `status` tersedia untuk badge FE nanti.
- [x] Guru bukan wali kelas (kelasIdWali null) — 403 dengan pesan jelas.
- [x] Guru wali kelas lain — hanya lihat siswa kelasnya sendiri, tidak bisa akses kelas lain (kelasId selalu dari JWT, tidak ada parameter override).
- [x] Build + type-check hijau, jest baru: wali kelas normal (berhasil), guru bukan wali (403), kelas kosong (array kosong).

## Validasi Claudian
- [x] Konfirmasi `kelasId` di query SELALU derive dari `user.kelasIdWali` (JWT), TIDAK ADA jalur yang menerima kelasId dari client — `requireKelasIdWali()` (private, sudah ada di controller) dipakai apa adanya, `getKelasWaliSiswa(kelasId)` cuma terima 1 parameter number dari situ.

## Temuan Kritis Saat Implementasi (2026-08-19) — Konflik dengan T220

Riset menemukan T220 (task sebelumnya, sudah selesai) otomatis **null-kan `Student.kelasId`** begitu siswa jadi nonaktif — snapshot nama kelas lama disimpan ke `kelasTerakhirNama` (field TEKS, bukan FK). Akibatnya query naif `where: { kelasId }` **TIDAK PERNAH** menangkap siswa nonaktif (field itu sudah kosong), padahal task ini eksplisit minta siswa nonaktif tetap tampil.

**Dikonfirmasi user**: siswa nonaktif TETAP tampil di Daftar Siswa (task ini), TAPI TIDAK tampil di Rekap Kehadiran (T224c, konsisten T220 by design — 2 perilaku berbeda, bukan inkonsistensi).

**Solusi diimplementasikan**: query `Student.findMany({ where: { OR: [{ kelasId }, { kelasTerakhirNama: kelas.nama }] } })` — tangkap siswa AKTIF (via `kelasId` FK) DAN siswa NONAKTIF yang dulu di kelas ini (via `kelasTerakhirNama` match nama teks). Matching by nama, bukan FK kedua — cukup akurat untuk kebutuhan tampilan histori tanpa perlu migrasi schema tambahan. `kelas.findUnique()` dipanggil dulu untuk resolve nama kelas wali (dari `kelasId`), return array kosong kalau kelas itu sendiri sudah tidak ada (edge case defensif, bukan skenario yang diminta task tapi mencegah crash).

## Implementasi (2026-08-19)

**Backend**:
- `journal.service.ts` — interface `KelasWaliSiswaRow` baru (`id`, `nisn`, `nama`, `status`, `jenisKelamin`), method `getKelasWaliSiswa(kelasId)` baru (lihat solusi OR di atas). Field biodata lengkap SENGAJA tidak disertakan (scope T224b saat siswa diklik).
- `journal-kelas-wali.controller.ts` — endpoint `GET /journal/kelas-wali-siswa` baru, REUSE `requireKelasIdWali()` private method yang sudah ada (pola identik 2 endpoint existing di file yang sama).

**Verifikasi**: 6 test baru (4 `journal.service.spec.ts` — OR query benar, campuran aktif+nonaktif tanpa filter status, kelas tidak ditemukan→array kosong, kelas kosong→array kosong; 2 `journal-kelas-wali.controller.spec.ts`, file BARU sebelumnya tidak ada — 403 guru bukan wali+service tidak terpanggil, kelasId diteruskan dari JWT). tsc bersih, `nest build` bersih. Frontend TIDAK disentuh sama sekali (scope T224d terpisah).

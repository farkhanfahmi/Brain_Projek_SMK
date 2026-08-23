# T196 — API+Web: Tombol Hapus Mapel (Hanya Jika Belum Terpakai)

## Depends on
Tidak ada dependency teknis. Independen.

## Objective
Tambah tombol **Hapus** di halaman Mapel — SAAT INI sengaja tidak ada (komentar kode lama "T050 — TIDAK ada aksi hapus"). Backend **TOLAK** penghapusan kalau mapel itu SUDAH terpakai di `Schedule`/`TeachingSession`/`GradeAssessment` manapun (mencegah kerusakan data historis).

## Context — Klarifikasi User (2026-08-16)

User testing melaporkan "mapel belum bisa CRUD" — setelah dicek, TERNYATA maksudnya spesifik **tidak ada tombol Hapus** (Create/Update/List semua SUDAH berfungsi normal). Dikonfirmasi ini memang desain LAMA yang disengaja (bukan bug T186/T187) — user SEKARANG eksplisit minta fitur Hapus ditambahkan, DENGAN syarat proteksi data (dikonfirmasi user).

## Spec Detail

### 1. Backend — endpoint delete dengan proteksi

- `MapelController` — tambah `DELETE /mapel/:id`, guard SAMA seperti create/update (`super_admin, admin_jurnal`).
- `MapelService.delete(id)` — SEBELUM hapus, cek apakah `id` ini direferensikan di:
  - `Schedule.mapelId` (jadwal mengajar manapun, historis maupun aktif).
  - `TeachingSession.mapelId`.
  - `GradeAssessment.mapelId` (T172).
  - Kalau ADA REFERENSI DI SALAH SATU — TOLAK dengan `ConflictException`, pesan JELAS ("Mapel ini sudah dipakai di N jadwal/pertemuan/penilaian, tidak bisa dihapus") — REKOMENDASI: sebutkan JUMLAH referensi yang ditemukan supaya admin paham skala pemakaian, TIDAK PERLU sebutkan detail per-baris.
  - Kalau TIDAK ADA referensi sama sekali — hapus BERHASIL.
- `@LogActivity` — WAJIB terpasang di endpoint delete (KONSISTEN aturan proyek, jangan lupa).

### 2. Frontend — tombol Hapus + konfirmasi

- `mapel-view.tsx` (dan duplikat) — tambah tombol Hapus (icon trash) di kolom Aksi, SEJAJAR tombol Edit yang sudah ada.
- Dialog konfirmasi SEBELUM eksekusi (KONSISTEN pola konfirmasi aksi destruktif lain di proyek).
- Kalau backend TOLAK (mapel terpakai) — tampilkan pesan error APA ADANYA dari backend (bukan generic "gagal hapus").

## Edge Cases
- Mapel yang TIDAK PERNAH terpakai sama sekali (baru dibuat, belum di-assign ke jadwal manapun) — hapus LANGSUNG berhasil tanpa halangan.
- Mapel yang PERNAH terpakai tapi SEMUA jadwal/sesi terkait sudah lama tidak aktif (misal semester lalu) — TETAP DITOLAK (referensi historis TETAP dihitung, tidak dibedakan aktif/tidak-aktif — data historis harus tetap utuh, JANGAN biarkan mapel yang PERNAH dipakai di rekap lama jadi hilang referensinya).

## Files
- **Modifikasi:** `apps/api/src/mapel/mapel.controller.ts`+`mapel.service.ts` (endpoint delete+validasi referensi), `apps/web/.../mapel/mapel-view.tsx` (dan duplikat admin, tombol Hapus+dialog konfirmasi).
- **Jangan sentuh:** create/update Mapel existing (TIDAK diubah), `Mapel.kode` opsional (TETAP opsional, tidak diwajibkan — desain lama yang benar, bukan bug).

## Acceptance Criteria
- [x] Mapel yang TIDAK terpakai — bisa dihapus.
- [x] Mapel yang SUDAH terpakai (di Schedule/TeachingSession/GradeAssessment manapun) — DITOLAK dengan pesan jelas menyebut jumlah referensi.
- [x] `@LogActivity` tercatat untuk penghapusan yang berhasil.
- [x] Build + type-check hijau, jest baru untuk skenario terpakai vs tidak terpakai.

## Validasi Claudian
- [x] Konfirmasi SEMUA 3 titik referensi (Schedule, TeachingSession, GradeAssessment) dicek — `Promise.all` 3 `count()` terpisah, dijumlah, ditolak kalau total > 0.
- [x] Konfirmasi `Mapel.kode` TETAP opsional — tidak disentuh sama sekali di task ini.

## Status Eksekusi (2026-08-16)

**Selesai.**

### Backend

- `MapelService.delete(id)` — cek referensi PARALEL (`Promise.all`) di `Schedule.mapelId`, `TeachingSession.mapelId`, `GradeAssessment.mapelId` — jumlah TOTAL digabung (bukan per-titik), referensi HISTORIS tetap dihitung (tidak ada filter aktif/tidak-aktif, sesuai edge case spec). Kalau total 0 → `prisma.mapel.delete()`. Kalau > 0 → `ConflictException` pesan "Mapel ini sudah dipakai di N jadwal/pertemuan/penilaian, tidak bisa dihapus."
- `DELETE /mapel/:id` — guard SAMA seperti create/update (`super_admin, admin_jurnal`), `@LogActivity` terpasang (snapshot-before dari `paramId` sebelum row dihapus, pola sama `school_holiday.delete`), `@HttpCode(204)`.
- Komentar file lama "T047 — mapel tidak boleh dihapus" diupdate — desain lama itu SENGAJA direvisi task ini, bukan bug.
- 6 unit test baru (404, berhasil tanpa referensi, ditolak Schedule-saja/TeachingSession-saja/GradeAssessment-saja masing-masing, jumlah gabungan 3 titik sekaligus) — 481/481 pass di seluruh suite.

### Frontend

- `MapelView` (dipakai admin+admin-jurnal via reuse T157) — tombol Hapus (ikon trash) sejajar tombol Ubah di kolom Aksi. Konfirmasi via `window.confirm()` — pola SAMA seperti `jam-pelajaran-view.tsx` `handleDelete`, bukan komponen dialog baru. Pesan error dari backend ditampilkan apa adanya (bukan generic "gagal hapus").

### Verifikasi

- `tsc --noEmit` bersih (kecuali error pre-existing tidak terkait di `dispen-view.tsx`).
- `jest apps/api`: 481/481 pass, 29/29 suite.
- Live-verify browser: belum dilakukan (konsisten pola T186-T194, verifikasi manual diserahkan ke user).

# T169 — API+Web: Halaman "Jadwal" Guru Terpisah + Field Ruangan di Kelas

## Depends on
**WAJIB setelah T168** (shell mobile-app harus ada dulu, halaman ini adalah salah satu dari 5 tab bottom-nav).

## Objective
Pisahkan tampilan "jadwal hari ini" dari halaman Jurnal (yang saat ini melebur jadi satu di `guru/jurnal/jurnal-view.tsx`) jadi halaman tersendiri `/guru/jadwal`, dengan kartu jadwal menampilkan **ruangan** dan **kampus** (belum ada saat ini) selain kelas/mapel/jam yang sudah ada.

## Context — Temuan Riset (2026-08-13)

- Halaman "jadwal hari ini" SAAT INI justru berjudul H1 "Jadwal Mengajar Hari Ini" tapi terletak di route `guru/jurnal/jurnal-view.tsx` — SATU halaman ini sekaligus jadi pintu masuk ke jurnal (tiap kartu sesi ada tombol Mulai Mengajar → masuk ke `guru/sesi/[sessionId]` yang berisi form jurnal + presensi). Task ini MEMISAHKAN peran itu: `/guru/jadwal` HANYA tampilkan daftar sesi hari ini (kartu read-only + tombol aksi), form jurnal pindah scope-nya ke T171.
- Endpoint `GET /teaching-sessions/my-today` (`teaching-sessions.controller.ts:59-86`) SUDAH sort ascending jam (`orderBy: { schedule: { jamMulai: "asc" } }`, `teaching-sessions.service.ts:143`) — TIDAK PERLU diubah sort-nya. TAPI response-nya BELUM include ruangan/kampus (field itu belum ada di database sama sekali).
- **`Kelas` model TIDAK PUNYA field `ruangan`** (`schema.prisma` — dikonfirmasi lewat grep, nihil). `Kampus` SUDAH ada relasi via `Kelas.kampusId`. Keputusan user: ruangan ditambahkan sebagai **field baru di `Kelas`** (asumsi 1 kelas = 1 ruangan tetap sepanjang semester — bukan per-Schedule, lebih simpel dan cukup untuk mayoritas kasus SMK).

## Spec Detail

### 1. Database — `Kelas.ruangan`

- Migration Prisma: tambah kolom `ruangan String? @map("ruangan")` di model `Kelas` (`schema.prisma`, dekat field `tingkat`) — NULLABLE (kelas lama belum tentu diisi langsung, admin isi belakangan bertahap, JANGAN wajibkan retroactive).

### 2. Backend — form Tambah/Edit Kelas + expose di endpoint jadwal

- Cari controller/service Kelas (`apps/api/src/core/...` — kemungkinan `kelas.service.ts` atau serupa, VERIFIKASI lokasi pasti saat implementasi) — tambah field `ruangan` opsional ke DTO create/update Kelas.
- `TeachingSessionsService.myToday()` (`teaching-sessions.service.ts` sekitar baris 143) — response per sesi TAMBAH `ruangan` (dari `kelas.ruangan`) dan `kampus` (nama, dari `kelas.kampus.nama` — relasi SUDAH ADA, tinggal di-include kalau belum).

### 3. Frontend — form Tambah/Edit Kelas

- Cari halaman admin Master Data Kelas (`apps/web/src/app/(admin)/...` — kemungkinan di grup "Master Data Sekolah") — tambah field input "Ruangan" (text, opsional) di form create/edit Kelas yang sudah ada.

### 4. Frontend — halaman baru `/guru/jadwal`

- File baru `apps/web/src/app/(guru)/guru/jadwal/page.tsx` + `jadwal-view.tsx` — REUSE `sesi-card.tsx` yang sudah ada (`apps/web/src/app/(guru)/guru/jurnal/components/sesi-card.tsx`) sebagai basis, PERKAYA dengan baris ruangan+kampus. Fetch `GET /teaching-sessions/my-today` (endpoint SAMA, sudah diperkaya field di langkah 2).
- Tombol "Mulai Belajar" di kartu — TETAP panggil `POST /teaching-sessions/:sessionId/start` (endpoint SAMA, TIDAK diubah), lalu navigate ke `/guru/sesi/[sessionId]` (route existing, TIDAK dipindah — form jurnal+presensi di sana tetap, hanya DIPERKAYA di T171).
- Halaman lama `guru/jurnal/page.tsx` — lihat catatan di T171 (path `/guru/jurnal` akan DIALIHFUNGSIKAN, bukan dihapus, jadi hindari konflik route saat kedua task ini jalan berurutan).

## Edge Cases
- Guru tanpa sesi hari ini (libur/tidak mengajar) — tampilkan empty state jelas ("Tidak ada jadwal mengajar hari ini"), bukan halaman kosong membingungkan (cek apakah state ini sudah ditangani di `sesi-card.tsx`/`jurnal-view.tsx` lama, REUSE penanganannya).
- Kelas yang `ruangan`-nya masih NULL (belum diisi admin) — tampilkan "-" atau "Ruangan belum diatur", JANGAN crash atau tampilkan "null"/"undefined" mentah.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`Kelas.ruangan`), Kelas service/controller (DTO), `apps/api/src/teaching-sessions/teaching-sessions.service.ts` (`myToday()` response), halaman admin Master Data Kelas (form ruangan).
- **Buat:** `apps/api/prisma/migrations/xxx_add_ruangan_kelas/`, `apps/web/src/app/(guru)/guru/jadwal/page.tsx` + `jadwal-view.tsx`.
- **Jangan sentuh:** logic sort/filter `myToday()` selain menambah field response, `POST /:sessionId/start` (reuse apa adanya).

## Acceptance Criteria
- [ ] Admin bisa isi field Ruangan saat Tambah/Edit Kelas.
- [ ] `/guru/jadwal` menampilkan daftar sesi hari ini ascending jam, tiap kartu tampilkan kelas, mapel, jam, **ruangan**, **kampus**.
- [ ] Tombol "Mulai Belajar" tetap berfungsi sama seperti sebelumnya (guard `sudahTapGerbang`+jendela waktu TIDAK berubah).
- [ ] Kelas dengan ruangan NULL tidak crash, tampil placeholder jelas.
- [ ] Build + type-check `apps/web`+`apps/api` hijau, jest existing tetap pass.

## Validasi Claudian
- [ ] Konfirmasi endpoint `/teaching-sessions/my-today` TIDAK mengubah sort/filter existing, HANYA menambah field response.
- [ ] Konfirmasi migration `ruangan` NULLABLE (tidak retroactive wajib).

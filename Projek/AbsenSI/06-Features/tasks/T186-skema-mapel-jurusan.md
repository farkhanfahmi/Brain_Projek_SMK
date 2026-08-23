# T186 — API+Web: Skema Mapel ↔ Jurusan (Umum vs Khusus Jurusan)

## Depends on
**WAJIB dikerjakan SEBELUM T187** (Import Excel Mapel) — field `jurusanId` harus ada dulu supaya template Excel mengikutsertakan kolomnya. Independen dari task lain.

## Objective
Tambah field `Mapel.jurusanId` (NULLABLE) — mapel **UMUM** (NULL, misal Matematika/PKN/Bahasa Indonesia — dipakai semua jurusan) vs mapel **KHUSUS jurusan tertentu** (terisi, misal "Produktif TKR" hanya relevan untuk jurusan TKR). Filter mapel di form jadwal mengajar/lain otomatis menyesuaikan jurusan kelas yang dipilih.

## Context — Temuan Riset (2026-08-15)

`Mapel` (`schema.prisma:379-390`) SAAT INI 100% GLOBAL — hanya `id, nama, kode?(unique), createdAt` — TIDAK ADA relasi ke `Jurusan` sama sekali. Ini artinya form Jadwal Mengajar SAAT INI menampilkan SEMUA mapel di dropdown, termasuk mapel produktif jurusan lain yang tidak relevan untuk kelas yang sedang diedit — user eksplisit menyebut ini "belum ter-skema dengan baik".

**Keputusan user**: `jurusanId` NULLABLE (opsional 1-ke-1, BUKAN many-to-many) — NULL = mapel umum berlaku semua jurusan, terisi = mapel khusus jurusan itu SAJA.

## Spec Detail

### 1. Schema (Prisma)

```prisma
// Tambah ke model Mapel yang sudah ada:
jurusanId Int? @map("jurusan_id")
jurusan   Jurusan? @relation(fields: [jurusanId], references: [id])
```
- Migration baru — `jurusanId` NULLABLE, data existing (SEMUA mapel yang sudah ada) otomatis NULL (dianggap UMUM) — TIDAK PERLU backfill wajib, admin bisa isi belakangan per-mapel yang memang khusus jurusan.

### 2. Backend — filter mapel berdasar jurusan

- `MapelService.findAll()` — TAMBAH parameter opsional `jurusanId` — kalau diisi, return mapel yang `jurusanId === null OR jurusanId === filterJurusanId` (umum + khusus jurusan itu). Kalau TIDAK diisi, return SEMUA (perilaku lama, untuk halaman admin kelola Mapel yang butuh lihat semua).
- **Titik pemakaian yang PERLU diaudit dan diperbarui** — SEMUA form yang punya dropdown pilih Mapel DALAM KONTEKS 1 Kelas tertentu (yang punya `jurusanId` jelas): form create/edit `Schedule` (Jadwal Mengajar), form create Modul Nilai (T172, `/guru/nilai` pilih mapel), MUNGKIN lokasi lain — GREP MENYELURUH semua pemanggil endpoint list Mapel, PUTUSKAN per titik apakah filter jurusan relevan diterapkan (KEBANYAKAN kasus YA relevan, kecuali halaman admin murni kelola master data Mapel yang memang perlu lihat semua).

### 3. Frontend — form Tambah/Edit Mapel

- `apps/web/src/app/(admin)/mapel/` (dan duplikat admin-jurnal) — form Tambah/Edit Mapel TAMBAH dropdown "Jurusan (kosongkan untuk mapel umum)" — opsional, pilih dari daftar Jurusan existing.
- Tabel daftar Mapel — TAMBAH kolom "Jurusan" (badge "Umum" kalau NULL, atau nama jurusan kalau terisi) — KONSISTEN aturan tabel wajib sortable.
- Form Jadwal Mengajar (dan form lain yang teridentifikasi poin 2) — dropdown Mapel OTOMATIS terfilter sesuai `jurusanId` dari `kelasId` yang sedang dipilih di form itu.

## Edge Cases
- Kelas TANPA jurusan yang jelas (seharusnya tidak mungkin, `Kelas.jurusanId` WAJIB terisi di schema) — tidak relevan, semua Kelas selalu punya jurusan.
- Mapel khusus jurusan A yang SUDAH TERLANJUR dipakai di `Schedule` kelas jurusan B (data lama sebelum task ini, tidak konsisten) — task ini TIDAK melakukan validasi retroaktif/pembersihan data lama, MURNI mengatur perilaku form KE DEPAN (filter dropdown saat input BARU) — data lama TIDAK diubah/divalidasi ulang.
- Admin ubah Mapel dari "khusus jurusan A" jadi "umum" (hapus `jurusanId`) SETELAH banyak dipakai — TIDAK ADA proteksi khusus diperlukan (mapel jadi lebih longgar cakupannya, bukan lebih ketat, tidak ada data yang jadi invalid).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`Mapel` +field), `MapelService`/`MapelController` (filter opsional), `apps/web/src/app/(admin)/mapel/mapel-view.tsx` (dan duplikat admin-jurnal), form Jadwal Mengajar + form Modul Nilai (filter dropdown mapel by jurusan kelas).
- **Buat:** migration Prisma baru.

## Acceptance Criteria
- [x] Form Tambah/Edit Mapel punya dropdown Jurusan opsional.
- [x] Tabel Mapel tampilkan kolom Jurusan (badge Umum/nama jurusan), sortable.
- [x] Form Jadwal Mengajar — dropdown Mapel terfilter sesuai jurusan kelas yang dipilih (mapel umum SELALU muncul, mapel khusus jurusan lain TIDAK muncul).
- [x] Form Modul Nilai — **tidak perlu perubahan**, sudah correct by design (lihat Status Eksekusi #1 — tidak pernah query `/mapel` langsung, mapelId derive dari TeachingSession yang sudah terikat ke kelas benar).
- [x] Data Mapel existing (jurusanId NULL) tetap tampil di SEMUA jurusan (dianggap umum) — regresi nol, migration tanpa backfill.
- [x] Build + type-check hijau, jest baru untuk filter mapel by jurusan.

## Validasi Claudian
- [x] Grep MENYELURUH semua pemanggil endpoint list Mapel — laporkan titik mana saja yang diperbarui filternya dan mana yang sengaja TIDAK (misal halaman admin kelola Mapel sendiri, yang perlu lihat semua).
- [x] Konfirmasi migration TIDAK memaksa backfill `jurusanId` — data lama aman NULL (umum) secara default.

## Status Eksekusi (2026-08-16)

**Selesai** (implementasi + otomatis terverifikasi; verifikasi manual browser diserahkan ke user sesuai preferensi sesi ini).

### 1. Grep audit — semua pemanggil `GET /mapel`

| Lokasi | Diperbarui filter? | Alasan |
|---|---|---|
| `apps/web/src/app/(admin-jurnal)/admin-jurnal/mapel/page.tsx` | **TIDAK** (sengaja) | Halaman admin kelola master data Mapel — WAJIB lihat semua mapel lintas jurusan, bukan konteks 1 kelas. |
| `apps/web/src/app/(admin)/mapel/page.tsx` | **TIDAK** (sengaja) | Duplikat T157 dari baris di atas (reuse `MapelView`) — alasan sama. |
| `apps/web/src/app/(admin-jurnal)/admin-jurnal/jadwal/page.tsx` → `JadwalFormModal` | **YA** | Form Jadwal Mengajar sudah tahu konteks kelas (jurusanId) — dropdown Mapel difilter CLIENT-SIDE dari `mapelList` yang sudah di-fetch penuh (bukan re-fetch server dengan query param), supaya tidak ada round-trip ekstra tiap ganti kelas. |
| `apps/web/src/app/(admin)/jadwal-mengajar/page.tsx` → `JadwalView` → `JadwalFormModal` | **YA** | Duplikat T157 dari baris jadwal admin-jurnal (reuse `JadwalView`+`JadwalFormModal` apa adanya) — otomatis ikut terfilter tanpa perubahan terpisah. |
| `apps/web/src/app/(guru)/guru/nilai/components/create-assessment-dialog.tsx` (Modul Nilai, T172) | **TIDAK ADA PERUBAHAN — sudah benar sejak awal** | **Temuan penting**: form ini TIDAK PERNAH memanggil `GET /mapel` sama sekali. `mapelId` di-derive dari `TeachingSession` yang dicentang guru (lewat `GET /grades/sessions?kelasId=`), dan `TeachingSession` SELALU sudah terikat ke 1 kelas+mapel nyata dari Jadwal Mengajar yang sudah difilter benar. Jadi Modul Nilai otomatis correct tanpa sentuhan apa pun — acceptance criteria spec soal ini adalah asumsi keliru, dikoreksi di sini. |
| `apps/web/src/app/(admin-jurnal)/admin-jurnal/jadwal/components/salin-jadwal-modal.tsx` | Tidak relevan | Tidak ada dropdown Mapel di form ini (hanya pilih semester sumber). |
| `apps/web/src/app/(guru)/guru/wali-kelas/components/rekap-mapel-tab.tsx` | Tidak relevan | Kolom Mapel di tabel laporan read-only, bukan dropdown pilih mapel. |
| `apps/api/src/import/import.service.ts` (`importMapel`) | Tidak disentuh (di luar scope T186, akan di T187) | Import CSV belum punya kolom jurusan di template — mapel yang diimport otomatis `jurusanId: null` (umum), konsisten default aman. T187 akan tambah kolom ini. |

### 2. Backend

- `Mapel.jurusanId` (nullable, FK ke `Jurusan`, indexed) — migration `20260815142147_mapel_jurusan_optional`, **TIDAK ADA backfill** (kolom baru langsung nullable, semua mapel existing otomatis NULL/umum, terverifikasi lewat `prisma migrate dev` yang apply tanpa prompt reset/backfill).
- `MapelService.findAll(jurusanId?)` — kalau diisi: `WHERE jurusanId IS NULL OR jurusanId = :filter`. Kalau tidak: semua mapel (perilaku lama, dipakai halaman admin).
- `GET /mapel?jurusanId=` — query param opsional baru (`ListMapelDto`), tapi FE Jadwal Mengajar tidak memakainya (filter client-side dari `mapelList` yang sudah ada, lihat tabel di atas) — param ini tersedia untuk konsumen API lain yang mungkin butuh server-side filter.
- `GET /jurusan` — role `admin_jurnal` ditambahkan ADDITIF ke `@Roles()` existing (dulu hanya super_admin/guru_piket/guru/pembina_ekstra/card_admin) supaya form Tambah/Edit Mapel bisa isi dropdown Jurusan.
- Pesan error P2003 (FK `jurusanId` tidak valid) ditangkap eksplisit → 400 pesan jelas, bukan 500 generik — sesuai aturan wajib CLAUDE.md terbaru.
- 4 unit test baru (`findAll` dengan/tanpa filter, error P2003, update `jurusanId: null` eksplisit) — semua pass.

### 3. Frontend

- `MapelView` (dipakai `(admin)/mapel` + `(admin-jurnal)/admin-jurnal/mapel`, reuse T157) — form Tambah/Edit dapat dropdown "Jurusan (kosongkan untuk mapel umum)"; tabel dapat kolom Jurusan (badge violet `status-processing` untuk nama jurusan, badge netral untuk "Umum").
- **Bonus di luar scope literal spec**: tabel Mapel sebelumnya TIDAK PATUH aturan wajib search+sort+kolom No (CLAUDE.md) — dibawa ke compliance BARENGAN penambahan kolom Jurusan (bukan menambah kolom baru di atas tabel yang sudah lama tidak patuh), memakai `SortableHeader` existing.
- `JadwalFormModal` — dropdown Mapel filter client-side sesuai `selectedKelas.jurusanId`, reset `mapelId` saat ganti kelas (mapel yang sudah kepilih di luar jurusan baru tidak nyangkut diam-diam), pesan jelas kalau 0 mapel valid untuk jurusan itu.

### 4. Verifikasi

- `tsc --noEmit` bersih di `apps/api` dan `apps/web` (2x dijalankan — sekali setelah implementasi T186, sekali lagi setelah perubahan eksternal T158/T159/T185 yang masuk ke `jadwal-view.tsx` di tengah sesi, keduanya clean).
- `jest` full suite `apps/api`: **437/437 pass**, 29 suites (termasuk 4 test baru T186).
- Live-verify browser: checklist langkah manual sudah diberikan ke user (login `adminjurnal`, buat mapel umum vs khusus jurusan TKR, cek badge+search+sort di tabel, cek dropdown Jadwal Mengajar terfilter benar per jurusan kelas) — **belum dikonfirmasi user saat dokumentasi ini ditulis** (tabel `mapel` masih kosong di dev DB), sesuai preferensi sesi ini untuk verifikasi manual bukan Playwright otomatis.

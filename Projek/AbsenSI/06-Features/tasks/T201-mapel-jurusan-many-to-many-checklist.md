# T201 — API+Web: Mapel ↔ Jurusan Many-to-Many (Checklist, Bukan Dropdown Single)

## Depends on
**MEREVISI T186** (SUDAH SELESAI — `Mapel.jurusanId` single nullable FK). Task ini MENGGANTI struktur itu jadi many-to-many. **WAJIB SEBELUM T187 disempurnakan lebih lanjut** (template Excel Mapel perlu kolom jurusan yang mendukung multi-value).

## Objective
`Mapel.jurusanId` (SAAT INI `Int?` — 1 mapel = 0 atau 1 jurusan) — GANTI jadi **many-to-many** via junction table — 1 mapel bisa terikat ke **beberapa jurusan sekaligus** (bukan cuma "1 jurusan" atau "semua/umum"). Form Tambah/Edit Mapel — GANTI dropdown single-select jadi **CHECKLIST** semua jurusan (bisa centang 0/beberapa/semua).

## Context — Revisi Keputusan User (2026-08-16)

T186 (SELESAI) mengimplementasikan `jurusanId` sebagai single nullable FK berdasarkan keputusan awal "opsional 1-ke-1". User SEKARANG merevisi: **"saya ingin ada mapel yang bisa digunakan untuk beberapa jurusan saja, semua jurusan, dan 1 jurusan"** — 3 skenario ini TIDAK BISA direpresentasikan dengan single FK (yang cuma bisa NULL=semua atau 1 jurusan spesifik, TIDAK BISA "beberapa tapi tidak semua").

**Riset kondisi kode SEKARANG** (2026-08-16, sebelum revisi ini):
- `Mapel.jurusanId` (`schema.prisma:384`) — `Int?` single FK.
- Form Tambah/Edit — `mapel-view.tsx:229` dropdown single-select.
- **2 titik filter yang PERLU DIUBAH BERSAMAAN** (logic single-match `jurusanId === null || jurusanId === X`):
  - Frontend: `(admin-jurnal)/admin-jurnal/jadwal/components/jadwal-form-modal.tsx:48`.
  - Backend: `apps/api/src/mapel/mapel.service.ts:20` (dipanggil via `GET /mapel?jurusanId=`).
  - Test: `mapel.service.spec.ts:100-106` (perlu diperbarui skenarionya).

## Spec Detail

### 1. Schema (Prisma) — junction table many-to-many

```prisma
// HAPUS dari Mapel:
// jurusanId Int? @map("jurusan_id")
// jurusan   Jurusan? @relation(...)

// TAMBAH model baru:
model MapelJurusan {
  id        Int    @id @default(autoincrement())
  mapelId   Int    @map("mapel_id")
  jurusanId Int    @map("jurusan_id")

  mapel   Mapel   @relation(fields: [mapelId], references: [id], onDelete: Cascade)
  jurusan Jurusan @relation(fields: [jurusanId], references: [id])

  @@unique([mapelId, jurusanId])
  @@map("mapel_jurusan")
}
```
- **Konvensi "umum" (semua jurusan)**: kalau `Mapel` TIDAK PUNYA baris `MapelJurusan` SAMA SEKALI (0 baris terkait) → dianggap UMUM (semua jurusan boleh pakai) — KONSISTEN makna lama "NULL = umum", HANYA representasinya berubah dari "field NULL" jadi "tidak ada baris junction".
- **Migration data lama**: SEMUA mapel yang SEBELUMNYA punya `jurusanId` terisi (dari T186) — buat 1 baris `MapelJurusan` untuk PASANGAN itu (migrasi 1-ke-1 jadi 1 baris many-to-many, TIDAK ADA DATA YANG HILANG). Mapel yang SEBELUMNYA `jurusanId: null` — TETAP 0 baris junction (tetap umum, tidak berubah maknanya).

### 2. Backend — filter & CRUD

- `MapelService.findAll(jurusanId?)` — GANTI query dari `WHERE jurusanId = X OR jurusanId IS NULL` jadi: `WHERE NOT EXISTS (SELECT 1 FROM mapel_jurusan) OR EXISTS (SELECT 1 FROM mapel_jurusan WHERE jurusanId = X)` (ATAU pola Prisma yang setara — REKOMENDASI: `{ OR: [{ jurusanRelations: { none: {} } }, { jurusanRelations: { some: { jurusanId: X } } }] }` — VERIFIKASI sintaks Prisma persis saat implementasi).
- `MapelService.create()`/`update()` — terima `jurusanIds: number[]` (array, BOLEH KOSONG = umum) — REPLACE SELURUH baris `MapelJurusan` untuk mapel itu tiap update (hapus semua lama, insert baru — pola REPLACE PENUH, KONSISTEN pola `JamPelajaranService.update()` untuk slot — lebih sederhana dan aman daripada diff-patch parsial).
- `mapel.service.spec.ts` — perbarui test yang mengasumsikan single `jurusanId` — tambah skenario BARU: mapel dengan 2+ jurusan sekaligus muncul benar untuk KEDUA jurusan itu, TIDAK muncul untuk jurusan ketiga yang tidak dicentang.

### 3. Frontend — checklist, bukan dropdown

- `mapel-view.tsx` — GANTI dropdown single-select Jurusan jadi **CHECKLIST** (daftar SEMUA jurusan, masing-masing dengan checkbox) — 0 dicentang = Umum, 1 dicentang = khusus 1 jurusan, beberapa dicentang = khusus beberapa jurusan, SEMUA dicentang = secara efektif sama seperti Umum tapi EKSPLISIT tercatat sebagai "semua jurusan dicentang satu-satu" (BUKAN otomatis dianggap "0 baris" — VERIFIKASI keputusan ini: apakah mencentang SEMUA jurusan harus SAMA PERSIS record-nya dengan tidak mencentang apa pun/Umum, atau dibedakan secara data meski maknanya sama secara fungsional — REKOMENDASI: biarkan BEDA secara data record TAPI SAMA secara filter-effect, supaya UI tidak perlu logic khusus "kalau semua dicentang, otomatis uncheck semua" yang membingungkan).
- Tabel daftar Mapel — kolom "Jurusan" tampilkan **badge multi** (misal "TKR, TKJ" kalau 2 jurusan, "Umum" kalau 0 baris) — BUKAN 1 badge tunggal seperti sebelumnya.
- `jadwal-form-modal.tsx:48` — GANTI filter dari `m.jurusanId === null || m.jurusanId === selectedKelas.jurusanId` jadi cek KEANGGOTAAN array (`m.jurusanIds.length === 0 || m.jurusanIds.includes(selectedKelas.jurusanId)`) — SESUAIKAN response API `GET /mapel` supaya expose `jurusanIds: number[]` (bukan `jurusanId` tunggal).

## Edge Cases
- Mapel yang PERNAH terikat 1 jurusan (data lama T186), lalu admin TAMBAH jurusan kedua — SUKSES, sekarang mapel itu terikat 2 jurusan sekaligus (bukan digantikan, ditambah).
- Hapus SEMUA centang jurusan dari mapel yang tadinya khusus — mapel jadi UMUM kembali (0 baris junction) — SESUAI konvensi.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (hapus `Mapel.jurusanId`, tambah `MapelJurusan`), `apps/api/src/mapel/mapel.service.ts`+`mapel.controller.ts`+DTO (many-to-many), `apps/web/.../mapel/mapel-view.tsx` (checklist), `apps/web/.../jadwal/components/jadwal-form-modal.tsx` (filter array).
- **Buat:** migration Prisma dengan migrasi data lama (1 jurusanId → 1 baris MapelJurusan).

## Acceptance Criteria
- [x] Form Tambah/Edit Mapel — checklist jurusan (0/beberapa/semua), bukan dropdown single.
- [x] Mapel bisa terikat ke 2+ jurusan sekaligus.
- [x] Data lama (mapel dengan 1 jurusan dari T186) ter-migrasi benar ke 1 baris `MapelJurusan`, TIDAK ada data hilang (dev DB kosong saat migrasi, tapi migration SQL diverifikasi benar untuk kasus data terisi).
- [x] Filter dropdown Mapel di form Jadwal Mengajar — benar untuk kasus 0/1/beberapa jurusan.
- [x] Tabel Mapel tampilkan badge multi-jurusan.
- [x] Build + type-check hijau, jest diperbarui+ditambah untuk skenario many-to-many.

## Validasi Claudian
- [x] **Migrasi data lama diverifikasi**: migration.sql tangan-edit (Prisma `--create-only`) — `CREATE TABLE mapel_jurusan` DULU, LALU `INSERT ... SELECT` dari `mapel.jurusan_id IS NOT NULL` SEBELUM `DROP COLUMN jurusan_id` — urutan ini KRITIKAL (kalau kolom di-drop duluan datanya hilang tak bisa dibaca lagi). Dev DB kebetulan 0 baris terisi saat migrasi, tapi SQL diverifikasi logic-nya benar untuk kasus produksi yang mungkin ada data.
- [x] Konfirmasi KEDUA titik filter diperbarui BERSAMAAN: `jadwal-form-modal.tsx` (`m.jurusan.some(...)`) DAN `mapel.service.ts` (`jurusanRelations: { some/none }`) — tidak ada yang terlewat.

## Status Eksekusi (2026-08-16)

**Selesai.**

### Schema & Migration

- `Mapel.jurusanId` (single `Int?`) DIHAPUS total. Model baru `MapelJurusan` (junction, `@@unique([mapelId, jurusanId])`, `onDelete: Cascade` dari sisi Mapel). `Jurusan.mapel` relation diganti `Jurusan.mapelRelations`.
- Migration `20260816081203_mapel_jurusan_many_to_many` — DIEDIT TANGAN dari hasil `prisma migrate dev --create-only` supaya urutan operasi benar: create table junction → migrasi data (`INSERT...SELECT WHERE jurusan_id IS NOT NULL`) → BARU drop kolom lama. Diverifikasi via `DESCRIBE mapel`/`DESCRIBE mapel_jurusan` di dev DB — kolom lama hilang, tabel junction ada dengan struktur benar.

### Backend

- `MapelService.findAll(jurusanId?)` — filter Prisma relational: `{ OR: [{ jurusanRelations: { none: {} } }, { jurusanRelations: { some: { jurusanId } } }] }` (mapel umum = 0 baris relasi, ATAU punya relasi ke jurusan yang diminta).
- `create()`/`update()` — terima `jurusanIds: number[]`. Create: insert baris `MapelJurusan` SETELAH mapel dibuat (butuh id). Update: REPLACE PENUH (`deleteMany` semua lama LALU `createMany` baru sesuai array terkini) — KONSISTEN pola `JamPelajaranService.update()` untuk slot, sesuai rekomendasi spec. `jurusanIds` undefined (field di-omit) = tidak diubah; `[]` eksplisit = jadi umum.
- Response API diratakan (`flattenJurusan()` private helper) — expose `jurusan: Jurusan[]` langsung, BUKAN `jurusanRelations[].jurusan` mentah — FE tidak perlu tahu junction table adalah detail implementasi.
- `ImportService.importMapel()` — kolom `jurusan` di CSV/Excel sekarang boleh berisi BEBERAPA nama dipisah koma (contoh: `"TKR, TSM"`), SEMUA nama harus ketemu atau seluruh baris ditolak (bukan partial). Template Excel diupdate dengan contoh multi-jurusan.
- 30 unit test baru/diperbarui (`mapel.service.spec.ts` 21 test + `import.service.spec.ts` bagian Mapel 9 test) — 491/491 pass di seluruh suite backend.

### Frontend

- `Mapel.jurusanId: number | null` → `jurusan: Jurusan[]` di `core-types.ts`.
- `MapelView` — dropdown single-select diganti CHECKLIST (daftar semua jurusan + checkbox), badge tabel jadi multi (`flex flex-wrap`, 1 badge per jurusan, "Umum" kalau array kosong).
- `jadwal-form-modal.tsx` — filter dropdown Mapel: `m.jurusan.length === 0 || m.jurusan.some((j) => j.id === selectedKelas.jurusanId)`.

### Verifikasi

- `tsc --noEmit` bersih di `apps/api` dan `apps/web` (tidak ada error tersisa sama sekali, termasuk error `dispen-view.tsx` yang sebelumnya pre-existing dari task lain — sudah resolve sendiri via kerja paralel).
- `jest apps/api`: 491/491 pass, 29/29 suite.
- Live-verify browser: belum dilakukan (konsisten pola T186-T200, verifikasi manual diserahkan ke user).

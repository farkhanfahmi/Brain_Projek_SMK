# T071 — Schema: Field Tinggal Kelas + Kolom Jumlah Siswa

## Depends on
T002 (schema dasar). Tidak depend Fase 2 manapun.

## Objective
Tambahkan kolom `tinggalKelasPada` ke `Student` (untuk fitur Kenaikan Kelas Massal, T073), dan siapkan endpoint hitung jumlah siswa per kelas (untuk kolom baru di tabel Kelas, T072).

## Context
- **App:** `apps/api`
- **Ref:** `Projek/AbsenSI/06-Features/plotting-siswa-kelas.md` — bagian "2️⃣ Kenaikan Kelas Massal"

## Spec Detail

### Tambah ke model `Student`
```prisma
model Student {
  // ... existing (termasuk field dari T063) ...
  tinggalKelasPada DateTime? @map("tinggal_kelas_pada") @db.Date
}
```
- Nullable, `null` = tidak pernah ditandai tinggal kelas. Ini FIELD TERPISAH dari `status`/`alasanNonaktif` (T063) — siswa tinggal kelas TETAP `status: aktif`

### Endpoint: `GET /kelas` (extend existing, JANGAN buat endpoint baru)
- Tambah field response `jumlahSiswa` (COUNT dari `students` where `kelasId` = kelas ini DAN `status: aktif`) — cek controller/service `kelas` existing, tambahkan `_count` relasi Prisma atau query terpisah

## JANGAN
- ❌ JANGAN gabungkan `tinggalKelasPada` dengan `alasanNonaktif` (T063) — dua konsep berbeda, siswa tinggal kelas tidak pernah jadi `nonaktif`
- ❌ JANGAN hitung `jumlahSiswa` termasuk siswa `nonaktif` — hanya siswa aktif yang relevan untuk kolom ini

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma`
- **Buat:** migration Prisma
- **Modifikasi:** `apps/api/src/core/kelas/kelas.service.ts` / controller — tambah `jumlahSiswa` ke response `GET /kelas`

## Acceptance Criteria
- [ ] `prisma migrate dev` berjalan tanpa error
- [ ] `GET /kelas` response tiap kelas punya `jumlahSiswa` yang cocok dengan `COUNT` manual via MySQL MCP
- [ ] Siswa `nonaktif` tidak ikut terhitung di `jumlahSiswa`

## Handoff ke T072, T073
T072 (menu Kelas — tabel + tombol aksi) pakai `jumlahSiswa` dari endpoint ini. T073 (Kenaikan Kelas Massal) pakai kolom `tinggalKelasPada`.

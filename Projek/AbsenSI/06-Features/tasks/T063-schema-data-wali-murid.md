# T063 — Schema: Tambah Data Wali Murid ke Student

## Depends on
T061 (schema tambahan biodata guru — dikerjakan bersamaan/berurutan, sama-sama prasyarat T062)

## Objective
Tambahkan field kontak wali murid (nama & no HP ayah/ibu, no HP siswa, RT/RW) ke `Student` — ditemukan hilang saat perbandingan dengan database lama, relevan untuk fitur Notifikasi Orang Tua (Fase 3, sudah ada di backlog `notifikasi-ortu.md`).

## Context
- **App:** `apps/api`
- **Ref:** `Projek/AbsenSI/06-Features/migrasi-database-lama.md`, `Projek/AbsenSI/06-Features/notifikasi-ortu.md` (konsumen masa depan field ini)

## Spec Detail

### Verifikasi sudah dilakukan (2026-07-22): `rtRw`, `namaAyah`, `namaIbu` SUDAH ADA di schema saat ini (baris 104-106 `apps/api/prisma/schema.prisma`, dari T028 biodata lengkap Fase 1) — TIDAK perlu ditambahkan lagi, JANGAN duplikat.

### Tambah ke model `Student` (HANYA 3 kolom ini yang benar-benar baru)
```prisma
model Student {
  // ... field existing (termasuk rtRw, namaAyah, namaIbu yang SUDAH ADA) tidak berubah ...
  noHpSiswa String? @map("no_hp_siswa")
  noHpAyah  String? @map("no_hp_ayah")
  noHpIbu   String? @map("no_hp_ibu")
}
```

## JANGAN
- ❌ JANGAN tambah ulang `rtRw`/`namaAyah`/`namaIbu` — sudah ada, menambah lagi akan menyebabkan konflik migration
- ❌ JANGAN buat field ini non-nullable — semua data kontak wali murid ini opsional (banyak siswa lama datanya kosong/NULL sesuai temuan sample data)

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma`
- **Buat:** migration Prisma (HANYA untuk kolom yang benar-benar baru)

## Acceptance Criteria
- [ ] Tidak ada kolom duplikat dengan yang sudah ada dari T028
- [ ] Semua kolom baru nullable
- [ ] `prisma migrate dev` berjalan tanpa error

## Handoff ke T062
T062 (ETL migrasi data) akan mengisi kolom-kolom ini dari `siswas.nama_ayah`, `nama_ibu`, `no_hp_ayah`, `no_hp_ibu`, `no_hp_siswa`, `rtrw` di data lama.

# T075 — Schema: Kolom Tingkat Terpisah di Kelas

## Depends on
T002 (schema dasar). Tidak depend Fase 2 manapun — sebaiknya dikerjakan SEBELUM T073 (revisi Kenaikan Kelas Massal butuh kolom ini untuk logic dropdown "Lulus").

## Objective
Tambahkan kolom `tingkat` terpisah ke model `Kelas` — ditemukan celah saat merancang Kenaikan Kelas Massal (T073): sistem butuh tahu "kelas ini tingkat berapa" (X/XI/XII) untuk logic seperti "opsi Lulus hanya muncul untuk kelas XII", tapi info itu sekarang cuma tersirat di dalam string `nama` (misal "X TKJ 1") — rapuh untuk diparsing dan gampang salah kalau penamaan tidak konsisten.

## Context
- **App:** `apps/api`
- **Ref:** Diskusi 2026-07-22 soal Kenaikan Kelas Massal (lihat `plotting-siswa-kelas.md` revisi T073)

## Spec Detail

### Tambah ke model `Kelas`
```prisma
enum Tingkat {
  X
  XI
  XII
}

model Kelas {
  // ... existing (id, nama, kampusId, jurusanId) TIDAK BERUBAH ...
  tingkat Tingkat
}
```
- **`nama` TETAP seperti sekarang** (misal "X TKJ 1", lengkap) — `tingkat` adalah kolom TAMBAHAN untuk logic, BUKAN pengganti nama. Tidak ada perubahan tampilan di halaman manapun yang sudah menampilkan `nama` apa adanya
- Kolom **wajib diisi** (non-nullable) untuk kelas BARU — tapi kelas EXISTING di database perlu diisi lewat migrasi data (lihat di bawah), tidak bisa langsung `NOT NULL` tanpa backfill

### Migrasi Data untuk Kelas Existing
- Setelah kolom ditambahkan (sementara nullable dulu saat migration), jalankan 1 script/query backfill: parse `nama` existing untuk tebak `tingkat` (cari substring "X"/"XI"/"XII" di awal nama — HATI-HATI "X" adalah substring dari "XI" dan "XII", parsing harus urutan XII → XI → X, atau split by whitespace ambil token pertama)
- Setelah backfill selesai dan diverifikasi manual (tampilkan hasil parsing ke user untuk dicek dulu SEBELUM constraint `NOT NULL` diterapkan), baru ubah kolom jadi non-nullable di migration berikutnya
- **JANGAN** langsung `NOT NULL` di migration pertama yang sama dengan tambah kolom — proses 2 tahap (tambah nullable → backfill → verifikasi manual dengan user → constraint NOT NULL) untuk mencegah data kelas existing rusak/salah tebak tanpa terverifikasi

### Update Form Kelas (Create/Edit) — extend existing, JANGAN buat form baru
- Tambah dropdown "Tingkat" (X/XI/XII) di form Tambah/Edit Kelas yang sudah ada

## JANGAN
- ❌ JANGAN ubah/hapus kolom `nama` — tetap ada apa adanya, tingkat murni tambahan
- ❌ JANGAN langsung set `NOT NULL` tanpa backfill+verifikasi manual dulu — kelas existing bisa rusak datanya kalau parsing string salah tebak tanpa dicek
- ❌ JANGAN parse substring "X" sebelum "XI"/"XII" — "XI TKJ 1" akan salah kebaca "X" kalau urutan pengecekan salah

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma`
- **Buat:** migration Prisma (2 tahap: tambah kolom nullable, lalu constraint NOT NULL setelah backfill)
- **Buat:** script backfill (`apps/api/scripts/backfill-tingkat-kelas.ts` atau serupa) — cetak hasil parsing untuk direview user sebelum apply
- **Modifikasi:** form Kelas existing (`apps/web/.../kelas`) — tambah dropdown Tingkat

## Acceptance Criteria
- [ ] Kolom `tingkat` ada di semua baris `Kelas` existing setelah backfill, terverifikasi manual cocok dengan `nama` masing-masing (misal "X TKJ 1" → tingkat `X`, "XI RPL 2" → tingkat `XI`)
- [ ] Kelas baru WAJIB isi tingkat lewat form (dropdown), tidak bisa disubmit tanpa itu
- [ ] Kolom `nama` tidak berubah nilainya untuk kelas manapun (backfill hanya ISI kolom baru, tidak menyentuh `nama`)

## Handoff ke T073 (revisi)
T073 (Kenaikan Kelas Massal, desain baru) memakai `tingkat` untuk menentukan baris kelas XII mana yang dapat opsi dropdown tujuan "Lulus".

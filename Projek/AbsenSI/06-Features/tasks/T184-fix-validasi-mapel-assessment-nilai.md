# T184 — API: Fix Validasi `mapelId` Saat Buat Penilaian (Bug T172)

## Depends on
Tidak ada dependency teknis. **Ditemukan saat audit kritis pasca-eksekusi T172 (2026-08-15)** — gap validasi kecil, severity sedang (integritas data, bukan celah keamanan lintas-guru).

## Objective
`GradesService.createAssessment()` — tambah validasi `dto.mapelId` HARUS cocok dengan mapel dari SEMUA `sessionIds` yang dipilih, sejajar dengan validasi `kelasId` yang SUDAH ADA untuk kasus serupa.

## Context — Bug Ditemukan Saat Audit (2026-08-15)

Audit kritis pasca-eksekusi T172 (Modul Nilai) menemukan: `apps/api/src/grades/grades.service.ts` (~baris 46-62) — `createAssessment()` SUDAH BENAR memvalidasi:
1. Semua `sessionIds` milik `teacherId` yang login (403 kalau tidak).
2. Semua sesi berasal dari `kelasId` YANG SAMA dengan yang diminta (400 kalau campuran) — SESUAI spec T172 Edge Cases.

**TAPI** `dto.mapelId` yang dikirim client langsung disimpan ke `GradeAssessment.mapelId` (~baris 76) **TANPA cross-check** terhadap `sessions[].mapelId` — padahal `TeachingSession.mapelId` ADA di schema dan field ini SEHARUSNYA divalidasi dengan pola YANG SAMA PERSIS seperti `kelasId`.

**Skenario kegagalan konkret**: Guru X mengajar MATEMATIKA dan FISIKA di kelas yang sama (XI TKR 3). Guru itu buat Penilaian, pilih 2 pertemuan MATEMATIKA (sessionIds valid, kelasId cocok) — TAPI `dto.mapelId` yang dikirim (baik sengaja salah pilih di dropdown UI, ATAU bug frontend, ATAU request dimanipulasi manual) adalah FISIKA. Backend TIDAK menolak — assessment tersimpan dengan `mapelId: Fisika` padahal cakupan pertemuannya MATEMATIKA. Laporan nilai jadi tercatat di mapel yang salah, membingungkan rekap/riwayat guru maupun siswa.

**BUKAN celah keamanan** (tetap guru yang sama, bukan lintas-tenant) — MURNI gap integritas data.

## Spec Detail

### 1. Backend — tambah validasi `mapelId`

- `apps/api/src/grades/grades.service.ts`, `createAssessment()` — SEJAJAR dengan pengecekan `sesiBedaKelas` yang SUDAH ADA (~baris 57-62), tambah pengecekan SERUPA:
  ```ts
  const sesiBedaMapel = sessions.some((s) => s.mapelId !== dto.mapelId);
  if (sesiBedaMapel) {
    throw new BadRequestException(
      "Semua pertemuan yang dipilih harus dari mapel yang sama dengan Penilaian ini",
    );
  }
  ```
  (SESUAIKAN nama variabel/pesan persis gaya kode existing di file itu — REPLIKASI pola `sesiBedaKelas`, JANGAN buat gaya validasi baru berbeda.)

## Edge Cases
- Assessment YANG SUDAH TERLANJUR DIBUAT dengan mapel salah (dari SEBELUM fix ini) — task ini TIDAK melakukan backfill/koreksi data lama (di luar scope, murni perbaikan validasi ke depan) — REKOMENDASI (opsional, laporkan ke user, bukan wajib dikerjakan): query manual untuk cari assessment yang `mapelId` tidak cocok dengan `sessions[].mapelId`-nya, supaya bisa dicek/dikoreksi manual kalau ada.

## Files
- **Modifikasi:** `apps/api/src/grades/grades.service.ts` (`createAssessment()`, tambah 1 validasi).
- **Jangan sentuh:** validasi `kelasId` yang sudah ada (REUSE pola-nya, TIDAK diubah), `updateEntries()`/endpoint lain (tidak terkait bug ini).

## Acceptance Criteria
- [x] Create assessment dengan `sessionIds` yang mapel-nya BEDA dari `dto.mapelId` → DITOLAK `BadRequestException`, pesan "Semua sesi yang dipilih harus dari mapel yang sama dengan Penilaian ini" (sejajar persis pesan kelasId).
- [x] Create assessment dengan `sessionIds` yang mapel-nya SAMA dengan `dto.mapelId` → TETAP berhasil (regresi nol, diverifikasi test eksplisit).
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `nest build` sukses), 2 test baru (bug + regresi) ditambahkan ke `grades.service.spec.ts` existing.

## Validasi Claudian
- [x] **Bug direproduksi SEBELUM fix** — test ditulis dan dijalankan dulu terhadap kode lama, GAGAL persis seperti diprediksi (`Received promise resolved instead of rejected` — mapel berbeda lolos tanpa ditolak), baru fix diterapkan dan test lulus.
- [x] **Regresi nol dikonfirmasi** — seluruh 18 test existing di `grades.service.spec.ts` tetap lulus (fixture `sessionFixture()` ditambah default `mapelId: 30` yang cocok `dto.mapelId`, tidak mengubah perilaku test manapun yang sudah ada). Full suite backend 426/426 (naik dari 424).

# T227 — Migrasi Data: Bersihkan kelasId Siswa Nonaktif Lama (Retroaktif) + Filter Pengaman di Query Rekap

## Depends on
Tidak ada dependency teknis. **PRIORITAS TINGGI, DESTRUKTIF (UPDATE massal ke production)** — WAJIB ikuti protokol backup lengkap CLAUDE.md "Commit di Dev = Auto-Deploy ke Production" DAN "Aturan WAJIB Sebelum Commit Migration Destruktif" sebelum eksekusi apa pun ke production.

## Konteks — Bug Nyata Ditemukan (2026-08-20)

User melaporkan siswa **"NABILA ICA FITRIANI" (id=1743, NISN 0102009079, kelas X GAME DEV)** yang berstatus `nonaktif` MASIH MUNCUL di Rekap Kehadiran Murid (admin) DAN Rekap Wali Kelas (T224c). Investigasi database production (port 3309) mengonfirmasi:

```
id=1743, nama="NABILA ICA FITRIANI", status="nonaktif",
kelas_id=48, kelas_terakhir_nama=NULL, alasan_nonaktif="lainnya"
```

**Root cause TERKONFIRMASI via `activity_log`**: Nabila Ica dinonaktifkan **2026-08-07 03:39:46** — **12 hari SEBELUM** fix T220 di-deploy ke production (commit `2da42f8`, 2026-08-19 10:28:15). `snapshot_after` di log aktivitas itu menunjukkan `kelasId=48` (TIDAK di-null-kan) dan `kelasTerakhirNama=null` (tidak di-snapshot) — persis bug yang diperbaiki T220, TAPI transisi ini terjadi sebelum kode fix itu ada.

**T220 SUDAH BENAR untuk transisi status BARU** (dikonfirmasi kode `StudentsService.update()` dan `KelasService.kenaikanMassal()` saat ini sudah null-kan `kelasId` dengan benar) — TAPI T220 **sengaja TIDAK melakukan migrasi retroaktif** ke data lama (keputusan sadar di spec T220, butuh konfirmasi terpisah). **Klaim T224c ("siswa nonaktif otomatis exclude berkat T220") KELIRU** — itu hanya berlaku untuk siswa yang dinonaktifkan SETELAH 2026-08-19, bukan SEMUA siswa nonaktif.

**Kemungkinan Nabila Ica BUKAN satu-satunya** — siswa nonaktif lain yang transisinya terjadi sebelum 2026-08-19 kemungkinan besar punya masalah yang sama.

## Spec Detail

### 1. WAJIB SEBELUM APAPUN — Backup Manual Production

```bash
bash /home/anunnaki/scripts/backup-absensi.sh
ls -lh /media/anunnaki/DataNvme/backups/absensi/ | tail -3   # VERIFIKASI file baru benar-benar muncul
```
TIDAK BOLEH DILEWATI — ini UPDATE massal ke tabel `students` production, konsisten protokol pasca-insiden 2026-07-30 dan 2026-08-19.

### 2. Identifikasi cakupan dampak SEBELUM migrasi

Jalankan query READ-ONLY dulu untuk tahu PERSIS berapa banyak siswa terdampak (JANGAN asumsi cuma 1 siswa seperti laporan awal):

```sql
SELECT id, nama, nisn, status, kelas_id, kelas_terakhir_nama, alasan_nonaktif
FROM students
WHERE status = 'nonaktif' AND kelas_id IS NOT NULL;
```

- Catat JUMLAH baris hasil query ini — SEBUTKAN angka pastinya sebelum lanjut (bukan "beberapa", angka konkret).
- Kalau jumlahnya sangat besar (ratusan+) — STOP, konfirmasi ke user dulu apakah ini sesuai ekspektasi (bisa jadi indikasi bug lain yang belum ditemukan) sebelum lanjut migrasi massal.

### 3. Migrasi data — null-kan kelasId + isi kelasTerakhirNama

**JANGAN pakai raw SQL UPDATE massal langsung** (risiko tinggi, tidak ada dry-run) — REKOMENDASI: script Node/ts-node terpisah (pola serupa `scripts/seed-jurnal-uji-coba.ts` dari T225, TAPI untuk migrasi bukan seed) yang:

1. Query SEMUA siswa `status='nonaktif' AND kelasId IS NOT NULL`, JOIN `kelas.nama` untuk tiap baris.
2. **DRY RUN dulu** — print daftar lengkap (nama siswa + nama kelas asal) yang AKAN diubah, TANPA eksekusi UPDATE — user/operator review daftar ini dulu.
3. Setelah konfirmasi eksplisit (flag `--execute` atau prompt konfirmasi terminal), baru jalankan UPDATE per-siswa (bukan `updateMany` — supaya `kelasTerakhirNama` benar per siswa sesuai kelas asalnya masing-masing, KONSISTEN pola per-individual-update yang sudah dipakai T220 di `kenaikanMassal()`):
   ```ts
   await prisma.student.update({
     where: { id: siswa.id },
     data: { kelasId: null, kelasTerakhirNama: siswa.kelas.nama },
   });
   ```
4. Print ringkasan akhir: jumlah baris berhasil diubah, dan verifikasi ulang query poin 2 sekarang harus return 0 baris.

### 4. Verifikasi row count SEBELUM/SESUDAH (wajib, protokol proyek)

```sql
-- SEBELUM migrasi
SELECT COUNT(*) FROM students WHERE status = 'nonaktif' AND kelas_id IS NOT NULL;
-- catat angka X

-- SESUDAH migrasi, harus 0
SELECT COUNT(*) FROM students WHERE status = 'nonaktif' AND kelas_id IS NOT NULL;

-- Total siswa TIDAK BOLEH BERUBAH (migrasi ini HANYA ubah kelasId+kelasTerakhirNama, tidak hapus/tambah baris)
SELECT COUNT(*) FROM students;
```

### 5. Verifikasi Nabila Ica spesifik setelah migrasi

```sql
SELECT id, nama, status, kelas_id, kelas_terakhir_nama FROM students WHERE id = 1743;
-- WAJIB hasil: kelas_id = NULL, kelas_terakhir_nama = 'X GAME DEV'
```

### 6. Lapisan pengaman kedua — filter `status` eksplisit di query rekap

Meski migrasi data (poin 1-5) sudah memperbaiki data yang ADA sekarang, **TAMBAHKAN JUGA filter defensif** di level query — supaya kalau ada celah lain yang belum ditemukan (jalur nonaktif yang tidak terdeteksi grep, race condition, dsb), rekap tetap benar TANPA bergantung 100% pada `kelasId` selalu null.

- `apps/api/src/attendance/attendance-report.service.ts` — `reportInternal()` (baris ~137) dan `reportSingleDay()` (baris ~386) — where clause SAAT INI `kelasId: query.kelasId ?? { not: null }` — TAMBAH `status: "aktif"` eksplisit ke where clause KEDUA method ini.
- **VERIFIKASI DAMPAK SEBELUM UBAH**: cek apakah ada consumer LAIN dari method ini yang SENGAJA butuh siswa nonaktif (misal laporan historis akhir tahun ajaran) — kalau ADA, TAMBAH parameter opsional `includeNonaktif?: boolean` (default `false`) ke `ReportQueryDto`, BUKAN hardcode filter tanpa opsi keluar.
- `journal-kelas-wali.controller.ts` (T224c, rekap wali kelas) — REUSE method yang sama, filter ini otomatis ikut tanpa perubahan tambahan di situ.
- **JANGAN ubah** `reportFlexible()` untuk keperluan LAIN di luar filter status ini — scope task ini SEMPIT, hanya tambahan filter pengaman.

## Edge Cases

- **Siswa nonaktif yang MEMANG SENGAJA masih perlu tampil di suatu laporan khusus** (kalau ada kebutuhan itu, TIDAK disebutkan user sejauh ini) — parameter `includeNonaktif` (poin 6) jadi jalan keluar, JANGAN hardcode filter tanpa opsi.
- **Siswa nonaktif dengan `kelasId` SUDAH null tapi `kelasTerakhirNama` juga null** (transisi terjadi sebelum field itu bahkan ada, kemungkinan sangat lama) — migrasi script (poin 3) TIDAK bisa isi `kelasTerakhirNama` untuk kasus ini (tidak ada `kelasId` untuk di-JOIN) — biarkan `kelasTerakhirNama` tetap null untuk kasus ini, TIDAK ERROR, cukup catat di log migrasi sebagai "tidak bisa di-backfill nama kelas".
- **Migrasi dijalankan 2x** — idempotent, run kedua harus 0 baris terdampak (query poin 2 sudah 0), TIDAK ERROR kalau dijalankan ulang.

## Files
- **Buat:** `apps/api/scripts/migrate-nonaktif-kelas-retroaktif.ts` (dry-run + execute mode).
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (`reportInternal()`/`reportSingleDay()` tambah filter `status`), `apps/api/src/attendance/dto/report-query.dto.ts` (`includeNonaktif?: boolean` opsional kalau dibutuhkan).

## Acceptance Criteria
- [ ] Backup production terverifikasi ADA sebelum migrasi dijalankan.
- [ ] Dry-run migrasi menampilkan daftar lengkap siswa terdampak SEBELUM eksekusi nyata.
- [ ] Setelah migrasi — `SELECT COUNT(*) FROM students WHERE status='nonaktif' AND kelas_id IS NOT NULL` = 0.
- [ ] Total row `students` TIDAK berubah sebelum/sesudah migrasi (hanya update field, bukan insert/delete).
- [ ] Nabila Ica (id=1743) spesifik terverifikasi `kelas_id=NULL`, `kelas_terakhir_nama='X GAME DEV'`.
- [ ] Rekap admin DAN rekap wali kelas — Nabila Ica TIDAK LAGI muncul, diverifikasi manual di UI (bukan cuma query DB).
- [ ] Filter `status: "aktif"` (atau `includeNonaktif` opsional) ditambahkan sebagai pengaman kedua di query rekap.
- [ ] Build + type-check hijau, jest baru untuk filter status di `reportInternal()`/`reportSingleDay()`.

## Validasi Claudian
- [ ] Konfirmasi backup dijalankan dan diverifikasi SEBELUM UPDATE apa pun ke production (bukan sesudah).
- [ ] Konfirmasi migrasi pakai per-individual update (bukan `updateMany`), supaya `kelasTerakhirNama` akurat per siswa.
- [ ] Konfirmasi jumlah siswa terdampak DILAPORKAN eksplisit (angka pasti) ke user sebelum eksekusi nyata, bukan diam-diam dijalankan.
- [ ] Konfirmasi filter pengaman kedua (poin 6) tidak menghilangkan akses ke siswa nonaktif untuk use-case yang mungkin memang membutuhkannya (opsi `includeNonaktif` tersedia kalau perlu).

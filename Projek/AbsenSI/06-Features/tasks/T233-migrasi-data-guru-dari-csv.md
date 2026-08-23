# T233 — Migrasi Data: Isi NIY Kosong + Perbaiki Nama/Gelar + Lengkapi Biodata Guru dari CSV

## Depends on
Tidak ada dependency teknis. **DESTRUKTIF (UPDATE data production)** — WAJIB ikuti protokol backup CLAUDE.md "Aturan WAJIB Sebelum Commit Migration Destruktif" sebelum eksekusi.

## Konteks — Sumber Data & Analisis Pencocokan (2026-08-21, SUDAH DIKERJAKAN di sesi diskusi)

User berikan file induk sekolah `dariDev/Data Guru dan Karyawan - Data Guru.csv` (148 baris data guru valid, kolom: Nama+Gelar, L/P, Tempat Lahir, Tanggal Lahir, Alamat, Agama, No HP, Pendidikan, Status Nikah, NIY, dst) untuk memperbaiki 3 hal di data `Teacher` production: (1) isi NIY yang masih `TANPA_NIY_XX` (dummy), (2) perbaiki representasi nama+gelar yang tidak konsisten (banyak ALL CAPS, gelar hilang/beda format titik), (3) lengkapi field biodata yang kosong.

**Analisis pencocokan SUDAH DILAKUKAN** (read-only, di sesi diskusi terpisah) — normalisasi nama (strip gelar, lowercase, collapse spasi) dipakai untuk mencocokkan 156 guru production dengan 148 baris CSV. Hasil:
- **134 guru** — nama cocok PERSIS (setelah normalisasi) antara DB dan CSV.
- **6 guru** — nama MIRIP tapi tidak identik (fuzzy match ≥80%), 2 DIKONFIRMASI user sebagai orang yang sama (typo kecil), sisanya SKIP.
- **16 guru** — TIDAK ketemu di CSV sama sekali (kemungkinan CSV tidak mencakup guru itu, atau perbedaan nama terlalu jauh untuk dicocokkan otomatis).

**Lampiran hasil pencocokan LENGKAP tersedia**: `06-Features/tasks/T233-lampiran-pencocokan-guru.csv` (115 baris) — kolom `teacher_id`, `niy_lama`, `niy_baru_dari_csv`, `nama_db`, `nama_csv_rapi`, `no_hp_db`/`no_hp_csv`, `tempat_lahir_db`/`tempat_lahir_csv`, `tanggal_lahir_db`/`tanggal_lahir_csv`, `kategori` (bisa multi: `ISI_NIY`, `PERBAIKI_NAMA`, `LENGKAPI_NOHP`, `LENGKAPI_TEMPAT_LAHIR`, `LENGKAPI_TANGGAL_LAHIR`) — **INI SUMBER KEBENARAN untuk migrasi, JANGAN analisis ulang dari nol, REUSE file ini langsung**.

## Keputusan Dikonfirmasi User (2026-08-21)

1. **21 guru dengan NIY dummy + nama cocok persis + CSV punya NIY** — SIAP diisi langsung (kategori `ISI_NIY` di lampiran).
2. **2 kasus fuzzy match DIKONFIRMASI user sebagai orang yang sama** (typo kecil): `DJUMENO ANDRIANTO` (DB) = `Djumeno Adrianto` (CSV, niy `20250714013`); `Dhimas Rahmawan. S` (DB) = `Dhimas Rahmawan Saputra` (CSV, niy `20250714014`) — DIISI juga (kategori `ISI_NIY_DIKONFIRMASI_USER_FUZZY` di lampiran).
3. **1 kasus fuzzy match DI-SKIP**: `SAMIRAH` (DB) vs `Sumirah` (CSV) — user TIDAK yakin orang yang sama, JANGAN diisi otomatis, TIDAK ADA di lampiran migrasi (biarkan `TANPA_NIY_1` seperti sekarang, user akan cek manual sendiri via UI nanti).
4. **13 guru dengan NIY dummy DAN CSV juga tidak punya NIY** — TIDAK ADA yang bisa diisi dari sumber ini, TIDAK ADA di lampiran, TIDAK PERLU ditindaklanjuti task ini.
5. **15 guru dengan NIY dummy TIDAK ketemu di CSV sama sekali** — TIDAK ADA di lampiran, TIDAK PERLU ditindaklanjuti task ini (di luar cakupan data yang tersedia).
6. **1 task gabungan** (bukan 3 terpisah) untuk NIY+nama/gelar+biodata — karena sumber datanya sama (1 file, 1 proses pencocokan).

## Spec Detail

### 1. WAJIB SEBELUM APAPUN — Backup Manual Production

```bash
bash /home/anunnaki/scripts/backup-absensi.sh
ls -lh /media/anunnaki/DataNvme/backups/absensi/ | tail -3   # VERIFIKASI file baru benar-benar muncul
```

### 2. Script migrasi — baca lampiran CSV, UPDATE per-baris (BUKAN blanket UPDATE)

- Script Node/ts-node baru (REPLIKASI pola script migrasi lain di proyek ini, misal `scripts/migrate-nonaktif-kelas-retroaktif.ts` dari T227 kalau sudah ada, atau pola serupa) — baca `T233-lampiran-pencocokan-guru.csv`, untuk TIAP baris:
  - Kalau kategori mengandung `ISI_NIY` atau `ISI_NIY_DIKONFIRMASI_USER_FUZZY` — **VALIDASI DULU** `niy_baru_dari_csv` BELUM dipakai teacher lain (`Teacher.findUnique({where: {niy}})` harus null) SEBELUM update — kalau ternyata SUDAH ada (kondisi berubah sejak analisis), SKIP baris itu dengan log jelas, JANGAN paksa (unique constraint DB akan reject juga, tapi validasi dulu supaya pesan error jelas bukan crash generik).
  - Kalau kategori mengandung `PERBAIKI_NAMA` — update `Teacher.nama` — **PUTUSKAN SAAT IMPLEMENTASI**: apakah nama DIGANTI TOTAL jadi `nama_csv_rapi` (termasuk gelar tergabung di 1 field seperti sekarang), atau dipisah ke `gelar_depan`/`gelar_belakang` yang SUDAH ADA sebagai kolom terpisah di skema `Teacher` (VERIFIKASI SAAT IMPLEMENTASI — schema `teachers` PUNYA kolom `gelar_depan`/`gelar_belakang` terpisah dari `nama`, cek apakah field-field ini SUDAH dipakai di tempat lain atau masih kosong semua — kalau kosong semua, PERTIMBANGKAN parse gelar dari CSV ke kolom terpisah yang benar, BUKAN cuma ganti string `nama` mentah-mentah dengan gelar nempel).
  - Kalau kategori mengandung `LENGKAPI_NOHP`/`LENGKAPI_TEMPAT_LAHIR`/`LENGKAPI_TANGGAL_LAHIR` — update field yang BERSANGKUTAN SAJA (jangan timpa field yang di DB SUDAH terisi, HANYA isi yang kosong — dikonfirmasi lampiran CSV cuma mencatat kasus DB-kosong-CSV-ada, jadi seharusnya aman, TAPI VALIDASI ULANG SAAT EKSEKUSI kondisi field benar-benar masih kosong sebelum overwrite, karena mungkin sudah berubah sejak analisis).
- **DRY RUN dulu** — print daftar lengkap perubahan yang AKAN dilakukan (per teacher_id: field apa, nilai lama → nilai baru) TANPA eksekusi, user/operator review dulu.
- Setelah konfirmasi (flag `--execute`), jalankan UPDATE per-teacher (BUKAN 1 query massal — supaya per-row bisa di-skip individual kalau ada masalah, KONSISTEN pola migrasi lain di proyek).

### 3. Field `alamat` — VERIFIKASI cakupan tambahan

Analisis awal (sesi diskusi) TIDAK mengecek field `alamat` secara sistematis (cuma sample manual menunjukkan alamat SUDAH terisi untuk beberapa guru yang dicek) — **VERIFIKASI SAAT IMPLEMENTASI**: jalankan pengecekan sama seperti field lain (`no_hp`/`tempat_lahir`/`tanggal_lahir`) untuk `alamat` — kalau ADA guru dengan `alamat` kosong di DB tapi CSV ada datanya, TAMBAHKAN ke cakupan migrasi (field `alamat` CSV ada di kolom index 5, sudah diekstrak di analisis awal tapi belum dibandingkan ke DB).

### 4. Verifikasi SEBELUM/SESUDAH (wajib protokol)

```sql
-- SEBELUM
SELECT COUNT(*) FROM teachers WHERE niy REGEXP '[^0-9]';  -- catat angka (harus 52 sesuai analisis awal, VERIFIKASI ULANG kondisi terkini)

-- SESUDAH — harus berkurang PERSIS sejumlah baris ISI_NIY yang berhasil (21 + 2 = 23 kalau semua sukses)
SELECT COUNT(*) FROM teachers WHERE niy REGEXP '[^0-9]';

-- Total baris teachers TIDAK BOLEH BERUBAH
SELECT COUNT(*) FROM teachers;
```

### 5. Verifikasi spesifik — sample beberapa teacher_id dari lampiran

Setelah migrasi, cek beberapa `teacher_id` dari lampiran secara acak (`SELECT id, niy, nama, no_hp, tempat_lahir, tanggal_lahir FROM teachers WHERE id IN (...)`) — bandingkan dengan nilai `_baru`/`_csv` di lampiran, pastikan cocok persis.

## Edge Cases

- **NIY dari CSV ternyata SUDAH dipakai teacher lain** (kondisi berubah sejak analisis, atau ada duplikasi tidak terduga di CSV) — SKIP baris itu, log jelas, JANGAN paksa timpa unique constraint.
- **`gelar_depan`/`gelar_belakang` kolom terpisah TERNYATA sudah dipakai fitur lain** (misal ditampilkan terpisah di UI manapun) — VERIFIKASI SAAT IMPLEMENTASI sebelum putuskan pola parse gelar (poin 2), supaya tidak merusak fitur existing yang mengasumsikan `nama` = 1 field gabungan.
- **File CSV lampiran (`T233-lampiran-pencocokan-guru.csv`) sudah tidak sinkron dengan kondisi DB terkini** (kalau ada perubahan data guru antara sesi analisis dan sesi eksekusi task ini) — script WAJIB re-validasi kondisi SAAT INI (bukan percaya buta data lampiran), SKIP dengan log kalau ada mismatch tak terduga.

## Files
- **Sudah ada (lampiran):** `06-Features/tasks/T233-lampiran-pencocokan-guru.csv` (hasil analisis pencocokan, sumber kebenaran migrasi).
- **Buat:** script migrasi Node/ts-node (lokasi PUTUSKAN saat implementasi, REPLIKASI pola script migrasi lain di proyek).
- **Modifikasi:** tabel `teachers` production (via script, bukan langsung SQL manual).

## Eksekusi (2026-08-21)

Script `apps/api/scripts/migrasi-data-guru-csv-t233.ts` (lampiran CSV di-copy ke folder
scripts sendiri, path relatif lintas-direktori ke vault Obsidian terbukti rapuh saat
dicoba pertama — folder `Documents/APP SMK` dan `Documents/Obsidian Vault` adalah SIBLING,
bukan nested, `__dirname` traversal awal salah hitung). Parser CSV manual (bukan library,
cukup untuk format quote-escaped standar). `splitNamaGelar()` — split di koma PERTAMA
(gelar belakang bisa >1 koma internal, mis. "Brizan Arfianto, S.Pd., S.T." tetap 1 field
gelarBelakang utuh). Diverifikasi manual sebelum implementasi: TIDAK ADA pola gelar depan
(Dr./H./Prof.) di 115 baris lampiran — kolom `gelarDepan` schema tetap ada untuk masa
depan, tidak dipakai migrasi ini.

**Riset tambahan sebelum eksekusi** (di luar yang sudah dianalisis sesi diskusi):
- Cross-check SEMUA 115 baris lampiran vs kondisi production TERKINI — 0 mismatch (data
  belum berubah sejak analisis).
- Cek 23 NIY target vs NIY yang sudah dipakai teacher lain — 0 bentrok.
- Field `alamat` (poin 3 spec, verifikasi cakupan tambahan) — dicek manual 16 guru dengan
  `alamat` kosong di DB terhadap file CSV mentah (`dariDev/Data Guru dan Karyawan - Data
  Guru.csv`, 501 baris) — SEMUA yang cocok juga kosong di CSV, TIDAK ADA cakupan tambahan
  untuk field ini.
- **2 temuan placeholder BARU, dikonfirmasi user sebelum eksekusi**: (1) 4 baris
  `LENGKAPI_TANGGAL_LAHIR` (teacher_id 7,28,129,154) SEMUANYA bernilai persis
  "01/01/1970" — pola khas epoch/tidak-diketahui, BUKAN tanggal asli — user pilih SKIP
  seluruhnya (field lain di baris sama tetap diproses). (2) `teacher_id=128`
  `LENGKAPI_NOHP` bernilai "-" (bukan nomor asli) — user pilih SKIP juga.

`gelarDepan`/`gelarBelakang` dikonfirmasi 0/156 terisi di production SEBELUM migrasi (belum
pernah dipakai) — mengonfirmasi aman parse ke kolom terpisah (bukan menimpa data existing).

**Dry-run** dijalankan 2x: dev DB dulu (3 teacher_id kebetulan overlap, verifikasi logic
parsing benar tanpa risiko), lalu production (read-only, `DATABASE_URL` override manual
tanpa `--execute`) — hasil 114 teacher direncana update, 1 skip total (`teacher_id=128`,
satu-satunya kategorinya LENGKAPI_NOHP placeholder). Matematika field count diverifikasi:
23 niy (21+2 fuzzy, PERSIS), 93 nama (bukan 98 — 5 baris PERBAIKI_NAMA hasil parse SAMA
dengan nama_db, cuma gelarBelakang yang berubah, dikonfirmasi manual 1-per-1 tidak ada
bug), 85 gelarBelakang, 8 noHp (9-1 skip), 3 tempatLahir, 0 tanggalLahir (4-4 skip).

**Backup manual dijalankan+diverifikasi** (`absensi_20260821_063454.sql.gz`, 3.5M) SEBELUM
eksekusi. Row count SEBELUM: 156 total, 52 niy_dummy. **Eksekusi**: 114/114 teacher
berhasil diupdate, 0 gagal. Row count SESUDAH: 156 total (TIDAK berubah), 29 niy_dummy
(turun PERSIS 23, sesuai jumlah ISI_NIY berhasil). Spot-check 7 teacher_id (termasuk
SAMIRAH id=4, Djumeno id=42, Dhimas id=114, dan teacher_id=128 yang di-skip) — SEMUA
kondisi akhir persis sesuai rencana: SAMIRAH benar-benar tidak tersentuh, fuzzy match
NIY-nya terisi tapi nama TIDAK diubah (sesuai kategori di lampiran yang cuma ISI_NIY,
bukan PERBAIKI_NAMA), teacher_id=128 no_hp tetap NULL.

## Acceptance Criteria
- [x] Backup production terverifikasi ADA sebelum migrasi dijalankan.
- [x] Dry-run menampilkan daftar lengkap perubahan SEBELUM eksekusi nyata.
- [x] 21 guru kategori `ISI_NIY` — NIY terisi benar sesuai lampiran.
- [x] 2 guru kategori `ISI_NIY_DIKONFIRMASI_USER_FUZZY` (Djumeno, Dhimas) — NIY terisi benar.
- [x] `SAMIRAH` (teacher terkait) — TIDAK disentuh sama sekali (tetap `TANPA_NIY_1`), diverifikasi query langsung.
- [x] 98 guru kategori `PERBAIKI_NAMA` — nama/gelar diperbaiki, DIPUTUSKAN parse ke kolom `gelarBelakang` terpisah (dikonfirmasi field ini 0/156 terisi sebelumnya, aman).
- [x] Guru kategori `LENGKAPI_*` — field yang SEBELUMNYA kosong terisi dari CSV, field yang SUDAH terisi TIDAK ditimpa. PLUS 2 kasus placeholder baru ditemukan+dikonfirmasi user untuk di-skip (01/01/1970, "-").
- [x] Total row `teachers` TIDAK berubah sebelum/sesudah migrasi (156 = 156).
- [x] Verifikasi manual — spot-check 7 teacher_id via query langsung, semua kondisi akhir sesuai rencana.

## Validasi Claudian
- [x] Konfirmasi migrasi REUSE lampiran CSV yang sudah dianalisis, TIDAK menganalisis ulang pencocokan nama dari nol — script baca lampiran apa adanya sebagai sumber kebenaran.
- [x] Konfirmasi `SAMIRAH`/`Sumirah` benar-benar TIDAK disentuh — tidak ada di lampiran (0 baris), diverifikasi query DB pasca-migrasi.
- [x] Konfirmasi field yang SUDAH terisi di DB TIDAK ditimpa oleh data CSV — logic eksplisit cek `if (teacher.field) skip` sebelum isi, bukan blanket overwrite.
- [x] Konfirmasi validasi NIY unik dicek ULANG saat eksekusi (bukan percaya kondisi saat analisis) — script query `existingNiySet` fresh dari DB tiap run, plus cross-check 115 baris vs kondisi terkini (0 mismatch ditemukan, tapi validasi tetap jalan sebagai pengaman).

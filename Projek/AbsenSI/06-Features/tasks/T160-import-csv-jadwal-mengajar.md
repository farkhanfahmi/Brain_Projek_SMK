# T160 — API+Web: Import CSV Jadwal Mengajar (Guru/Mapel/Kelas/Jam Ke/Minggu)

## Depends on
**WAJIB T158 (Model Opsi Jam Pelajaran) DAN T159 (Toggle Mode Blok/Normal Per Kelas) SELESAI DULU** — import ini menghasilkan `Schedule` yang butuh resolusi "Jam Ke → jam sungguhan" (T158) dan butuh tahu apakah kelas tujuan mode Blok/Normal untuk validasi kolom Minggu (T159). JANGAN kerjakan sebelum keduanya selesai.

## Objective
Admin bisa **import jadwal mengajar sekaligus banyak baris** lewat file CSV, dengan format PERSIS seperti contoh yang diberikan user (kolom: No, Minggu, Hari, Nama Guru, Mata Pelajaran, Kelas, Jam Ke Awal, Jam Ke Akhir, Catatan) — mengikuti pola infrastruktur import CSV yang SUDAH ADA dan TERBUKTI di sistem (siswa/guru/kartu).

## Context — Format CSV (Contoh Asli dari User, 2026-08-11)

```
No | Minggu | Hari | Nama Guru | Mata Pelajaran | Kelas | Jam Ke Awal | Jam Ke Akhir | Catatan
1  | Minggu A | Senin | Ir. Mohammad Bastomi | Konsentrasi Keahlian TKR 3 | XI TKR 6 | 2 | 7 |
```

Referensi lengkap ada di `dariDev/Sistem Jurnal Baru 2026-2027.xlsx` (sheet `Penjadwalan Perguru`, `List`, `Alokasi Waktu`) — SUDAH dianalisis, poin-poin penting:
- **1 baris CSV = 1 sesi mengajar** (rentang Jam Ke Awal-Akhir, KONSISTEN konsep `TeachingSession`/`Schedule` yang SUDAH ADA — 1 jadwal bisa mencakup beberapa jam pelajaran berturut, BUKAN 1 baris per jam individual).
- **Kolom "Minggu"** berisi teks `"Minggu A"` / `"Minggu B"` (ATAU kosong/tidak ada untuk kelas mode Normal) — dikonfirmasi user: **kalau kolom ini terisi A/B untuk suatu kelas, IMPORT OTOMATIS MENGUBAH `Kelas.modeJadwal` kelas itu jadi `blok`** (CSV jadi sumber kebenaran, admin TIDAK PERLU setting manual dulu sebelum import).
- **"Jam Ke Awal"/"Jam Ke Akhir"** — ANGKA (bukan jam:menit) — DIRESOLVE ke waktu sungguhan lewat `JamPelajaranOption` yang SEDANG AKTIF (T158, Opsi A) SAAT `TeachingSession` di-generate — TIDAK diresolve saat import (import SIMPAN angka jam-ke apa adanya ke `Schedule`, sesuai keputusan arsitektur T158 Opsi A).

## Spec Detail

### 1. Backend — modul import baru, REUSE pola `ImportService` existing

- Tambah `apps/api/src/import/import.service.ts` method BARU `importSchedules(buffer: Buffer): Promise<ImportReport>` — REUSE `parseCsv<T>()` yang SUDAH ADA (jangan tulis parser baru), REUSE pola validasi-per-baris+error-collection yang SUDAH TERBUKTI di `importStudents`/`importTeachers`/`importCards`.
- **Preload lookup list SEKALI di luar loop** (pola existing, N+1-safe):
  - Semua `Teacher` (untuk matching by `niy` ATAU `nama` — lihat poin 3 soal keputusan matching).
  - Semua `Mapel` (matching by `nama`).
  - Semua `Kelas` (matching by `nama`, INCLUDE relasi `jurusan`/`kampus` untuk disambiguasi kalau perlu — lihat poin 3).
- **Validasi per baris**:
  1. Kolom wajib terisi: `Hari`, `Nama Guru`, `Mata Pelajaran`, `Kelas`, `Jam Ke Awal`, `Jam Ke Akhir`. Kolom `Minggu`/`Catatan` OPSIONAL.
  2. `Hari` — mapping teks Indonesia (`"Senin"`, `"Selasa"`, dst) ke angka `DAYOFWEEK` MySQL (1=Minggu..7=Sabtu) — REUSE mapping yang SUDAH ADA di codebase (cek `Schedule.hari` existing usage untuk konsistensi konversi, JANGAN buat mapping baru yang mungkin beda).
  3. `Nama Guru` — lookup ke `Teacher` preloaded list. **KEPUTUSAN MATCHING** (lihat poin 3 di bawah, PENTING).
  4. `Mata Pelajaran` — lookup ke `Mapel` by `nama` EXACT MATCH (case-insensitive, konsisten pola `importStudents` untuk field non-unique) — GAGAL kalau tidak ketemu, ATAU (PERTIMBANGKAN) auto-create `Mapel` baru kalau belum ada — PUTUSKAN saat implementasi, REKOMENDASI: GAGAL dengan pesan jelas ("Mapel 'X' tidak ditemukan, buat dulu di menu Mata Pelajaran") — LEBIH AMAN daripada auto-create yang bisa menghasilkan Mapel duplikat/typo tanpa sadar.
  5. `Kelas` — lookup ke `Kelas` by `nama` EXACT MATCH. Kalau AMBIGU (>1 kelas dengan nama sama, misal beda kampus) — GAGAL dengan pesan jelas menyebutkan ambiguitas (JANGAN asal pilih salah satu).
  6. `Jam Ke Awal`/`Jam Ke Akhir` — angka, `Awal <= Akhir`, keduanya harus ADA sebagai `jamKe` valid di `JamPelajaranSlot` untuk opsi jam pelajaran yang SEDANG AKTIF DAN hari yang sesuai (VALIDASI SILANG dengan T158 — kalau opsi jam pelajaran aktif tidak punya "Jam Ke 7" untuk hari Senin, baris CSV yang minta Jam Ke Akhir=7 untuk Senin harus GAGAL dengan pesan jelas).
  7. **Kolom `Minggu`** — kalau terisi (`"Minggu A"`/`"Minggu B"`, parsing fleksibel case-insensitive dan toleran spasi) → set `Schedule.minggu = A/B` UNTUK BARIS INI, DAN (SEKALI per kelas per proses import, bukan per baris) **update `Kelas.modeJadwal = blok`** untuk kelas yang dimaksud KALAU belum blok (sesuai keputusan user). Kalau kolom KOSONG → `Schedule.minggu = null`, kelas TIDAK diubah mode-nya (biarkan apa adanya, TIDAK dipaksa Normal kalau sebelumnya sudah Blok karena baris lain).
  8. **Cek bentrok jadwal** — REUSE validasi `ensureNoBentrok` yang SUDAH ADA di `SchedulesService` (jangan tulis ulang logic overlap-check, panggil method yang sama/serupa) — baris CSV yang bentrok dengan jadwal existing ATAU bentrok ANTAR BARIS DALAM FILE YANG SAMA harus GAGAL dengan pesan jelas menyebut baris mana yang bentrok.
- Semua kegagalan per-baris masuk ke `errors: ImportRowError[]` (format SAMA seperti import lain), baris yang GAGAL TIDAK menghentikan proses baris lain (proses SEMUA baris, kumpulkan semua error, laporkan di akhir — pola existing).
- Endpoint baru: `POST /import/schedules` (`apps/api/src/import/import.controller.ts`) — `@Roles(UserRole.super_admin)`, `FileInterceptor` SAMA seperti endpoint import lain, `@LogActivity` ringkasan (pola existing).

### 2. Frontend — tambah opsi baru di halaman Import

- `apps/web/src/app/(admin)/import/import-view.tsx` (halaman Import existing, SUDAH punya UI untuk siswa/guru/kartu) — TAMBAH TAB/OPSI BARU "Jadwal Mengajar" — REUSE komponen upload+preview-error yang SUDAH ADA, cuma tambah 1 target endpoint baru.
- Sertakan LINK/TOMBOL "Unduh Template CSV" (contoh file kosong dengan header kolom yang benar) — KONSISTEN pola bantuan import lain kalau sudah ada di halaman ini (VERIFIKASI saat implementasi apakah fitur unduh template sudah ada pola/komponennya, reuse kalau ada).

### 3. KEPUTUSAN — matching "Nama Guru" dari CSV: `niy` vs `nama`

Riset mengonfirmasi `Teacher.niy` UNIQUE (kandidat matching AMAN), sementara `Teacher.nama` TIDAK UNIQUE (berisiko ambigu kalau ada 2 guru nama sama/mirip). **TAPI contoh CSV user HANYA punya kolom "Nama Guru" (teks nama), TIDAK ADA kolom NIY.**

**KEPUTUSAN**: matching PRIMER pakai `nama` EXACT MATCH (case-insensitive, trim whitespace) KARENA itu satu-satunya data yang tersedia di format CSV yang diberikan — TAPI:
- Kalau ditemukan >1 guru dengan `nama` PERSIS SAMA di database — GAGAL dengan pesan jelas ("Nama guru 'X' ambigu, ditemukan N guru dengan nama sama — tidak bisa diimport otomatis, edit manual atau tambah NIY di file") — JANGAN asal pilih salah satu.
- **PERTIMBANGKAN** (opsional, tingkatkan robustness): kalau format CSV BISA diperluas untuk sertakan kolom NIY opsional (kolom TAMBAHAN, bukan pengganti "Nama Guru") — pakai NIY sebagai matching UTAMA kalau kolom itu ADA DAN TERISI untuk baris itu, fallback ke nama kalau NIY kosong — INI PENINGKATAN OPSIONAL, TIDAK WAJIB untuk v1 karena format asli user TIDAK menyebut NIY sama sekali, JANGAN memaksa user mengubah format CSV mereka tanpa diminta.

## Edge Cases
- CSV dengan baris "No" yang meloncat/tidak berurutan (dari contoh Excel, kolom "No" kadang berupa desimal `2.0`/`3.0` bukan integer bersih) — kolom "No" HANYA untuk referensi manusia, TIDAK dipakai sebagai identifier sistem — JANGAN validasi ketat terhadap format/urutan kolom ini, abaikan nilainya kecuali untuk ditampilkan di pesan error ("Baris dengan No=X gagal karena...").
- File CSV yang sebagian barisnya untuk kelas mode Normal (kolom Minggu kosong) dan sebagian untuk kelas mode Blok (kolom Minggu terisi) DALAM FILE YANG SAMA — HARUS BISA diproses bersamaan tanpa masalah (per-baris independen, sesuai poin 1.7 di atas).
- Guru yang jadwalnya (dari CSV) BENTROK dengan jadwal MANUAL yang sudah diinput admin sebelumnya (bukan bentrok sesama baris CSV) — GAGAL dengan pesan jelas (poin 1.8), JANGAN timpa/override jadwal existing secara diam-diam.

## Files
- **Modifikasi:** `apps/api/src/import/import.service.ts` (method baru `importSchedules`), `apps/api/src/import/import.controller.ts` (endpoint baru), `apps/web/src/app/(admin)/import/import-view.tsx` (tab/opsi baru).
- **Jangan sentuh:** `importStudents`/`importTeachers`/`importCards` (existing, TIDAK diubah — task ini MENAMBAH method baru, REUSE helper `parseCsv` saja).

## Acceptance Criteria
- [ ] Admin bisa upload CSV format PERSIS seperti contoh user (No, Minggu, Hari, Nama Guru, Mata Pelajaran, Kelas, Jam Ke Awal, Jam Ke Akhir, Catatan) lewat halaman Import.
- [ ] Baris dengan kolom Minggu terisi ("Minggu A"/"Minggu B") → `Schedule.minggu` terisi BENAR, DAN `Kelas.modeJadwal` kelas terkait otomatis jadi `blok` kalau belum.
- [ ] Baris dengan kolom Minggu kosong → `Schedule.minggu = null`, mode kelas TIDAK dipaksa berubah.
- [ ] Baris dengan Guru/Mapel/Kelas yang TIDAK ditemukan → gagal dengan pesan jelas per baris, TIDAK menghentikan proses baris lain.
- [ ] Baris dengan Jam Ke yang TIDAK ADA di opsi jam pelajaran aktif untuk hari itu → gagal dengan pesan jelas.
- [ ] Baris yang bentrok jadwal (sesama file ATAU vs existing) → gagal dengan pesan jelas menyebut baris/jadwal yang bentrok.
- [ ] Ringkasan hasil import (total/berhasil/gagal + detail error per baris) ditampilkan ke admin setelah proses selesai.
- [ ] `@LogActivity` mencatat ringkasan operasi import.
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [ ] **JANGAN** kerjakan sebelum T158 dan T159 selesai — dependency ini KRITIKAL, import ini butuh keduanya untuk validasi Jam Ke dan mode kelas.
- [ ] **REUSE** `parseCsv<T>()`, pola validasi-per-baris, dan `ensureNoBentrok` yang SUDAH ADA — JANGAN tulis ulang logic yang sudah teruji.
- [ ] Matching Guru pakai `nama` (bukan NIY) KARENA format CSV asli user tidak menyebut NIY — JANGAN memaksa user mengubah format tanpa diminta, tapi TETAP tangani kasus ambigu (>1 guru nama sama) dengan gagal jelas, bukan asal pilih.
- [ ] Auto-set `Kelas.modeJadwal = blok` HANYA terjadi kalau kolom Minggu di CSV terisi untuk kelas itu — JANGAN mengubah mode kelas untuk baris yang kolom Mingu-nya kosong.

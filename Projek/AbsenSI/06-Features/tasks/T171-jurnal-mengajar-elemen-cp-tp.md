# T171 — API+Web: Redesign Jurnal Mengajar — Pisah Field Elemen/CP/TP

## Depends on
**WAJIB setelah T168** (shell mobile-app) **dan T169** (karena route `/guru/jurnal` akan dialihfungsikan, perlu jadwal sudah pindah ke `/guru/jadwal` dulu supaya tidak tabrakan makna).

## Objective
1. `JournalEntry` — pecah field `tujuanPembelajaran` (satu field bebas) jadi 3 field terpisah sesuai istilah Kurikulum Merdeka: **Elemen**, **Capaian Pembelajaran (CP)**, **Tujuan Pembelajaran (TP)** — plus `materi` dan `catatan` yang sudah ada TETAP dipertahankan.
2. Alihfungsikan route `/guru/jurnal` jadi tab "Jurnal Mengajar" di bottom-nav (bukan lagi "jadwal hari ini" yang sudah pindah ke `/guru/jadwal` di T169) — isinya daftar sesi yang SUDAH/SEDANG diajar dengan status jurnal (sudah diisi/belum), klik → form jurnal lengkap.

## Context — Temuan Riset (2026-08-13)

- `JournalEntry` (`schema.prisma:484-503`) SAAT INI: `sessionId` (unique, 1:1 `TeachingSession`), `materi`, `tujuanPembelajaran`, `tugasPenilaian`, `catatan` — SEMUA `String? @db.Text` bebas.
- Form jurnal SUDAH ADA: `apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/journal-form.tsx`, dipakai dalam `sesi-detail-view.tsx` bersama `attendance-table.tsx`.
- Keputusan user: field `tujuanPembelajaran` DIPECAH jadi field terpisah untuk Elemen dan CP (istilah resmi Kurikulum Merdeka: Elemen → Capaian Pembelajaran → Tujuan Pembelajaran, 3 tingkat berbeda) — bukan digabung 1 kolom teks bebas.
- Field `tugasPenilaian` — **KEMUNGKINAN BENTROK KONSEP** dengan modul Nilai baru (T172). VERIFIKASI saat implementasi: apakah field ini (teks bebas, deskripsi tugas) tetap relevan sebagai catatan kualitatif terpisah dari `GradeEntry` (nilai numerik T172) — KEMUNGKINAN BESAR tetap relevan (field ini mendeskripsikan APA tugasnya, bukan NILAI siswa), TAPI kalau ragu saat implementasi, TANYAKAN ke user daripada asumsi sepihak.

## Spec Detail

### 1. Database — migration `JournalEntry`

- Tambah kolom baru: `elemen String? @db.Text`, `capaianPembelajaran String? @map("capaian_pembelajaran") @db.Text`.
- Kolom `tujuanPembelajaran` TETAP ADA (jangan drop) — sekarang murni untuk TP saja (makna dipersempit, bukan field baru).
- Data JournalEntry LAMA yang sudah ada (`elemen`/`capaianPembelajaran` otomatis NULL untuk baris lama) — TIDAK PERLU backfill/migrasi data (field baru, wajar kosong untuk histori lama, TIDAK mengganggu tampilan riwayat kalau UI menangani null dengan baik).

### 2. Backend — DTO & service

- Cari `journal.service.ts`/DTO create-update JournalEntry — tambah `elemen`, `capaianPembelajaran` sebagai field opsional di payload PATCH/PUT yang sudah ada (endpoint TIDAK berubah, cuma payload-nya diperkaya).

### 3. Frontend — form jurnal diperkaya

- `journal-form.tsx` — tambah 2 field baru (Elemen, CP) di ATAS field TP yang sudah ada (urutan Elemen → CP → TP → Materi → Catatan, sesuai hierarki Kurikulum Merdeka + urutan yang disebut user "Elemen, CP, TP, Materi yang diberikan, catatan").
- Textarea sederhana untuk masing-masing (konsisten style field lain di form ini), TIDAK PERLU rich text editor (di luar scope, field lama juga plain text).

### 4. Frontend — alihfungsi route `/guru/jurnal`

- `guru/jurnal/page.tsx` + `jurnal-view.tsx` — UBAH ISI (bukan hapus file, REUSE path): dari "daftar jadwal hari ini" (sudah pindah murni ke `/guru/jadwal` di T169) jadi **"daftar sesi yang perlu/sudah diisi jurnalnya"** — tampilkan sesi (hari ini + mungkin beberapa hari terakhir yang jurnalnya belum diisi, putuskan rentang wajar saat implementasi, misal 7 hari terakhir) dengan badge status "Jurnal belum diisi" / "Jurnal sudah diisi". Klik kartu → masuk ke `/guru/sesi/[sessionId]` (route SAMA, TIDAK berubah) yang berisi `journal-form.tsx` (diperkaya) + `attendance-table.tsx` (TIDAK disentuh langkah ini, presensi tetap di sana untuk EDIT — beda dari T170 yang read-only viewer).
- `sesi-card.tsx` — evaluasi apakah reusable lagi di sini (kemungkinan besar YA dengan variasi kecil, badge status jurnal ditambahkan) atau perlu varian baru — putuskan saat implementasi, HINDARI duplikasi kalau bisa cukup reuse+props tambahan.

## Edge Cases
- Sesi lampau yang jurnalnya BELUM diisi sama sekali — tetap bisa diisi kapan saja (TIDAK ada gate "hanya hari ini", beda dari presensi yang lebih strict) — KECUALI ada instruksi lain dari user saat implementasi, ini asumsi wajar karena user tidak menyebut jurnal read-only setelah ganti hari (beda eksplisit dari Presensi di poin diskusi awal).
- Field baru (Elemen/CP) kosong untuk jurnal lama — tampil placeholder "-" atau field kosong biasa di form (biarkan guru isi belakangan kalau mau), JANGAN error.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`JournalEntry` +2 kolom), journal service/DTO, `apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/journal-form.tsx`, `apps/web/src/app/(guru)/guru/jurnal/jurnal-view.tsx` (alihfungsi isi).
- **Buat:** migration Prisma baru.
- **Jangan sentuh:** `attendance-table.tsx` (edit presensi di sesi aktif TETAP jalur ini, tidak diubah task ini), route `/guru/sesi/[sessionId]` (path sama).

## Acceptance Criteria
- [ ] Form jurnal punya 5 field: Elemen, CP, TP, Materi, Catatan — tersimpan dan termuat kembali benar.
- [ ] `/guru/jurnal` (bottom-nav "Jurnal Mengajar") menampilkan daftar sesi dengan status jurnal (sudah/belum diisi), BUKAN lagi daftar jadwal murni (itu sudah di `/guru/jadwal`).
- [ ] Jurnal lama (field baru NULL) tampil rapi tanpa error di form edit.
- [ ] Build + type-check hijau, jest existing tetap pass.

## Validasi Claudian
- [ ] Konfirmasi field `tugasPenilaian` (existing) TIDAK dihapus/diubah maknanya — tetap ada berdampingan dengan modul Nilai baru (T172), BUKAN digantikan.
- [ ] Konfirmasi route `/guru/sesi/[sessionId]` tidak berubah path/struktur, hanya isi `journal-form.tsx` yang diperkaya.

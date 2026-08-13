# T172 — API+Web: Modul Nilai — Penilaian Siswa per Pertemuan (Bisa Gabung Beberapa Pertemuan)

## Depends on
**WAJIB setelah T168** (shell mobile-app). **REKOMENDASI setelah T171** (karena "pilih pertemuan dari jurnal yang telah dibuat" butuh daftar `JournalEntry`/`TeachingSession` yang statusnya sudah jelas — tidak WAJIB blocking, tapi lebih natural dikerjakan setelahnya).

## Objective
Modul BARU sepenuhnya (tidak ada model existing sama sekali) — guru bisa membuat **Penilaian** (judul + pilih 1 atau lebih pertemuan/sesi sebagai konteks cakupan materi) lalu isi **1 nilai (0-100) per siswa** untuk penilaian itu. Bisa diedit kapan saja (tidak seperti Presensi yang read-only setelah lewat hari).

## Context — Keputusan Diskusi (2026-08-13)

- **Tidak ada model Grade/Nilai/Penilaian sama sekali di schema saat ini** (dikonfirmasi grep menyeluruh) — murni fitur baru.
- **Bentuk relasi Penilaian↔Pertemuan**: MANY-TO-MANY. 1 Penilaian (misal "UH Bab 3") BISA mencakup beberapa `TeachingSession` sekaligus (pertemuan minggu 1+2 gabung) — pertemuan yang dipilih HANYA untuk konteks/cakupan materi (opsional ditampilkan sebagai info), BUKAN untuk memecah nilai per pertemuan.
- **Bentuk nilai per siswa**: **1 nilai per siswa untuk SELURUH Penilaian** (bukan matrix siswa×pertemuan) — sesuai deskripsi asli user: form berisi judul + pilih pertemuan + daftar semua nama siswa dengan 1 field nilai di sampingnya + tombol Save.
- **Tipe nilai**: Angka 0-100 (decimal, bukan bebas teks).
- **Daftar siswa yang muncul di form**: berdasarkan KELAS dari pertemuan yang dipilih (kalau semua pertemuan yang dipilih sama kelasnya — kasus umum). Guru TIDAK "pilih siswa manual" — daftar otomatis dari kelas terkait.

## Spec Detail

### 1. Database — model baru

```prisma
model GradeAssessment {
  id          Int      @id @default(autoincrement())
  teacherId   Int      @map("teacher_id")
  kelasId     Int      @map("kelas_id")
  mapelId     Int      @map("mapel_id")
  judul       String
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  teacher   Teacher                    @relation(fields: [teacherId], references: [id])
  kelas     Kelas                      @relation(fields: [kelasId], references: [id])
  mapel     Mapel                      @relation(fields: [mapelId], references: [id])
  sessions  GradeAssessmentSession[]
  entries   GradeEntry[]

  @@index([teacherId])
  @@index([kelasId])
  @@map("grade_assessments")
}

model GradeAssessmentSession {
  id            Int @id @default(autoincrement())
  assessmentId  Int @map("assessment_id")
  sessionId     Int @map("session_id")

  assessment GradeAssessment @relation(fields: [assessmentId], references: [id], onDelete: Cascade)
  session    TeachingSession @relation(fields: [sessionId], references: [id])

  @@unique([assessmentId, sessionId])
  @@map("grade_assessment_sessions")
}

model GradeEntry {
  id            Int      @id @default(autoincrement())
  assessmentId  Int      @map("assessment_id")
  studentId     Int      @map("student_id")
  nilai         Decimal? @db.Decimal(5, 2)
  updatedAt     DateTime @updatedAt @map("updated_at")

  assessment GradeAssessment @relation(fields: [assessmentId], references: [id], onDelete: Cascade)
  student    Student         @relation(fields: [studentId], references: [id])

  @@unique([assessmentId, studentId])
  @@map("grade_entries")
}
```

**CATATAN saat implementasi**: field/nama di atas adalah USULAN, VERIFIKASI dan sesuaikan dengan konvensi penamaan yang SUDAH DIPAKAI di model lain sekitarnya (`JournalEntry`, `ClassAttendanceMark`) sebelum finalize — terutama relasi `mapelId` (apakah `GradeAssessment` perlu simpan `mapelId` eksplisit, atau cukup derive dari `sessions[0].mapelId` karena semua pertemuan yang dipilih SEHARUSNYA 1 mapel yang sama — putuskan saat implementasi, REKOMENDASI simpan eksplisit di `GradeAssessment` untuk query lebih mudah tanpa join berlapis).

- `Decimal(5,2)` untuk nilai 0-100 dengan 2 desimal (mis. 87.50) — VERIFIKASI apakah sekolah perlu presisi desimal atau cukup integer 0-100 bulat, KLARIFIKASI ke user kalau ragu saat implementasi (default aman: izinkan desimal, tidak memaksa bulat).
- `nilai` NULLABLE di `GradeEntry` — siswa yang belum dinilai TETAP punya baris (auto-created saat assessment dibuat, dari daftar siswa kelas) dengan `nilai: null`, BUKAN baris hilang — supaya form selalu tampilkan SEMUA siswa kelas meski belum diisi.

### 2. Backend — modul baru `apps/api/src/grades/` (nama final diputuskan saat implementasi, konsisten pola modul lain)

- `POST /grades/assessments` — body: `kelasId, mapelId, judul, sessionIds[]` (minimal 1). Service: buat `GradeAssessment` + `GradeAssessmentSession` (loop `sessionIds`) + auto-create `GradeEntry` untuk SEMUA siswa aktif di `kelasId` itu (nilai null) — dalam 1 `$transaction`.
- `GET /grades/assessments?kelasId=&mapelId=` — role `guru`, WAJIB filter `teacherId` dari JWT (guru cuma lihat penilaian buatan sendiri) — list assessment + ringkasan (jumlah siswa sudah/belum dinilai).
- `GET /grades/assessments/:id` — detail + daftar `GradeEntry` (join nama siswa) untuk form isi nilai.
- `PATCH /grades/assessments/:id/entries` — body: `entries: [{studentId, nilai}]` (batch update, bisa isi sebagian/semua sekaligus) — **BISA DIPANGGIL KAPAN SAJA, TIDAK ADA GATE WAKTU** (beda dari presensi), validasi `nilai` dalam rentang 0-100 kalau tidak null.
- `GET /teaching-sessions/mine-for-grading?kelasId=&mapelId=` (atau reuse endpoint T170 `my-classes`/list sesi yang sudah ada) — untuk dropdown "pilih pertemuan" di form create assessment, hanya sesi milik guru itu sendiri.
- Guard role SEMUA endpoint: `@Roles(UserRole.guru)` + filter `teacherId` dari JWT di SETIAP query (bukan trust body/query param client untuk ownership).
- `@LogActivity` di endpoint create assessment DAN update entries (konsisten aturan proyek, CEK jangan lupa pasang — riwayat insiden 14/22 controller lupa decorator ini).

### 3. Frontend — halaman `/guru/nilai`

- File baru `apps/web/src/app/(guru)/guru/nilai/page.tsx` + `nilai-view.tsx`.
- List Penilaian existing (judul, kelas, mapel, ringkasan sudah/belum dinilai) + tombol "Buat Penilaian Baru".
- Form Buat Penilaian: pilih kelas (dari kelas yang diajar guru — reuse sumber data T170 kalau ada), pilih mapel (kalau guru ajar >1 mapel di kelas itu), input judul, **pilih pertemuan** (multi-select checklist dari sesi yang sudah/sedang berjalan untuk kelas+mapel itu, tampilkan tanggal+materi ringkas biar guru gampang kenali) — submit → create assessment, auto-redirect ke halaman isi nilai.
- Halaman isi nilai (`/guru/nilai/[assessmentId]`): tabel nama siswa + input angka 0-100 di kolom sebelahnya + tombol Save (batch, kirim semua entries sekaligus atau per-baris — putuskan UX saat implementasi, REKOMENDASI 1 tombol Save di bawah untuk semua perubahan sekaligus, lebih hemat request dan konsisten "form" bukan "auto-save per field").
- Validasi FE: nilai harus 0-100, tolak input di luar rentang sebelum submit (selain validasi BE).

## Edge Cases
- Siswa baru pindah masuk kelas SETELAH assessment dibuat — TIDAK otomatis muncul di assessment lama (snapshot siswa saat assessment dibuat) — TIDAK PERLU auto-sync, ini wajar untuk penilaian historis (assessment adalah snapshot waktu itu).
- Siswa keluar/pindah kelas SETELAH dinilai — `GradeEntry` TETAP ada (histori nilai tidak hilang), TIDAK PERLU cascade delete kalau siswa masih ada di DB (hanya pindah kelas) — TAPI kalau `Student` benar-benar dihapus, `onDelete` behavior ikut default relasi lain di schema (VERIFIKASI konsistensi).
- Assessment dengan sessionIds dari 2 KELAS BERBEDA (guru salah pilih) — TOLAK di validasi backend (`sessionIds` yang dipilih harus semua merujuk `kelasId` yang sama dengan yang dipilih di form) — pesan error jelas kalau ditemukan campuran.

## Files
- **Buat:** migration Prisma baru (3 model), `apps/api/src/grades/` (module baru: controller, service, DTO), `apps/web/src/app/(guru)/guru/nilai/page.tsx` + `nilai-view.tsx` + `[assessmentId]/page.tsx`.
- **Jangan sentuh:** `JournalEntry.tugasPenilaian` (field teks bebas lama, TETAP ada terpisah, bukan pengganti modul ini).

## Acceptance Criteria
- [ ] Guru bisa buat Penilaian baru: pilih kelas, mapel, judul, 1+ pertemuan.
- [ ] Setelah dibuat, SEMUA siswa kelas itu otomatis muncul di daftar dengan nilai kosong.
- [ ] Guru bisa isi/ubah nilai 0-100 per siswa, simpan, dan BUKA LAGI KAPAN SAJA untuk edit (tidak ada gate waktu).
- [ ] Guru A tidak bisa lihat/edit Penilaian buatan guru B (verified filter teacherId dari JWT).
- [ ] Assessment dengan sessions dari kelas berbeda ditolak dengan pesan jelas.
- [ ] `@LogActivity` terpasang di create assessment + update entries — verified tercatat.
- [ ] Build + type-check hijau, jest baru untuk modul ini pass.

## Validasi Claudian
- [ ] Konfirmasi SEMUA endpoint filter `teacherId` dari JWT, bukan trust client — cross-tenant leak check (guru lain tidak bisa akses assessment orang lain via tebak ID).
- [ ] Konfirmasi `@LogActivity` terpasang, bukan asumsi otomatis tercatat (cek [[feedback_wajib_log_activity]]).
- [ ] Konfirmasi validasi rentang nilai 0-100 di BACKEND (bukan cuma FE) — FE bisa dibypass.

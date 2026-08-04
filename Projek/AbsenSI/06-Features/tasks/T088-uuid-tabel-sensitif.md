# T088 — Schema+API: Migrasi PK Sequential Int → UUID untuk Tabel Sensitif (Student, Teacher, Card)

## ⚠️ Status: TIDAK MENDESAK — jangan eksekusi tanpa instruksi eksplisit user
Dibuat 2026-07-26 atas permintaan user setelah audit keamanan menyeluruh (lihat "Konteks & Hasil Audit" di bawah) **membuktikan tidak ada celah IDOR nyata di sistem ini** — semua endpoint yang seharusnya dibatasi kepemilikan/kampus sudah benar dijaga di level service (bukan mengandalkan ID sulit ditebak). User tetap ingin migrasi ini disiapkan sebagai task untuk dieksekusi **di kemudian hari kalau memang dibutuhkan** (bukan alasan keamanan murni — kemungkinan preferensi konsistensi dengan gaya database lama). **Jangan jadikan task ini prioritas** dibanding task lain yang punya manfaat fungsional/keamanan nyata.

## Depends on
Tidak ada — tapi WAJIB dikerjakan di jam sepi/maintenance window (downtime terjadwal), TIDAK bisa dilakukan tanpa jeda ke aplikasi yang sedang dipakai live.

## Context & Hasil Audit (2026-07-26)
- Kekhawatiran awal user: ID sekuensial (`1,2,3,...`) bisa ditebak orang luar (IDOR/enumerasi), berbeda dari database lama (Laravel) yang pakai UUID.
- Audit menyeluruh dilakukan ke SEMUA endpoint `:id` di semua controller (students, teachers, cards, permits, users, kelas, jurusan, attendance, teaching-sessions, teacher-permits). Hasil: SEMUA endpoint yang seharusnya scoped (guru ke datanya sendiri — `submitTugas` cek `permit.teacherId !== teacherId`; guru_piket ke kampusnya — `ensurePermitInKampus`/`ensureYesterdayRecordInKampus`; wali kelas ke kelasnya) **sudah benar ditegakkan di level service**. Endpoint tanpa scoping lainnya (super_admin/card_admin akses lintas kampus, admin_jurnal lintas kampus untuk domain jurnal-mengajar) adalah **desain sengaja secara konsisten di seluruh sistem**, bukan celah yang terlewat.
- **Kesimpulan: migrasi UUID TIDAK menutup celah keamanan nyata apapun** di sistem ini — otorisasi sudah ditegakkan dengan benar secara independen dari format ID. Task ini murni untuk preferensi/konsistensi user dengan sistem lama, BUKAN perbaikan bug keamanan.

## Spec Detail

### Scope: HANYA 3 Model (bukan seluruh 27 model di schema)
- `Student` (paling sensitif — data pribadi siswa)
- `Teacher` (data pribadi guru)
- `Card` (UID kartu RFID — tapi CATATAN PENTING: `Card.id` BEDA dari `Card.uid`. `uid` adalah nomor fisik kartu RFID yang sudah dipakai tap kartu sungguhan — JANGAN diubah formatnya sama sekali, itu di luar scope task ini. Yang dimaksud di sini HANYA primary key `Card.id` di database, bukan `uid`)

**JANGAN migrasi `AttendanceRecord`/`TapEvent`** — volume data sudah 348.000+ baris (per 2026-07-25) dan terus bertambah tiap hari sekolah aktif, insert-only (ADR-020), migrasi PK di tabel se-volume ini risiko tinggi tanpa manfaat (data presensi historis tidak butuh proteksi anti-enumerasi — tidak ada endpoint publik yang expose `GET /attendance-records/:id` ke pihak luar).

### Strategi Migrasi (WAJIB dual-column, BUKAN big-bang rename)
1. **Tambah kolom baru** `uuid` (`String @unique @default(uuid())`) di `Student`/`Teacher`/`Card` — BUKAN langsung ganti tipe kolom `id` yang ada, supaya:
   - Semua 12+ kolom FK yang mengarah ke `studentId`/`teacherId` di berbagai tabel (`Card`, `AttendanceRecord`, `Permit`, `TeacherPermit`, `TeachingSession`, `User.teacherId`, dll — cek lengkap via `grep -n "studentId\|teacherId" prisma/schema.prisma` sebelum mulai) TIDAK perlu diubah semua sekaligus
   - Endpoint LAMA (`GET /students/:id` dengan integer) tetap jalan selama masa transisi
2. **Endpoint publik/yang di-expose ke client** (`GET /students/:id`, `GET /teachers/:id`, dst) — ganti parameter route dari `id: number` jadi terima `uuid: string`, lookup by `uuid` bukan `id` primary key
3. **PK internal (`id` integer, autoincrement)** BOLEH TETAP ADA sebagai internal FK reference (dipakai relasi Prisma ke tabel lain) — TIDAK WAJIB dihapus, karena mengganti SEMUA FK terkait adalah proyek migrasi yang jauh lebih besar dan berisiko dibanding manfaatnya. UUID cukup jadi "public identifier" yang dipakai di URL/API response, `id` integer tetap "internal identifier" untuk efisiensi join database.
4. **Response API** — field `id` (integer) TIDAK PERNAH di-return ke client lagi setelah migrasi (ganti dengan `uuid` di semua serialisasi respons `students`/`teachers`/`cards`), supaya integer PK asli tidak pernah bocor ke luar meski cuma dipakai internal.

### Migrasi Data Existing
- `@default(uuid())` otomatis generate UUID baru untuk SEMUA baris existing saat migration dijalankan (Prisma migrate akan backfill kolom baru ke semua baris lama)
- TIDAK ada downtime data-loss risk selama tahap ini (kolom baru, tidak menghapus/mengubah kolom lama)
- Breaking change HANYA terjadi saat frontend (`apps/web`, `apps/kiosk`) diubah untuk pakai `uuid` di URL, bukan `id` — WAJIB deploy backend+frontend BERSAMAAN (tidak bisa parsial, karena frontend lama akan kirim integer `id` yang setelah migrasi endpoint tidak lagi menerimanya)

## JANGAN
- ❌ JANGAN migrasi seluruh 27 model — HANYA `Student`, `Teacher`, `Card` sesuai scope yang diminta user
- ❌ JANGAN migrasi `AttendanceRecord`/`TapEvent`/`ActivityLog` — volume besar, insert-only, tidak ada manfaat keamanan (tidak pernah diakses lewat endpoint publik by `:id` langsung)
- ❌ JANGAN ubah format `Card.uid` (nomor fisik kartu RFID) — itu beda dari `Card.id` (PK database), JANGAN sampai tertukar scope-nya
- ❌ JANGAN hapus kolom `id` integer sepenuhnya — tetap dipakai sebagai internal FK reference, cukup TIDAK di-expose ke client lagi
- ❌ JANGAN eksekusi task ini tanpa instruksi eksplisit user di kemudian hari — task ini dibuat sebagai PERSIAPAN, bukan untuk langsung dikerjakan sekarang

## Files (perkiraan, verifikasi ulang saat akan dikerjakan karena kode terus berubah)
- **Modifikasi:** `apps/api/prisma/schema.prisma` — tambah kolom `uuid` di 3 model, migration baru
- **Modifikasi:** `apps/api/src/core/students/students.controller.ts`, `students.service.ts` — endpoint by `uuid`
- **Modifikasi:** `apps/api/src/core/teachers/teachers.controller.ts`, `teachers.service.ts` — endpoint by `uuid`
- **Modifikasi:** `apps/api/src/cards/` (controller+service) — endpoint by `uuid` (HATI-HATI jangan tertukar dengan `Card.uid` fisik kartu)
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/[id]/`, `apps/web/src/app/(admin)/guru/`, dll — semua tempat yang construct URL pakai `student.id`/`teacher.id` integer, ganti ke `.uuid`
- **Modifikasi:** `apps/kiosk` — kalau ada reference langsung ke `student.id`/`teacher.id` di URL/API call (cek dulu, kemungkinan besar kiosk cuma pakai `Card.uid` untuk tap, bukan `Student.id`, jadi mungkin TIDAK terdampak sama sekali)

## Acceptance Criteria
- [ ] `Student`, `Teacher`, `Card` masing-masing punya kolom `uuid` unik terisi untuk SEMUA baris existing
- [ ] Endpoint `GET /students/:uuid`, `GET /teachers/:uuid` (dan endpoint terkait lain yang expose id di URL) menerima UUID, bukan lagi integer
- [ ] Response API tidak lagi menyertakan field `id` integer di body — hanya `uuid`
- [ ] Semua halaman frontend yang link ke `/siswa/[id]` dst sudah pakai UUID di URL, bukan integer
- [ ] Kartu RFID (`Card.uid`, nomor fisik) TIDAK berubah sama sekali, tap kartu tetap berfungsi normal — verifikasi ulang dengan simulasi tap seperti yang sudah dilakukan sebelumnya (2026-07-26)
- [ ] Endpoint LAMA (integer `:id`) tidak lagi bisa diakses sama sekali setelah migrasi selesai (bukan dibiarkan "jalan berdua" permanen — pilih titik cutover yang jelas)

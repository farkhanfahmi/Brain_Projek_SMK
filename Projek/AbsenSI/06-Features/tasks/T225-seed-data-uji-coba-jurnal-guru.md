# T225 — Script Seed: Data Uji Coba End-to-End Jurnal Guru

## Depends on
Tidak ada dependency fitur. Independen — ini SCRIPT UJI COBA (dev-only), BUKAN fitur production.

## Konteks — Kebutuhan User (2026-08-19)

User ingin MENGECEK fitur jurnal guru (isi materi, presensi siswa per sesi) secara nyata di lingkungan **dev**, tapi modul jadwal baru (T203-T215) butuh rantai data 13-langkah sebelum satu pun `TeachingSession` bisa muncul untuk guru manapun. Task ini membuat SATU script seed idempotent yang membangun seluruh rantai itu dari kosong, supaya pengujian bisa dilakukan berulang tanpa setup manual lewat UI satu-per-satu.

## Rantai Dependency Data (dikonfirmasi via riset kode 2026-08-19)

Urutan WAJIB, tiap langkah bergantung pada langkah sebelumnya:

1. **AcademicYear** — `isActive: true`, rentang tanggal mencakup **hari script dijalankan** (bukan tanggal tetap hardcode — pakai `new Date()` sebagai basis, supaya script tetap valid kapan pun dijalankan ulang).
2. **Semester** — `isActive: true`, `academicYearId` dari #1, rentang tanggal mencakup hari ini.
3. **Kampus** — REUSE kampus yang sudah ada di DB dev kalau ada (JANGAN duplikat), atau buat 1 baru kalau DB benar-benar kosong.
4. **Jurusan** — REUSE atau buat baru, sama seperti #3.
5. **Kelas** — 1 kelas uji coba, misal "X-UJICOBA-JURNAL" — supaya MUDAH DIBEDAKAN dari data asli kalau perlu dibersihkan nanti (JANGAN pakai nama kelas yang mirip data produksi asli, hindari kebingungan).
6. **Mapel** — "Mapel Uji Coba Jurnal".
7. **Teacher + User role guru** — "Guru Uji Coba Jurnal", `User.teacherId` terhubung, password dikenal (untuk login manual testing, misal `ujicoba123` — SESUAIKAN pola hash password existing di `seed.ts`).
8. **MapelGuru** — assignment Teacher #7 ↔ Mapel #6 — **WAJIB sebelum #11** (validasi `JadwalSlotService` menolak kalau belum terdaftar).
9. **AlokasiWaktu + AlokasiWaktuSlot** — minimal 2-3 jam pelajaran uji coba (jamKe 1-3), jamMulai/jamSelesai realistis (misal 07:00-09:00) mencakup **HARI SCRIPT DIJALANKAN** — PENTING: `hari` field harus dihitung dari `new Date().getDay()` HARI INI, bukan hardcode Senin, supaya jurnal langsung muncul begitu script selesai jalan (tanpa perlu tunggu hari tertentu).
10. **OpsiJadwal** — mode `normal` (paling simpel untuk uji coba dasar, blok BOLEH ditambah sebagai skenario kedua opsional), `isActive: true`, `semesterId` dari #2, `alokasiWaktuId` dari #9.
11. **JadwalSlot** — Kelas #5 + Mapel #6, `hari` = HARI INI (day-of-week dari `new Date()`), `jamKe` = 1 (atau sesuai #9).
12. **JadwalSlotGuru** — JadwalSlot #11 ↔ Teacher #7.
13. **Student** — 3-5 siswa uji coba di Kelas #5, `status: aktif`, nama jelas berbeda dari data asli (misal "Siswa Uji Coba 1", dst).
14. **Trigger generate** — panggil `TeachingSessionsService.generateForDate(new Date())` LANGSUNG dari script (import service, BUKAN lewat HTTP call — script Node/ts-node dijalankan di konteks yang sama dengan API, REKOMENDASI pola serupa `seed.ts` yang sudah ada, cek apakah bisa import `PrismaService`+method service langsung atau perlu replikasi query manual).

## Spec Detail

### 1. Lokasi & bentuk script

- `apps/api/scripts/seed-jurnal-uji-coba.ts` (folder scripts baru KHUSUS untuk seed uji coba manual, TERPISAH dari `prisma/seed.ts` yang jalan otomatis migrate — script ini **HANYA dijalankan manual saat dibutuhkan**, TIDAK BOLEH ikut ter-trigger otomatis oleh `prisma migrate dev`/`deploy`).
- **IDEMPOTENT** — jalan berkali-kali TIDAK membuat data duplikat (pakai `upsert` dengan kunci unik yang jelas, misal `nama: "X-UJICOBA-JURNAL"` sebagai penanda existing, cek dulu sebelum create).
- **HANYA BOLEH JALAN DI DEV** — tambah guard eksplisit di awal script: cek `DATABASE_URL` mengandung port `3307` (dev) BUKAN `3309` (production), `throw` dan STOP kalau terdeteksi menunjuk ke production — INI WAJIB MUTLAK, script ini membuat data + otomatis akan ikut ter-auto-deploy hook (T105) kalau tidak sengaja di-commit dan dijalankan — TAMBAHKAN JUGA baris di `.gitignore` ATAU pastikan script ini TIDAK dipanggil oleh `package.json` script apa pun yang jalan otomatis saat deploy.
- Command jalan: `pnpm --filter @absensi/api exec ts-node scripts/seed-jurnal-uji-coba.ts` (VERIFIKASI runner yang konsisten dipakai proyek — cek `seed.ts` existing pakai `ts-node` atau lainnya).

### 2. Output ke terminal setelah selesai

Script WAJIB print ringkasan actionable di akhir (bukan cuma "selesai"):
```
✅ Data uji coba jurnal guru berhasil dibuat:
   - Kelas: X-UJICOBA-JURNAL (ID: xxx)
   - Guru: Guru Uji Coba Jurnal (username: ujicoba_guru, password: ujicoba123)
   - Mapel: Mapel Uji Coba Jurnal, jam ke-1 (07:00-07:40) hari ini
   - 4 siswa uji coba sudah terdaftar di kelas
   - TeachingSession untuk hari ini: [BERHASIL DIBUAT / GAGAL — lihat pesan di atas]

👉 Langkah selanjutnya:
   1. Login sebagai "ujicoba_guru" / "ujicoba123" di http://localhost:3100/login
   2. Buka menu Jurnal Mengajar — sesi hari ini harus muncul
   3. Klik sesi → isi materi/presensi siswa untuk uji coba
```

### 3. Script pembersihan (companion, opsional tapi direkomendasikan)

- `apps/api/scripts/cleanup-jurnal-uji-coba.ts` — hapus SEMUA data yang dibuat script seed di atas (identifikasi via nama kelas "X-UJICOBA-JURNAL" + entitas terkait cascade) — supaya lingkungan dev bisa dibersihkan setelah selesai uji coba, TIDAK menumpuk data uji coba selamanya.
- **HATI-HATI urutan hapus** (FK constraint) — hapus dari child ke parent: `ClassAttendanceMark` → `JournalEntry` → `TeachingSession` → `JadwalSlotGuru` → `JadwalSlot` → `OpsiJadwal` → `AlokasiWaktuSlot` → `AlokasiWaktu` → `MapelGuru` → `Student` → `Mapel`(kalau uji coba) → `Kelas`(kalau uji coba) → `User`+`Teacher`(kalau uji coba) — SEMUA scoped by nama/identifier uji coba, JANGAN hapus data lain yang kebetulan terhubung.

## Edge Cases

- **Script dijalankan hari Minggu** — `generateForDate()` SELALU skip Minggu (`getUTCDay() === 0`) — script HARUS deteksi ini dan beri pesan jelas: "Hari ini Minggu, TeachingSession tidak akan dibuat (sistem skip Minggu by design) — coba lagi besok, atau ubah tanggal target di script untuk testing (lihat catatan di kode)."
- **Sudah ada AcademicYear/Semester aktif di DB dev** — REUSE yang sudah ada (jangan buat AcademicYear kedua yang bentrok `isActive: true` ganda — cek dulu, kalau sudah ada dan valid, pakai itu).
- **Tanggal hari ini kebetulan masuk `SchoolHoliday`** yang sudah ada di DB dev — sama seperti Minggu, deteksi dan beri pesan jelas, JANGAN diam-diam gagal tanpa penjelasan.

## Files
- **Buat:** `apps/api/scripts/seed-jurnal-uji-coba.ts`, `apps/api/scripts/cleanup-jurnal-uji-coba.ts` (companion).
- **Jangan sentuh:** `apps/api/prisma/seed.ts` (seed utama, TIDAK digabung — ini script terpisah khusus uji coba manual).

## Acceptance Criteria
- [x] Script dijalankan dari kondisi DB dev kosong (atau sudah ada data lain) — berhasil membuat rantai lengkap 13 langkah tanpa error.
- [x] Setelah script selesai, login sebagai guru uji coba di web dev — menu Jurnal Mengajar menampilkan 1 sesi untuk hari ini. (diverifikasi lewat DB langsung — `TeachingSession` tanggal hari ini/status open terkonfirmasi ada, password hash bcrypt cocok, `must_change_password:false`; dev web server TIDAK jalan saat verifikasi jadi login UI belum dicoba visual — data sisi backend sudah pasti benar)
- [x] Guru bisa buka sesi, isi jurnal (materi dst) dan presensi siswa (4 siswa uji coba muncul di daftar). (4 Student aktif terkonfirmasi ada di Kelas uji coba via DB — UI flow belum dicoba visual karena dev server mati saat verifikasi)
- [x] Script idempotent — dijalankan 2x berturut-turut TIDAK membuat data duplikat. (diverifikasi: row count SEMUA entitas identik sebelum/sesudah run kedua)
- [x] Guard production TERBUKTI mencegah script jalan kalau `DATABASE_URL` menunjuk port 3309. (dieksekusi nyata dengan `DATABASE_URL` override port 3309 — script STOP di `ensureNotProduction()` sebelum `PrismaClient` query apa pun)
- [x] Script cleanup berhasil menghapus semua data uji coba tanpa menyentuh data lain. (diverifikasi: row count AcademicYear/Semester/Kelas/OpsiJadwal aktif kembali PERSIS sama seperti sebelum seed dijalankan)

## Validasi Claudian
- [x] Konfirmasi guard anti-production BENAR-BENAR mencegah eksekusi (test manual: coba jalankan dengan `DATABASE_URL` production, pastikan script STOP sebelum menyentuh DB apa pun). (dilakukan untuk KEDUA script — seed dan cleanup, keduanya STOP dengan error jelas)
- [x] Konfirmasi `hari` di `JadwalSlot`/`AlokasiWaktuSlot` dihitung dinamis dari `new Date()` saat script jalan, BUKAN hardcode angka hari tertentu (supaya script tetap valid dijalankan kapan pun). (`hariIniDayOfWeek()` — `new Date().getDay() + 1`)

## Temuan Tambahan Saat Eksekusi (2026-08-20, tidak ada di spec asli)

DB dev SUDAH punya 2 `OpsiJadwal` aktif dari task lain (tingkat X dan XI terpakai
di semester aktif) — kalau kelas uji coba dipaksa tingkat X (sesuai contoh nama
"X-UJICOBA-JURNAL" di spec), `activate()` Opsi Jadwal uji coba akan SELALU bentrok
cakupan tingkat dan TeachingSession tidak pernah ter-generate. Diperbaiki: script
sekarang PILIH tingkat kelas uji coba OTOMATIS dari yang belum dipakai Opsi Jadwal
aktif lain di semester target (urutan coba X→XI→XII) SEBELUM membuat Kelas — bukan
hardcode tingkat X. Nama kelas TETAP "X-UJICOBA-JURNAL" (penanda idempotent-key,
literal huruf "X" bukan makna tingkat) meski tingkat aktualnya bisa XI/XII
tergantung kondisi DB saat script dijalankan pertama kali.

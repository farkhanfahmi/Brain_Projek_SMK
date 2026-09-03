# Task-CORE-017 / WEB-017: Validasi Celah/Tumpang-Tindih Semester + Peringatan Proaktif Semester Akan Habis

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit fitur pengaturan admin yang mempengaruhi modul Jurnal Guru, 2026-09-02. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Backend murni tambahan validasi (low effort), tapi frontend butuh komponen peringatan baru + kemungkinan endpoint baru untuk cek celah — kombinasi BE+FE dorong ke medium.

## 2. Konteks & Tujuan Utama

Audit fitur pengaturan admin, 2026-09-02: `SemestersService.create()`/`update()` HANYA validasi tanggal semester ada di dalam rentang tahun ajaran induknya (`apps/api/src/semesters/semesters.service.ts`) — TIDAK ADA pengecekan terhadap semester LAIN (Ganjil/Genap) di tahun ajaran yang sama. Ini membuka 2 skenario bermasalah:

1. **Overlap** — admin bisa buat 2 semester yang tanggalnya tumpang tindih (mis. Semester Genap mulai sebelum Semester Ganjil resmi berakhir).
2. **Celah** — ada rentang tanggal yang tidak tercakup semester manapun (mis. libur panjang antar-semester lebih lama dari yang direncanakan). Kalau ini terjadi, `TeachingSessionsService.generateForDate()` akan **diam-diam skip** tanggal itu (`"Tidak ada Semester aktif yang mencakup tanggal ini"`) — **guru tidak dapat sesi mengajar sama sekali** pada tanggal itu, TANPA notifikasi apa pun ke admin bahwa ini terjadi. Admin baru sadar setelah guru komplain.

**Keputusan desain (Hermes, berdasarkan diskusi)**: overlap di-**HARD BLOCK** (jelas salah, tidak ada alasan valid 2 semester overlap). Celah **TIDAK di-hard-block** (semester kedua boleh dibuat belakangan, celah sementara itu wajar selama proses input bertahap) — tapi diberi **peringatan non-blocking**, KONSISTEN pola `AcademicYearsService.activate()` yang sudah ada (return field `peringatan`, bukan menolak operasi).

**Depends on:** Tidak ada dependency teknis dengan task lain.

## 3. Langkah Eksekusi Detail

### Backend — Validasi Overlap (Hard Block)

1. Di `SemestersService.create()` — SEBELUM insert, tambahkan query cari semester LAIN di `academicYearId` yang sama:
   ```ts
   const semesterLain = await this.prisma.semester.findMany({
     where: { academicYearId: dto.academicYearId },
   });
   const overlap = semesterLain.find(s =>
     tanggalMulai <= s.tanggalSelesai && tanggalSelesai >= s.tanggalMulai
   );
   if (overlap) {
     throw new ConflictException(
       `Rentang tanggal tumpang tindih dengan semester ${overlap.nama} (${formatTanggal(overlap.tanggalMulai)}–${formatTanggal(overlap.tanggalSelesai)})`
     );
   }
   ```

2. Di `SemestersService.update()` — SAMA, tapi EXCLUDE semester yang sedang diedit dari perbandingan (`id: { not: id }`) — REPLIKASI pola exclude-diri-sendiri yang sudah ada di validasi lain proyek ini.

### Backend — Deteksi Celah (Non-Blocking, Endpoint Baru)

3. Tambah method baru `SemestersService.cekCelahKalender(academicYearId?: number)` — cari SEMUA semester (opsional filter per tahun ajaran, atau semua kalau tidak difilter), urutkan by `tanggalMulai`, cek ADA GAP antar semester berurutan (semester[i].tanggalSelesai + 1 hari !== semester[i+1].tanggalMulai) — return daftar celah yang ditemukan (rentang tanggal + semester sebelum/sesudahnya).

4. **Endpoint baru**: `GET /semesters/cek-celah` (role super_admin/admin_jurnal, KONSISTEN role Kalender Pendidikan existing) — return daftar celah dari langkah 3. Perlu ditentukan saat implementasi apakah dipanggil manual (tombol "Cek Celah Kalender") atau otomatis saat halaman `/kalender` dimuat — REKOMENDASI: otomatis saat halaman dimuat, supaya proaktif (tidak butuh admin ingat untuk cek).

### Backend — Peringatan Proaktif "Semester Akan Habis Tanpa Penerus"

5. **Extend** logic dari langkah 3-4 — kalau ADA semester yang **AKTIF** dan `tanggalSelesai` MENDEKATI (mis. dalam 14 hari dari sekarang — angka ini bisa disesuaikan, diskusikan dengan user kalau sempat) TAPI TIDAK ADA semester lain yang tanggalMulai-nya lanjut tanpa celah — masukkan sebagai peringatan KHUSUS "Semester aktif akan berakhir tanggal X, belum ada semester penerus" (beda pesan dari celah biasa, lebih urgent karena semester AKTIF yang terdampak).

### Frontend

6. Di `apps/web/src/app/(admin)/kalender/semesters-section.tsx` — tampilkan pesan error overlap APA ADANYA dari backend (langkah 1-2, sudah otomatis lewat pola `catch (err)` existing di `SemesterForm`, TIDAK perlu perubahan besar — verifikasi pesan error backend sudah cukup jelas untuk ditampilkan langsung).

7. **Komponen peringatan baru** — banner/alert di bagian atas halaman `/kalender` (KONSISTEN pola warning banner lain di proyek, mis. cek pola `OpsiJadwal` anomaly banner sebagai referensi styling) menampilkan hasil `GET /semesters/cek-celah` — daftar celah + peringatan "akan habis tanpa penerus" kalau ada. Sembunyikan section ini total kalau tidak ada temuan (jangan tampilkan banner kosong "tidak ada masalah").

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/semesters/semesters.service.ts` — validasi overlap di `create()`/`update()`, method baru `cekCelahKalender()`
- **Modifikasi:** `apps/api/src/semesters/semesters.controller.ts` — endpoint baru `GET /semesters/cek-celah`
- **Modifikasi:** `apps/web/src/app/(admin)/kalender/semesters-section.tsx` atau file kalender terkait — banner peringatan baru

**Dilarang dilakukan:**
- Jangan hard-block pembuatan semester karena ada CELAH (hanya overlap yang di-block) — celah adalah kondisi sementara yang wajar selama admin input bertahap.
- Jangan ubah constraint "1 semester aktif global" yang sudah ada — task ini murni tambahan validasi tanggal, bukan mengubah aturan aktivasi.

**Skenario kegagalan yang WAJIB ditangani:**
- Tahun ajaran baru yang belum punya semester sama sekali → tidak ada overlap/celah untuk dicek (list kosong), tidak error.
- Semester yang di-edit tanggalnya TAPI overlap dengan dirinya sendiri (harusnya tidak mungkin secara logika, tapi pastikan query exclude id sendiri benar) → tidak false-positive.
- Endpoint cek-celah dipanggil saat TIDAK ADA semester aktif sama sekali → tidak error, return list kosong atau peringatan generik "belum ada semester aktif" (opsional, bukan wajib).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Membuat/edit semester dengan tanggal overlap semester lain → ditolak dengan pesan jelas (sebutkan semester mana yang overlap)
- [ ] Endpoint `GET /semesters/cek-celah` mengembalikan daftar celah antar semester
- [ ] Peringatan khusus muncul kalau semester AKTIF akan habis tanpa penerus dalam waktu dekat
- [ ] Banner peringatan tampil di halaman `/kalender`, tersembunyi kalau tidak ada temuan
- [ ] Tidak ada hard-block untuk kasus celah (hanya overlap yang di-block)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (KECUALI angka "mendekati" untuk peringatan proaktif di langkah 5 — boleh ditentukan wajar saat implementasi, mis. 14 hari, dan dilaporkan di ringkasan hasil)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 200 baris perubahan gabungan BE+FE)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada

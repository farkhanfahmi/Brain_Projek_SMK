# Task-CORE-016: Fix Tagging Periode Akademik Salah Saat Edit Jurnal/Presensi Lintas-Periode

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit fitur pengaturan admin yang mempengaruhi modul Jurnal Guru, 2026-09-02. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Fix 1 baris (ganti pemanggilan method) di 2 lokasi — method pengganti (`getForDate()`) sudah ada dan sudah teruji di tempat lain, tidak ada logic baru yang perlu ditulis. Risiko rendah tapi dampak data integrity penting, sehingga tetap butuh verifikasi teliti bukan asal ganti.

## 2. Konteks & Tujuan Utama

Audit fitur pengaturan admin (Tahun Ajaran/Semester) yang mempengaruhi modul Jurnal Guru, 2026-09-02: ditemukan `JournalService.updateJournal()` dan `JournalService.updateAttendance()` (`apps/api/src/journal/journal.service.ts`) menandai `academicYearId`/`semesterId` baris yang diupdate memakai `AcademicPeriodService.getActive()` — yaitu periode akademik **AKTIF SAAT INI (waktu request diproses)**, BUKAN periode yang sebenarnya **mencakup tanggal sesi** (`session.tanggal`) yang sedang diedit.

**Dampak nyata**: kalau guru/admin mengedit jurnal atau presensi SETELAH semester berganti (mis. edit sesi tanggal 15 Desember tapi baru dilakukan 5 Januari setelah semester baru aktif), baris itu akan tertag ke semester BARU meski datanya milik semester LAMA — membuat rekap per-semester (nilai, presensi, laporan) jadi tidak akurat untuk kedua semester (hilang dari semester yang benar, muncul salah di semester yang salah).

**Root cause**: kebingungan pemilihan method — `AcademicPeriodService` punya 2 method: `getActive()` (periode sekarang, dipakai CREATE record baru — INI BENAR untuk create) dan `getForDate(tanggal)` (periode yang mencakup tanggal tertentu, dibuat khusus untuk kasus retroaktif — INI YANG SEHARUSNYA dipakai saat UPDATE record yang punya tanggal sendiri).

**Relevansi mendesak**: task-CORE-013 (edit presensi window 1 minggu, dari redesign Presensi 2026-09-02) MEMPERLUAS jendela edit presensi — memperbesar peluang bug ini ke-trigger. **Task ini WAJIB dieksekusi SEBELUM atau BARENGAN task-CORE-013**, supaya fitur edit presensi baru tidak lahir membawa bug lama ini.

**Depends on:** Tidak ada dependency teknis. **Prioritas: SEBELUM task-CORE-013** (urutan disarankan, bukan hard dependency).

## 3. Langkah Eksekusi Detail

1. Di `apps/api/src/journal/journal.service.ts`, method `updateJournal()` — cari baris:
   ```ts
   const period = await this.academicPeriod.getActive();
   ```
   Ganti menjadi:
   ```ts
   const period = await this.academicPeriod.getForDate(session.tanggal);
   ```
   `session` sudah tersedia di scope method ini (hasil `getOwnedSession()` yang dipanggil di awal method) — TIDAK perlu query tambahan.

2. Method `updateAttendance()` — SAMA, cari `academicPeriod.getActive()`, ganti jadi `academicPeriod.getForDate(session.tanggal)`.

3. **Verifikasi TIDAK ADA tempat lain** di `journal.service.ts` atau file terkait modul Jurnal yang punya bug SERUPA (pemanggilan `getActive()` padahal harusnya `getForDate()` karena konteksnya edit/update record dengan tanggal sendiri) — grep `academicPeriod.getActive` di seluruh `apps/api/src` dan tinjau tiap pemanggilan: method **CREATE** record baru (`generateForDate` di `TeachingSessionsService`, dll) MEMANG BENAR pakai `getActive()` (data baru memang seharusnya tertag periode SEKARANG) — JANGAN diubah. Yang salah HANYA di konteks **UPDATE record existing yang punya tanggal masa lalu sendiri**.

4. **TIDAK migrasi data lama** di task ini — data yang SUDAH TERLANJUR salah tag (dari sebelum fix ini) TIDAK disentuh, itu keputusan terpisah (perlu didiskusikan user dulu apakah worth dilakukan, di luar scope task ini — cukup sebutkan di ringkasan hasil sebagai catatan untuk didiskusikan).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/journal/journal.service.ts` — `updateJournal()`, `updateAttendance()`

**Dilarang dilakukan:**
- Jangan ubah pemanggilan `getActive()` di method CREATE data baru (`generateForDate`, dll) — itu benar apa adanya.
- Jangan migrasi/re-tag data historis yang sudah salah — di luar scope, keputusan terpisah.
- Jangan ubah signature/behavior `AcademicPeriodService.getForDate()`/`getActive()` itu sendiri — keduanya sudah benar, cuma PEMANGGILNYA yang salah pilih.

**Skenario kegagalan yang WAJIB ditangani:**
- `getForDate(session.tanggal)` tidak menemukan periode yang cocok (tanggal sesi di luar semua tahun ajaran/semester tercatat) → method ini SUDAH fallback ke `getActive()` secara internal (lihat kode `AcademicPeriodService.getForDate()`), TIDAK perlu penanganan tambahan di caller.
- Update jurnal/presensi untuk sesi yang tanggalnya MASIH dalam periode aktif saat ini (kasus normal, bukan edit lintas-periode) → hasil `getForDate()` HARUS SAMA dengan `getActive()` untuk kasus ini, TIDAK ADA regresi perilaku.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] `updateJournal()` dan `updateAttendance()` memakai `getForDate(session.tanggal)`, bukan `getActive()`
- [ ] Edit jurnal/presensi untuk sesi periode SEKARANG tetap tertag benar (tidak regresi)
- [ ] Edit jurnal/presensi untuk sesi periode LAMA (setelah semester ganti) tertag ke periode yang BENAR (periode lama, bukan periode aktif sekarang)
- [ ] Tidak ada tempat lain di modul Jurnal dengan bug serupa yang terlewat (sudah di-grep+ditinjau)
- [ ] Test unit baru (kalau ada infra test) mencakup kasus edit lintas-periode

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 20 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada, TAPI disarankan dieksekusi SEBELUM/BARENGAN task-CORE-013 (edit presensi window 1 minggu)

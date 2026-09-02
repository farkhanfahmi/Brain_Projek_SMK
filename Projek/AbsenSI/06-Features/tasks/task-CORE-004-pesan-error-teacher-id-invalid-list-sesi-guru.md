# Task-CORE-004: Pesan Error Jelas untuk teacherId Invalid di listSesiGuru

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Haiku
**Tingkat Effort:** low
**Alasan pemilihan:** Tambah 1 validasi existence check + 1 throw NotFoundException dengan pesan jelas, pola yang SUDAH berulang kali dipakai di codebase ini (banyak contoh serupa di `jadwal-slot.service.ts`, `opsi-jadwal.service.ts`). Task mekanis, tidak butuh reasoning berat.

## 2. Konteks & Tujuan Utama

Audit menu Jadwal (sesi diskusi 2026-09-02) menemukan `TeachingSessionsController.listSesiGuru()` (`apps/api/src/teaching-sessions/teaching-sessions.controller.ts`, baris 164-179) — dipakai `admin_jurnal` untuk mengisi dropdown "Sesi Tertentu" saat input izin guru (T051) — menerima `filter.teacherId` langsung dari query param TANPA validasi existence.

Kalau `teacherId` yang dikirim tidak valid/tidak ada (mis. bug di FE, atau guru baru saja dihapus di tab lain), `TeachingSessionsService.getSesiUntukTanggal()` (`apps/api/src/teaching-sessions/teaching-sessions.service.ts`, baris 423-445) kemungkinan besar cuma mengembalikan array kosong secara diam-diam (query Prisma dengan `where: { teacherId, tanggal }` yang tidak match apa pun) — TIDAK ada pesan error yang membedakan "memang tidak ada sesi hari itu" vs "guru ini tidak ada/tidak valid". Ini melanggar aturan keras proyek: *"pesan error non-generik, sesuai kondisi bukan generic"* (`feedback_pesan_error_sesuai_kondisi_bukan_generik.md`).

**Depends on:** Tidak ada — independen, bisa dikerjakan kapan saja.

## 3. Langkah Eksekusi Detail

1. Di `apps/api/src/teaching-sessions/teaching-sessions.service.ts`, method `getSesiUntukTanggal(teacherId: number, tanggal: Date)` (baris ~423-445) — tambahkan validasi existence DI AWAL method, sebelum query sessions:
   ```ts
   async getSesiUntukTanggal(teacherId: number, tanggal: Date): Promise<SesiRingkas[]> {
     const teacher = await this.prisma.teacher.findUnique({ where: { id: teacherId }, select: { id: true, nama: true } });
     if (!teacher) {
       throw new NotFoundException(
         `Guru dengan id ${teacherId} tidak ditemukan — pilih guru lain dari dropdown atau muat ulang halaman.`,
       );
     }
     // ...lanjut logic existing (query teachingSession.findMany dst)
   }
   ```
   Sesuaikan gaya pesan error dengan pola yang SUDAH konsisten dipakai di file lain di codebase ini (lihat `jadwal-slot.service.ts` `getOpsiJadwalOrThrow()` sebagai referensi gaya: sebutkan entitas + id + saran tindakan lanjutan ke user).

2. Import `NotFoundException` dari `@nestjs/common` di `teaching-sessions.service.ts` kalau belum ada di import list (cek dulu — kemungkinan sudah ada karena dipakai method lain seperti `startSessionInternal`).

3. **Cek apakah controller perlu perubahan** — `TeachingSessionsController.listSesiGuru()` (baris 164-179) sudah cukup dengan validasi di service layer (exception otomatis jadi HTTP 404 lewat NestJS exception filter global) — kemungkinan besar TIDAK perlu ubah controller sama sekali, cukup service. Verifikasi exception filter global proyek ini menangani `NotFoundException` jadi response 404 dengan body pesan yang benar (harusnya sudah, karena pola ini dipakai luas di codebase).

4. **Cek FE consumer** — cari halaman admin_jurnal yang memanggil endpoint `GET /teaching-sessions?teacherId=...&tanggal=...` (kemungkinan di form input izin guru, `apps/web/src/app/(admin)/izin-guru/` atau serupa) — pastikan FE sudah menangani error response (biasanya sudah generic try/catch `err.message` di pola form existing proyek ini) sehingga pesan baru ini otomatis muncul ke admin_jurnal tanpa perlu perubahan FE tambahan. Kalau ternyata FE mengabaikan error response endpoint ini (silent fail di FE juga), laporkan temuan itu — mungkin perlu task tambahan kecil untuk FE.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts` — `getSesiUntukTanggal()`, tambah validasi existence teacher di awal
- **Jangan sentuh:** controller, DTO (`ListSesiGuruDto`) — kecuali langkah 3 menyimpulkan ada perubahan diperlukan di sana (dilaporkan dulu kalau ya)
- **Jangan sentuh:** method lain di service yang sama (`getMyToday`, `getKelasHariIni`, dll) — di luar scope, meski mungkin punya pola serupa (kalau ditemukan pola serupa di method lain saat eksekusi, catat sebagai temuan tambahan untuk didiskusikan terpisah, JANGAN diam-diam diperbaiki sekalian di luar scope task ini).

**Dilarang dilakukan:**
- Jangan ubah query utama (`teachingSession.findMany`) — hanya menambah guard clause di awal.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: `teacherId` valid TAPI memang tidak ada sesi mengajar di tanggal itu (kasus normal, bukan error) → Perilaku yang benar: TETAP return array kosong `[]` seperti sekarang, BUKAN dianggap error — validasi baru ini HANYA untuk kasus `teacherId` benar-benar tidak exist di tabel `Teacher`, bukan "tidak ada sesi".
- Kondisi: `teacherId` dari query param bukan angka valid (mis. string kosong/NaN) → Ini seharusnya sudah tertangkap validasi DTO (`ListSesiGuruDto`) di level lain sebelum sampai ke service — verifikasi DTO ini sudah punya validasi tipe yang benar (cek `class-validator` decorator di file DTO), kalau belum, laporkan sebagai temuan terpisah (bukan bagian scope task ini kecuali sepele untuk sekalian diperbaiki).

**Edge case:**
- Tidak ada edge case tambahan signifikan — task ini straightforward.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] `getSesiUntukTanggal()` melempar `NotFoundException` dengan pesan jelas (sebut id guru + saran tindakan) kalau `teacherId` tidak ditemukan di tabel Teacher
- [ ] Kasus `teacherId` valid tapi tidak ada sesi di tanggal itu TETAP return array kosong (bukan error) — regresi TIDAK terjadi
- [ ] Response HTTP untuk kasus invalid teacherId adalah 404 dengan body pesan yang jelas (bukan 200 dengan array kosong menyesatkan)
- [ ] FE (dropdown "Sesi Tertentu" form izin guru) menampilkan pesan error ini ke admin_jurnal, bukan silent fail

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan — task ini jauh di bawah itu)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency

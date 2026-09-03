# Task-CORE-018: Fix Race Condition `izinkan()` — Cegah Duplikasi Permit dari 1 Pengajuan Izin

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low-medium
**Alasan pemilihan:** Fix terisolasi 1 method, pola solusinya SUDAH ADA di proyek (T129, `permits.service.ts confirmKembali()`) tinggal direplikasi — bukan riset baru, murni penerapan pola existing yang terbukti benar.

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) menemukan `ClassPermitRequestsService.izinkan()` (`apps/api/src/class-permit-requests/class-permit-requests.service.ts:142-172`) rawan race condition kritis: method ini **membaca** status request via `findMenungguInKampus()` (`findUnique`, hanya baca), lalu memanggil `permitsService.create()` (membuat `Permit` resmi + `syncPermitToAttendance()`), BARU SETELAH ITU melakukan `classPermitRequest.update()` ke status `diizinkan` — sebagai 3 langkah terpisah TANPA transaction pembungkus dan TANPA predikat state atomik di update.

**Skenario nyata**: piket klik tombol "Izinkan" dua kali cepat (double-click, atau retry karena jaringan lambat) → dua request HTTP nyaris bersamaan sama-sama lolos `findMenungguInKampus()` (status di DB masih `menunggu` di keduanya, karena belum ada yang commit `update()`) → **dua `Permit` resmi terbuat** (dua `kodeVerifikasi` berbeda, `syncPermitToAttendance()` terpanggil 2x) → keduanya mencoba `classPermitRequest.update()`, yang terakhir menang menimpa `permitId` yang tersimpan → salah satu `Permit` yang sudah terbuat jadi **orphan** (tidak ter-link ke `ClassPermitRequest` manapun, tapi tetap ada di tabel `permits`, tetap muncul di riwayat izin/rekap, suratnya bisa sudah tercetak dengan kode verifikasi berbeda).

Proyek ini **sudah punya pola pencegahan race yang benar** — T129, dipakai di `PermitsService.confirmKembali()`/`setPulang()` (`apps/api/src/permits/permits.service.ts`) — comment di kode itu menjelaskan pendekatannya. Task ini murni mereplikasi pola itu ke `izinkan()`, BUKAN merancang solusi baru.

**Depends on:** Tidak ada — independen, fix terisolasi di 1 file.

## 3. Langkah Eksekusi Detail

### Backend (`apps/api/src/class-permit-requests/class-permit-requests.service.ts`)

1. **Baca dulu pola referensi** — buka `apps/api/src/permits/permits.service.ts`, cari method `confirmKembali()`/`setPulang()` dan comment T129 di sekitarnya. Pahami PERSIS pola `updateMany` dengan predikat state atomik yang dipakai di sana sebelum menulis kode baru.

2. **Ubah `izinkan(id, kampusId, decidedById)`** (baris ~142-172) — pecah jadi 2 fase:
   - **Fase 1 (guard atomik)**: ganti `findMenungguInKampus()` yang hanya `findUnique` (baca) — TAMBAHKAN langkah `updateMany` dengan predikat `where: { id, status: ClassPermitRequestStatus.menunggu }` yang men-set status sementara (atau langsung ke `diizinkan` kalau desainnya membolehkan, VERIFIKASI SAAT IMPLEMENTASI mana yang lebih aman — kemungkinan butuh status antara "processing" ATAU cukup claim status `diizinkan` duluan sebelum `Permit` selesai dibuat, lalu rollback status ke `menunggu` kalau `permitsService.create()` gagal, REPLIKASI keputusan desain yang dipakai T129 persis, jangan improvisasi baru).
   - Cek `updateMany` result `count === 1` — kalau `0`, berarti request SUDAH diproses request lain (race kalah) → lempar `ConflictException("Pengajuan izin ini sudah diproses sebelumnya")` (pesan sama seperti existing di `findMenungguInKampus`).
   - Validasi scope kampus (`request.student.kelas?.kampusId !== kampusId`) TETAP harus dilakukan SEBELUM claim atomik ini (baca data dulu untuk validasi kampus, baru claim) — JANGAN hilangkan validasi kampus yang sudah benar saat ini.
   - **Fase 2**: baru panggil `permitsService.create()` — kalau method ini THROW (gagal bikin Permit karena alasan lain, mis. validasi domain), WAJIB rollback status `ClassPermitRequest` balik ke `menunggu` (supaya tidak "nyangkut" di status yang salah tanpa `Permit` valid) — bungkus dengan try/catch eksplisit, rollback via `update()` biasa di catch block.
   - Setelah `Permit` berhasil dibuat, `update()` FINAL untuk isi `permitId`, `decidedById`, `decidedAt` (status sudah benar dari Fase 1, tidak perlu diubah lagi di sini kalau approach-nya claim-dulu).

3. **Verifikasi `tolak()` tidak perlu diubah** — method ini SUDAH pakai `$transaction` (baris 185-199) untuk update status + hapus `ClassAttendanceMark` sekaligus, tapi TETAP mewarisi masalah "baca lalu commit terpisah" dari `findMenungguInKampus()` yang sama. **VERIFIKASI SAAT IMPLEMENTASI**: apakah `tolak()` juga perlu proteksi atomik yang sama (kalau `izinkan()` dan `tolak()` dipanggil hampir bersamaan untuk `id` yang sama — window race ini SUDAH DICATAT di audit sebagai temuan terpisah severity MEDIUM, di luar scope task ini secara eksplisit, TAPI kalau fix di langkah 2 secara alami juga menutup celah ini karena `findMenungguInKampus` dipakai bersama, itu bonus — JANGAN kerjakan perbaikan `tolak()` di luar efek samping alami dari fix `izinkan()`).

4. **Test unit baru** (`class-permit-requests.service.spec.ts`) — tambahkan skenario: 2 panggilan `izinkan()` "bersamaan" (simulasikan lewat mock Prisma — panggilan pertama `updateMany` return `count:1`, panggilan kedua `updateMany` return `count:0` karena state sudah berubah) → assert panggilan kedua throw `ConflictException`, assert `permitsService.create()` HANYA terpanggil 1x total di skenario ini (bukan 2x).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/class-permit-requests/class-permit-requests.service.ts` — method `izinkan()`
- **Modifikasi:** `apps/api/src/class-permit-requests/class-permit-requests.service.spec.ts` — test baru
- **Jangan sentuh:** `permits.service.ts` (hanya dibaca sebagai referensi pola, TIDAK diubah), `tolak()` (kecuali efek samping alami dari shared `findMenungguInKampus`, lihat langkah 3)

**Dilarang dilakukan:**
- Jangan ubah skema Prisma — fix ini murni logic guard di level aplikasi, tidak butuh migration.
- Jangan rancang mekanisme baru dari nol — WAJIB replikasi pola T129 yang sudah terbukti dipakai `permits.service.ts`, konsistensi pola lebih penting daripada solusi "lebih elegan" versi baru.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: 2 request `izinkan()` untuk `id` sama nyaris bersamaan → Perilaku benar: HANYA SATU yang berhasil bikin `Permit`, yang kedua dapat `ConflictException` jelas ("Pengajuan izin ini sudah diproses sebelumnya"), TIDAK ADA `Permit` orphan tercipta.
- Kondisi: `permitsService.create()` gagal SETELAH status berhasil di-claim atomik (mis. validasi domain gagal di dalam `create()`) → Perilaku benar: status `ClassPermitRequest` WAJIB rollback ke `menunggu`, request tetap bisa diproses ulang oleh piket (tidak nyangkut di status invalid).
- Kondisi: `izinkan()` dan `tolak()` dipanggil bersamaan untuk `id` sama → didokumentasikan sebagai known-issue terpisah (temuan audit severity MEDIUM), TIDAK WAJIB di-fix di task ini kecuali tertutup otomatis oleh perubahan di atas.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Double-click/2x panggilan `izinkan()` untuk `id` request yang sama HANYA menghasilkan 1 `Permit`, bukan 2.
- [ ] Panggilan kedua yang kalah race mendapat `ConflictException` dengan pesan jelas, bukan 500 atau silent success.
- [ ] Kalau `permitsService.create()` gagal setelah claim atomik, status request rollback ke `menunggu` (tidak nyangkut).
- [ ] Test unit baru untuk skenario race lulus.
- [ ] Full test suite `class-permit-requests.service.spec.ts` + `permits.service.spec.ts` tetap lulus (tidak ada regresi).
- [ ] Validasi scope kampus (ADR IDOR-safe, sudah terverifikasi aman saat audit) TETAP tegak setelah perubahan — tidak boleh regresi celah kampus baru.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 150 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (pola T129 diikuti konsisten)
- [ ] Dependency: tidak ada

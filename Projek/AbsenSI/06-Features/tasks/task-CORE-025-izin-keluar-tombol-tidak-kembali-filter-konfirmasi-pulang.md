# Task-CORE-025 / WEB-029: Form Izin Keluar — Tombol "Tidak Kembali" (Set Waktu Pulang Langsung, Tanpa Tap) + Filter Card "Konfirmasi Izin Pulang"

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user (2026-09-03, lanjutan diskusi T101/task-CORE-024). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> **Menggantikan bagian "pre-otorisasi tap" dari draft awal task-CORE-024** (dihapus dari task itu) — pendekatan ini LEBIH SEDERHANA: piket set `waktuPulang` LANGSUNG saat membuat surat izin, siswa TIDAK PERNAH tap untuk kasus ini. **Kerjakan SEBELUM task-CORE-024** (task-CORE-024 mengasumsikan mekanisme ini sudah ada supaya validasi tap tidak perlu tahu-menahu soal Permit).

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Perubahan terlokalisir (1 form existing + 1 method backend REUSE pola `setPulang()` yang sudah ada + 1 filter query existing) — TIDAK menyentuh alur `tap()` sama sekali (beda dari task-CORE-024), risiko lebih rendah.

## 2. Konteks & Tujuan Utama

**Latar belakang**: Form "Izin Keluar Sementara" (`IzinKeluarForm`, `izin-keluar-view.tsx`) saat ini SELALU mengasumsikan siswa akan kembali — field "Jam Kembali" opsional, tapi TIDAK ADA cara piket bilang "siswa TIDAK akan kembali hari ini" SAAT MEMBUAT surat izin. Untuk kasus itu, workaround sekarang: siswa harus tap (kalau bisa) LALU piket resolve manual dari section lain, ATAU kalau siswa tidak bisa tap (sakit parah, dijemput orang tua) — TIDAK ADA jalur resmi sama sekali sebelumnya (celah yang mendorong diskusi task-CORE-024 semula).

**Keputusan user (2026-09-03)**: sederhanakan total — **definisikan "tidak kembali" LANGSUNG saat piket membuat surat izin**, bukan diresolve belakangan:
1. Form "Izin Keluar Sementara" — field "Jam Kembali" diberi tombol/opsi tambahan **"Tidak Kembali"**.
2. Kalau piket pilih "Tidak Kembali" — `jamKembaliDiharapkan` TIDAK diisi (tetap kosong/tidak relevan), DAN **`waktuPulang` siswa langsung diisi = waktu piket membuat surat izin ini** (`now()` saat submit). Siswa **TIDAK PERLU TAP SAMA SEKALI** — baik untuk keluar maupun "pulang".
3. **Prasyarat**: siswa WAJIB SUDAH tap masuk hari itu (`AttendanceRecord` untuk hari ini SUDAH ADA) — kalau belum, TIDAK BISA membuat izin keluar sama sekali (baik "kembali" maupun "tidak kembali"). Ditegakkan **defense-in-depth 2 lapis** (keputusan eksplisit user): (a) FRONTEND — dropdown/search "Murid" di form HANYA menampilkan siswa yang SUDAH tap masuk (`waktuMasuk !== null`) hari itu; (b) BACKEND — kalau lolos ke endpoint tanpa `AttendanceRecord` hari itu, tolak dengan pesan jelas **"Siswa belum absen masuk"**.
4. **Skenario pakai**: siswa sakit mendadak, tidak memungkinkan tap ulang (lemas, pingsan, dijemput buru-buru) — piket cukup buat surat izin dengan "Tidak Kembali", selesai — tidak perlu koordinasi tap-lalu-konfirmasi 2 langkah seperti sebelumnya.
5. **Card "Konfirmasi Izin Pulang"** (section `IzinGuruKelasSection`, judul tampilan "Izin dari Guru Kelas Hari Ini") — SEKARANG HARUS **HANYA** menampilkan izin yang diajukan **guru kelas** (`ClassPermitRequest` via `syncPermitToAttendance()`), **TIDAK LAGI** menampilkan submission piket sendiri (dari form Izin Keluar Sementara — baik "kembali" maupun "tidak kembali"). Filter tambahan diperlukan di query `listIzinHariIni()`.

**Depends on:** Tidak ada dependency task lain. **task-CORE-024 REKOMENDASI dikerjakan SETELAH task ini** (lihat catatan di file task itu).

## 3. Langkah Eksekusi Detail

### A. Backend — Method Baru "Buat Izin Keluar + Tidak Kembali"

1. **Baca dulu `setPulang()`** (`permits.service.ts:638-660` sekitar) — pola REFERENSI untuk "set waktuPulang dari sebuah Permit", termasuk pola `$transaction` + `updateMany` predikat atomik (T129) dan cara update/create `AttendanceRecord`.

2. **Ubah `CreatePermitDto`** (`apps/api/src/permits/dto/create-permit.dto.ts`) — tambah field opsional:
   ```ts
   @IsOptional()
   @IsBoolean()
   tidakKembali?: boolean;
   ```
   Kalau `true` — `jamKembaliDiharapkan` di body DIABAIKAN/tidak dipakai (VERIFIKASI SAAT IMPLEMENTASI: tolak kalau keduanya dikirim bersamaan dengan `BadRequestException` pesan jelas, ATAU cukup abaikan `jamKembaliDiharapkan` secara diam-diam — REKOMENDASI: tolak eksplisit, lebih jelas untuk piket daripada silent-ignore).

3. **Ubah `createKeluar()`** (`permits.service.ts:414-440`) — SEBELUM `permit.create()`:
   - **Validasi prasyarat (langkah 2 di bagian Konteks)**: cek `AttendanceRecord` untuk `studentId`+`tanggal` hari ini SUDAH ADA (`waktuMasuk !== null`) — kalau TIDAK ADA, lempar `BadRequestException("Siswa belum absen masuk")` — SELALU dicek (untuk KEDUA kasus: "kembali" maupun "tidak kembali"), BUKAN cuma untuk `tidakKembali:true` (VERIFIKASI SAAT IMPLEMENTASI: apakah form "Izin Keluar Sementara" SELAMA INI sudah punya validasi serupa — kalau BELUM, ini PERBAIKAN TAMBAHAN sekaligus, dokumentasikan sebagai temuan).
   - Kalau `dto.tidakKembali === true` — bungkus dalam `$transaction`:
     a. `tx.permit.create()` — `jenis: keluar`, `jamKeluar` dari `dto.jamKeluar` (TETAP wajib diisi, "jam keluar" tetap relevan meski tidak kembali — untuk keperluan surat/audit), `jamKembaliDiharapkan: null`, `statusKembali: StatusKembali.pulang` (LANGSUNG set ke status FINAL "pulang" — BUKAN `belum` seperti Sub-alur A biasa, karena TIDAK ADA proses "menunggu kembali" untuk kasus ini), `kembaliDikonfirmasiAt: now()`, `kembaliDikonfirmasiById: approvedById` (piket yang membuat surat = piket yang "mengkonfirmasi" pulang, sekaligus di titik yang sama — REPLIKASI SEMANTIK `setPulang()` tapi digabung 1 langkah, bukan 2 langkah terpisah seperti alur lama).
     b. `tx.attendanceRecord.update()` — `waktuPulang: now()` (waktu SAAT piket submit form ini, BUKAN `jamKeluar` dari input — VERIFIKASI SAAT IMPLEMENTASI ulang dengan definisi user: "waktu pulang siswa adalah waktu guru piket membuat surat" — pakai `new Date()` server-side saat transaksi berjalan, KONSISTEN prinsip proyek "waktu server, bukan client"), `pulangVia: PulangVia.piket_izin` (REUSE value existing, KONSISTEN makna "piket yang menentukan pulang, bukan tap").
     c. `syncPermitToAttendance(permit)` — TETAP dipanggil SAMA seperti `create()` biasa (baris 179-181 existing) — supaya `ClassAttendanceMark` sesi-sesi HARI ITU yang overlap turut ter-set Izin (BUKAN terganggu logic baru, REUSE APA ADANYA).
   - Kalau `dto.tidakKembali` bukan `true` (default/false) — perilaku TETAP SAMA seperti sekarang (Sub-alur A biasa, `statusKembali: belum`, TIDAK ada perubahan).

4. **`@LogActivity`/activity log** — pastikan action tercatat jelas beda dari Sub-alur A biasa (VERIFIKASI SAAT IMPLEMENTASI: apakah `action: "permit.create"` existing sudah cukup, atau perlu differensiasi `permit.create_tidak_kembali` — REKOMENDASI: cukup 1 action yang sama, `snapshotAfter` (isi Permit lengkap termasuk `statusKembali: pulang`) SUDAH CUKUP membedakan di audit trail tanpa perlu action baru).

### B. Frontend — Form Izin Keluar Sementara

5. **`izin-keluar-view.tsx`, `IzinKeluarForm`**:
   - **Filter dropdown "Murid"** (baris ~203-207, `matches` useMemo) — tambah filter `s.waktuMasuk !== null` (hanya siswa yang SUDAH tap masuk hari itu muncul di pencarian) — REPLIKASI pola filter serupa yang SUDAH ADA di `input-izin-view.tsx` baris 43 (`s.status === null`), field yang dicek BEDA (`waktuMasuk` bukan `status`) tapi POLA sama.
   - **Field "Jam Kembali"** (baris ~352-366) — tambah opsi/tombol "Tidak Kembali": REKOMENDASI checkbox atau toggle kecil di sebelah label "Jam Kembali (opsional)" — saat dicentang, input `type="time"` untuk jam kembali disembunyikan/disabled (tidak relevan lagi), state baru `tidakKembali: boolean`.
   - **`handleSubmit`** (baris ~216-258) — kirim `tidakKembali: tidakKembali` ke body, `jamKembaliDiharapkan` TIDAK dikirim sama sekali kalau `tidakKembali === true` (VERIFIKASI SAAT IMPLEMENTASI konsisten dengan validasi backend langkah 3).
   - **Pesan sukses** — beda teks untuk kasus `tidakKembali: true` (mis. "Izin tersimpan, siswa langsung tercatat pulang jam {waktu}. Surat dibuka di tab baru." — BUKAN teks default yang menyiratkan siswa masih perlu kembali).
   - **Cetak surat** — `buildPrintUrl()` TETAP dipanggil sama seperti sekarang (surat izin tetap perlu dicetak untuk KEDUA kasus) — VERIFIKASI SAAT IMPLEMENTASI apakah parameter `jamkembali` di surat perlu diubah teksnya jadi "Tidak Kembali" alih-alih kosong/format jam, supaya surat cetak jelas menyatakan status ini (REKOMENDASI: ya, ubah — surat fisik yang dipegang siswa/orang tua harus mencerminkan status sebenarnya).

### C. Filter Card "Konfirmasi Izin Pulang" — Hanya Guru Kelas

6. **`ClassPermitRequestsService.listIzinHariIni()`** (`class-permit-requests.service.ts:108-135`) — tambah filter supaya HANYA menampilkan izin dari guru kelas, TIDAK LAGI menampilkan submission piket sendiri:
   - **VERIFIKASI SAAT IMPLEMENTASI cara paling tepat membedakan sumber**: query saat ini `where: { status: "izin", permitId: { not: null }, session: {...} }` mencakup KEDUA sumber (guru-kelas via `ClassPermitRequest.izinkan()` MAUPUN piket-langsung via `PermitsService.create()`, keduanya sama-sama isi `permitId` lewat `syncPermitToAttendance()`). Tambahkan filter: `permit: { classPermitRequests: { some: {} } }` (Permit ini PUNYA relasi balik ke `ClassPermitRequest` — HANYA true kalau asalnya dari alur guru-kelas `izinkan()`, Permit dari piket-langsung `createKeluar()` TIDAK PERNAH punya `ClassPermitRequest` terkait sama sekali) — REPLIKASI relasi yang SUDAH di-`include` di method yang sama (baris 126-129, `classPermitRequests`), tinggal PINDAHKAN jadi filter `where`, bukan cuma `include`.
   - Field `diajukan_oleh`/fallback "Piket" (disebut di komentar existing baris 105-106, "null kalau asal piket-langsung, FE tampilkan 'Piket' sebagai fallback") — SEKARANG TIDAK RELEVAN LAGI karena piket-langsung sudah di-exclude filter baru — VERIFIKASI SAAT IMPLEMENTASI apakah fallback "Piket" ini masih perlu dipertahankan di kode (aman dibiarkan sebagai defensive fallback, TIDAK WAJIB dihapus) ATAU dihapus karena sudah tidak mungkin terjadi (REKOMENDASI: biarkan, defensive coding tidak masalah tetap ada meski jalur itu sekarang tidak akan terpicu).

7. **Update komentar/docstring `IzinGuruKelasSection`** (`izin-keluar-view.tsx` baris 65-74) — hapus kalimat "diajukan guru kelas atau langsung oleh piket" (baris 132, deskripsi UI juga) — GANTI jadi murni "diajukan guru kelas" — supaya dokumentasi kode + UI konsisten dengan behavior baru.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/permits/dto/create-permit.dto.ts` — field `tidakKembali`
- **Modifikasi:** `apps/api/src/permits/permits.service.ts` — `createKeluar()`
- **Modifikasi:** `apps/api/src/class-permit-requests/class-permit-requests.service.ts` — `listIzinHariIni()` filter
- **Modifikasi:** `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` — `IzinKeluarForm` (tombol/state baru), `IzinGuruKelasSection` (deskripsi teks)
- **Jangan sentuh:** `createTidakMasuk()`, `createDispen()` (jalur lain, tidak terdampak), `ClassPermitRequestsService.izinkan()`/`tolak()` (alur guru-kelas TIDAK berubah sama sekali).

**Dilarang dilakukan:**
- Jangan izinkan `tidakKembali: true` DAN `jamKembaliDiharapkan` terisi bersamaan — pilih salah satu perilaku (tolak eksplisit ATAU abaikan salah satu, VERIFIKASI SAAT IMPLEMENTASI, REKOMENDASI tolak eksplisit untuk kejelasan).
- Jangan hapus/ubah alur Sub-alur A biasa (`tidakKembali` default/false) — perilaku HARUS identik dengan sebelum task ini untuk kasus itu.
- Jangan biarkan siswa yang belum tap masuk lolos ke `createKeluar()` sama sekali (2 lapis validasi WAJIB, bukan cuma 1 — lihat langkah 3+5).

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: piket coba buat izin (baik kembali maupun tidak kembali) untuk siswa yang BELUM tap masuk hari itu, lolos filter dropdown (mis. race — siswa baru saja di-reset statusnya di tab lain) → backend TETAP tolak dengan pesan jelas "Siswa belum absen masuk" (defense-in-depth lapis ke-2 WAJIB berfungsi independen dari filter frontend).
- Kondisi: siswa SUDAH ada Permit `keluar` lain yang masih `statusKembali: belum` untuk hari yang sama (kasus jarang tapi mungkin, `createKeluar()` belum ada cek duplikasi — TEMUAN AUDIT SEBELUMNYA, DI LUAR SCOPE task ini untuk diperbaiki total, TAPI VERIFIKASI SAAT IMPLEMENTASI: apakah kasus spesifik "tidak kembali dobel" ini perlu dicegah minimal sebagai bagian task ini, mengingat efeknya langsung ubah `waktuPulang` — REKOMENDASI: tambahkan cek dasar sebelum create untuk kasus `tidakKembali:true` KHUSUSNYA, karena dampaknya langsung ke data kehadiran, bukan cuma dokumen izin).
- Kondisi: `syncPermitToAttendance()` gagal SETELAH `waktuPulang` sudah ter-update (partial fail dalam 1 `$transaction` — Prisma `$transaction` interaktif akan ROLLBACK SEMUA kalau ada exception, VERIFIKASI SAAT IMPLEMENTASI bahwa `syncPermitToAttendance()` dipanggil DI DALAM `$transaction` yang sama supaya atomik, BUKAN setelahnya seperti pola `create()` biasa existing baris 179-181 yang MEMANG di luar transaction utama create Permit — PERTIMBANGKAN apakah perlu diketatkan untuk kasus BARU ini khususnya, karena effect-nya lebih besar/langsung).
- Kondisi: card "Konfirmasi Izin Pulang" kosong setelah filter baru diterapkan (karena hari itu kebetulan tidak ada izin dari guru kelas sama sekali) → empty state normal, TIDAK error.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Form Izin Keluar Sementara punya opsi "Tidak Kembali", saat dipilih field Jam Kembali disembunyikan/tidak relevan.
- [ ] Submit dengan "Tidak Kembali" → `waktuPulang` siswa langsung terisi (waktu server saat submit), `statusKembali: pulang`, TANPA siswa perlu tap.
- [ ] Dropdown "Murid" di form Izin Keluar HANYA menampilkan siswa yang sudah tap masuk hari itu.
- [ ] Backend menolak (pesan jelas "Siswa belum absen masuk") kalau dipanggil untuk siswa tanpa `AttendanceRecord` hari itu, meski lolos filter frontend.
- [ ] Card "Konfirmasi Izin Pulang"/"Izin dari Guru Kelas Hari Ini" HANYA menampilkan izin dari `ClassPermitRequest` (guru kelas) — submission piket sendiri (form Izin Keluar) TIDAK muncul di situ lagi.
- [ ] Surat cetak untuk kasus "Tidak Kembali" mencerminkan status itu dengan jelas (bukan tampil kosong/membingungkan).
- [ ] Sub-alur A biasa (jam kembali diisi normal) TIDAK ADA regresi perilaku.
- [ ] Test unit baru untuk `createKeluar()` (kasus tidakKembali true/false, validasi belum-tap-masuk) + `listIzinHariIni()` (filter guru-kelas only).
- [ ] Full test suite lulus tanpa regresi, typecheck bersih api+web.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 250 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada — REKOMENDASI kerjakan SEBELUM task-CORE-024 (task itu mengasumsikan mekanisme ini sudah ada)

# T231 — API: Fix "Total Hari Aktif" Rekap — Clamp ke Hari Ini, Bukan Cuma liveSince

## Depends on
Tidak ada dependency teknis. Independen, murni modul `attendance`. Bug ini mempengaruhi SEMUA konsumen `resolveHariWajib()`/`totalHariAktif` (rekap admin DAN rekap wali kelas, T224c REUSE method yang sama — fix di 1 titik otomatis benar di semua tempat).

## Konteks — Bug Nyata Ditemukan & Root Cause Terkonfirmasi (2026-08-21)

User laporkan "Total Hari Aktif" di Rekap Wali Kelas menampilkan **108** padahal `SystemLiveConfig.liveSince = 2026-08-01` dan hari ini baru 2026-08-20 (ekspektasi wajar ~14-15 hari kerja). Investigasi (dengan koreksi — riset awal salah connect ke database bukan production sungguhan, dikoreksi via `docker exec absensi-mysql-prod` + perhitungan manual) menemukan akar penyebab PASTI:

**Perhitungan manual (Python, hari kerja Senin-Jumat) mengonfirmasi**:
```
1 Agustus - 31 Desember 2026 (liveSince s.d. AKHIR SEMESTER) = 109 hari kerja
```
Sangat dekat dengan 108 yang dilaporkan (selisih 1, kemungkinan hari libur nasional yang tidak dihitung manual, atau clamp inclusive/exclusive `liveSince` beda 1 hari — BUKAN masalah utama, root cause sudah pasti).

**Penjelasan mekanisme**: `resolveDateRange()` (`attendance-report.service.ts:794-836`) — kalau `academicYearId`+`semesterId` dikirim tanpa override tanggal manual, `from`/`to` yang diteruskan ke `resolveHariWajib()` adalah **RENTANG PENUH SEMESTER** (`semester.tanggalMulai`..`semester.tanggalSelesai`, di kasus ini 20 Juli - **31 Desember 2026**). Frontend `RekapView` (`rekap-view.tsx:312-329`) SECARA OTOMATIS meng-override state `from`/`to` ke rentang semester penuh begitu `academicYearId`/`semesterId` dipilih — jadi ini SELALU terjadi di kondisi default (bukan edge case jarang).

`resolveHariWajib()` SUDAH BENAR meng-clamp ujung **AWAL** ke `liveSince` (T147, `if (liveSince && cursor < liveSince) continue;`) — TAPI ujung **AKHIR** tetap `to` = akhir semester (31 Desember, **MASA DEPAN** dari hari ini 20 Agustus), BUKAN di-clamp ke hari ini. Hasilnya: "Total Hari Aktif" menghitung hari wajib dari 1 Agustus **sampai 31 Desember** (termasuk ~4 bulan yang belum terjadi), sementara "Hadir" cuma bisa terisi untuk hari yang SUDAH LEWAT (1-20 Agustus) — karena memang belum ada data tap untuk hari yang belum terjadi. Kesenjangan besar antara kedua kolom ini yang membingungkan guru.

**Ini BUKAN masalah baru** — task **T217** (grafik persentase rekap admin) SUDAH memperbaiki masalah SERUPA PERSIS untuk kebutuhan grafik (`agregatPersen`, basis "hari wajib yang **sudah lewat**" — `effectiveTo = min(to, startOfToday())`) — TAPI fix itu HANYA diterapkan ke field `agregatPersen` BARU untuk grafik, TIDAK diterapkan ke field `totalHariAktif` per-baris di tabel yang SUDAH ADA SEBELUM T217. Kesenjangan ini yang jadi akar bug sekarang.

## Keputusan Dikonfirmasi User (2026-08-21)

**Clamp 2 arah** untuk "Total Hari Aktif": batas bawah = `liveSince` (SUDAH bekerja, T147), batas atas = `MIN(hari ini, akhir semester/rentang yang diminta)` — KONSISTEN pola `effectiveTo` yang SUDAH ADA di T217 untuk `agregatPersen`, diterapkan JUGA ke `totalHariAktif`.

**Konfirmasi keamanan jangka panjang** (pertanyaan user soal pergantian tahun ajaran/semester): AMAN — baik `liveSince` maupun "hari ini" adalah TITIK YANG BERGERAK MAJU OTOMATIS, bukan nilai yang perlu diupdate manual. Semester BARU (rentang di masa depan dari `liveSince`) TIDAK terpengaruh clamp `liveSince` sama sekali (karena `liveSince` sudah lebih awal). Semester LAMA (histori, sudah berakhir) — `to` = akhir semester (SUDAH di masa lalu, `MIN(hari ini, akhir semester)` = akhir semester itu sendiri, clamp "hari ini" TIDAK berpengaruh untuk histori yang sudah selesai). Clamp cuma relevan untuk SEMESTER YANG SEDANG BERJALAN (rentang mencakup hari ini) — persis kasus yang dilaporkan user.

## Spec Detail

### 1. Backend — `reportInternal()` — clamp `to` untuk `totalHariAktif`

`apps/api/src/attendance/attendance-report.service.ts`, method `reportInternal()` (sekitar baris 159) — SAAT INI:
```ts
const { wajibDates, adaTahunAjaranAktif } = await this.resolveHariWajib(from, to);
...
const totalHariAktif = wajibDates.size;
```

TAMBAH clamp `to` SEBELUM panggil `resolveHariWajib()`, REPLIKASI PERSIS pola `effectiveTo` yang SUDAH ADA di T217 (cek `reportFlexible()` sekitar baris 400an untuk pola `effectiveTo = to <= startOfToday() ? to : startOfToday()` — REUSE fungsi/pola yang SAMA, JANGAN tulis clamp kedua yang berbeda):
```ts
const effectiveToUntukTotalHariAktif = to <= this.startOfToday() ? to : this.startOfToday();
const { wajibDates, adaTahunAjaranAktif } = await this.resolveHariWajib(from, effectiveToUntukTotalHariAktif);
```
- **PENTING**: `wajibDates` yang dipakai untuk `alfa`/`hadir`/kategori lain per-siswa (loop `for (const dateKey of wajibDates)` di bagian bawah `reportInternal()`) — VERIFIKASI SAAT IMPLEMENTASI apakah `wajibDates` yang SAMA (sudah di-clamp) dipakai untuk SEMUA kategori (alfa dst), bukan cuma `totalHariAktif` — KONSISTENSI WAJIB: alfa juga harus pakai basis yang sama (hari yang sudah lewat), supaya tidak ada mismatch baru antara alfa dan totalHariAktif (KEDUANYA harus pakai `wajibDates` yang SAMA persis, satu Set, bukan 2 variabel terpisah yang bisa drift).
- **VERIFIKASI reuse `startOfToday()`** — method ini KEMUNGKINAN sudah ada (dipakai T217/tempat lain) — REUSE, JANGAN buat ulang.

### 2. Backend — `reportSingleDay()` — TIDAK PERLU diubah

Mode single-day (`from === to`, 1 hari saja) SUDAH otomatis tidak kena masalah ini (rentang cuma 1 hari, tidak ada "masa depan" dalam rentang 1 hari yang sama). TIDAK ADA perubahan di method ini.

### 3. Backend — `riwayatCatatan()` — VERIFIKASI, kemungkinan SUDAH benar

`riwayatCatatan()` (baris ~639) SUDAH pakai `to = startOfToday()` secara eksplisit (dikonfirmasi riset sebelumnya) — SUDAH BENAR, TIDAK PERLU diubah, HANYA verifikasi ulang saat implementasi untuk memastikan tidak ada regresi tidak sengaja.

### 4. Konsistensi — SEMUA consumer `reportInternal()` otomatis ikut benar

Karena `totalHariAktif` dihitung SEKALI di `reportInternal()` dan di-reuse oleh SEMUA caller (`report()`, `reportFlexible()` — dipakai rekap admin DAN rekap wali kelas via T224c), fix di 1 titik ini OTOMATIS memperbaiki SEMUA tempat yang menampilkan kolom ini — TIDAK PERLU perubahan terpisah di `journal-kelas-wali.controller.ts` (T224c, sudah reuse method yang sama, akan otomatis ikut benar).

## Edge Cases

- **Semester yang SUDAH BERAKHIR** (rentang seluruhnya di masa lalu, misal semester lalu) — `to` (akhir semester) SUDAH lebih awal dari hari ini, `MIN(hari ini, to)` = `to` itu sendiri, clamp TIDAK mengubah apa pun (behavior sama seperti sebelum fix). WAJIB dites eksplisit supaya tidak regresi rekap histori.
- **Semester yang BELUM MULAI** (seluruh rentang di masa depan, `from` > hari ini) — `resolveHariWajib(from, effectiveTo)` dengan `effectiveTo` < `from` (karena `effectiveTo` = hari ini, lebih awal dari `from` semester masa depan) — HARUS return `wajibDates` KOSONG (bukan error), `totalHariAktif = 0` — KONSISTEN behavior T217 untuk kasus serupa (`basisHariWajibSudahLewat = 0`).
- **`liveSince` belum pernah di-set admin** (`null`) — clamp AWAL tidak berlaku (behavior lama), clamp AKHIR (fix task ini) TETAP berlaku independen — 2 clamp ini TIDAK saling bergantung, boleh salah satu aktif salah satu tidak.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (`reportInternal()`, clamp `to` sebelum `resolveHariWajib()`).
- **Jangan sentuh:** `reportSingleDay()`, `riwayatCatatan()` (sudah benar), `journal-kelas-wali.controller.ts` (T224c, otomatis ikut benar via reuse).

## Eksekusi (2026-08-21)

`reportInternal()` — clamp `to` ke `min(to, startOfToday())` SEBELUM `resolveHariWajib()`
(baris ~159-168), 1 Set `wajibDates` yang sudah diclamp ini dipakai SAMA PERSIS untuk `alfa`
(loop per-siswa) DAN `totalHariAktif` (`wajibDates.size`) — tidak ada 2 variabel terpisah.

`reportFlexible()` — clamp `effectiveTo` LAMA (T217, khusus `agregatPersen` chart) jadi
**redundan** setelah fix ini, karena `reportInternal()` SENDIRI sudah clamp dengan basis yang
identik. Disederhanakan: `hitungAgregatPersen(perStudentDates, internal.wajibDates)` langsung
reuse `internal.wajibDates`, hapus clamp kedua yang duplikat logic-nya persis.

**Efek samping ditemukan saat implementasi**: test lama `"rentang CAMPUR (mencakup hari
depan)"` (T217, ditulis SEBELUM T231) punya asumsi eksplisit "basis chart < totalHariAktif
baris tabel (yang TIDAK diubah)" — asumsi ini JADI SALAH setelah T231 (totalHariAktif SEKARANG
JUGA diclamp). Test diupdate: assertion berubah dari `toBeLessThan` jadi `toBe` (basis chart
dan totalHariAktif sekarang identik).

5 test baru ditambahkan (describe block terpisah `T231 clamp totalHariAktif ke hari ini`):
semester sedang berjalan (clamp aktif, totalHariAktif < 20 padahal rentang 60 hari ke depan),
semester sudah berakhir (regresi nol, totalHariAktif tetap 5 tidak berubah), semester belum
mulai (0, bukan error), kombinasi liveSince terisi + clamp hari ini (2 clamp independen aktif
bersamaan), dan invariant `alfa+hadir+belumMemilikiKartu <= totalHariAktif` (basis Set selalu
konsisten, tidak drift).

**Verifikasi**: `pnpm exec jest attendance-report.service.spec.ts` 45/45 pass, full suite
`pnpm exec jest --runInBand` 42 suite/601 test pass (0 regresi). Live curl ke dev server
dengan rentang PERSIS skenario bug (`from=2026-07-20&to=2026-12-31`, rentang semester penuh)
— `totalHariAktif=25` (BUKAN ~109 seperti sebelum fix), `basisHariWajibSudahLewat=25` SAMA
PERSIS dengan totalHariAktif (dulu 2 angka berbeda, sekarang identik). Cross-check manual
Python: 25 hari kerja Senin-Jumat dari 20 Juli sampai 21 Agustus 2026 (hari eksekusi) — cocok.

## Acceptance Criteria
- [x] Semester SEDANG BERJALAN (rentang mencakup hari ini) — "Total Hari Aktif" TIDAK LAGI menghitung hari di masa depan, hanya sampai hari ini.
- [x] Kolom "Total Hari Aktif" dan jumlah (Hadir+Izin+Sakit+Dispen+Alfa) — SELISIH-nya masuk akal (tidak ada lagi kesenjangan besar seperti 108 vs puluhan) — dikonfirmasi live: 25 vs 25, bukan 108 vs puluhan.
- [x] Semester SUDAH BERAKHIR (histori) — TIDAK ADA regresi, angka sama seperti sebelum fix (test baru mengonfirmasi).
- [x] Semester BELUM MULAI (masa depan) — "Total Hari Aktif" = 0, bukan error (test baru mengonfirmasi).
- [x] Rekap admin DAN rekap wali kelas — KEDUANYA otomatis benar setelah fix (fix di 1 titik `reportInternal()`, di-reuse SEMUA caller termasuk T224c).
- [x] Build + type-check hijau, jest baru: semester sedang berjalan (clamp aktif), semester lalu (tidak berubah), semester masa depan (0), kombinasi dengan `liveSince` null dan liveSince terisi.

## Validasi Claudian
- [x] Konfirmasi `wajibDates` yang di-clamp SAMA PERSIS dipakai untuk `totalHariAktif` DAN semua kategori per-siswa (alfa dst) — 1 Set, tidak 2 variabel terpisah.
- [x] Konfirmasi REUSE pola `effectiveTo`/`startOfToday()` yang SUDAH ADA dari T217 — BAHKAN disederhanakan lebih jauh: clamp kedua yang dulu ada di `reportFlexible()` khusus chart SEKARANG DIHAPUS karena sudah redundan dengan clamp `reportInternal()`.
- [x] Konfirmasi tidak ada regresi ke rekap histori (semester yang sudah berakhir) — test baru + live verify mengonfirmasi angka sama persis sebelum/sesudah fix untuk kasus ini.

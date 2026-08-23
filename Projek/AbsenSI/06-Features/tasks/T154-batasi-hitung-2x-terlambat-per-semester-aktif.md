# T154 — Batasi Hitung "2x Terlambat" (Lock Otomatis) ke Semester Aktif, Bukan All-Time

## Depends on
**WAJIB T139 (tag academicYearId/semesterId di AttendanceRecord) selesai dan diverifikasi dulu** — task ini butuh kolom `semesterId` di `AttendanceRecord` untuk memfilter query hitung strike per semester. JANGAN kerjakan sebelum T139 selesai. Independen dari T153 (bisa dikerjakan sebelum/sesudah/paralel, tidak saling bergantung secara teknis, TAPI KEDUANYA menyentuh method yang sama `applyLateStrikeLock()` — KALAU T153 sudah dikerjakan lebih dulu, WAJIB baca hasil implementasinya dulu supaya tidak konflik/menimpa perbaikan atomicity yang sudah ada).

## Objective
Hitung "2x terlambat" (pemicu lock otomatis siswa) **DIBATASI ke SEMESTER AKTIF SAAT INI** — bukan lagi seluruh riwayat sejak kapan pun. Siswa yang terlambat 1x di semester lalu lalu terlambat lagi di semester ini TIDAK otomatis dianggap "2x", karena periode hitungnya sudah berbeda.

## Context — Kenapa Task Ini Ada (Ditemukan 2026-08-10)

Investigasi insiden "20 siswa terkunci bersamaan, toggle sudah dimatikan tapi tetap banyak yang terkunci" (bukan bug toggle — expected behavior, lihat T153 untuk bug TERPISAH yang ditemukan di insiden yang sama) — menemukan AKAR kenapa lock massal itu bisa terjadi bersamaan:

`applyLateStrikeLock()` (`apps/api/src/attendance/attendance.service.ts:272-298`) menghitung jumlah `AttendanceRecord.status = "terlambat"` untuk seorang siswa dengan rentang:
```ts
const resetDateFloor = student.lateStrikeResetAt ? this.startOfDay(student.lateStrikeResetAt) : null;
const lateCount = await this.prisma.attendanceRecord.count({
  where: {
    studentId,
    status: AttendanceStatus.terlambat,
    tanggal: resetDateFloor ? { gte: resetDateFloor } : undefined,
  },
});
```
Kalau `student.lateStrikeResetAt` masih `null` (siswa BELUM PERNAH di-unlock sebelumnya) — **TIDAK ADA BATASAN TANGGAL SAMA SEKALI** — dihitung dari SELURUH histori siswa itu, bisa mencakup semester/tahun ajaran sebelumnya. `lateStrikeResetAt` hanya di-set saat UNLOCK MANUAL terjadi (bukan otomatis per pergantian semester) — jadi siswa yang belum pernah kena lock sebelumnya, "counter"-nya efektif berjalan TANPA BATAS WAKTU sampai pertama kali dia di-unlock.

**Ini yang menyebabkan lock massal bisa terjadi tiba-tiba**: banyak siswa yang masing-masing punya 1 catatan terlambat LAMA (misal dari semester/bulan sebelumnya) — begitu MEREKA SEMUA kebetulan terlambat LAGI di hari yang sama (strike ke-2, all-time), SEMUANYA otomatis terkunci BERSAMAAN di hari itu — bukan karena ada yang salah secara teknis, tapi karena definisi "2x" ini terlalu longgar rentang waktunya, membuat efek "ledakan lock" yang tidak terduga admin.

**Keputusan final** (dikonfirmasi user 2026-08-10): definisi "2x terlambat" untuk pemicu lock HARUS dibatasi ke **SEMESTER AKTIF SAAT INI** — konsisten dengan struktur akademik yang sudah ada di sistem (`AcademicYear`/`Semester`, dan pekerjaan T139 yang menambahkan tag `semesterId` eksplisit ke `AttendanceRecord`).

## Spec Detail

### 1. Ganti sumber batas rentang — dari `lateStrikeResetAt` murni ke `semesterId` yang sedang aktif

- `applyLateStrikeLock()` — GANTI logic filter tanggal, jadi filter berdasarkan **`AttendanceRecord.semesterId`** (kolom yang SUDAH ADA dari T139) **SAMA DENGAN** semester yang `isActive: true` SAAT INI (bukan lagi berdasarkan `lateStrikeResetAt`/rentang tanggal manual):
  ```ts
  const activePeriod = await this.academicPeriod.getActive(); // helper T139, sudah ada
  const lateCount = await this.prisma.attendanceRecord.count({
    where: {
      studentId,
      status: AttendanceStatus.terlambat,
      semesterId: activePeriod.semesterId, // BARU — filter by semester aktif, BUKAN tanggal
    },
  });
  if (lateCount < 2) return false;
  ```
- **Kalau `activePeriod.semesterId` adalah `null`** (tidak ada semester aktif SAAT INI — kondisi yang SAMA dikonfirmasi T144 pernah terjadi di production) — JANGAN biarkan `lateCount` menghitung SEMUA data tanpa filter (itu balik ke bug lama) — sebaliknya, kalau tidak ada semester aktif, `applyLateStrikeLock()` HARUS return `false` LANGSUNG (tidak pernah mengunci siswa) — KONSISTEN dengan filosofi T144/T147 (kalau syarat periode tidak terpenuhi, jangan diam-diam salah hitung, mending tidak beraksi sama sekali).

### 2. Field `lateStrikeResetAt` — evaluasi apakah masih dibutuhkan

- Field ini SEBELUMNYA berfungsi ganda: (a) sebagai floor tanggal untuk hitung count, (b) sebagai penanda "siswa ini pernah di-unlock, jadi mulai hitung ulang dari titik ini". Dengan perubahan poin 1, fungsi (a) SUDAH DIGANTIKAN oleh filter `semesterId`. TAPI fungsi (b) MASIH RELEVAN untuk kasus: siswa di-unlock DI TENGAH semester yang sama (misal orang tua sudah datang bicara ke piket) — kalau TIDAK ADA reset apa pun, siswa itu masih akan kena `lateCount >= 2` LAGI di semester yang sama (karena 2 catatan terlambat lama itu tetap ada, cuma statusnya sekarang unlocked). PUTUSKAN saat implementasi:
  - **Opsi A (direkomendasikan)**: PERTAHANKAN `lateStrikeResetAt` SEBAGAI TAMBAHAN filter (bukan pengganti) — kombinasikan DENGAN `semesterId`: `tanggal: resetDateFloor ? { gte: resetDateFloor } : undefined` DITAMBAH `semesterId: activePeriod.semesterId` — supaya SETELAH unlock manual, siswa itu benar-benar "clean slate" DALAM semester yang sama, TIDAK otomatis kena lagi dari 2 catatan lama yang sudah "dimaafkan" piket.
  - **Opsi B**: hapus total ketergantungan pada `lateStrikeResetAt`, biarkan reset OTOMATIS terjadi tiap pergantian semester (SEMUA siswa "clean slate" tiap semester baru, tanpa perlu unlock manual eksplisit) — TAPI ini menghilangkan kemampuan "reset di tengah semester" yang MUNGKIN masih dibutuhkan (misal kasus khusus orang tua sudah konfirmasi, piket ingin kasih kesempatan lagi TANPA menunggu semester ganti). REKOMENDASI KUAT: pilih Opsi A, JANGAN Opsi B, kecuali user secara eksplisit mengonfirmasi ingin menghapus kemampuan reset-di-tengah-semester (KLARIFIKASI KE USER kalau ragu, JANGAN putuskan sepihak menghapus kapabilitas existing).

### 3. Endpoint `unlock()` — evaluasi apakah perlu penyesuaian (kemungkinan TIDAK, tapi verifikasi)

- `StudentsService.unlock()` (`apps/api/src/core/students/students.service.ts`) yang men-set `lateStrikeResetAt` saat piket melakukan unlock manual — KEMUNGKINAN BESAR TIDAK PERLU DIUBAH (perilaku "set `lateStrikeResetAt` ke waktu unlock" tetap masuk akal dengan Opsi A di atas) — TAPI VERIFIKASI saat implementasi bahwa kombinasi filter baru (poin 1+2 Opsi A) benar-benar menghasilkan perilaku yang diinginkan: siswa yang di-unlock DALAM semester yang sama tidak langsung kena lock lagi dari sisa catatan lama DALAM semester itu.

## Edge Cases
- Siswa terlambat 1x di semester GANJIL, lalu terlambat lagi di semester GENAP (semester sudah ganti, tapi ini bukan pergantian TAHUN AJARAN) — dengan filter `semesterId`, INI DIANGGAP "1x" di semester baru (BENAR, sesuai keputusan final — reset per semester, bukan per tahun ajaran).
- Semester AKTIF berganti DI TENGAH HARI (kasus sangat jarang, misal admin mengaktifkan semester baru siang hari) — siswa yang tap pagi (semester lama, terlambat) lalu tap lagi sore (semester baru sudah aktif) — count akan terpecah otomatis sesuai `semesterId` yang tercatat di masing-masing `AttendanceRecord` (konsisten dengan T139, TIDAK PERLU logic tambahan, murni ikut apa yang sudah tercatat).
- Siswa yang SUDAH terlanjur locked SEBELUM task ini di-deploy (data lama, `lateStrikeResetAt` mungkin `null`) — task ini TIDAK OTOMATIS meng-unlock siapa pun (perubahan ini hanya mempengaruhi PERHITUNGAN untuk lock BARU ke depan, bukan status lock yang sudah ada) — konsisten dengan filosofi toggle T112 (perubahan config tidak retroaktif membuka yang sudah terkunci).

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (`applyLateStrikeLock()`, ganti sumber filter rentang).
- **Jangan sentuh:** `StudentsService.unlock()` KECUALI verifikasi di poin 3 menemukan perlu penyesuaian kecil (evaluasi dulu, jangan asumsi perlu diubah), toggle `AttendanceLockConfig` (di luar scope, sudah benar).

## Acceptance Criteria
- [x] Siswa terlambat 1x di semester SEBELUMNYA + 1x di semester AKTIF SAAT INI → TIDAK dianggap "2x", TIDAK terkunci (regresi dari bug lama, ini yang diperbaiki).
- [x] Siswa terlambat 2x DALAM semester aktif yang SAMA → TETAP terkunci seperti sebelumnya (perilaku inti tidak berubah, cuma rentang hitungnya yang dipersempit).
- [x] Siswa yang di-unlock manual DI TENGAH semester aktif → tidak langsung kena lock lagi dari sisa catatan lama DALAM semester yang sama (Opsi A, `lateStrikeResetAt` tetap jadi filter tambahan).
- [x] Kondisi TIDAK ADA semester aktif sama sekali → `applyLateStrikeLock()` TIDAK PERNAH mengunci siapa pun (fail-safe, bukan fallback ke hitung tanpa batas seperti bug lama).
- [x] Build + type-check `apps/api` hijau. Test suite existing lulus 100% (TAMBAH test baru untuk skenario "terlambat lintas semester tidak dianggap 2x").

## Validasi Claudian
- [x] **JANGAN** menghapus kapabilitas reset-di-tengah-semester (`lateStrikeResetAt`) tanpa konfirmasi eksplisit user — Opsi A dipilih (dikombinasikan dengan filter semesterId, TIDAK diganti total).
- [x] **JANGAN** kerjakan sebelum T139 selesai — sudah selesai (sesi-sesi sebelumnya), kolom `semesterId` sudah ada dan ditandai di `AttendanceRecord`.
- [x] **CEK DULU apakah T153 sudah dikerjakan** — SUDAH (dikerjakan tepat sebelum T154 di sesi yang sama), `applyLateStrikeLock()` sudah dibungkus `$transaction`. Filter `semesterId` ditambahkan DI DALAM query `count()` yang sama, TIDAK menyentuh struktur `$transaction`/`activityLog.record(tx)` dari T153 — kedua perbaikan hidup berdampingan tanpa konflik.
- [x] Tidak ada ambiguitas Opsi A vs B yang perlu diverifikasi ke user — spec sudah rekomendasi kuat Opsi A dan tidak ada indikasi user ingin B.

## Status Eksekusi (2026-08-14)

**Selesai.** Digabung dengan hasil T153 di method yang sama tanpa konflik.

- `apps/api/src/attendance/attendance.service.ts` — `applyLateStrikeLock()`: tambah `const activePeriod = await this.academicPeriod.getActive();` lalu `if (activePeriod.semesterId === null) return false;` (fail-safe) SEBELUM query count. Query `attendanceRecord.count()` ditambah `semesterId: activePeriod.semesterId` di `where`, DIKOMBINASIKAN (bukan menggantikan) dengan filter `tanggal: resetDateFloor ? {gte} : undefined` yang sudah ada (Opsi A) — supaya unlock manual di tengah semester tetap efektif "clean slate". `StudentsService.unlock()` TIDAK diubah (sudah benar, set `lateStrikeResetAt` saat unlock otomatis, cukup dikombinasikan dengan filter baru).
- `apps/api/src/attendance/attendance.service.spec.ts` — 4 test baru (`applyLateStrikeLock` diakses via bracket-notation karena private): (1) tidak ada semester aktif → false, count tidak dipanggil; (2) terlambat lintas semester → count dipanggil dengan `semesterId` di where (regresi utama yang diperbaiki); (3) 2x dalam semester aktif sama → tetap lock, transaksi dipanggil; (4) `lateStrikeResetAt` tetap dikombinasikan dengan `semesterId` di where (Opsi A).

**Verifikasi live** (dev DB port 3307, API dev port 3101, production tidak disentuh):
1. Setup: 1 semester LAMA (`isActive: false`) + 1 semester AKTIF, 1 catatan terlambat ditag ke semester lama, tap hari ini dipaksa terlambat (via `client_timestamp`) → response TIDAK ada `justLocked`, `Student.lockedAt` tetap NULL — walau ini "2x terlambat all-time", TIDAK dianggap 2x karena beda semester.
2. Ganti catatan lama jadi tag semester AKTIF (simulasi 2 kali terlambat DALAM semester yang sama) → tap kedua `justLocked: true`, siswa terkunci normal seperti sebelumnya.
3. Data uji (attendance_records, activity_log, lock state siswa test, 2 semester test, schedule jam_sekolah sementara, `kiosks.allowed_ip` override) dibersihkan, dikonfirmasi via re-query — dev DB kembali ke state semula (0 semester, sesuai temuan stray state yang sudah dilaporkan di T153).
4. `tsc --noEmit` bersih, `jest` — 22 suite / 277 test lulus 100% (4 test baru T154 + semua test lama termasuk T153 tetap hijau).

# T170 — API+Web: Halaman "Presensi" Guru — Kalender Per Kelas + Riwayat Read-Only

## Depends on
**WAJIB setelah T168** (shell mobile-app). Independen dari T169/T171 (bisa dikerjakan paralel/urutan bebas di antara ketiganya setelah T168 selesai).

## Objective
Halaman baru `/guru/presensi` — guru pilih salah satu kelas yang diajar, lihat **kalender bulan** dengan tanda hari yang sudah ada presensinya, klik tanggal tertentu → lihat **daftar nama siswa + status** (read-only murni, TIDAK BISA diedit) untuk sesi di tanggal itu.

## Context — Temuan Riset (2026-08-13)

- **Model presensi kelas SUDAH ADA**: `ClassAttendanceMark` (`schema.prisma:510-527`) — `sessionId` (FK `TeachingSession`), `studentId`, `status` (enum `hadir | tidak_ada_di_kelas`), `markedById`, `markedAt`. Ini DIISI guru mapel via `attendance-table.tsx` yang SUDAH ADA (`apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx`, PATCH `/teaching-sessions/:id/attendance` per siswa). Task ini TIDAK mengubah cara isi presensi — MURNI menambah cara MELIHAT riwayatnya per kelas per bulan.
- **Kalender bulan bertanda hari-ada-presensi TIDAK ADA sama sekali** — bukan endpoint, bukan UI. Perlu dibangun dari nol.
- **Endpoint terdekat**: `GET /teaching-sessions` (`teaching-sessions.controller.ts:89-104`, `ListSesiGuruDto`) — SAAT INI khusus role `admin_jurnal`, filter per-tanggal TUNGGAL, bukan per-kelas-per-bulan. TIDAK BISA dipakai langsung apa adanya — perlu endpoint BARU atau extend yang ini dengan hati-hati (role guru harus bisa akses versi milik sendiri saja).

## Spec Detail

### 1. Backend — endpoint baru untuk guru: daftar kelas yang diajar

- `GET /teaching-sessions/my-classes` (nama final diputuskan saat implementasi) — role `guru`, balikin daftar KELAS UNIK (id, nama, kampus) yang PERNAH/SEDANG diajar guru ini (distinct dari `TeachingSession` milik `teacherId` dari JWT, ATAU dari `Schedule` aktif — pilih sumber yang lebih akurat, VERIFIKASI saat implementasi apakah `Schedule` cukup atau perlu gabungan keduanya untuk kelas yang jadwalnya sudah berubah/lampau).

### 2. Backend — endpoint baru: kalender bulan per kelas

- `GET /teaching-sessions/calendar?kelasId=X&bulan=YYYY-MM` — role `guru`, **WAJIB filter `teacherId` dari JWT** (guru A tidak boleh lihat kalender kelas yang diajar guru B untuk mapel lain, meski kelasnya sama — filter kombinasi `kelasId + teacherId` dalam rentang tanggal bulan itu). Response: daftar tanggal dalam bulan itu yang punya `TeachingSession` dengan status `selesai` (atau yang punya minimal 1 `ClassAttendanceMark` — putuskan definisi "sudah ada presensinya" saat implementasi, REKOMENDASI: status sesi selesai DAN closedAt tidak null, karena itu paling jelas menandakan "sudah beres").
- Response per tanggal: `{ tanggal, sessionId, jumlahHadir, jumlahTidakAdaDiKelas }` (ringkasan cukup untuk badge di kalender, detail lengkap di-fetch terpisah saat tanggal diklik).

### 3. Backend — endpoint detail 1 hari (reuse)

- Detail presensi 1 sesi — REUSE endpoint existing yang dipakai `attendance-table.tsx` (`GET /teaching-sessions/:id/detail` atau serupa yang SUDAH ADA) — TIDAK PERLU endpoint baru untuk ini, cukup mode tampilan FE yang read-only.

### 4. Frontend — halaman `/guru/presensi`

- File baru `apps/web/src/app/(guru)/guru/presensi/page.tsx` + `presensi-view.tsx`.
- Step 1: List kelas yang diajar (dari endpoint langkah 1) — mirip kartu/list sederhana, klik kelas → masuk step 2.
- Step 2: Kalender bulan (komponen kalender — cek apakah sudah ada primitif `Calendar` di `packages/ui`/shadcn yang bisa direuse, JANGAN bangun dari nol kalau sudah tersedia) dengan tanda visual (dot/badge) di tanggal yang punya presensi. Navigasi bulan sebelum/sesudah.
- Step 3: Klik tanggal bertanda → tampilkan daftar siswa + status (REUSE tampilan dari `attendance-table.tsx`, TAPI dalam **mode read-only**: hilangkan semua kontrol edit/toggle, cukup tampilkan status sebagai teks/badge statis). PERTIMBANGKAN: extract komponen `attendance-table.tsx` jadi terima prop `readOnly?: boolean` daripada duplikasi komponen baru — cek dulu apakah ini masuk akal secara struktur sebelum diputuskan.

## Edge Cases
- Guru mengajar 1 kelas untuk BEBERAPA mapel berbeda (jadwal berbeda jam) — kalender per kelas ini implisit PER (kelas × guru), bukan per (kelas × mapel) — kalau guru itu ajar 2 mapel di kelas yang sama, kalender akan gabung tanda dari kedua mapel itu (TIDAK dipisah per mapel di v1, kecuali user vokal minta lebih granular — catat sebagai kemungkinan follow-up, JANGAN over-engineer sekarang).
- Bulan yang belum py sesi apa pun — kalender tampil kosong tanpa tanda, TIDAK error.
- "Read-only kalau sudah ganti hari" (istilah user) — DIINTERPRETASIKAN sebagai: seluruh halaman `/guru/presensi` MEMANG selalu read-only (tidak ada mode edit sama sekali di sini) — edit presensi TETAP HANYA lewat `/guru/sesi/[sessionId]` (T169/existing) SAAT sesi masih berjalan hari itu. Task ini TIDAK membuka jalur edit retroaktif — sesuai instruksi eksplisit user ("sementara tidak bisa diedit, kita perbaiki nanti saja").

## Files
- **Buat:** endpoint baru di `apps/api/src/teaching-sessions/` (`my-classes`, `calendar`), `apps/web/src/app/(guru)/guru/presensi/page.tsx` + `presensi-view.tsx`.
- **Modifikasi (mungkin):** `attendance-table.tsx` kalau diputuskan extract mode `readOnly` untuk reuse.
- **Jangan sentuh:** `PATCH /teaching-sessions/:id/attendance` (cara isi presensi TIDAK berubah), `ClassAttendanceMark` model (reuse apa adanya).

## Acceptance Criteria
- [ ] `/guru/presensi` menampilkan daftar kelas yang diajar guru login.
- [ ] Pilih kelas → kalender bulan tampil dengan tanda di tanggal yang sudah ada presensi (sesi selesai).
- [ ] Klik tanggal bertanda → tampil daftar siswa + status, MURNI READ-ONLY (tidak ada kontrol edit sama sekali di halaman ini).
- [ ] Guru A tidak bisa lihat kalender/presensi kelas yang diajar guru lain (verified via filter teacherId dari JWT, bukan trust client).
- [ ] Bulan kosong tidak error, tampil kalender polos.
- [ ] Build + type-check hijau, jest existing tetap pass.

## Validasi Claudian
- [ ] Konfirmasi endpoint kalender WAJIB filter `teacherId` dari JWT (bukan dari query param client) — cross-tenant leak check.
- [ ] Konfirmasi tampilan detail tanggal TIDAK punya jalur submit/PATCH apa pun (murni read-only, tidak ada tombol simpan tersembunyi).

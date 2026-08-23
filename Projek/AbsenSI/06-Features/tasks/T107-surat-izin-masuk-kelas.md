# T107 — Schema+API+Web: Surat Izin Masuk Kelas untuk Siswa Terlambat

## Depends on
Tidak ada — model baru terpisah (`LateEntrySlip`, lihat di bawah), tidak menyentuh `Permit` yang sedang berpotensi diubah T098. Kalau T098 dikerjakan berdekatan, tidak ada file yang sama disentuh (T098 murni `permits/`, T107 murni model+route baru), aman dikerjakan urutan bebas atau paralel.

## Objective
Piket bisa mencetak surat bukti "izin masuk kelas" untuk siswa yang tap masuk terlambat hari itu (`status: terlambat`) — surat ini yang dibawa siswa ke guru mata pelajaran sebagai bukti keterlambatannya sudah sepengetahuan/tanggung jawab piket, ditandatangani PIKET (bukan guru, kebalikan dari struk izin keluar yang existing).

## Context
- **App:** `apps/api` (model+endpoint baru) + `apps/web` (tombol di Piket Board + modal + route cetak baru)
- **Diskusi 2026-08-05** — hasil riset kode existing (Explore agent, 2026-08-05):
  - `status: terlambat` sudah otomatis terhitung saat tap masuk (`determineStatus()`, `apps/api/src/attendance/attendance.service.ts:535-554` — bandingkan `scannedAt` vs `schedule.jamMulai` hari itu). **Tidak perlu logic baru untuk deteksi terlambat**, tinggal baca `AttendanceRecord.status`.
  - Belum ada alur "piket memproses siswa terlambat" sama sekali di kode — yang ada cuma soal *lock* 2x-terlambat (`applyLateStrikeLock()`, independen dari task ini) dan `UnlockForm` untuk siswa yang sudah locked. T107 ini FITUR BARU, bukan modifikasi alur existing.
  - `Permit` model (`apps/api/prisma/schema.prisma:787-810`) sengaja TIDAK dipakai untuk ini — field-nya (`jamKembaliDiharapkan`, `statusKembali`, dst) spesifik untuk konsep "keluar lalu kembali", tidak relevan untuk "masuk kelas" yang tidak ada konsep kembali. **Keputusan user: model baru terpisah**, bukan extend `PermitJenis`.
  - Pola cetak yang sudah ada (`apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` fungsi `buildPrintUrl()` baris ±24-35, POST → dapat record → `window.open(printUrl)`) — **reuse pola yang SAMA persis**, jangan reinvent.
  - Petugas piket yang login sudah tersedia dari session server-side (`user!.username`, lihat `apps/web/src/app/(piket)/piket/izin-keluar/page.tsx:10`) — pola sama dipakai di sini untuk isi "TTD Piket" otomatis, piket tidak perlu ketik nama sendiri.

## Keputusan Final (dikonfirmasi user 2026-08-05)

1. **Trigger**: tombol "Cetak Surat Masuk" di baris siswa berstatus `terlambat` pada Piket Board (bukan halaman form terpisah/search-to-select seperti Input Izin) — data nama/kelas/jam tap sudah otomatis ada dari row yang sedang ditampilkan, tidak perlu piket mencari siswa manual.
2. **Alasan & catatan**: modal kecil muncul saat tombol diklik — field "Alasan Terlambat" (opsional, teks bebas) + "Catatan Piket" (opsional, teks bebas) → tombol "Cetak" di modal → submit → buka tab cetak. Modal ini analog `LockForm`/permit dialog yang sudah ada di file yang sama, ikuti pola visual yang sama (`Dialog`/`DialogContent` dari `@absensi/ui`, sudah diimport di `piket-board-view.tsx`).
3. **Cakupan siswa**: SEMUA siswa `status === "terlambat"` hari itu, TERMASUK yang sudah `isLocked` (locked karena 2x-terlambat) — tombol cetak surat ini TIDAK memblokir/tidak terkait proses unlock, keduanya independen. Piket bisa cetak surat masuk kelas untuk siswa locked (misal sambil menunggu proses unlock terpisah selesai) tanpa harus unlock dulu.
4. **Disimpan sebagai record permanen** — ada riwayat siapa dicetakkan surat, kapan, oleh piket siapa, konsisten dengan pola `tap_events`/`activity_log`/`permits` yang semuanya insert-tercatat di proyek ini.
5. **Model baru terpisah** dari `Permit` (lihat alasan di Context).

## Spec Detail

### Schema (Prisma)
Model baru `LateEntrySlip` (nama sementara, boleh disesuaikan saat implementasi kalau ada konvensi penamaan lain yang lebih konsisten dengan model lain di proyek — cek pola penamaan existing dulu, misal apakah proyek pakai snake_case atau PascalCase untuk nama tabel DB via `@@map`):

```prisma
model LateEntrySlip {
  id               Int      @id @default(autoincrement())
  studentId        Int
  student          Student  @relation(fields: [studentId], references: [id])
  attendanceRecordId Int?   // FK opsional ke AttendanceRecord hari itu, kalau ada relasi yang masuk akal — cek apakah AttendanceRecord punya id yang bisa dipakai unik per tap-in hari itu
  tanggal          DateTime // tanggal kejadian (bukan waktu cetak)
  jamTap           DateTime // waktu tap masuk (dari AttendanceRecord.scannedAt hari itu)
  alasan           String?  @db.Text // opsional, diisi piket di modal
  catatan          String?  @db.Text // opsional, diisi piket di modal
  kodeVerifikasi   String   // pola sama seperti Permit.kodeVerifikasi (T024)
  printedById      Int      // FK ke User (piket yang mencetak) — WAJIB, ini "TTD Piket"
  printedBy        User     @relation(fields: [printedById], references: [id])
  createdAt        DateTime @default(now())

  @@map("late_entry_slips")
}
```
- Tambah back-relation di `Student` dan `User` model sesuai kebutuhan Prisma (`lateEntrySlips LateEntrySlip[]`).
- **Insert-only** — konsisten dengan pola proyek untuk record bukti/audit (`tap_events`, `activity_log`, dan `Permit` yang sifatnya append meski punya field update untuk resolusi). TIDAK ADA endpoint UPDATE/DELETE untuk model ini — kalau piket salah cetak, cetak ulang (record baru), jangan edit yang lama.
- Migration baru: `pnpm --filter @absensi/api exec prisma migrate dev` (dev dulu, lalu production sesuai alur T105 auto-deploy).

### Backend
- Modul baru `apps/api/src/late-entry-slips/` (ikuti pola modul existing: `*.controller.ts`, `*.service.ts`, `*.module.ts`, `dto/`) — ATAU taruh sebagai sub-bagian dari modul `attendance/` kalau reviewer merasa lebih related ke situ (pertimbangkan saat implementasi, tidak strict harus modul terpisah).
- `POST /late-entry-slips` — body: `{ studentId, alasan?, catatan? }`. Server-side: ambil `AttendanceRecord` hari ini untuk `studentId`, validasi `status === "terlambat"` (tolak kalau bukan terlambat — jangan biarkan surat dicetak untuk siswa yang tidak terlambat), ambil `jamTap` dari situ, generate `kodeVerifikasi`, set `printedById` dari JWT user yang request (piket yang login), `tanggal` = hari ini server-side (**JANGAN dari request body**, konsisten dengan aturan proyek "timestamp selalu dari server" seperti `tap_events.scanned_at`).
- **Guard**: `@Roles(UserRole.guru_piket)` — hanya piket yang boleh cetak surat ini, pola sama seperti endpoint permit lainnya. Cek juga `PiketOnDutyGuard` (ADR-024) kalau pola existing di modul lain mewajibkan piket sedang bertugas untuk aksi ini — putuskan konsisten dengan aksi piket lain yang serupa (lock/unlock/permit).
- `@LogActivity` — WAJIB dipasang di endpoint mutasi ini (lihat memory proyek: 14/22 controller pernah lupa pasang decorator ini, jangan sampai terulang).

### Frontend
- `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — tambah tombol aksi "Cetak Surat Masuk" untuk baris dengan `row.status === "terlambat"` (kolom baru di tabel board, atau di area yang sama dengan tombol aksi lain kalau board sudah punya kolom Aksi — cek struktur tabel terkini saat implementasi, kemungkinan sudah berubah dari T099/T106 kalau dikerjakan lebih dulu).
- Modal baru (state `lateEntryTarget`, pola sama seperti `permitTarget`/`lockTarget`/`unlockTarget` yang sudah ada di file ini baris ±113-118) — form 2 field opsional (Alasan, Catatan) + tombol submit.
- Submit: `POST /late-entry-slips` → dapat response (termasuk `kodeVerifikasi`) → `buildPrintUrl()` versi baru → `window.open()`.
- Route cetak baru: `apps/web/src/app/print/surat-masuk-kelas/route.ts` — **copy struktur dari `apps/web/src/app/print/struk-izin/route.ts`**, sesuaikan:
  - Judul: **"SURAT IZIN MASUK KELAS"** (bukan "BUKTI IZIN KELUAR").
  - Data: Nama Petugas Piket, Tanggal & Jam Tap, Nama Siswa, Kelas, Alasan Terlambat (kalau diisi, tampilkan "-" kalau kosong), Catatan Piket (kalau diisi).
  - **Bagian tanda tangan DIBALIK** dari struk izin keluar — bukan "Persetujuan Guru Pengajar", tapi **"Diketahui/Disetujui Piket:"** dengan nama petugas piket (dari `printedBy`) + kolom TTD di bawahnya. Hapus baris "( ........................ )" generik, ganti dengan nama piket eksplisit di atas garis TTD supaya jelas siapa yang bertanggung jawab (bukan blangko kosong seperti punya guru).
  - Footer: instruksi ke guru, misal *"Mohon siswa diizinkan mengikuti pelajaran. Serahkan surat ini ke guru mata pelajaran."*
  - Kode verifikasi tetap ada di footer, pola sama.
  - Query params: `petugas`, `tgl`, `jamtap`, `nama`, `kls`, `alasan`, `ket`, `kode`.

## Business Rules
- Surat HANYA bisa dicetak untuk siswa `status === "terlambat"` hari itu — validasi di backend, bukan cuma disembunyikan di UI (jangan percaya frontend saja).
- Boleh dicetak berkali-kali untuk siswa yang sama hari yang sama (misal surat hilang/rusak) — tiap cetak = record baru, tidak ada pembatasan jumlah. Kalau ternyata perlu dibatasi (mis. mencegah penyalahgunaan), ini keputusan terpisah yang perlu diklarifikasi ulang ke user, JANGAN diasumsikan saat implementasi.
- Tidak menghalangi/berinteraksi dengan proses lock/unlock siswa — independen total.

## Edge Cases
- Siswa terlambat tapi `AttendanceRecord` untuk hari itu somehow tidak ditemukan saat submit (race condition/data berubah) → tolak dengan pesan jelas, jangan crash.
- Siswa yang statusnya berubah dari terlambat ke lain (harusnya tidak mungkin dalam 1 hari, tapi kalau ada edge case data correction manual) → cetak surat lama tetap valid sebagai record historis, tidak perlu di-invalidate.

## Files
- **Buat:** `apps/api/src/late-entry-slips/` (modul baru lengkap), migration Prisma baru, `apps/web/src/app/print/surat-masuk-kelas/route.ts`.
- **Modifikasi:** `apps/api/prisma/schema.prisma` (model `LateEntrySlip` + back-relations), `apps/api/src/app.module.ts` (registrasi modul baru), `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (tombol + modal baru).
- **Jangan sentuh:** `Permit` model/`permits/` module sama sekali — ini fitur independen, tidak boleh menambah field/enum baru ke situ.

## Acceptance Criteria
- [ ] Piket bisa klik "Cetak Surat Masuk" pada baris siswa terlambat (termasuk yang locked) di Piket Board.
- [ ] Modal menampilkan field Alasan + Catatan (keduanya opsional), submit berhasil membuka tab cetak baru.
- [ ] Surat cetak menampilkan TTD Piket (nama petugas yang login), bukan kolom TTD guru.
- [ ] Backend menolak percobaan cetak untuk siswa yang statusnya bukan `terlambat` hari itu.
- [ ] Record `LateEntrySlip` tersimpan permanen tiap kali surat dicetak, dengan `printedById` terisi benar dari user yang login.
- [ ] `@LogActivity` terpasang di endpoint `POST /late-entry-slips`.
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.
- [ ] Verifikasi visual (Playwright): surat tercetak terlihat rapi di ukuran 58mm, konsisten gaya dengan struk izin keluar existing.

## Validasi Claudian
- [ ] Konfirmasi ke user nama model final (`LateEntrySlip` masih sementara) dan lokasi modul (`late-entry-slips/` terpisah vs sub-bagian `attendance/`) sebelum migration dijalankan — schema change butuh kepastian nama sebelum commit ke migration history.
- [ ] Konfirmasi apakah `PiketOnDutyGuard` perlu diterapkan di endpoint ini juga (konsisten dengan aksi piket lain) — cek pola existing dulu, jangan asumsikan ya/tidak.
- [ ] Pastikan tidak ada pembatasan jumlah cetak yang diam-diam ditambahkan tanpa diminta (lihat Business Rules) — kalau terasa perlu, tanya dulu, jangan putuskan sepihak saat implementasi.

# T016 — End-of-Day Job: Deteksi Siswa Tidak Tap Pulang

## Depends on
T012 (attendance_records sudah ada), T010 (BullMQ sudah setup dari T012)

## Objective
Buat scheduled job yang jalan setiap hari setelah jam pulang sekolah untuk mengidentifikasi siswa yang tap masuk tapi tidak tap pulang, sehingga bisa muncul di antrian klarifikasi Dashboard Piket keesokan paginya.

## Context
- **App:** `apps/api`
- **Tables:** `attendance_records`
- **Queue:** BullMQ (sudah setup di T012)
- **Ref:** `Projek/AbsenSI/06-Features/absensi-gerbang.md` — bagian "Siswa Tidak Tap Pulang"

## Spec Detail

### Scheduled job:
```typescript
// Jalan setiap hari pukul 17:00 (atau jam yang bisa dikonfigurasi via env)
// Gunakan BullMQ repeateable job atau @nestjs/schedule (pilih salah satu, tidak perlu keduanya)

// Rekomendasi: @nestjs/schedule lebih sederhana untuk cron job
// pnpm add @nestjs/schedule --filter api
```

### Logic job:
```typescript
@Cron('0 17 * * 1-6') // Setiap hari Senin-Sabtu jam 17:00
async detectMissingPulang() {
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  // Cari semua attendance_records hari ini:
  // - student_id IS NOT NULL (hanya siswa, bukan guru)
  // - waktu_pulang IS NULL
  // - tanggal = hari ini
  
  // Hasil: daftar siswa yang tap masuk tapi tidak tap pulang
  // Log ke console untuk visibility
  // Data ini akan di-query langsung oleh Dashboard Piket via GET endpoint
  // TIDAK perlu disimpan ke tabel terpisah — query langsung dari attendance_records
}
```

### API endpoint untuk Dashboard Piket:
`GET /attendance/missing-pulang?tanggal=YYYY-MM-DD&kampus_id=X`
- Return: list siswa yang `waktu_pulang IS NULL` pada tanggal tersebut, di-scope ke kampus tertentu
- Default `tanggal`: kemarin (karena piket lihat ini pagi hari berikutnya)
- Akses: `guru_piket` (scope ke kampus mereka)

### Catatan implementasi:
Tidak perlu tabel baru atau flag tambahan. Data `waktu_pulang IS NULL` sudah cukup sebagai indikator. Dashboard Piket (T025) akan query endpoint ini dan menampilkan hasilnya sebagai antrian klarifikasi.

## JANGAN
- ❌ JANGAN auto-close `waktu_pulang` dengan jam default — tidak ada auto-close (keputusan eksplisit)
- ❌ JANGAN buat tabel baru `missing_pulang` — data sudah ada di `attendance_records`, cukup query
- ❌ JANGAN kirim notifikasi otomatis ke orang tua — itu Fase 3
- ❌ JANGAN lock siswa secara otomatis — lock hanya manual oleh piket (ADR-017)
- ❌ JANGAN jalankan job ini pada hari Minggu (non-sekolah)

## Files
- **Buat:** `apps/api/src/attendance/jobs/end-of-day.job.ts`
- **Modifikasi:** `apps/api/src/attendance/attendance.controller.ts` — tambah endpoint `GET /attendance/missing-pulang`
- **Modifikasi:** `apps/api/src/app.module.ts` — import `@nestjs/schedule`, tambah `ScheduleModule.forRoot()`

## Acceptance Criteria
- [ ] Job terdaftar di scheduler dan bisa dipicu manual untuk testing
- [ ] `GET /attendance/missing-pulang?tanggal=2026-07-03&kampus_id=1` return hanya siswa kampus 1 yang tidak tap pulang pada tanggal itu
- [ ] Siswa yang sudah punya `waktu_pulang` tidak masuk ke hasil
- [ ] Guru tidak muncul di hasil (hanya siswa)
- [ ] Response include: nama siswa, kelas, kampus, waktu_masuk (untuk konteks)

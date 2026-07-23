# T042 — Config Toleransi Keterlambatan Mengajar

## Depends on
T038 (schema)

## Objective
Buat mekanisme konfigurasi "toleransi keterlambatan" (dalam menit) untuk perhitungan status terlambat guru mulai mengajar — configurable oleh `admin_jurnal`, dipakai T043 (start-session) untuk menghitung `terlambat_menit`.

## Context
- **App:** `apps/api`
- **Table:** `schedule_config` (dari T038 — **catatan revisi 2026-07-22:** tabel ini sekarang HANYA berisi `toleransi_terlambat_menit`, field `mode`/rentang minggu A/B sudah dipindah ke `semesters`/`block_week_ranges` di T054, jangan bingung dengan versi lama)
- **Role:** `admin_jurnal`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Jendela Waktu Jadwal", open question "Nilai default toleransi keterlambatan (menit)"

## Spec Detail

### Keputusan desain: toleransi GLOBAL, bukan per-guru/per-mapel
Konsisten dengan pola threshold terlambat guru Fase 1 (`schedules.threshold_terlambat_menit` global, lihat T004) — task ini pakai pola yang sama untuk konteks mengajar: **1 nilai toleransi berlaku untuk semua guru, semua mapel**. Kalau nanti kebutuhan berubah jadi per-guru/per-mapel, itu perubahan skema terpisah di luar scope task ini.

### Kolom `schedule_config.toleransi_terlambat_menit` (sudah dibuat di T038, task ini implementasikan endpoint & logic-nya)
```
toleransi_terlambat_menit   Int   @default(10)
```

### Endpoint — buat controller BARU khusus config ini (BUKAN bagian dari `ScheduleResolverService`/T039 — resolver T039 murni baca `semesters`/`block_week_ranges`, tidak ada urusan dengan toleransi keterlambatan)

**GET `/schedule-config`** — response:
```json
{ "toleransi_terlambat_menit": 10 }
```

**PATCH `/schedule-config`** — body:
```json
{ "toleransi_terlambat_menit": 15 }
```
- Validasi: `toleransi_terlambat_menit >= 0`, integer

### Service helper: `ScheduleResolverService.hitungTerlambatMenit(jamMulaiJadwal: string, waktuMulaiAktual: Date): number`
- `deadline = jamMulaiJadwal + toleransi_terlambat_menit`
- Kalau `waktuMulaiAktual <= deadline` → return `0` (tidak terlambat)
- Kalau lebih → return selisih menit dari `jamMulaiJadwal` (bukan dari deadline) ke `waktuMulaiAktual`, dibulatkan ke bawah

## JANGAN
- ❌ JANGAN buat tabel terpisah untuk config toleransi — pakai `schedule_config` yang sudah dibuat T038
- ❌ JANGAN buat toleransi per-guru atau per-mapel — di luar scope, sudah diputuskan global
- ❌ JANGAN hitung "terlambat" dari selisih ke deadline toleransi — perhitungan menit terlambat harus dari `jam_mulai` asli jadwal (supaya angka yang ditampilkan konsisten dengan makna "berapa menit setelah jadwal mulai", toleransi hanya menentukan APAKAH dianggap terlambat, bukan mengubah cara hitung selisihnya)
- ❌ JANGAN gabungkan endpoint ini dengan resolver mode blok (T039) — dua concern yang sudah dipisah sejak revisi 2026-07-22, `schedule_config` (task ini) murni toleransi, `semesters`/`block_week_ranges` (T054) murni mode blok

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` — kolom `toleransi_terlambat_menit` sudah ada dari T038, task ini tidak perlu ubah schema lagi
- **Buat:** `apps/api/src/schedule-resolver/schedule-resolver.service.ts` — tambah method `hitungTerlambatMenit` (boleh reuse module `schedule-resolver` dari T039 untuk method ini SAJA, karena secara konsumen sama-sama dipakai T043, tapi query datanya independen dari resolver mode blok)
- **Buat:** `apps/api/src/schedule-config/schedule-config.controller.ts`, `schedule-config.service.ts`, `schedule-config.module.ts` — controller BARU khusus toleransi, terpisah dari apapun yang berurusan dengan semester/mode blok

## Acceptance Criteria
- [ ] Toleransi default 10 menit sudah ada dari migration/seed
- [ ] `hitungTerlambatMenit("07:00", waktu 07:08)` dengan toleransi 10 → return `0`
- [ ] `hitungTerlambatMenit("07:00", waktu 07:15)` dengan toleransi 10 → return `15` (selisih dari jam_mulai asli, bukan dari deadline)
- [ ] `GET /schedule-config` HANYA return `toleransi_terlambat_menit`, tidak ada field mode/minggu apapun
- [ ] `PATCH /schedule-config` dengan `toleransi_terlambat_menit: -5` → 400 error
- [ ] Update toleransi via `admin_jurnal` → langsung berlaku ke sesi berikutnya yang di-start (tidak perlu restart server)

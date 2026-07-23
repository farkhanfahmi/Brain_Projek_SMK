# T039 — Service Resolver: Jadwal Aktif Hari Ini (Lookup Rentang Blok A/B)

## Depends on
T038 (schema dasar jurnal), T054 (schema `semesters` + `block_week_ranges` — **WAJIB selesai dulu**, resolver ini murni membaca dari struktur yang dibuat T054)

> **Revisi total 2026-07-22:** task ini SEBELUMNYA dirancang untuk menghitung "minggu aktif" dari selisih minggu terhadap 1 titik acuan tanggal (`minggu_acuan_tanggal`/`minggu_acuan_nilai`). **Mekanisme itu DIBATALKAN sepenuhnya** — diganti admin input rentang tanggal eksplisit (`block_week_ranges`, dibuat di T054) untuk menghindari kesalahan sistem dari perhitungan otomatis. Kalau Anda pernah baca versi lama task ini, buang semua asumsi soal "hitung selisih minggu" — resolver sekarang murni LOOKUP, bukan ARITMETIKA.

## Objective
Buat service inti yang menentukan "jadwal mana yang aktif hari ini" dengan cara mencari tanggal hari ini di tabel `block_week_ranges` milik semester yang sedang aktif — bukan menghitung dari rumus. Semua fitur berikutnya (dashboard guru, job generate sesi, dst) bergantung ke service ini sebagai satu-satunya sumber kebenaran "sekarang minggu A atau B".

## Context
- **App:** `apps/api`
- **Tables:** `semesters`, `block_week_ranges` (keduanya dari T054), `schedules`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "📅 Jadwal: Mode Blok (Minggu A/B) vs Mode Normal" (baca ULANG, ini sudah direvisi total 2026-07-22)

## Spec Detail

### Service: `ScheduleResolverService`

**Method `getMingguAktif(tanggal: Date): 'A' | 'B' | null`**
- Cari `semesters` yang `is_active: true` DAN `tanggal` berada dalam rentang `tanggal_mulai`–`tanggal_selesai` semester itu
  - Kalau tidak ketemu semester aktif yang mencakup tanggal ini → return `null` (lihat penanganan di bawah, ini kondisi abnormal yang harus di-log sebagai warning, bukan silent)
- Kalau semester ketemu dan `semesters.mode === 'normal'` → return `null` (tidak relevan, konsisten dengan semantik lama)
- Kalau `semesters.mode === 'blok'`:
  - Query `block_week_ranges` where `semester_id = {semester ini}` AND `tanggal_mulai <= tanggal <= tanggal_selesai`
  - **Harus ketemu TEPAT 1 baris** (karena validasi T054 menjamin tidak ada overlap/gap dalam 1 semester) — kalau ketemu 0 baris: ini kondisi "lubang tanggal belum lengkap", return `null` DAN log warning eksplisit (`"Tanggal {tanggal} belum punya block_week_range untuk semester {id}"`) — ini bukan error aplikasi, tapi data admin yang belum lengkap
  - Kalau ketemu >1 baris: ini BUG (validasi T054 seharusnya mencegah ini) — throw error, jangan diam-diam pakai salah satu
  - Kalau ketemu tepat 1 baris → return `minggu` dari baris itu (`A` atau `B`)

**Method `getJadwalHariIni(teacherId: number, tanggal: Date): Schedule[]`**
- Cari `semesters` yang `is_active: true` dan mencakup `tanggal` (sama seperti di atas) — kalau tidak ketemu, return `[]` (tidak ada jadwal apapun, konsisten dengan tidak adanya semester aktif untuk tanggal itu)
- Ambil semua `schedules` dengan `teacher_id = teacherId`, `semester_id = {semester aktif itu}`, dan `hari` (basis `DAYOFWEEK`) cocok dengan hari dari `tanggal`
- Kalau `semesters.mode === 'normal'` → return semua yang cocok hari, abaikan kolom `minggu` sepenuhnya
- Kalau `semesters.mode === 'blok'`:
  - Panggil `getMingguAktif(tanggal)` — kalau hasilnya `null` (lubang tanggal belum lengkap) → **return `[]`** (tidak ada jadwal yang dianggap valid untuk tanggal ini, ini yang mewujudkan "hard block" di spec: semua sesi terkunci karena sistem tidak tahu ini minggu A atau B)
  - Kalau dapat `A`/`B` → filter tambahan: `schedule.minggu === mingguAktifHariIni OR schedule.minggu === 'setiap_minggu'`
- Urutkan hasil berdasarkan `jam_mulai` ascending

**Method `getMingguAktifSaatIni(): { semester_id: number | null, mode: 'blok' | 'normal' | null, minggu: 'A' | 'B' | null, tanggal_belum_lengkap: boolean }`**
- Shortcut untuk `getMingguAktif(new Date())` + info tambahan — dipakai UI untuk tampilkan badge "Minggu A" / "Minggu B" / "Mode Normal" / **"⚠️ Jadwal hari ini belum lengkap, hubungi Admin Jurnal"** (kalau `tanggal_belum_lengkap: true`, yaitu semester aktif ketemu, mode blok, tapi `block_week_ranges` tidak ketemu untuk hari ini)

## JANGAN
- ❌ JANGAN hitung minggu aktif dari rumus/selisih tanggal apapun — HARUS query `block_week_ranges`, tidak ada aritmetika sama sekali di method ini
- ❌ JANGAN buat fallback diam-diam kalau `block_week_ranges` tidak ketemu untuk suatu tanggal (misal "asumsikan minggu A", atau "pakai minggu terakhir yang diketahui") — WAJIB return `null`/`[]` (hard block) dan log warning, sesuai keputusan eksplisit "celah tanggal = hard block total"
- ❌ JANGAN taruh logic resolver ini di controller atau di frontend — harus 1 service tunggal di backend, dipakai ulang oleh semua consumer (T040 job generator, T041 endpoint jadwal guru, dst) supaya tidak ada 2 implementasi yang bisa berbeda hasil
- ❌ JANGAN cache hasil `getMingguAktif` — harus dihitung ulang tiap panggilan (query murah, dan admin bisa mengubah `block_week_ranges` kapan saja)

## Files
- **Buat:** `apps/api/src/schedule-resolver/schedule-resolver.module.ts`
- **Buat:** `apps/api/src/schedule-resolver/schedule-resolver.service.ts`
- **Buat:** `apps/api/src/schedule-resolver/schedule-resolver.service.spec.ts` — unit test WAJIB, minimal kasus berikut:
  - Tanggal masuk rentang `A` → return `A`
  - Tanggal masuk rentang `B` → return `B`
  - Tanggal TIDAK masuk rentang manapun (lubang, simulasikan dengan sengaja tidak insert rentang untuk tanggal itu di test data) → return `null`, tidak throw error
  - Semester `mode: normal` → selalu return `null` terlepas dari `block_week_ranges` apapun
  - Tidak ada semester `is_active` yang mencakup tanggal tsb sama sekali → return `null`
  - `getJadwalHariIni` untuk tanggal yang "lubang" → return `[]` (bukan sebagian data, benar-benar kosong)

## Acceptance Criteria
- [ ] Unit test semua kasus di atas hijau
- [ ] `getMingguAktif` tidak pernah melakukan perhitungan tanggal (selisih hari/minggu) — murni 1 query lookup
- [ ] `getJadwalHariIni` untuk tanggal dengan `block_week_ranges` lengkap → hanya return schedule yang `minggu` cocok ATAU `setiap_minggu`
- [ ] `getJadwalHariIni` untuk tanggal "lubang" (belum ada di `block_week_ranges` manapun) → return array kosong, TIDAK partial/fallback ke apapun

## Handoff ke T040 & T041
T040 (job generate `teaching_sessions` harian) akan memanggil `getJadwalHariIni()` untuk SEMUA guru — kalau hasilnya `[]` untuk suatu tanggal karena lubang, job TIDAK generate sesi apapun untuk tanggal itu (bukan error, tapi harus di-log sebagai warning supaya admin_jurnal sadar ada lubang yang perlu dilengkapi). T041 (endpoint jadwal guru) harus meneruskan sinyal "jadwal belum lengkap" ini ke response supaya UI (T048) bisa tampilkan pesan yang jelas ke guru, bukan sekadar "tidak ada jadwal hari ini" yang membingungkan (guru bisa salah kira dianggap tidak ada jam ngajar, padahal sebenarnya data admin belum lengkap).

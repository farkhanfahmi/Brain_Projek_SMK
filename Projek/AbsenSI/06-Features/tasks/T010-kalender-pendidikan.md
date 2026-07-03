# T010 — Kalender Pendidikan (Tahun Ajaran + Hari Libur)

## Depends on
T003 (auth), T004 (kampus/kelas sudah ada — bukan dependency teknis tapi urutan logis)

## Objective
Buat API dan UI untuk manajemen tahun ajaran aktif dan input hari libur (blok dan mendadak). Data ini digunakan modul Rekap untuk menghitung alfa.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `academic_years`, `school_holidays`
- **Role akses:** `super_admin`
- **Ref:** `Projek/AbsenSI/06-Features/kalender-pendidikan.md`, `Projek/AbsenSI/04-Database-Schema.md`

## Spec Detail

### API Endpoints:

**Academic Years:**
- `GET /academic-years` — list semua tahun ajaran
- `POST /academic-years` — `{ nama, tanggal_mulai, tanggal_selesai }`
- `PATCH /academic-years/:id/activate` — set `is_active: true`, set semua yang lain `is_active: false` (atomic)
- `GET /academic-years/active` — return tahun ajaran yang sedang aktif

**School Holidays:**
- `GET /school-holidays?academic_year_id=&month=` — list hari libur (filter by tahun ajaran dan bulan untuk tampilan kalender)
- `POST /school-holidays` — `{ academic_year_id?, tanggal_mulai, tanggal_selesai, jenis, keterangan }`
  - Hari tunggal: `tanggal_mulai == tanggal_selesai`
  - `jenis`: `libur_nasional | libur_semester | libur_sekolah | libur_mendadak`
- `PATCH /school-holidays/:id` — edit keterangan atau tanggal
- `DELETE /school-holidays/:id` — hapus entri libur (ini data konfigurasi, bukan log — boleh dihapus)

### Admin UI (`/admin/kalender`):

**Section 1: Manajemen Tahun Ajaran**
- List tahun ajaran (card sederhana: nama, tanggal mulai-selesai, badge "Aktif")
- Form buat tahun ajaran baru
- Tombol "Aktifkan" per tahun ajaran

**Section 2: Kalender Visual (bulanan)**
- Grid kalender standar (7 kolom hari, baris per minggu)
- Navigasi prev/next bulan, pilih bulan/tahun
- Pewarnaan:
  - **Putih:** hari aktif sekolah (Senin–Jumat, bukan libur)
  - **Abu-abu muda:** Sabtu (opsional — hadir dicatat tapi bukan hari wajib)
  - **Abu-abu gelap:** Minggu
  - **Merah muda:** hari dalam `school_holidays`
- Klik tanggal yang merah muda → popup: tampilkan info libur + tombol Edit/Hapus
- Klik tanggal putih/abu Sabtu → popup: form "Tandai Libur Mendadak" (pre-fill `tanggal_mulai = tanggal_selesai = tanggal diklik`, `jenis = libur_mendadak`)

**Section 3: Form Input Libur Blok**
- Form di bawah kalender: date range picker (mulai–selesai), dropdown jenis, input keterangan
- Tombol "Tambahkan ke Kalender" → refetch kalender

### Logic "hari wajib" (untuk dipakai modul Rekap di T019):
Buat helper function/service `SchoolCalendarService.getWajibDays(from, to)`:
```typescript
// Return array of dates yang merupakan "hari wajib absen"
// = Senin-Jumat (TIDAK Sabtu), dalam range academic_year aktif,
//   TIDAK overlap dengan school_holidays
```

## JANGAN
- ❌ JANGAN buat logika "hari wajib" di dalam controller rekap — buat sebagai service terpisah yang bisa dipakai modul lain
- ❌ JANGAN buat Sabtu jadi hari wajib — Sabtu opsional (hadir dicatat, tidak hadir bukan alfa)
- ❌ JANGAN hardcode libur nasional — admin input manual semua libur
- ❌ JANGAN hapus `academic_years` yang tidak aktif — data historis harus tetap ada untuk rekap tahun sebelumnya
- ❌ JANGAN gunakan library kalender yang berat — implementasi grid kalender sendiri dengan Tailwind, cukup sederhana untuk kebutuhan ini

## Files
- **Buat:** `apps/api/src/calendar/calendar.module.ts`
- **Buat:** `apps/api/src/calendar/academic-year.service.ts`
- **Buat:** `apps/api/src/calendar/school-holiday.service.ts`
- **Buat:** `apps/api/src/calendar/school-calendar.service.ts` (helper `getWajibDays`)
- **Buat:** `apps/api/src/calendar/calendar.controller.ts`
- **Buat:** `apps/web/app/(admin)/kalender/page.tsx`

## Acceptance Criteria
- [ ] `POST /academic-years` + `PATCH /activate` → tahun ajaran baru jadi aktif, yang lama tidak aktif
- [ ] `POST /school-holidays` dengan range 5 hari → 5 hari terwarna merah di kalender
- [ ] Klik tanggal putih di kalender → form "Libur Mendadak" muncul dengan tanggal pre-filled
- [ ] `SchoolCalendarService.getWajibDays('2025-11-01', '2025-11-30')` return array tanggal Senin-Jumat yang bukan libur
- [ ] Sabtu TIDAK muncul di hasil `getWajibDays`
- [ ] Hari libur di `school_holidays` TIDAK muncul di hasil `getWajibDays`

## Handoff ke T019
T019 (Rekap) akan memanggil `SchoolCalendarService.getWajibDays()` untuk menghitung alfa. Pastikan method ini sudah ada dan exported dari CalendarModule sebelum T019 dimulai.

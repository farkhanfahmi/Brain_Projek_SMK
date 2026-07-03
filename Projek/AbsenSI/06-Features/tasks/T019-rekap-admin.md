# T019 — Admin Dashboard: Rekap Kehadiran Siswa

## Depends on
T010 (SchoolCalendarService.getWajibDays harus ada), T012 (attendance_records harus terisi), T022 (permits harus ada — tapi bisa dikerjakan paralel, rekap tanpa permits data tetap valid untuk testing)

## Objective
Buat halaman rekap kehadiran siswa di admin dashboard dengan filter tanggal/kelas/jurusan dan perhitungan alfa yang akurat menggunakan kalender pendidikan.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `attendance_records`, `permits`, `students`, `kelas`, `jurusan`, `academic_years`, `school_holidays`
- **Role akses:** `super_admin`
- **Ref:** `Projek/AbsenSI/06-Features/rekap-kehadiran.md`

## Spec Detail

### API `GET /attendance/report`:

Query params:
- `from` (required): YYYY-MM-DD
- `to` (required): YYYY-MM-DD
- `kelas_id` (optional)
- `jurusan_id` (optional)

**Logika query per siswa:**

```typescript
// 1. Ambil daftar siswa sesuai filter
// 2. Ambil "hari wajib" dalam range dari SchoolCalendarService.getWajibDays(from, to)
// 3. Per siswa:
//    - hadir: attendance_records dengan waktu_masuk IS NOT NULL dan status = 'hadir'
//    - terlambat: attendance_records dengan status = 'terlambat'
//    - izin: permits dengan jenis=tidak_masuk dan alasan_kategori='izin'
//    - sakit: permits dengan jenis=tidak_masuk dan alasan_kategori='sakit'
//    - alfa: hari_wajib - hadir - terlambat - izin - sakit
//      (alfa = tidak ada record di keduanya, di hari yang wajib)
```

**Response:**
```json
{
  "meta": {
    "from": "2026-07-01",
    "to": "2026-07-31",
    "total_hari_wajib": 23,
    "total_siswa": 150
  },
  "data": [
    {
      "student_id": "xxx",
      "nama": "Budi Santoso",
      "nisn": "1234567890",
      "kelas": "XI-RPL-1",
      "jurusan": "RPL",
      "hadir": 20,
      "terlambat": 2,
      "izin": 1,
      "sakit": 0,
      "alfa": 2
    }
  ],
  "pagination": { "page": 1, "per_page": 50, "total": 150 }
}
```

### Admin UI (`/admin/rekap`):

**Filter bar:**
- Date range picker (dari – sampai), default: bulan berjalan
- Dropdown Kelas (opsional, "Semua Kelas")
- Dropdown Jurusan (opsional, "Semua Jurusan")
- Tombol "Tampilkan"

**Tabel hasil:**
| Nama | Kelas | Jurusan | Hadir | Terlambat | Izin | Sakit | Alfa |
- Sort by kolom (klik header)
- Jumlah total hari wajib ditampilkan di atas tabel sebagai konteks
- Pagination 50 per halaman

**Loading state:** skeleton table saat query berjalan

**Empty state:** pesan "Belum ada data kehadiran untuk periode ini" kalau result kosong

### Perhatian performa:
Query rekap bisa berat untuk range 1 bulan + semua siswa. Pastikan:
- Index `(tanggal, student_id)` ada di `attendance_records` (cek via `prisma migrate`)
- Index `(tanggal, student_id)` ada di `permits`
- Query pakai `findMany` dengan `include` yang minimal — jangan include relasi yang tidak dibutuhkan di rekap

## JANGAN
- ❌ JANGAN hitung alfa sebagai kolom di database — alfa dihitung saat query (kondisi absennya data)
- ❌ JANGAN buat rekap guru di task ini — rekap guru adalah Fase 2
- ❌ JANGAN buat export Excel/PDF — Fase 2
- ❌ JANGAN include Sabtu dalam `total_hari_wajib` — Sabtu bukan hari wajib
- ❌ JANGAN query semua data tanpa limit kalau filter tidak diset — wajib ada pagination

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.controller.ts` — tambah endpoint `GET /attendance/report`
- **Buat:** `apps/api/src/attendance/attendance-report.service.ts` (logic rekap terpisah dari tap logic)
- **Buat:** `apps/web/app/(admin)/rekap/page.tsx`

## Acceptance Criteria
- [ ] `GET /attendance/report?from=2026-07-01&to=2026-07-31` return data semua siswa
- [ ] Filter `kelas_id` hanya return siswa di kelas itu
- [ ] `alfa` = 0 untuk siswa yang hadir semua hari wajib
- [ ] Hari libur yang diinput di kalender (T010) tidak dihitung sebagai hari wajib (tidak menambah alfa)
- [ ] Sabtu tidak masuk hitungan `total_hari_wajib`
- [ ] Halaman rekap di UI: pilih filter → klik Tampilkan → tabel muncul dengan data yang benar

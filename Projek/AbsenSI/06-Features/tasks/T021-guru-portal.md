# T021 — Guru Portal: Riwayat Kehadiran Diri Sendiri

## Depends on
T020 (akun guru harus bisa dibuat), T012 (attendance_records harus terisi)

## Objective
Buat halaman untuk guru melihat riwayat kehadiran dirinya sendiri setelah login.

## Context
- **App:** `apps/web`
- **Route:** `/riwayat` (atau bisa juga `/guru/riwayat`)
- **Role:** `guru`
- **Ref:** `Projek/AbsenSI/06-Features/akun-guru.md`

## Spec Detail

### API `GET /attendance/my-history`:
- Auth: JwtAuthGuard (role `guru`)
- Ambil `teacher_id` dari JWT payload (`req.user.teacher_id`)
- Query: `attendance_records` where `teacher_id = req.user.teacher_id`
- Filter opsional: `from`, `to` (date range)
- Response:
```json
[
  {
    "tanggal": "2026-07-03",
    "waktu_masuk": "07:15:00",
    "waktu_pulang": "15:30:00",
    "status": "hadir"
  },
  {
    "tanggal": "2026-07-02",
    "waktu_masuk": "07:45:00",
    "waktu_pulang": "15:30:00",
    "status": "terlambat"
  }
]
```

### UI (`/riwayat`):
- Layout: sama seperti admin tapi sidebar minimal (hanya link "Riwayat Kehadiran" dan "Logout")
- Atau bisa layout terpisah yang sederhana — tidak perlu full admin layout

Konten:
- Sapaan: "Selamat datang, [Nama Guru]"
- Filter: date range picker (default: bulan ini)
- Tabel: Tanggal | Masuk | Pulang | Status
- Badge status: Hadir (hijau), Terlambat (kuning), tidak ada data (abu)
- **Tidak ada tombol edit apapun** — read-only

### Protected route:
- Kalau `role` bukan `guru` → redirect ke dashboard yang sesuai rolenya
- Kalau tidak login → redirect ke `/login`

## JANGAN
- ❌ JANGAN tampilkan data siswa atau guru lain
- ❌ JANGAN buat tombol edit atau koreksi kehadiran — guru tidak punya wewenang ubah data sendiri
- ❌ JANGAN buat halaman rekap per kelas — itu Fase 2 (wali kelas feature)
- ❌ JANGAN ambil data lewat `teacher_id` dari query param — harus dari JWT payload (mencegah guru lihat data guru lain)

## Files
- **Buat:** `apps/api/src/attendance/` — tambah endpoint `GET /attendance/my-history`
- **Buat:** `apps/web/app/(guru)/riwayat/page.tsx`
- **Buat:** `apps/web/app/(guru)/layout.tsx` (layout sederhana untuk guru, beda dari admin)

## Acceptance Criteria
- [ ] Login sebagai guru → akses `/riwayat` → tampilkan data kehadiran guru tersebut
- [ ] Login sebagai guru A, akses `GET /attendance/my-history` → hanya data guru A (bukan guru B)
- [ ] Filter date range berfungsi
- [ ] Tidak ada tombol edit/ubah di halaman ini
- [ ] Akses `/riwayat` tanpa login → redirect ke `/login`

# T005 — Core Module: Students & Teachers CRUD

## Depends on
T004 (kelas dan jurusan harus sudah ada)

## Objective
Buat API dan halaman admin untuk melihat dan mengelola data siswa dan guru yang sudah ada di database (setelah import CSV di T006).

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `students`, `teachers`
- **Role akses:** `super_admin`
- **ADR:** ADR-010 (students & teachers terpisah)

## Spec Detail

### API Endpoints:

**Students:**
- `GET /students` — list dengan filter: `kelas_id?`, `jurusan_id?`, `kampus_id?`, `status?` (aktif/nonaktif), `search?` (nama/nisn), pagination
- `GET /students/:id` — detail siswa + riwayat kartu (`cards` yang pernah dimiliki) + status lock
- `PATCH /students/:id` — update data (nama, kelas — NISN tidak boleh diubah)
- `PATCH /students/:id/status` — aktifkan/nonaktifkan siswa

**Teachers:**
- `GET /teachers` — list dengan filter: `status?`, `search?` (nama/nip), pagination
- `GET /teachers/:id` — detail guru + riwayat kartu
- `PATCH /teachers/:id` — update data (nama — NIP tidak boleh diubah)
- `PATCH /teachers/:id/status` — aktifkan/nonaktifkan

### Admin UI:

**Halaman `/admin/siswa`:**
- Tabel: Nama | NISN | Kelas | Jurusan | Kampus | Status | Aksi
- Filter bar: search nama/NISN, dropdown kelas, dropdown jurusan, dropdown kampus, toggle aktif/nonaktif
- Pagination (25 per halaman)
- Badge status: `aktif` (hijau), `nonaktif` (abu), `terkunci` (merah)
- Klik baris → modal detail (bukan halaman baru) yang tampilkan info lengkap + riwayat kartu

**Halaman `/admin/guru`:**
- Tabel: Nama | NIP | Status | Kartu Aktif | Aksi
- Lebih sederhana dari siswa — guru tidak punya kelas
- Badge status lock tidak ada (lock hanya untuk siswa)

### Kolom lock di detail siswa:
Tampilkan informasi lock kalau `locked_at IS NOT NULL`:
- Terkunci sejak: `locked_at`
- Alasan: `locked_reason`
- Dikunci oleh: `locked_by` (nama user)

**Catatan:** aksi lock/unlock siswa bukan dari halaman ini — itu dari Dashboard Piket (T026). Halaman ini hanya menampilkan status lock sebagai informasi.

## JANGAN
- ❌ JANGAN buat form create siswa/guru manual di sini — data masuk via import CSV (T006). Halaman ini adalah view + edit, bukan create
- ❌ JANGAN buat endpoint DELETE siswa/guru — data tidak boleh dihapus, hanya dinonaktifkan
- ❌ JANGAN izinkan ubah NISN atau NIP — ini identifier permanen
- ❌ JANGAN tampilkan tombol lock/unlock di halaman admin — itu eksklusif di Dashboard Piket (ADR-019)
- ❌ JANGAN buat halaman detail terpisah (`/admin/siswa/[id]`) — gunakan modal/drawer supaya user tidak kehilangan konteks filter yang sudah diset

## Files
- **Buat:** `apps/api/src/core/students/`
- **Buat:** `apps/api/src/core/teachers/`
- **Buat:** `apps/web/app/(admin)/siswa/page.tsx`
- **Buat:** `apps/web/app/(admin)/guru/page.tsx`

## Acceptance Criteria
- [ ] `GET /students?kampus_id=X` hanya return siswa dari kampus X
- [ ] `GET /students?search=budi` return siswa dengan nama atau NISN mengandung "budi"
- [ ] Detail siswa menampilkan riwayat kartu (aktif dan nonaktif)
- [ ] Badge "Terkunci" muncul di siswa yang `locked_at IS NOT NULL`
- [ ] `PATCH /students/:id` dengan body `{ nisn: "..." }` diabaikan (NISN tidak bisa diubah)
- [ ] Pagination berjalan (halaman 2 return 25 data berikutnya)

## Handoff ke T006
T006 (import CSV) akan memanggil endpoint yang sama untuk validasi (`kelas`, `jurusan` exists). Pastikan response `GET /kelas` dan `GET /jurusan` sudah bisa dipakai sebagai data dropdown di UI import.

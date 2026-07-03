# T004 — Core Module: Master Data Manual (Kampus, Jurusan, Kelas, Jadwal)

## Depends on
T003 (auth harus sudah jalan)

## Objective
Buat API dan UI admin untuk input manual master data: Kampus, Jurusan, Kelas, dan pengaturan jadwal/threshold. Ini data yang harus diisi sebelum import siswa/guru bisa dilakukan.

## Context
- **App:** `apps/api` (endpoints) + `apps/web` (UI admin)
- **Tables:** `kampus`, `jurusan`, `kelas`, `schedules`
- **Role akses:** `super_admin` saja
- **ADR:** ADR-015 (struktur kampus)

## Spec Detail

### API Endpoints (`apps/api`):

**Kampus:**
- `GET /kampus` — list semua kampus
- `POST /kampus` — buat kampus baru `{ nama: string }`
- `PATCH /kampus/:id` — edit nama

**Jurusan:**
- `GET /jurusan` — list semua jurusan
- `POST /jurusan` — `{ nama: string }`
- `PATCH /jurusan/:id` — edit nama

**Kelas:**
- `GET /kelas` — list semua kelas (include relasi kampus + jurusan)
- `POST /kelas` — `{ nama: string, kampus_id: string, jurusan_id: string }`
- `PATCH /kelas/:id` — edit (tidak boleh ubah kampus_id setelah dibuat — ada siswa yang bergantung)
- `GET /kelas/:id/students` — list siswa di kelas ini (dipakai di UI)

**Jadwal (Schedules):**
- `GET /schedules` — list jadwal
- `POST /schedules` — buat jadwal masuk sekolah
- `PATCH /schedules/:id`
- **Yang paling penting untuk Fase 1:** jam masuk sekolah (`type: jam_sekolah`) dan `threshold_terlambat_menit` global untuk guru

### Admin UI (`apps/web/app/(admin)/`):

**Halaman `/admin/master-data`** dengan tabs:
1. **Tab Kampus:** tabel 2 kolom (nama, aksi) + form create/edit inline — data kecil, tidak perlu pagination
2. **Tab Jurusan:** sama seperti Kampus
3. **Tab Kelas:** tabel dengan kolom nama, kampus, jurusan + filter dropdown kampus + tombol create (form modal: nama, pilih kampus, pilih jurusan)
4. **Tab Jadwal:** form set jam masuk sekolah + threshold terlambat guru (dalam menit) — ini **bukan** tabel besar, hanya form setting

### Validasi penting:
- Kelas: `kampus_id` dan `jurusan_id` harus sudah ada di DB
- Kelas: nama unik per kampus (boleh ada "X-RPL-1" di Kampus 1 dan Kampus 2 sekaligus)
- Jadwal: validasi format jam (`HH:mm`)

## JANGAN
- ❌ JANGAN buat endpoint DELETE untuk Kelas — kelas tidak boleh dihapus kalau sudah ada siswa. Kalau perlu "nonaktifkan" bisa tambah kolom `status` nanti, tapi itu bukan scope Fase 1
- ❌ JANGAN buat UI untuk mapel (mata pelajaran) — itu Fase 2
- ❌ JANGAN buat endpoint per-guru untuk threshold terlambat — threshold adalah **global**, satu nilai untuk semua guru
- ❌ JANGAN buat halaman terpisah untuk masing-masing entity — semua dalam 1 halaman `/admin/master-data` dengan tabs
- ❌ JANGAN implement pagination untuk kampus/jurusan — jumlahnya kecil (< 10 kampus, < 20 jurusan)

## Files
- **Buat:** `apps/api/src/core/kampus/`
- **Buat:** `apps/api/src/core/jurusan/`
- **Buat:** `apps/api/src/core/kelas/`
- **Buat:** `apps/api/src/core/schedules/`
- **Buat:** `apps/web/app/(admin)/master-data/page.tsx` + komponen terkait
- **Modifikasi:** `apps/api/src/app.module.ts` — import CoreModule

## Acceptance Criteria
- [ ] `POST /kampus` berhasil buat kampus baru, login sebagai `super_admin`
- [ ] `POST /kampus` return 403 kalau login sebagai `guru`
- [ ] `POST /kelas` dengan `jurusan_id` tidak valid return error yang jelas
- [ ] UI tab Kelas: bisa create kelas baru, kelas muncul di tabel setelah create
- [ ] UI tab Jadwal: form tersimpan, nilai threshold bisa diambil via `GET /schedules`

## Handoff ke T005
T005 butuh endpoint `GET /kelas` yang sudah jalan untuk dropdown saat input/import siswa.

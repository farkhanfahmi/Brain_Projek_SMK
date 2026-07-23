# T076 — UI+API: Direktori Siswa (Search & Filter) di Dashboard Piket

## Depends on
Tidak ada — reuse endpoint/halaman existing (`students.controller.ts`, `riwayat-catatan`), murni menambah akses role + 1 halaman list baru.

## Objective
Tambahkan menu "Direktori Siswa" di dashboard piket — daftar siswa yang bisa dicari (nama) dan difilter (kelas/jurusan), tiap baris tampilkan foto+nama+kelas, klik baris membuka halaman detail siswa **yang sudah ada** (`/admin/siswa/[id]`, dibuka aksesnya untuk `guru_piket`, BUKAN dibuat ulang).

## Context
- **App:** `apps/api` + `apps/web`
- **Route baru:** `/piket/siswa` (menu baru di sidebar piket)
- **Ref:** Diskusi 2026-07-22 — celah ditemukan: dashboard piket cuma punya daftar kontekstual (tap hari ini, belum kembali, terkunci), tidak ada direktori pencarian siswa

## Spec Detail

### Keputusan Final
- **Scope: LINTAS KAMPUS, bukan per-kampus** — pengecualian SADAR dari pola scope kampus yang dipakai semua fitur piket lain (ADR-015). Alasan: guru piket kadang perlu cari/verifikasi identitas siswa dari kampus lain (siswa titipan lintas kampus, verifikasi identitas, dst). **Catat ini eksplisit di kode/komentar** supaya tidak ada yang "memperbaiki" jadi scoped-kampus di masa depan tanpa sadar ini keputusan sengaja
- **Reuse halaman detail siswa existing** (`apps/web/.../siswa/[id]/siswa-detail-view.tsx`) — SUDAH punya foto, biodata, riwayat catatan (terlambat/izin/sakit/alfa via `riwayat-catatan`). TIDAK buat halaman detail baru, cukup buka akses guard-nya
- **"Riwayat pelanggaran"**: untuk sekarang CUKUP riwayat kehadiran yang sudah ada (terlambat/izin/sakit/alfa). **Dicatat untuk masa depan** (BUKAN scope task ini): akan ada fitur guru BK/PDS mencatat pelanggaran tata tertib terpisah, yang nanti akan MUNCUL juga di halaman detail siswa yang sama — jangan desain halaman detail ini sedemikian sempit sehingga sulit menambah section baru nanti (tapi jangan bangun section pelanggaran KOSONG sekarang, itu bukan scope task ini)

### API: Buka Akses Role
**`GET /students`** (existing, `students.controller.ts`) — tambah `guru_piket` ke `@Roles` di level controller ATAU override per-method kalau method lain di controller itu memang harus tetap eksklusif admin (cek dulu method apa saja yang ada, generalisasi hanya untuk `GET`, method mutasi TETAP eksklusif `super_admin`/`card_admin`)
```typescript
@Get()
@Roles(UserRole.super_admin, UserRole.card_admin, UserRole.guru_piket)
findAll(@Query() filter: ListStudentsDto) { ... }
```
- Pastikan `ListStudentsDto` (existing) sudah mendukung filter `nama` (search), `kelasId`, `jurusanId` — cek dulu, tambahkan kalau belum ada

**`GET /students/:id`** — tambah `guru_piket` ke `@Roles`

**`GET /attendance/students/:id/riwayat-catatan`** — tambah `guru_piket` ke `@Roles`

### UI: `/piket/siswa` (menu baru sidebar piket)
- Search input (nama) + 2 dropdown filter (Kelas, Jurusan) — pola sama seperti filter yang sudah ada di `(admin)/siswa/siswa-view.tsx`, REUSE komponen filter itu kalau strukturnya cukup generik untuk dipakai ulang
- List/tabel hasil: **foto (avatar bulat kecil) + Nama Lengkap + Kelas** per baris — pakai `DataTableCard`/`DataTableEntityCell` (T058, generik) kalau cocok
- Klik baris → navigasi ke `/admin/siswa/[id]` (halaman detail existing) — **read-only untuk piket**: pastikan halaman detail itu sendiri tidak menampilkan tombol edit/hapus/upload foto untuk role `guru_piket` (cek behavior existing, `siswa-detail-view.tsx` punya tombol upload foto & hapus yang HARUS disembunyikan/disabled kalau yang login adalah `guru_piket`)

## JANGAN
- ❌ JANGAN buat halaman detail siswa BARU — WAJIB reuse `/admin/siswa/[id]` yang sudah lengkap (foto, biodata, riwayat)
- ❌ JANGAN scope pencarian ini per-kampus — LINTAS KAMPUS adalah keputusan final yang disengaja, bukan bug untuk diperbaiki
- ❌ JANGAN buka akses `guru_piket` ke endpoint MUTASI siswa (`POST`, `PATCH`, `DELETE /students`) — HANYA `GET` (list, detail, riwayat-catatan). Guru piket read-only mutlak, konsisten aturan tegas yang sudah ada
- ❌ JANGAN tampilkan tombol edit/upload foto/hapus di halaman detail siswa untuk role `guru_piket` — halaman ini dishare dengan admin, guard-nya harus tetap membedakan wewenang di level UI (disabled/hidden) MESKI backend sudah menolak mutasinya
- ❌ JANGAN bangun section "riwayat pelanggaran BK/PDS" kosong di task ini — itu fitur masa depan, dicatat sebagai konteks bukan scope pengerjaan sekarang

## Files
- **Modifikasi:** `apps/api/src/core/students/students.controller.ts` — tambah `guru_piket` ke `@Roles` untuk `GET` methods saja
- **Modifikasi:** `apps/api/src/attendance/attendance.controller.ts` — tambah `guru_piket` ke `@Roles` endpoint `riwayat-catatan`
- **Buat:** `apps/web/app/(piket)/piket/siswa/page.tsx`
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` — sembunyikan tombol edit/upload/hapus untuk role `guru_piket` (cek role dari context/JWT yang sudah tersedia di app)
- **Modifikasi:** sidebar piket (`(piket)/piket-sidebar.tsx`) — tambah menu "Direktori Siswa"

## Acceptance Criteria
- [ ] Login sebagai `guru_piket`, buka `/piket/siswa` → search "Budi" → hasil terfilter, termasuk siswa dari KAMPUS LAIN (verifikasi lintas kampus benar-benar berfungsi)
- [ ] Filter dropdown Kelas/Jurusan berfungsi
- [ ] Klik 1 baris → navigasi ke halaman detail siswa, foto+biodata+riwayat catatan tampil
- [ ] Di halaman detail yang dibuka dari piket, tombol upload foto/hapus siswa TIDAK terlihat (dibanding saat dibuka dari akun admin, yang terlihat)
- [ ] `guru_piket` mencoba `POST /students` atau `PATCH /students/:id` langsung (bypass UI) → tetap 403, hanya `GET` yang terbuka

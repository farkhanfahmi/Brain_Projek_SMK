# T211 — Web: Sub-Menu "Tampilan Jadwal" — View Per Kelas & Per Guru (Read-Only + Edit Inline)

## Depends on
**WAJIB SETELAH T205 (field lantai), T209 (TeachingSession sudah pindah), T212 (Workspace input sudah bisa isi data)**. Bagian dari rangkaian T203-T215.

## Objective
Sub-menu BARU di bawah "Jadwal Pelajaran" — akses cepat KHUSUS jadwal yang **benar-benar aktif SAAT INI** (Opsi Jadwal aktif DAN Semester aktif DAN Tahun Ajaran aktif, 3 syarat sekaligus) — 2 tampilan: **Per Kelas** (accordion, header berisi info lokasi lengkap) dan **Per Guru** (list nama, klik → tabel jadwal). Read-only dengan tombol Edit inline per baris (BUKAN navigasi ke Workspace T212 lagi).

## Konteks — Detail Dikonfirmasi User (2026-08-16)

- **View Per Kelas**: accordion/collapsible card, HEADER format `[Nama Kelas] - Ruang: [Nama Ruang] (Lantai [X], Kampus [Y])` (data diwarisi dari `Kelas`, T169+T205 — TIDAK diinput ulang). Expand → tabel jadwal (Jam Ke, Hari, Waktu, Mapel, Guru). Mode Blok → toggle switch Minggu A/B DI DALAM card itu.
- **View Per Guru**: daftar nama guru → klik → tabel (No, Kelas, Hari, Jam Ke, Waktu, Ruang, Kampus, Lantai — SEMUA field lokasi diwarisi dari Kelas terkait tiap baris, KARENA 1 guru bisa mengajar BANYAK kelas berbeda lokasi). Mode Blok → toggle filter Minggu A/B.
- **Akses**: SAMA seperti Menu Utama (super_admin + admin_jurnal), BUKAN untuk role lain.
- **Sifat**: read-only DEFAULT, tapi tombol Edit INLINE per baris jadwal untuk revisi cepat (bukan navigasi ke form Workspace T212 penuh) — Toast Notification saat berhasil update.

## Spec Detail

### 1. Backend — endpoint khusus "jadwal aktif sekarang"

- `GET /jadwal-pelajaran/aktif/per-kelas` — return SEMUA `JadwalSlot` dari Opsi Jadwal yang MEMENUHI 3 SYARAT (`OpsiJadwal.isActive=true` AND `Semester.isActive=true` AND `AcademicYear.isActive=true`), grouped by Kelas (dengan info Kampus+Ruangan+Lantai ter-include).
- `GET /jadwal-pelajaran/aktif/per-guru` — SAMA, grouped by Guru (join lewat `JadwalSlotGuru`).
- **VALIDASI 3-SYARAT INI HARUS 1 HELPER TERPUSAT** (dipakai KEDUA endpoint di atas, DAN dipakai lagi di T209 generate TeachingSession kalau relevan — REUSE, JANGAN duplikasi logic 3-syarat di banyak tempat berbeda yang berisiko tidak sinkron).

### 2. Backend — endpoint edit inline

- `PATCH /jadwal-slot/:id` — endpoint YANG SAMA dari T206 (`JadwalSlotService.update()`), REUSE APA ADANYA (validasi bentrok+guru terdaftar+dst SEMUA TETAP BERLAKU untuk edit inline ini, TIDAK ADA jalur pintas yang skip validasi).

### 3. Frontend — View Per Kelas

- Path BARU `apps/web/src/app/(admin)/jadwal-pelajaran/tampilan/` (sub-route, atau tab terpisah — PUTUSKAN struktur navigasi saat implementasi, KONSISTEN pola breadcrumb/tab yang sudah ada di proyek).
- Accordion per Kelas — header PERSIS format yang diminta, expand → tabel jadwal (KONSISTEN aturan tabel wajib sortable+search — TAPI untuk accordion per-kelas yang sudah scoped 1 kelas, search mungkin tidak relevan, PUTUSKAN saat implementasi apakah search tetap perlu untuk tabel kecil ini).
- Mode Blok → toggle Minggu A/B DI DALAM card (state lokal, ganti data tabel yang ditampilkan tanpa collapse accordion).
- Tombol Edit inline per baris → buka mini-form (Popover/Dialog kecil, BUKAN Workspace penuh) untuk ubah Mapel/Guru/dst baris itu → submit `PATCH /jadwal-slot/:id` → Toast sukses → refresh baris itu saja (optimistic update ATAU refetch, KONSISTEN pola yang sudah established di proyek).

### 4. Frontend — View Per Guru

- List nama guru (dari SEMUA guru yang PUNYA minimal 1 JadwalSlot aktif) — klik → tabel (No, Kelas, Hari, Jam Ke, Waktu, Ruang, Kampus, Lantai).
- SAMA pola Edit inline + toggle Minggu A/B seperti View Per Kelas.

## Edge Cases
- Kelas/Guru TANPA jadwal aktif sama sekali (semua Opsi Jadwal terkait nonaktif/induk tidak aktif) — TIDAK MUNCUL di sub-menu ini sama sekali (KONSISTEN definisi "jadwal aktif SAAT INI") — BUKAN tampil dengan tabel kosong.
- Edit inline yang GAGAL validasi (misal ganti guru jadi yang sedang bentrok) — pesan error backend (T206) ditampilkan APA ADANYA di mini-form, JANGAN silent fail.

## Files
- **Buat:** endpoint backend 2 baru (`jadwal-pelajaran.controller.ts` atau service terpisah untuk agregasi "aktif sekarang"), `apps/web/src/app/(admin)/jadwal-pelajaran/tampilan/` (+duplikat admin-jurnal) — view Per Kelas + Per Guru + mini-form edit inline.
- **Jangan sentuh:** `JadwalSlotService.update()` (T206, REUSE apa adanya untuk edit inline).

## Acceptance Criteria
- [x] View Per Kelas — accordion dengan header lokasi lengkap, expand tampilkan tabel jadwal.
- [x] View Per Guru — list nama, klik tampilkan tabel dengan info lokasi per baris.
- [x] HANYA jadwal yang 3-syarat aktif (Opsi+Semester+TahunAjaran) yang muncul — verified via jest (`jadwal-pelajaran.service.spec.ts`, 6 test, assert WHERE clause 3-syarat sekaligus).
- [x] Toggle Minggu A/B berfungsi untuk kelas/guru dengan jadwal mode Blok.
- [x] Edit inline berfungsi, validasi SAMA seperti create (bentrok dicek via `GuruDropdownRealtime` T212 + `PATCH /jadwal-slot/:id` T206 apa adanya), Toast sukses (`SaveSuccessToast` T212, reuse).
- [x] Build + type-check hijau (tsc api+web, `nest build`, `next build`, jest 557/557).

## Validasi Claudian
- [x] Konfirmasi helper "3-syarat aktif" DIPAKAI BERSAMA oleh kedua endpoint (per-kelas, per-guru) — `getOpsiJadwalAktifSekarang()` private method di `JadwalPelajaranService`, 1 sumber kebenaran, tidak diduplikasi.
- [x] Konfirmasi edit inline REUSE validasi PERSIS sama dengan T206 `JadwalSlotService.update()`, tidak ada jalur pintas yang skip bentrok-check — `EditInlineDialog` submit langsung ke `PATCH /jadwal-slot/:id`, tidak ada endpoint/logic terpisah.

## Implementasi (2026-08-17)

**Backend** (`apps/api/src/jadwal-pelajaran/`):
- `jadwal-pelajaran.service.ts` — `getOpsiJadwalAktifSekarang()` (private, 1 sumber kebenaran 3-syarat), `perKelas()`, `perGuru()`.
- `jadwal-pelajaran.controller.ts` — `GET /jadwal-pelajaran/aktif/per-kelas`, `GET /jadwal-pelajaran/aktif/per-guru`, `@Roles(super_admin, admin_jurnal)`.
- `jadwal-pelajaran.module.ts`, terdaftar di `app.module.ts`.
- `jadwal-pelajaran.service.spec.ts` — 6 test baru (query 3-syarat sama persis di kedua endpoint, grouping, sort, skip slot tanpa guru).
- **Catatan penting**: T204 awalnya spec team-teaching many-to-many (`JadwalSlotGuru`), tapi T209 (2026-08-17) sudah putuskan 1 guru per slot (`MapelGuru` cuma filter dropdown). T211 mengikuti keputusan TERBARU — `teacher` di response singular (`{id,nama} | null`), bukan array.

**Frontend**:
- `apps/web/src/app/(admin)/jadwal-pelajaran/tampilan/page.tsx` + `tampilan-view.tsx` — 2 tab (Per Kelas/Per Guru) via query-param, pola sama T189.
- `components/per-kelas-view.tsx` — accordion manual (state `Set<number>`, TIDAK ADA primitive Accordion di `packages/ui`, ikut pola `openSection` piket-board-view.tsx), header format persis `[Nama Kelas] - Ruang: [Ruangan] (Lantai [X], Kampus [Y])`.
- `components/per-guru-view.tsx` — sama pola, tabel kolom No/Kelas/Hari/Jam Ke/Waktu/Mapel/Ruang/Kampus/Lantai/Aksi (field lokasi dari `Kelas` tiap baris, karena 1 guru bisa lintas kelas/lokasi).
- `components/edit-inline-dialog.tsx` — Dialog mini-form, reuse `GuruDropdownRealtime` (T212) via cross-directory import, submit `PATCH /jadwal-slot/:id`.
- Duplikat ke `(admin-jurnal)/admin-jurnal/jadwal-pelajaran/tampilan/page.tsx` (pola T157, reuse `TampilanJadwalView` apa adanya).
- Link "Tampilan Jadwal" ditambahkan di toolbar `jadwal-pelajaran-view.tsx` (pakai `basePath` yang sudah ada dari T210, otomatis benar di kedua route tree).

**Verifikasi**: tsc api+web bersih, `nest build` bersih, `next build` bersih (kedua route `/jadwal-pelajaran/tampilan` dan `/admin-jurnal/jadwal-pelajaran/tampilan` muncul di output), jest 557/557 passing. Live browser verify TIDAK dilakukan (kredensial login gagal, dikonfirmasi user untuk lanjut tanpa live-verify) — curl ke kedua route mengonfirmasi 307 redirect ke login (tidak ada 500).

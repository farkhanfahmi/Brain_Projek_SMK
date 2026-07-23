# T047 — RBAC `admin_jurnal` + API Kelola Mapel & Jadwal Mengajar

## Depends on
T038 (schema), T039 (schedule resolver untuk validasi mode blok saat assign jadwal), T054 (schema `semesters` + `schedules.semester_id`)

## Objective
Pastikan role `admin_jurnal` ditegakkan sebagai guard backend (bukan cuma UI) yang terkunci ke domain jurnal, dan buat endpoint CRUD mapel + assign jadwal mengajar (guru-kelas-mapel-jam, termasuk kolom `minggu` untuk mode blok) khusus role ini.

## Context
- **App:** `apps/api`
- **Tables:** `mapel`, `schedules`
- **Role:** `admin_jurnal`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Role Baru: admin_jurnal"

## Spec Detail

### Guard: `AdminJurnalGuard` (atau reuse `RolesGuard` existing dengan `@Roles('admin_jurnal')`)
- Pola sama seperti guard existing Fase 1 (`RolesGuard` cek role dari JWT payload vs `@Roles()` decorator) — **cek dulu apakah `RolesGuard` yang sudah ada bisa langsung dipakai** dengan `@Roles('admin_jurnal')` di controller, kemungkinan besar TIDAK perlu guard baru sama sekali, hanya perlu pastikan enum `role` sudah punya nilai `admin_jurnal` (sudah dilakukan T038) dan decorator dipasang benar di controller-controller task ini

### API Mapel — `apps/api/src/mapel/`

**GET `/mapel`** — list semua mapel (akses: `admin_jurnal`, `super_admin` read juga boleh untuk keperluan lain, tapi write HANYA `admin_jurnal`)

**POST `/mapel`** — role `admin_jurnal` saja
```json
{ "nama": "Basis Data", "kode": "RPL-DB" }
```

**PATCH `/mapel/:id`** — role `admin_jurnal` saja, update `nama`/`kode`

**JANGAN buat endpoint DELETE** — konsisten pola Fase 1 (kelas, dst tidak bisa dihapus kalau sudah dipakai), mapel yang sudah dipakai `schedules` tidak boleh dihapus. Kalau perlu "nonaktifkan", itu perluasan skema terpisah di luar scope task ini.

### API Jadwal Mengajar — extend `apps/api/src/core/schedules/` (module existing Fase 1, JANGAN buat module baru terpisah)

**Perubahan endpoint existing:**
- `POST /schedules` dan `PATCH /schedules/:id` — akses ditambah: sekarang bisa diakses `admin_jurnal` (sebelumnya cuma `super_admin`) KHUSUS untuk `type = jam_mengajar`. Guard-nya: kalau `role = admin_jurnal`, request WAJIB `type: jam_mengajar` (tolak 403 kalau `admin_jurnal` coba buat `jam_sekolah`/`jadwal_khusus` — itu tetap wewenang `super_admin` saja)
- Body `POST /schedules` tambah field: `mapel_id` (wajib kalau `type: jam_mengajar`), `minggu` (wajib kalau `schedule_config.mode === 'blok'` DAN `type: jam_mengajar`, harus salah satu dari `A`/`B`/`setiap_minggu`), **`semester_id` (WAJIB kalau `type: jam_mengajar`, dari T054)**

**Validasi tambahan:**
- Kalau `mode` sistem = `blok` dan `minggu` tidak dikirim untuk `type: jam_mengajar` → 400 `"Field minggu wajib diisi saat mode jadwal blok aktif"`
- Kalau `mode` sistem = `normal` dan `minggu` dikirim → abaikan/null-kan (tidak error, tapi tidak disimpan — mode normal tidak butuh field ini)
- **`semester_id` WAJIB untuk `type: jam_mengajar`** — 400 kalau kosong. Validasi `semester_id` yang dikirim benar-benar ada di tabel `semesters` (tidak perlu harus semester aktif — admin_jurnal boleh siapkan jadwal semester yang BELUM aktif, lihat bagian "Salin Jadwal" di bawah)
- Cek bentrok jadwal: guru yang sama tidak boleh punya 2 `schedules` (`type: jam_mengajar`) di hari+jam+minggu+**semester** yang sama (overlap jam_mulai-jam_selesai) — 409 kalau bentrok, response sertakan detail schedule yang bentrok. **Bentrok HANYA dicek dalam `semester_id` yang sama** — jadwal semester berbeda boleh punya guru+jam yang identik tanpa dianggap bentrok (itu memang set data terpisah)

**GET `/schedules?type=jam_mengajar&teacher_id=&kelas_id=&semester_id=`** — sudah ada dari Fase 1, pastikan filter tambahan `mapel_id` dan **`semester_id`** juga didukung untuk keperluan UI T050

### API Baru: Salin Jadwal dari Semester Sebelumnya

**POST `/schedules/copy-from-semester`** — role `admin_jurnal`
```json
{ "dari_semester_id": 3, "ke_semester_id": 4 }
```
- Validasi: `ke_semester_id` harus BELUM punya `schedules` (`type: jam_mengajar`) sama sekali — tolak 409 kalau sudah ada data (`"Semester tujuan sudah punya jadwal, salin hanya untuk semester kosong"`), supaya tidak menimpa/duplikat jadwal yang sudah diedit manual
- Duplikasi SEMUA `schedules` (`type: jam_mengajar`) dari `dari_semester_id` ke `ke_semester_id` — copy `teacher_id`, `kelas_id`, `mapel_id`, `minggu`, `hari`, `jam_mulai`, `jam_selesai`, `threshold_terlambat_menit`; `semester_id` diganti ke `ke_semester_id`; `id` baru (bukan re-use PK lama)
- Response: `{ "disalin": number }` — jumlah baris yang berhasil disalin
- Log ke `activity_log`, action `schedules.copy_from_semester`

## JANGAN
- ❌ JANGAN buat role baru selain `admin_jurnal` di task ini — role sudah dibuat di T038, task ini hanya menegakkan guard & endpoint
- ❌ JANGAN kasih `admin_jurnal` akses ke `POST /schedules` dengan `type: jam_sekolah` atau `jadwal_khusus` — itu tetap eksklusif `super_admin`
- ❌ JANGAN kasih `admin_jurnal` akses ke endpoint `users`, `cards`, `academic_years`, `school_holidays`, atau rekap kehadiran siswa — sesuai batasan tegas di spec ("murni terkunci ke domain jurnal")
- ❌ JANGAN hapus/ubah validasi existing untuk `type: jam_sekolah`/`jadwal_khusus` yang sudah ada dari T004 — task ini HANYA menambah cabang baru untuk `jam_mengajar`, bukan mengubah yang lama
- ❌ JANGAN izinkan `POST /schedules` dengan `type: jam_mengajar` tanpa `semester_id` — wajib, bukan opsional
- ❌ JANGAN cek bentrok jadwal LINTAS semester berbeda — jadwal semester ganjil dan genap adalah set data independen, guru yang sama boleh punya jam identik di kedua semester tanpa dianggap konflik
- ❌ JANGAN izinkan `copy-from-semester` menimpa jadwal yang sudah ada di semester tujuan — hanya untuk semester yang benar-benar kosong (0 baris `schedules` type jam_mengajar), cegah data hilang tidak sengaja

## Files
- **Buat:** `apps/api/src/mapel/mapel.module.ts`
- **Buat:** `apps/api/src/mapel/mapel.service.ts`
- **Buat:** `apps/api/src/mapel/mapel.controller.ts`
- **Modifikasi:** `apps/api/src/core/schedules/schedules.controller.ts` — tambah `@Roles('super_admin', 'admin_jurnal')` dengan validasi tambahan di service untuk cabang `admin_jurnal` + `type`
- **Modifikasi:** `apps/api/src/core/schedules/schedules.service.ts` — validasi `mapel_id`/`minggu` wajib, cek bentrok jadwal

## Acceptance Criteria
- [ ] `admin_jurnal` bisa `POST /schedules` dengan `type: jam_mengajar` lengkap `mapel_id`+`minggu` (saat mode blok) → berhasil
- [ ] `admin_jurnal` coba `POST /schedules` dengan `type: jam_sekolah` → 403
- [ ] `admin_jurnal` coba akses `GET /users` atau `POST /cards` → 403
- [ ] Mode blok aktif, `POST /schedules` tanpa `minggu` untuk `jam_mengajar` → 400
- [ ] Mode normal aktif, `POST /schedules` dengan `minggu` untuk `jam_mengajar` → berhasil, `minggu` tersimpan `null`
- [ ] Assign guru yang sama ke 2 jadwal bentrok jam (hari+minggu+semester sama, jam overlap) → 409 dengan detail schedule yang bentrok
- [ ] Assign guru yang sama ke jam identik tapi `semester_id` BEDA → berhasil, tidak dianggap bentrok
- [ ] `POST /schedules` dengan `type: jam_mengajar` tanpa `semester_id` → 400
- [ ] `POST /mapel` dari role `guru` → 403
- [ ] `POST /schedules/copy-from-semester` ke semester yang sudah punya jadwal → 409, tidak ada perubahan data
- [ ] `POST /schedules/copy-from-semester` ke semester kosong → semua schedules tersalin dengan `semester_id` baru, `id` unik baru (bukan duplikat PK)

## Handoff ke T050
T050 (UI dashboard admin_jurnal) konsumsi endpoint mapel & schedules ini untuk form assign jadwal — termasuk dropdown pilih semester dan tombol "Salin dari Semester Sebelumnya".

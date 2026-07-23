# T046 — API: Izin Guru Tidak Mengajar (Admin Input + Guru Submit Tugas)

## Depends on
T043 (sesi harus ada untuk direferensikan izin per-sesi), T038 (schema `teacher_permits`)

## Objective
Buat endpoint 2-tahap: `admin_jurnal` menginput status izin guru (tanpa request dari guru — sesuai laporan manual WA/lisan) LENGKAP dengan kategori dan file bukti wajib, lalu guru bersangkutan membuka form upload tugas titipan yang hanya aktif setelah admin approve.

## Context
- **App:** `apps/api`
- **Tables:** `teacher_permits`, `teaching_sessions`
- **Role:** `admin_jurnal` (buat izin, upload/replace bukti), `guru` (submit tugas)
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Izin Guru Tidak Mengajar" (baca lengkap, termasuk sub-bagian "Kategori Izin" dan alur 5 langkah — **perhatikan langkah 2 yang WAJIB sertakan file bukti**)

## Spec Detail

### API: `POST /teacher-permits` — role `admin_jurnal`
Multipart form-data (karena WAJIB sertakan file bukti sekaligus saat create — bukan JSON biasa):
```
teacher_id: 45
tanggal: "2026-07-22"
session_id: 501          // opsional, lihat catatan di bawah
kategori: "sakit"         // wajib: sakit | izin_pribadi | tugas_dinas | pelatihan
bukti_file: <file upload> // WAJIB, tidak boleh kosong
```
- `session_id` **opsional** — kalau diisi, izin berlaku HANYA untuk sesi itu; kalau tidak dikirim/`null`, izin berlaku SEHARIAN PENUH (semua sesi guru itu tanggal tsb)
- Validasi: kalau `session_id` diisi, harus milik `teacher_id` yang sama dan `tanggal` yang sama — reject kalau tidak konsisten
- Validasi: tidak boleh ada `teacher_permits` lain yang tumpang tindih untuk kombinasi guru+tanggal+sesi yang sama (cegah duplikat approval)
- **Validasi `bukti_file`: WAJIB ada, request tanpa file → 400 `"File bukti wajib diupload"`.** Tidak ada pengecualian untuk kategori apapun — sesuai keputusan eksplisit, admin yang mengatur ritme (boleh upload bukti sementara seperti screenshot WA, lalu ganti nanti lewat endpoint replace di bawah)
- Validasi file (sama seperti T046 upload tugas): tipe `pdf`/`jpg`/`jpeg`/`png`/`docx`, maks 10MB
- Simpan file ke disk (pola sama seperti `students.foto`, ADR-023 — path relatif), set `bukti_file_path`, `bukti_updated_at = now()`
- Set `kategori`, `status: diizinkan`, `approved_by: req.user.id`, `approved_at: now()`
- Response 201 dengan data permit yang dibuat
- Log ke `activity_log`, action `teacher_permit.create`

### API: `PATCH /teacher-permits/:permitId/bukti` — role `admin_jurnal`
- Multipart, field `bukti_file` — **replace** file bukti yang sudah ada (untuk kasus: admin upload bukti sementara/screenshot WA dulu saat approve cepat, lalu ganti dengan surat dokter/nota dinas resmi begitu tersedia)
- Validasi file sama seperti create (tipe & ukuran)
- Update `bukti_file_path` (file lama **ditimpa**, tidak perlu disimpan sebagai riwayat versi — cukup 1 file aktif), `bukti_updated_at = now()`
- Boleh dipanggil kapan saja, tidak ada batas waktu setelah `approved_at`
- Log ke `activity_log`, action `teacher_permit.update_bukti`

### API: `GET /teacher-permits?teacher_id=&tanggal=` — role `admin_jurnal`
- List permit, filter opsional by `teacher_id` dan/atau `tanggal` — untuk keperluan admin lihat riwayat/status

### API: `GET /teacher-permits/my-active-today` — role `guru`
- `teacher_id` dari JWT
- Return semua `teacher_permits` untuk guru ini, `tanggal = hari ini` — dipakai FE guru untuk tahu tombol "Izin" mana yang aktif/terbuka
```json
[
  { "permit_id": 12, "session_id": 502, "submitted_at": null, "tugas_file_path": null }
]
```

### API: `POST /teacher-permits/:permitId/submit-tugas` — role `guru`
- Multipart form-data: file upload + field `keterangan` (text)
- Validasi kepemilikan: `teacher_permits.teacher_id === req.user.teacher_id` → 403 kalau bukan
- Validasi: `submitted_at` harus masih `null` (belum pernah submit) — kalau sudah pernah, terapkan sebagai UPDATE (guru boleh revisi sebelum jam sesi lewat), bukan ditolak. **Tapi** kalau `follow_up_needed = true` sudah di-set job T044 (artinya sudah lewat jam sesi), submit setelah itu tetap DITERIMA (lebih baik telat diisi daripada tidak sama sekali) — cukup biarkan `follow_up_needed` tetap `true` sebagai jejak historis bahwa ini telat diisi, JANGAN di-reset ke `false` otomatis (piket sudah mungkin sudah menindaklanjuti manual)
- Simpan file ke disk (pola sama seperti `students.foto`/`teachers.foto`, ADR-023 — path relatif, bukan BLOB), set `tugas_file_path`, `tugas_keterangan`, `submitted_at = now()`
- Validasi file: tipe yang diterima `pdf`, `jpg`, `jpeg`, `png`, `docx` — tolak tipe lain dengan pesan jelas. Batas ukuran **10MB**
- Response 200 dengan data permit ter-update

## JANGAN
- ❌ JANGAN buat endpoint untuk GURU mengajukan/request izin — sesuai keputusan eksplisit, alur ini SATU ARAH: admin input duluan, guru cuma submit tugas setelahnya. Tidak ada "request izin" dari sisi guru di scope ini
- ❌ JANGAN izinkan `POST /teacher-permits` tanpa `bukti_file` — WAJIB untuk semua kategori tanpa pengecualian, ini keputusan eksplisit (bukan bug untuk "diperbaiki" dengan membuatnya opsional)
- ❌ JANGAN simpan riwayat versi file bukti (semacam file history/log tiap replace) — cukup 1 file aktif yang ditimpa, `bukti_updated_at` sebagai jejak kapan terakhir diganti sudah cukup untuk kebutuhan saat ini
- ❌ JANGAN izinkan guru submit tugas untuk permit yang bukan miliknya — cek `teacher_id` dari JWT vs `teacher_permits.teacher_id`
- ❌ JANGAN reset `follow_up_needed` ke `false` otomatis saat guru submit telat — biarkan sebagai jejak historis (lihat catatan di atas), clear manual adalah tanggung jawab piket/admin_jurnal (task terpisah kalau dibutuhkan, belum dispec)
- ❌ JANGAN terima file upload tanpa validasi tipe & ukuran — celah keamanan (arbitrary file upload)
- ❌ JANGAN buat validasi "izin harus disetujui minimal H-1" atau aturan waktu lainnya yang tidak diminta — admin bisa input izin kapan saja, termasuk untuk hari ini/dadakan (sesuai konteks sekolah nyata dari interview)

## Files
- **Buat:** `apps/api/src/teacher-permits/teacher-permits.module.ts`
- **Buat:** `apps/api/src/teacher-permits/teacher-permits.service.ts`
- **Buat:** `apps/api/src/teacher-permits/teacher-permits.controller.ts`
- **Modifikasi:** `apps/api/src/app.module.ts` — import module baru

## Acceptance Criteria
- [ ] `admin_jurnal` buat izin 1 sesi spesifik → hanya sesi itu yang `sudah_diizinkan` di response T041, sesi lain guru itu hari ini tetap normal
- [ ] `admin_jurnal` buat izin tanpa `session_id` → semua sesi guru itu tanggal tsb jadi `sudah_diizinkan` (via logic di T041, bukan bikin banyak baris `teacher_permits` — cukup 1 baris `session_id: null`)
- [ ] `POST /teacher-permits` tanpa `bukti_file` → 400, permit TIDAK tercipta
- [ ] `POST /teacher-permits` dengan `kategori: tugas_dinas` + file bukti → berhasil, `kategori` tersimpan benar
- [ ] `PATCH /teacher-permits/:id/bukti` dengan file baru → `bukti_file_path` berubah ke file baru, `bukti_updated_at` ter-update, file lama tidak lagi bisa diakses (ditimpa)
- [ ] Role selain `admin_jurnal` coba `POST /teacher-permits` atau `PATCH .../bukti` → 403
- [ ] Guru submit tugas untuk permit miliknya → berhasil, `submitted_at` terisi
- [ ] Guru B coba submit tugas ke permit milik guru A → 403
- [ ] Upload file `.exe` (baik di `bukti_file` maupun `tugas_file`) → ditolak 400 dengan pesan jelas tipe file tidak didukung
- [ ] Upload file 15MB → ditolak 400, pesan batas ukuran
- [ ] Submit ulang (revisi) sebelum jam sesi lewat → berhasil update, tidak error "sudah submit"

## Handoff ke T047 & T048
T047 (UI dashboard guru — tombol Izin) konsumsi `GET my-active-today` dan `POST submit-tugas`. T048 (UI admin_jurnal) konsumsi `POST /teacher-permits` dan `GET /teacher-permits`.

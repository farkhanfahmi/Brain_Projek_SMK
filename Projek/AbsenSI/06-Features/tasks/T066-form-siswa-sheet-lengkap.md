# T066 — UI: Form Siswa — Migrasi ke Sheet + Field Wali Murid & Status Lulus

## Depends on
T064 (primitif Sheet/Tabs), T063 (schema field wali murid & status lulus baru sudah ada di backend)

## Objective
Ganti `SiswaForm` existing (popup `Dialog` 11 field) jadi form `Sheet` bertab yang mencakup field tambahan dari T063 (no HP siswa, no HP ayah/ibu, tahun lulus, alasan nonaktif) — form sudah cukup padat sekarang (11 field dalam 1 Dialog kolom tunggal), akan makin tidak proporsional dengan penambahan ini.

## Context
- **App:** `apps/web`
- **File existing:** `apps/web/src/app/(admin)/siswa/siswa-view.tsx` — `SiswaForm` sudah 11 field (`nisn`, `nama`, `kelasId`, `tanggalLahir`, `tempatLahir`, `jenisKelamin`, `agama`, `alamat`, `rtRw`, `namaAyah`, `namaIbu`), semua dalam 1 kolom vertikal di Dialog
- **Ref:** `Projek/AbsenSI/06-Features/design-system/03-components.md` bagian "Form Input Panjang" (WAJIB), `Projek/AbsenSI/06-Features/migrasi-database-lama.md`

## Spec Detail

### Ganti kontainer: `Dialog` → `Sheet`

### Struktur Tab (4 section — lebih banyak dari form Guru karena field lebih padat)
1. **"Data Pokok"** — NISN, Nama, Kelas (dropdown existing, tidak berubah), Status (Aktif/Nonaktif), Alasan Nonaktif (dropdown: Lulus/Mengundurkan Diri/Lainnya — **HANYA muncul/aktif kalau Status = Nonaktif**, disabled/hidden kalau Aktif), Tahun Lulus (number input, **HANYA relevan kalau Alasan Nonaktif = Lulus**, sama logic show/hide)
2. **"Biodata"** — Tempat Lahir, Tanggal Lahir (`DatePicker`), Jenis Kelamin, Agama
3. **"Kontak & Alamat"** — No HP Siswa (BARU), Alamat, RT/RW
4. **"Data Wali"** — Nama Ayah, Nama Ibu, No HP Ayah (BARU), No HP Ibu (BARU)

### Logic Kondisional (Penting)
- Field "Alasan Nonaktif" dan "Tahun Lulus" **disembunyikan atau di-disable** kalau `status = aktif` — tidak masuk akal siswa aktif punya alasan keluar/tahun lulus. Pakai conditional render React biasa berdasarkan state `status` yang dipilih di tab yang sama
- "Tahun Lulus" lebih spesifik lagi hanya relevan kalau `alasanNonaktif = lulus` (siswa yang mengundurkan diri tidak punya "tahun lulus")

### Grid Layout
- Sama seperti T065: grid 2 kolom untuk field pendek per tab, field panjang (Alamat) full-width

## JANGAN
- ❌ JANGAN buat field wali murid/kontak jadi wajib — semua opsional, konsisten dengan data lama yang banyak NULL (lihat sample data di `migrasi-database-lama.md`, banyak field kontak siswa NULL di data lama)
- ❌ JANGAN tampilkan "Alasan Nonaktif"/"Tahun Lulus" tanpa logic kondisional — field ini harus disembunyikan/disabled kalau tidak relevan (status aktif), bukan selalu tampil membingungkan admin
- ❌ JANGAN pindahkan field existing (`kelasId`, dst) ke luar tab "Data Pokok" — field yang sudah ada tetap di grouping yang masuk akal, jangan diacak tanpa alasan
- ❌ JANGAN hapus validasi `required` yang SUDAH ADA di field existing (NISN, Nama, Kelas) — task ini menambah field baru opsional, bukan mengubah validasi field lama yang sudah benar

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/siswa-view.tsx` — ganti `Dialog` jadi `Sheet`, tambah field baru dengan struktur 4 Tab, tambah logic kondisional show/hide

## Acceptance Criteria
- [ ] Klik "Tambah Siswa" → Sheet geser dari kanan, 4 tab berfungsi
- [ ] Pilih Status = Aktif → field Alasan Nonaktif & Tahun Lulus tersembunyi/disabled
- [ ] Pilih Status = Nonaktif → Alasan Nonaktif muncul; pilih Alasan = Lulus → Tahun Lulus ikut muncul; pilih Alasan = Mengundurkan Diri → Tahun Lulus tetap tersembunyi
- [ ] Submit dengan hanya field wajib lama (NISN, Nama, Kelas) → berhasil, konsisten dengan behavior sebelumnya
- [ ] Submit dengan field wali murid baru terisi → tersimpan, terverifikasi lewat MySQL MCP
- [ ] Form EDIT siswa (`apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx`, 424 baris — cek apakah ada form edit di sana) mengikuti pola sama kalau relevan

## Handoff
Setelah T065+T066 selesai, jadikan pola ini rujukan WAJIB untuk semua form input data panjang berikutnya — jangan buat `Dialog` baru untuk form >6 field di halaman manapun ke depan.

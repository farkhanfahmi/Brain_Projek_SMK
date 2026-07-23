# T055 — Rekap Kehadiran: Filter Loopback Tahun Ajaran & Semester

## Depends on
T054 (schema `semesters`), Rekap Admin Fase 1 harus sudah ada (`Projek/AbsenSI/06-Features/rekap-kehadiran.md`, modul existing dari T019)

## Objective
Tambah filter dropdown "Tahun Ajaran" dan "Semester" (opsional) ke halaman Rekap Kehadiran Admin — data historis sudah tersimpan sejak Fase 1 (tidak pernah dihapus), yang belum ada cuma UI untuk memilihnya secara eksplisit.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `academic_years` (existing), `semesters` (baru, T054), `attendance_records` (existing)
- **Role:** `super_admin` (dan siapa pun yang sudah punya akses Rekap Admin dari Fase 1)
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "📚 Semester & Loopback Tahun Ajaran", `Projek/AbsenSI/06-Features/rekap-kehadiran.md`

## Spec Detail

### API: extend endpoint Rekap existing (dari T019, `GET /attendance/rekap` atau nama endpoint aktualnya — cek kode existing)
Tambah query parameter opsional:
- `academic_year_id` — filter rentang tanggal implisit ke `tanggal_mulai`–`tanggal_selesai` tahun ajaran itu (menggantikan/melengkapi filter tanggal manual yang sudah ada — kalau `academic_year_id` diisi, batasi rentang tanggal picker within tahun ajaran itu, JANGAN biarkan admin pilih rentang di luar tahun ajaran yang dipilih)
- `semester_id` — persempit lebih lanjut ke rentang tanggal semester itu (harus valid: `semesters.academic_year_id` cocok dengan `academic_year_id` yang dipilih, kalau keduanya diisi)
- Kalau tidak ada parameter ini sama sekali → behavior tetap seperti sebelumnya (default filter tanggal manual, tanpa scope tahun ajaran) — **backward compatible**, tidak mengubah default existing

### UI: extend halaman Rekap Admin existing (`apps/web/app/(admin)/rekap/page.tsx` atau path aktual dari T019)
- Tambah 2 dropdown baru di atas filter yang sudah ada: "Tahun Ajaran" (list dari `GET /academic-years`, default: yang `is_active: true`) dan "Semester" (opsional, list dari `GET /semesters?academic_year_id=`, muncul setelah tahun ajaran dipilih, ada opsi "Semua Semester")
- Pilih tahun ajaran → date range picker otomatis ter-set ke rentang tahun ajaran itu (tapi admin tetap bisa persempit manual di dalam rentang itu)
- Pilih semester → date range picker ter-set ke rentang semester itu

## JANGAN
- ❌ JANGAN tulis ulang logic hitung alfa/hadir/izin/sakit — task ini HANYA menambah filter/scope tanggal ke endpoint yang sudah ada, bukan membuat endpoint rekap baru
- ❌ JANGAN ubah default behavior kalau `academic_year_id`/`semester_id` tidak dikirim — harus tetap sama seperti sebelum task ini (backward compatible untuk konsumen lain yang mungkin sudah pakai endpoint ini, misal TV Piket nanti)
- ❌ JANGAN izinkan pilih rentang tanggal di luar `academic_year_id` yang dipilih — validasi ini di FE (date picker constraint) DAN BE (reject request kalau rentang tanggal keluar dari batas tahun ajaran yang di-filter, kalau parameter itu dikirim)

## Files
- **Modifikasi:** endpoint Rekap existing di `apps/api/src/attendance/` (cek nama file aktual dari implementasi T019) — tambah parameter opsional
- **Modifikasi:** halaman Rekap Admin di `apps/web/app/(admin)/` (cek path aktual dari T019) — tambah 2 dropdown

## Acceptance Criteria
- [ ] Rekap tanpa parameter tahun ajaran/semester → hasil identik dengan sebelum task ini (regression check)
- [ ] Pilih tahun ajaran lama (bukan yang aktif) → tabel rekap tampilkan data dari rentang tahun ajaran itu, bukan tahun ajaran aktif
- [ ] Pilih tahun ajaran + semester tertentu → rentang tanggal makin sempit sesuai semester
- [ ] Pilih semester yang `academic_year_id`-nya tidak cocok dengan tahun ajaran yang dipilih di dropdown lain → dropdown semester otomatis reset/disable opsi yang tidak valid (tidak mengizinkan kombinasi salah)

# T050 — UI: Dashboard Admin Jurnal — Mapel, Jadwal Mengajar & Config Mode Blok

## Depends on
T047 (API mapel & jadwal admin_jurnal, termasuk `copy-from-semester`), T042 (API config toleransi), T054 (schema & API `semesters` + `block_week_ranges`)

## Objective
Buat dashboard baru khusus role `admin_jurnal`: kelola mapel, assign jadwal mengajar (guru-kelas-mapel-jam-minggu-semester), dan konfigurasi toleransi keterlambatan. (Konfigurasi mode blok/normal dan rentang minggu A/B **bukan lagi di halaman ini** — sudah pindah ke level `semesters`/`block_week_ranges`, lihat T056 untuk UI kalender bloknya.)

## Context
- **App:** `apps/web`
- **Route:** `/admin-jurnal` (route group baru, layout terpisah dari `/admin` existing)
- **Role:** `admin_jurnal`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Role Baru: admin_jurnal", "Jadwal: Mode Blok vs Mode Normal" (**revisi 2026-07-22**, baca ulang)
- **⚠️ Ref WAJIB dibaca sebelum menulis UI:** `Projek/AbsenSI/06-Features/design-system/MASTER.md` + companion files — banner peringatan "lubang jadwal" pakai `--color-danger-bg`/`--color-danger-text` (satu-satunya tempat warna danger tampil persisten di layout, karena ini sinyal operasional serius), semua card/tabel/tombol lain ikut spec `03-components.md`

> **Revisi penting:** halaman "Konfigurasi Jadwal" versi lama (toggle mode global + tanggal acuan) **DIBATALKAN** — mode blok/normal dan rentang minggu A/B sekarang per-semester (`semesters.mode`, dikelola `super_admin`; `block_week_ranges`, dikelola `admin_jurnal` lewat kalender visual di T056). Task ini HANYA menyisakan konfigurasi toleransi keterlambatan sebagai halaman terpisah kecil.

### Layout `apps/web/app/(admin-jurnal)/layout.tsx`
- Sidebar khusus, menu: "Toleransi Keterlambatan", "Mata Pelajaran", "Semester" (read-only), "Jadwal Blok Minggu" (T056), "Jadwal Mengajar", "Izin Guru" (T051), "Wali Kelas" (T052), "Monitor Jurnal" (opsional lanjutan, boleh ditunda)
- **TIDAK** ada menu untuk users/kartu/kalender/rekap siswa — konsisten batasan role. Menu "Semester" HANYA read-only, bukan pengecualian ke batasan ini
- **Banner peringatan global (2026-07-22, celah proaktif):** di layout ini (tampil di SEMUA halaman admin_jurnal, bukan cuma 1 halaman spesifik), panggil `GET /block-week-ranges/upcoming-gaps?hari=7` saat layout dimuat — kalau `ada_lubang_mendekat: true`, tampilkan banner merah mencolok di atas seluruh konten: "⚠️ Ada {N} hari dalam 7 hari ke depan yang jadwal blok-nya belum lengkap — [Lihat Detail]" dengan link ke T056. Ini supaya admin_jurnal tidak perlu ingat buka halaman kalender secara manual untuk sadar ada masalah yang akan terjadi

### Halaman `/admin-jurnal/toleransi`
- Field "Toleransi Keterlambatan Mengajar (menit)" — number input
- Tombol "Simpan" → `PATCH /schedule-config` (T042, sekarang HANYA berisi field ini, lihat revisi T038)
- Halaman ini jauh lebih sederhana dari versi lama — tidak ada lagi toggle mode/tanggal acuan di sini

### Halaman `/admin-jurnal/mapel`
- Tabel sederhana: Nama | Kode | (tidak ada aksi hapus)
- Form create/edit modal: nama (wajib), kode (opsional)
- Pola sama seperti tab "Kampus"/"Jurusan" di T004 (tabel kecil tanpa pagination, CRUD inline)

### Halaman `/admin-jurnal/jadwal`
- **Dropdown "Semester" di paling atas** (WAJIB pilih sebelum tabel/form muncul) — list dari `GET /semesters`, default terpilih: semester yang `is_active: true`. Semua isi halaman di bawah (tabel, form create) di-scope ke `semester_id` yang dipilih ini
- Tabel jadwal mengajar (`schedules` where `type=jam_mengajar AND semester_id={dipilih}`): Guru | Kelas | Mapel | Hari | Jam | Minggu (kolom ini hanya tampil kalau **semester yang dipilih** punya `mode: blok` — cek dari data semester yang dipilih, BUKAN dari config global manapun karena mode sekarang per-semester)
- Filter tambahan: dropdown guru, dropdown kelas (dalam scope semester yang dipilih)
- **Kalau semester yang dipilih `mode: blok` TAPI belum lengkap rentang blok-nya** (`GET /block-week-ranges/coverage-check` return `lengkap: false`) → tampilkan banner peringatan di atas tabel: "Rentang minggu A/B semester ini belum lengkap — lengkapi di halaman Jadwal Blok Minggu sebelum jadwal bisa aktif penuh" dengan link ke T056
- **Kalau semester yang dipilih BELUM punya jadwal sama sekali (tabel kosong)** → tampilkan tombol besar **"Salin Jadwal dari Semester Sebelumnya"** di atas tabel kosong, dengan dropdown pilih semester sumber (default: semester lain yang `is_active` terakhir/terbaru sebelum semester ini) → klik → `POST /schedules/copy-from-semester` → refresh tabel menampilkan hasil salinan
- Form create (modal): pilih guru (autocomplete dari daftar teachers — **read-only lookup**, admin_jurnal tidak bisa create teacher baru dari sini, hanya pilih existing), pilih kelas, pilih mapel, hari, jam mulai-selesai, dan kalau semester yang dipilih `mode: blok` → dropdown Minggu (A/B/Setiap Minggu, wajib dipilih). `semester_id` otomatis dari dropdown semester yang sedang dipilih di atas, tidak perlu dipilih ulang di form
- Submit → `POST /schedules` — kalau backend return 409 (bentrok jadwal), tampilkan pesan jelas termasuk detail schedule yang bentrok dari response

### Halaman `/admin-jurnal/semester` (baru, menu tambahan di sidebar)
- **Read-only** (semester dikelola `super_admin`, bukan `admin_jurnal` — lihat T054) — tabel: Tahun Ajaran | Semester | Tanggal Mulai | Tanggal Selesai | Status (badge "Aktif" untuk yang `is_active: true`)
- Tujuan: admin_jurnal perlu tahu semester apa saja yang tersedia untuk dipilih di halaman Jadwal, tanpa perlu akses ke modul kalender pendidikan penuh
- **Tidak ada tombol create/edit/activate di sini** — kalau admin_jurnal butuh semester baru dibuat atau diaktifkan, itu request ke `super_admin` (di luar sistem, sesuai batasan domain)

## JANGAN
- ❌ JANGAN buat halaman/form untuk create akun guru baru di dashboard ini — admin_jurnal hanya PILIH dari guru yang sudah ada (dibuat oleh super_admin di `/admin/akun`), sesuai batasan "tidak akses modul users"
- ❌ JANGAN tampilkan menu/link ke halaman admin lain (`/admin/akun`, `/admin/kartu`, dst) di sidebar ini — kalau `admin_jurnal` mengetik langsung URL admin lain, itu harus 403 dari middleware/guard routing, BUKAN cuma disembunyikan
- ❌ JANGAN buat kolom "Minggu" jadi wajib tampil kalau mode = normal — sembunyikan sepenuhnya dari form & tabel saat mode normal aktif, supaya tidak membingungkan admin
- ❌ JANGAN implement endpoint DELETE mapel di UI ini — sesuai T047, tidak ada delete
- ❌ JANGAN buat halaman `/admin-jurnal/semester` bisa create/edit/activate — read-only murni, siklus hidup semester tetap wewenang `super_admin` (T054)
- ❌ JANGAN tampilkan tombol "Salin Jadwal" kalau semester yang dipilih SUDAH punya jadwal — hanya muncul untuk semester yang benar-benar kosong, sesuai validasi backend T047

## Files
- **Buat:** `apps/web/app/(admin-jurnal)/layout.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/toleransi/page.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/mapel/page.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/semester/page.tsx` (read-only)
- **Buat:** `apps/web/app/(admin-jurnal)/jadwal/page.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/jadwal/components/jadwal-form-modal.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/jadwal/components/salin-jadwal-modal.tsx`
- **Modifikasi:** middleware routing (`apps/web/middleware.ts` atau setara) — tambah proteksi route `/admin-jurnal/*` untuk role `admin_jurnal` saja, redirect role lain

## Acceptance Criteria
- [ ] Login sebagai `admin_jurnal` → landing default ke halaman utama dashboard ini
- [ ] Login sebagai `admin_jurnal`, coba akses `/admin/akun` langsung via URL → redirect/403, bukan cuma sembunyi menu
- [ ] Pilih semester `mode: blok` di dropdown jadwal → form jadwal menampilkan field Minggu; pilih semester `mode: normal` → field Minggu tersembunyi
- [ ] Simpan toleransi keterlambatan → tersimpan, terbaca kembali saat halaman dibuka ulang
- [ ] Buat mapel baru → muncul di dropdown mapel saat assign jadwal
- [ ] Semester `mode: blok` dengan rentang blok belum lengkap → banner peringatan muncul di halaman Jadwal
- [ ] Assign jadwal guru yang bentrok jam (dalam semester yang sama) → pesan error jelas dari response 409 ditampilkan, form tidak ke-submit
- [ ] Pilih semester yang belum ada jadwalnya → tombol "Salin Jadwal dari Semester Sebelumnya" muncul; pilih semester yang sudah ada jadwalnya → tombol ini tidak muncul
- [ ] Klik salin jadwal → tabel terisi hasil salinan dengan `semester_id` baru, jadwal semester sumber tidak berubah
- [ ] Halaman `/admin-jurnal/semester` menampilkan daftar semester, tidak ada tombol create/edit/activate di UI
- [ ] Login sebagai `super_admin` mencoba akses `/admin-jurnal/*` → **403/redirect**, sama seperti role lain. Route ini eksklusif `admin_jurnal` — akses read-only `super_admin` ke jurnal (sesuai spec "Hak Akses Lihat Jurnal") adalah fitur TERPISAH yang belum dispec detail UI-nya (bukan scope task ini, jangan diimplementasikan sebagai akses ke dashboard ini)

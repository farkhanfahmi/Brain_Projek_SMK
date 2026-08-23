# T210 — Web: Menu Utama "Jadwal Pelajaran" (Hierarki Tahun Ajaran → Semester → Opsi Jadwal)

## Depends on
**WAJIB SETELAH T206, T208** (butuh backend CRUD Opsi Jadwal + Date Generator sudah ada). Bagian dari rangkaian T203-T215.

## Objective
Halaman BARU "Jadwal Pelajaran" — menggantikan halaman "Jadwal Mengajar" (T157/T189) — dengan hierarki: dropdown Tahun Ajaran (default aktif) → dropdown Semester (default aktif sesuai tahun ajaran dipilih) → daftar Opsi Jadwal (bisa banyak, masing-masing toggle Aktif/Nonaktif sendiri, Warning Banner kalau aktif tapi induk tidak aktif) → klik 1 Opsi Jadwal masuk Workspace (T212).

## Konteks — Keputusan Dikonfirmasi User (2026-08-16)

- Warning Banner (KUNING) WAJIB tampil kalau Opsi Jadwal `isActive: true` TAPI Semester ATAU Tahun Ajaran induknya SEDANG TIDAK AKTIF — bunyi PERSIS pola dari referensi: *"Perhatian: Opsi jadwal ini aktif, namun disembunyikan dari pengguna karena Semester Induk sedang tidak aktif."* (SESUAIKAN teks kalau yang tidak aktif Tahun Ajaran, bukan Semester — pesan HARUS jelas MANA yang jadi penyebab).
- Mode Opsi Jadwal (Normal/Blok) DIPILIH SAAT create, PERMANEN, TIDAK BISA diubah.

## Spec Detail

### 1. Frontend — halaman utama

- Path BARU `apps/web/src/app/(admin)/jadwal-pelajaran/` (dan duplikat `(admin-jurnal)/admin-jurnal/jadwal-pelajaran/`, KONSISTEN pola T157 — REUSE komponen yang SAMA).
- **Header filter**: dropdown Tahun Ajaran (default `isActive: true`), dropdown Semester (default `isActive: true` DALAM tahun ajaran yang dipilih — VERIFIKASI: kalau tahun ajaran yang dipilih BUKAN yang aktif, apakah tetap ada 1 semester "default" yang dipilihkan otomatis, atau kosong sampai admin pilih manual — REKOMENDASI: pilih semester PERTAMA dalam tahun ajaran itu kalau tidak ada yang aktif, JANGAN biarkan dropdown Semester kosong tanpa pilihan).
- **Daftar Opsi Jadwal** untuk kombinasi Tahun Ajaran+Semester yang dipilih — tabel/card list: Nama Opsi, Mode (badge Normal/Blok), Cakupan Tingkat, Toggle Aktif/Nonaktif, tombol Hapus (disabled kalau `isActive: true` atau ada JadwalSlot terkait, KONSISTEN validasi T206).
- **Warning Banner** — tampil di ATAS list kalau ADA Opsi Jadwal `isActive: true` yang induknya (Semester/Tahun Ajaran) tidak aktif — SEBUTKAN NAMA Opsi Jadwal yang terdampak, JANGAN generic.
- **Tombol "Buat Opsi Jadwal Baru"** — buka form (Dialog/Sheet, PUTUSKAN saat implementasi berdasarkan jumlah field — nama, semester [sudah dari konteks dropdown], alokasi waktu [dropdown pilih dari yang sudah ada], mode [radio Normal/Blok, WARNING teks jelas "tidak bisa diubah setelah dibuat"], cakupan tingkat [checklist opsional]) — REKOMENDASI Dialog cukup (field tidak terlalu banyak, ≤6 field).
- **Klik baris Opsi Jadwal** (bukan tombol Hapus/Toggle) → navigasi ke Workspace (`T212`, path `/jadwal-pelajaran/[opsiJadwalId]`).

### 2. Frontend — halaman kelola Alokasi Waktu (Full Page, T158 lama pindah dari Sheet)

- Path BARU `apps/web/src/app/(admin)/alokasi-waktu/` — list Alokasi Waktu (ringkas), tombol "Buat Baru"/"Edit" → **navigasi ke halaman TERPISAH** `apps/web/src/app/(admin)/alokasi-waktu/baru/` dan `apps/web/src/app/(admin)/alokasi-waktu/[id]/` (Full Page View, BUKAN Sheet lagi — dikonfirmasi user, tabel Senin-Sabtu terlalu sempit di Sheet 480-560px).
- Form Full Page — tabel per hari (Senin-Sabtu, Sabtu opsional/boleh kosong), tiap baris: Jam Ke (angka, kosongkan untuk istirahat + WAJIB isi Keterangan kalau kosong, KONSISTEN validasi T206), Jam Mulai, Jam Selesai, Keterangan. Tombol "Tambah Baris"/"Hapus Baris" per hari.

## Edge Cases
- Tahun Ajaran/Semester yang TIDAK PUNYA Opsi Jadwal sama sekali — empty state jelas ("Belum ada Opsi Jadwal untuk semester ini — buat yang baru").
- Alokasi Waktu yang SEDANG DIRUJUK Opsi Jadwal manapun — edit-nya DIBATASI (T206: tolak hapus jamKe yang terpakai) — UI HARUS tampilkan pesan error itu APA ADANYA dari backend (bukan generic "gagal simpan").

## Files
- **Buat:** `apps/web/src/app/(admin)/jadwal-pelajaran/` (+duplikat admin-jurnal), `apps/web/src/app/(admin)/alokasi-waktu/` (+duplikat admin-jurnal, list+form full page).
- **Jangan sentuh:** halaman "Jadwal Mengajar" LAMA (T189, `(admin)/jadwal-mengajar/`) — TIDAK dihapus di task ini, akan di-redirect/dihapus di **T215** SETELAH semua fitur baru terverifikasi berfungsi.

## Acceptance Criteria
- [x] Hierarki dropdown Tahun Ajaran → Semester berfungsi, default ke yang aktif (fallback semester pertama kalau tidak ada yang aktif dalam tahun ajaran terpilih).
- [x] List Opsi Jadwal tampil dengan toggle aktif, badge mode, cakupan tingkat, nama Alokasi Waktu terkait.
- [x] Warning Banner tampil TEPAT kondisi (Opsi aktif tapi induk Semester ATAU Tahun Ajaran tidak aktif), sebutkan nama Opsi + penyebab spesifik.
- [x] Buat Opsi Jadwal baru — mode WAJIB dipilih saat create (radio Normal/Blok), form menjelaskan permanen (teks danger eksplisit).
- [x] Halaman Alokasi Waktu Full Page (bukan Sheet) berfungsi — 3 route (list, `baru/`, `[id]/`), Sabtu opsional (boleh 0 baris).
- [x] Klik baris Opsi Jadwal navigasi ke `[basePath]/[opsiJadwalId]` — path Workspace (T212) sudah benar, halamannya sendiri belum dibuat (task terpisah, di luar scope T210).
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi Warning Banner SEBUTKAN NAMA Opsi Jadwal spesifik — teks: `Perhatian: Opsi Jadwal "{nama}" aktif, namun disembunyikan dari pengguna karena {Semester Induk|Tahun Ajaran Induk} sedang tidak aktif.` — penyebab dibedakan (Semester dicek dulu, baru Tahun Ajaran, sesuai urutan hierarki).
- [x] Konfirmasi form Buat Opsi Jadwal Baru — teks danger eksplisit di bawah pilihan mode: "Mode TIDAK BISA diubah setelah Opsi Jadwal ini dibuat — kalau salah pilih, harus buat Opsi Jadwal baru."

## Catatan Implementasi (2026-08-17)

- **Dependency terpenuhi tanpa hambatan**: sesi paralel sudah menyelesaikan T206 (`OpsiJadwalService`/`AlokasiWaktuService`/`JadwalSlotService` lengkap dengan controller+DTO+test), T208 (Date Generator), dan bahkan T209 (`TeachingSession.jadwalSlotId`) SEBELUM T210 dimulai — T210 murni mengonsumsi API yang sudah stabil, tidak ada blocking wait.
- **Pola `basePath` prop**: `AlokasiWaktuListView`, `AlokasiWaktuForm`, `AlokasiWaktuEditView`, `JadwalPelajaranView` semua terima `basePath` opsional (default ke path admin biasa) — supaya duplikasi route admin-jurnal (T157) REUSE component yang SAMA PERSIS tanpa hardcode 1 path, konsisten filosofi "component generic, bukan asumsi 1 route" yang sudah dipakai `MapelView` dkk.
- **Warning Banner warna**: dipakai token `status-shipped` (amber, SUDAH disetujui design-system sebagai kategori intermediate-severity untuk tabel/banner workflow non-binary) — BUKAN aksen warna baru, sesuai larangan eksplisit "tidak ada aksen kedua" di DESIGN.md.
- **Form Buat Opsi Jadwal tetap Dialog** (bukan Sheet) — 5 field (nama, alokasi waktu, mode, cakupan tingkat — semester sudah dari konteks dropdown), sesuai aturan design-system "≤6 field → Dialog kecil tetap dipertahankan".
- **Sidebar**: 2 entry baru ("Jadwal Pelajaran", "Alokasi Waktu") ditambahkan ADDITIF ke `nav-items.ts` (admin) dan `admin-jurnal-sidebar.tsx`, SEJAJAR "Jadwal Mengajar" lama (tidak menggantikan) — dikonfirmasi sesuai spec task, penghapusan menu lama menunggu T215.
- **T212 (Workspace) belum dibuat**: klik baris Opsi Jadwal navigasi ke `[basePath]/[opsiJadwalId]`, path-nya sudah benar sesuai spec tapi halaman tujuannya sendiri di luar scope T210 (akan 404 sampai T212 dikerjakan) — sesuai instruksi task, tidak diperluas scope.

### Verifikasi

- `tsc --noEmit` bersih, `next build` sukses — 8 route baru terkonfirmasi di output (`/jadwal-pelajaran`, `/alokasi-waktu`, `/alokasi-waktu/baru`, `/alokasi-waktu/[id]` ×2 untuk admin+admin-jurnal).
- `jest apps/api`: 551/551 pass (naik dari 534, T206/T208/T209 lanjut menambah test paralel) — task ini TIDAK mengubah backend sama sekali, regresi nol dijamin otomatis.
- Smoke test via curl: 5 route utama (`/jadwal-pelajaran`, `/alokasi-waktu`, `/alokasi-waktu/baru`, `/admin-jurnal/jadwal-pelajaran`, `/admin-jurnal/alokasi-waktu`) semua 307 (redirect login, bukan 500) — dev server dikonfirmasi restart bersih setelah `next build` (pola insiden berulang di sesi ini, `.next` cache dev server ketimpa production build).
- `git status`/`git diff` dikonfirmasi scope T210 hanya file baru + `nav-items.ts`/`admin-jurnal-sidebar.tsx`/`core-types.ts`, tidak ada overlap dengan file T206.

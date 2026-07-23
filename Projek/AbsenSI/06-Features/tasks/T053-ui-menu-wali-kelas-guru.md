# T053 — UI: Menu Wali Kelas di Dashboard Guru

## Depends on
T052 (kelas_id_wali harus bisa di-assign), T045 (data jurnal & presensi kelas untuk breakdown per mapel), T046 (data teacher_permits untuk daftar sesi "Guru Izin")

## Objective
Tambahkan menu "Wali Kelas" ke sidebar dashboard guru — muncul HANYA untuk akun guru yang `kelas_id_wali`-nya terisi — berisi ringkasan kehadiran kelas, rekap per mapel, dan catatan/kendala dari guru mapel lain. Read-only murni.

## Context
- **App:** `apps/web` + `apps/api`
- **Route:** `/guru/wali-kelas`
- **Role:** `guru` dengan `kelas_id_wali IS NOT NULL`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "🏫 Wali Kelas", terutama "Isi Menu Wali Kelas (v1 — final, siap eksekusi)" — baca detail 3 sub-bagiannya
- **⚠️ Ref WAJIB dibaca sebelum menulis UI:** `Projek/AbsenSI/06-Features/design-system/MASTER.md` + companion files — 3 tab di halaman ini pakai Card/Table/Badge spec dari `03-components.md`, konsisten dengan halaman Rekap Admin yang di-reuse (Tab 1)

## Spec Detail

### Sidebar (extend `apps/web/app/(guru)/layout.tsx` dari T048)
- Kalau `req.user.kelas_id_wali` terisi (dari JWT payload) → tampilkan menu tambahan "Wali Kelas" di sidebar guru, link ke `/guru/wali-kelas`
- Kalau tidak terisi → menu ini tidak muncul sama sekali (bukan disabled, benar-benar tidak ada)
- **Proteksi route wajib di backend juga**: kalau guru tanpa `kelas_id_wali` mengetik URL `/guru/wali-kelas` langsung → redirect/403, jangan cuma sembunyi di sidebar

### Halaman `/guru/wali-kelas` — 3 tab/section

**Tab 1: Ringkasan Kehadiran Kelas**
- **API: `GET /rekap/kelas-wali`** (endpoint baru, TAPI wajib reuse service Rekap Admin yang sudah ada dari Fase 1 — JANGAN tulis ulang logic hitung alfa/hadir/izin/sakit)
  - `kelas_id` dari `req.user.kelas_id_wali` (JWT), BUKAN dari query param — cegah guru akses kelas lain lewat manipulasi URL
  - Filter: rentang tanggal (default bulan berjalan)
  - Response sama bentuknya dengan Rekap Admin (`Projek/AbsenSI/06-Features/rekap-kehadiran.md`): angka ringkas hadir/izin/sakit/alfa + tabel per siswa
- UI: sama persis pola tampilan Rekap Admin (kalau UI Rekap Admin sudah ada dari Fase 1, styling & komponen bisa di-reuse), tapi tanpa filter kelas/jurusan (sudah terkunci ke 1 kelas)

**Tab 2: Rekap Per Mapel**
- **API: `GET /class-attendance/kelas-wali-breakdown`** (endpoint baru)
  - Untuk tiap siswa di kelas: hitung jumlah `class_attendance_marks.status = tidak_ada_di_kelas` per `mapel_id`, dalam rentang tanggal filter
  - Response: per siswa, breakdown count per mapel — misal `{ student: "Budi", breakdown: [{ mapel: "Matematika", count: 3 }, { mapel: "Pemrograman Web", count: 1 }] }`
- **API: `GET /teacher-permits/kelas-wali`** (endpoint baru)
  - List `teacher_permits` yang `session_id` mengarah ke sesi dengan `kelas_id = req.user.kelas_id_wali`, dalam rentang tanggal filter
  - Response per baris: tanggal, mapel, guru, status tugas (`submitted_at` terisi/tidak), link/preview file kalau ada
- UI: 2 sub-tabel dalam 1 tab — tabel breakdown bolos per mapel, lalu tabel daftar sesi "Guru Izin"

**Tab 3: Catatan/Kendala dari Guru Mapel**
- **API: `GET /journal/kelas-wali-catatan`** (endpoint baru)
  - Query `journal_entries` untuk sesi dengan `kelas_id = req.user.kelas_id_wali`, **SELECT HANYA kolom `catatan`** (bukan `materi`/`tujuan_pembelajaran`/`tugas_penilaian`) yang tidak null, urut tanggal terbaru dulu
  - Response per baris: tanggal, mapel, nama guru, isi `catatan`
- UI: list kronologis sederhana (bukan tabel kompleks) — tanggal, mapel+guru sebagai heading kecil, isi catatan sebagai body text

## JANGAN
- ❌ JANGAN tulis ulang logic hitung alfa/rekap — panggil ulang service yang sama dipakai Rekap Admin (cek `apps/api/src/attendance/` atau modul rekap yang sudah ada dari Fase 1), cukup tambah parameter/guard baru untuk scope by `kelas_id_wali`
- ❌ JANGAN tampilkan `materi`/`tujuan_pembelajaran`/`tugas_penilaian` di Tab 3 — HANYA `catatan`. Kalau butuh field lengkap nanti, itu perubahan terpisah (sudah dicatat di spec sebagai "dicatat untuk masa depan")
- ❌ JANGAN buat SATU PUN tombol edit/aksi tulis di halaman ini — read-only murni, sesuai keputusan eksplisit ("Wali kelas... TIDAK ada aksi tulis apapun")
- ❌ JANGAN ambil `kelas_id` dari query param di endpoint manapun task ini — selalu dari `req.user.kelas_id_wali` (JWT), sama prinsip dengan scope `guru_piket.kampus_id` dan `teacher_id` di riwayat guru (T021)
- ❌ JANGAN buat endpoint ini bisa diakses guru yang `kelas_id_wali IS NULL` — 403 di backend, bukan cuma disembunyikan di FE

## Files
- **Modifikasi:** `apps/web/app/(guru)/layout.tsx` — tambah menu conditional "Wali Kelas"
- **Buat:** `apps/web/app/(guru)/guru/wali-kelas/page.tsx`
- **Buat:** `apps/web/app/(guru)/guru/wali-kelas/components/ringkasan-kehadiran-tab.tsx`
- **Buat:** `apps/web/app/(guru)/guru/wali-kelas/components/rekap-mapel-tab.tsx`
- **Buat:** `apps/web/app/(guru)/guru/wali-kelas/components/catatan-tab.tsx`
- **Modifikasi:** `apps/api/src/attendance/` (atau modul rekap Fase 1 yang relevan) — tambah endpoint `GET /rekap/kelas-wali` yang reuse service existing dengan guard scope `kelas_id_wali`
- **Modifikasi:** `apps/api/src/journal/` — tambah endpoint `GET /class-attendance/kelas-wali-breakdown`, `GET /teacher-permits/kelas-wali`, `GET /journal/kelas-wali-catatan`
- **Modifikasi:** JWT payload generation (`apps/api/src/auth/`) — pastikan `kelas_id_wali` ikut ter-include di token guru (kalau belum otomatis dari T052)

## Acceptance Criteria
- [ ] Guru tanpa `kelas_id_wali` → menu "Wali Kelas" tidak muncul di sidebar, akses URL langsung → 403/redirect
- [ ] Guru dengan `kelas_id_wali` terisi → menu muncul, Tab 1 tampilkan rekap kehadiran HANYA kelas yang diampu
- [ ] Angka rekap Tab 1 cocok dengan hasil query manual/Rekap Admin untuk kelas yang sama (memverifikasi reuse logic yang sama, bukan implementasi terpisah yang bisa beda hasil)
- [ ] Tab 2 tampilkan siswa yang sering ditandai `tidak_ada_di_kelas` di mapel tertentu, plus daftar sesi guru izin di kelas tsb
- [ ] Tab 3 HANYA tampilkan isi field `catatan`, tidak ada field materi/tujuan/tugas yang bocor ke response maupun UI
- [ ] Tidak ada tombol edit/hapus/aksi tulis apapun di ketiga tab
- [ ] Guru A (wali kelas X) tidak bisa lihat data kelas Y lewat manipulasi apapun di request (uji coba ubah parameter request manual kalau ada celah)

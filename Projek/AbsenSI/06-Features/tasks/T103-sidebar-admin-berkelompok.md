# T103 — UI: Sidebar Admin Berkelompok (Accordion) + Pisah Upload Foto Siswa/Guru

## Depends on
Tidak ada secara teknis. Konsisten pola dengan [[06-Features/tasks/T097-sidebar-guru-berkelompok|T097]] (sidebar guru) dan [[Projek/AbsenSI/TASKS-POLISH-3|T099a]] (sidebar piket) — **reuse komponen accordion yang sama** kalau salah satu dari itu sudah dikerjakan lebih dulu, jangan bikin pola accordion berbeda-beda di 3 sidebar berbeda.

## Objective
Kelompokkan 16 menu flat sidebar admin (`apps/web/src/components/shell/nav-items.ts`) jadi 6 grup accordion collapsible, DAN pisahkan halaman "Upload Foto" (sekarang 1 halaman dengan tab Siswa/Guru) jadi 2 menu terpisah — foto siswa masuk grup Siswa, foto guru masuk grup Guru.

## Context
- **App:** `apps/web`
- Diskusi 2026-07-31 (lanjutan diskusi Manajemen Akun berbasis section, [[06-Features/tasks/T104-akun-section-berbasis-role|T104]]) — user diminta pendapat soal pengelompokan menu admin, hasil diskusi di bawah.
- **File nav existing**: `apps/web/src/components/shell/nav-items.ts` — 14 item `primaryNav` + 2 item `secondaryNav`, semua FLAT tanpa pengelompokan sama sekali.
- **Halaman Upload Foto existing**: `apps/web/src/app/(admin)/foto/foto-view.tsx` — SUDAH punya struktur `Tabs` dengan `TabsContent value="siswa"` dan `value="guru"` masing-masing merender `<UploadPanel identifierLabel="NISN siswa" .../>` dan `<UploadPanel identifierLabel="NIY guru" .../>` (baris ±155-183) — pemisahan ini LEBIH MUDAH dari perkiraan awal karena logic sudah terpisah lewat tab, tinggal dipecah jadi 2 route berbeda, bukan ditulis ulang dari nol.

## Keputusan Final (dikonfirmasi user 2026-07-31)

### Pengelompokan 6 grup:
| Grup | Menu di dalamnya |
|---|---|
| **Master Data Sekolah** | Kampus, Kelas & Jurusan, Kalender |
| **Siswa** | Siswa, PKL Siswa, **Upload Foto Siswa** (baru, pecahan dari `/foto`) |
| **Guru** | Guru, **Upload Foto Guru** (baru, pecahan dari `/foto`) |
| **Kartu & Perangkat** | Kartu, Mesin (Kiosk) |
| **Absensi & Rekap** | Rekap Kehadiran, Jadwal, Jadwal Piket |
| **Ekstrakurikuler** | Daftar Ekstrakurikuler, Monitoring Ekstra |
| *(secondary, TIDAK berubah, tetap terpisah di bawah seperti sekarang)* | Manajemen Akun, Log Aktivitas |

### Bentuk visual: **Accordion collapsible**, KONSISTEN dengan pola T097 (sidebar guru) — grup yang match halaman aktif (`pathname`) default terbuka, grup lain default tertutup, state accordion TIDAK disimpan (reset per navigasi, ditentukan murni dari `pathname` — pola sama T097, JANGAN pakai localStorage).

### Perubahan halaman Upload Foto (BARU, disepakati 2026-07-31)
- Pisahkan `apps/web/src/app/(admin)/foto/` (1 halaman, 2 tab) jadi **2 route terpisah**: kemungkinan `apps/web/src/app/(admin)/siswa/foto/` (atau `/foto/siswa`) dan `apps/web/src/app/(admin)/guru/foto/` (atau `/foto/guru`) — **putuskan struktur URL final saat implementasi**, pertimbangkan konsistensi dengan pola nested route yang sudah ada di codebase (misal `/siswa/pkl` sebagai sub-halaman `/siswa`, kemungkinan `/siswa/foto` lebih konsisten daripada `/foto/siswa`).
- Server Component `page.tsx` masing-masing HANYA fetch data yang relevan untuk foto siswa ATAU foto guru (BUKAN keduanya sekaligus seperti sekarang, `Promise.all([studentsPage, allStudents, teachers])` di `foto/page.tsx` baris ±16-20 fetch SEMUA termasuk `teachers` walau halaman itu foto siswa — pisahkan supaya masing-masing cuma fetch datanya sendiri).
- Komponen `UploadPanel` (dipakai kedua tab, lihat kode existing) — REUSE APA ADANYA di kedua halaman baru, JANGAN duplikasi kodenya. `FotoView` yang sekarang berisi Tabs perlu dipecah jadi 2 komponen (`FotoSiswaView`/`FotoGuruView` atau nama serupa), masing-masing render `UploadPanel` + bagian daftar/laporan yang relevan (cek bagian bawah file `foto-view.tsx` untuk `StatCard`, laporan hasil upload, `AssignPicker` dropdown assign manual — semua ini perlu dipisah per konteks siswa/guru, bukan sekadar dihapus tab-nya doang).

## Files
- **Modifikasi:** `apps/web/src/components/shell/nav-items.ts` — restrukturisasi jadi grup (pola `NavGroup` sama seperti T097).
- **Modifikasi:** `apps/web/src/components/shell/sidebar.tsx` — render grup accordion (reuse komponen dari T097 kalau sudah ada, JANGAN duplikasi implementasi accordion).
- **Buat:** 2 route baru untuk foto siswa/guru (path final diputuskan saat implementasi), pecahan dari `apps/web/src/app/(admin)/foto/`.
- **Hapus (setelah migrasi selesai):** `apps/web/src/app/(admin)/foto/` (route lama) — pastikan tidak ada link/redirect lama yang masih mengarah ke `/foto` sebelum dihapus (grep dulu).
- **Jangan sentuh:** `apps/web/src/app/(admin)/photos/...` backend (`apps/api/src/photos/`) — tidak ada perubahan API, murni restrukturisasi halaman+navigasi frontend.

## Acceptance Criteria
- [ ] Sidebar admin menampilkan 6 grup accordion + 2 menu secondary (tidak berubah).
- [ ] Grup yang match halaman aktif otomatis terbuka, grup lain tertutup, konsisten pola T097.
- [ ] Menu "Upload Foto" lama hilang, tergantikan "Upload Foto Siswa" (di grup Siswa) dan "Upload Foto Guru" (di grup Guru), masing-masing berfungsi identik dengan tab yang sebelumnya ada (upload, laporan hasil, assign manual).
- [ ] Tidak ada link mati ke `/foto` (route lama) di tempat lain dalam kode.
- [ ] Build + type-check `apps/web` hijau.
- [ ] Verifikasi visual (Playwright/manual) — upload foto siswa dan guru masing-masing tetap berfungsi normal setelah dipecah.

## Validasi Claudian
- [ ] Reuse komponen accordion T097/T099a — cek dulu apakah salah satunya sudah dikerjakan sebelum menulis implementasi accordion baru dari nol.
- [ ] Grep semua referensi `/foto` (route lama) di codebase sebelum menghapusnya — termasuk kemungkinan link dari halaman lain, breadcrumb, atau redirect.
- [ ] `foto-view.tsx` (561 baris) punya logic lebih dari sekadar tab (assign-picker, laporan, stat card) — baca SELURUH file dulu sebelum memutuskan cara memecahnya, jangan asumsikan dari potongan kode yang sudah dilihat di sesi diskusi ini saja.

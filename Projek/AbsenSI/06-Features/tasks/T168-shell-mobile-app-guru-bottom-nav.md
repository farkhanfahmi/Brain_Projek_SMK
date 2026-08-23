# T168 — Web: Shell Mobile-App Guru (Bottom Navbar + Drawer Profil)

## Depends on
Tidak ada dependency teknis. **WAJIB dikerjakan PERTAMA** dari rangkaian T168-T173 (Jurnal Guru) — semua task lain menempel ke shell ini.

## Objective
Ganti total shell `(guru)/*` dari sidebar desktop (`guru-sidebar.tsx`) jadi pola mobile-app: **bottom navbar 5 menu** (Jadwal, Presensi, Jurnal Mengajar, Nilai, Riwayat) + **top bar** dengan judul halaman aktif (kiri) dan avatar profil (kanan) yang saat diklik membuka **drawer/sheet** berisi menu tambahan (Wali Kelas, Ekstrakurikuler — kondisional) + Logout.

## Context — Keputusan Diskusi (2026-08-13)

Riset kode mengonfirmasi: TIDAK ADA preseden pola "bottom navbar mobile app" di codebase (`apps/web` maupun `apps/kiosk`) — semua route group (`(guru)`, `(admin)`, `(piket)`, dst) pakai sidebar collapsible (`components/shell/sidebar.tsx`). Task ini adalah **pola UI pertama** jenisnya di proyek, jadi TIDAK ADA komponen shell existing yang bisa direuse langsung — perlu dibangun dari nol (boleh reuse primitif shadcn/ui seperti `Sheet`/`Drawer`/`Avatar` yang sudah ada di `packages/ui`).

**Keputusan yang sudah difinalkan bersama user**:
- Redesign TOTAL — bukan modul paralel. `(guru)/*` existing (`guru-sidebar.tsx`, layout lama) DIGANTI, bukan didampingi 2 versi.
- Avatar profil di **KANAN ATAS** top bar (konvensi mobile app dominan — Gojek/Instagram/WhatsApp/Google), BUKAN kiri. Kiri atas dicadangkan untuk judul halaman aktif.
- Drawer/sheet yang muncul saat avatar diklik berisi: foto+nama+role, lalu menu kondisional (Wali Kelas kalau `kelasIdWali` terisi, Ekstrakurikuler kalau `isPembinaEkstra`), lalu Logout paling bawah.
- Gate akses fitur TETAP di level "mulai sesi" (`sudahTapGerbang` di `TeachingSessionsService`, sudah ada dan TIDAK diubah) — BUKAN gate di level LOGIN aplikasi. Guru tetap bisa login & lihat semua menu kapan saja; yang di-gate hanya tombol "Mulai Belajar".

## Spec Detail

### 1. Layout baru `apps/web/src/app/(guru)/layout.tsx`

- Guard role TETAP SAMA (`user.role !== "guru"` → redirect) — TIDAK diubah, HANYA shell di dalamnya yang diganti.
- Fetch `GET /users/me` (untuk `isPembinaEkstra`) TETAP dipanggil sama seperti sekarang — dilempar sebagai props ke shell baru.
- Struktur baru: `<MobileShell>` (client component baru) yang membungkus: top bar (fixed top) + `{children}` (scrollable, padding bawah cukup untuk bottom nav) + bottom nav (fixed bottom).

### 2. Komponen baru — Bottom Navbar

- File baru `apps/web/src/app/(guru)/components/bottom-nav.tsx` — 5 item fixed: Jadwal (`/guru/jadwal`), Presensi (`/guru/presensi`), Jurnal (`/guru/jurnal-mengajar` — **PERHATIAN nama path**: route lama `/guru/jurnal` akan BERUBAH MAKNA di T169, lihat catatan di bawah), Nilai (`/guru/nilai`), Riwayat (`/guru/riwayat` — path lama, TETAP dipakai).
- Tiap item: icon (lucide-react, pilih yang representatif — Calendar/ClipboardCheck/BookOpen/GraduationCap/History atau serupa, konsisten style icon lain di proyek) + label singkat + highlight state aktif (bandingkan `usePathname()`).
- Style: fixed bottom, full width, background solid (bukan transparan), shadow atas tipis untuk pemisah dari konten, WAJIB ikuti `DESIGN.md`/vault design-system (beige `#EEE6D9` + aksen oranye `#F5841F` tunggal untuk item aktif, radius, dll — BACA vault `06-Features/design-system/{MASTER,03-components}.md` SEBELUM styling, terutama untuk pola "tab bar" kalau ada rujukannya, atau adaptasi dari pola button/badge yang sudah dijelaskan).

### 3. Komponen baru — Top Bar + Drawer Profil

- File baru `apps/web/src/app/(guru)/components/top-bar.tsx` — kiri: judul halaman aktif (terima sebagai prop dari tiap page, ATAU derive dari pathname ke label statis — putuskan yang lebih simpel saat implementasi). Kanan: `Avatar` (foto profil kalau ada field fotonya di `User`/`Teacher`, atau inisial nama sebagai fallback — VERIFIKASI dulu apakah ada field foto, kalau tidak ada cukup inisial, JANGAN tambah field foto baru di luar scope task ini).
- Klik avatar → buka `Sheet`/`Drawer` (shadcn/ui, cek `packages/ui` primitif apa yang sudah ada dan reuse) dari sisi kanan atau bawah (putuskan sesuai konsistensi shadcn default) — isi:
  1. Header: foto/inisial + nama + role "Guru" (+ nama mapel utama kalau relevan/tersedia, opsional).
  2. Menu "Wali Kelas" — HANYA tampil kalau `kelasIdWali` terisi (reuse logic `isWaliKelas` yang sudah ada di layout lama), link ke `/guru/wali-kelas` (path lama, TIDAK berubah).
  3. Menu "Ekstrakurikuler" — HANYA tampil kalau `isPembinaEkstra` true (reuse logic existing), link ke `/guru/ekstrakurikuler` (path lama, TIDAK berubah, termasuk sub-halaman `peserta`/`setting`/`sesi/[sesiId]` — TIDAK disentuh sama sekali, hanya PINTU MASUKNYA yang pindah dari sidebar section ke drawer).
  4. Tombol "Keluar" (logout) — reuse logic logout existing (cari implementasinya di sidebar lama sebelum dihapus).

### 4. Hapus shell lama

- `apps/web/src/app/(guru)/guru-sidebar.tsx` — DIHAPUS setelah `top-bar.tsx`+`bottom-nav.tsx` terbukti menggantikan semua fungsinya (termasuk logic `isWaliKelas`/`isPembinaEkstra` kondisional, logout).
- Pastikan TIDAK ADA import lain yang masih merujuk `guru-sidebar.tsx` sebelum dihapus (grep dulu).

## Edge Cases
- **REVISI 2026-08-15** (semula "shell mobile-app TETAP dipakai di desktop, bukan hybrid" — user KEMUDIAN eksplisit minta hybrid): Halaman guru dibuka di layar desktop lebar (>=768px, breakpoint `md` Tailwind) SEKARANG menampilkan sidebar desktop statis (`guru-sidebar-desktop.tsx`, pola sama `guru-sidebar.tsx` lama yang sempat dihapus, ditambah 3 menu baru Jadwal/Presensi/Nilai) + top bar reuse `TopBarWithTitle`/`TopBar` existing (dropdown avatar, BUKAN drawer Sheet). Di bawah 768px tetap shell mobile-app (bottom-nav+top-bar+drawer) seperti versi awal task ini. Breakpoint TUNGGAL, kedua shell dirender sekaligus di DOM disembunyikan via Tailwind `hidden`/`md:hidden` (bukan render kondisional JS).
- Konten halaman yang panjang (misal kalender presensi T170) — pastikan padding bawah cukup supaya konten TERAKHIR tidak ketutup bottom nav (masalah klasik fixed-bottom-nav).
- User TIDAK punya `kelasIdWali` DAN TIDAK `isPembinaEkstra` — drawer tetap muncul, cukup tanpa 2 menu kondisional itu, langsung Header→Logout (TIDAK boleh drawer kosong/error).

## Files
- **Buat:** `apps/web/src/app/(guru)/components/bottom-nav.tsx`, `apps/web/src/app/(guru)/components/top-bar.tsx`, `apps/web/src/app/(guru)/components/profile-drawer.tsx`, `apps/web/src/app/(guru)/components/guru-shell.tsx` (penggabung, awalnya `mobile-shell.tsx`, di-rename saat revisi hybrid), `apps/web/src/app/(guru)/components/guru-sidebar-desktop.tsx` (BARU, revisi hybrid 2026-08-15).
- **Modifikasi:** `apps/web/src/app/(guru)/layout.tsx` (render shell baru, guard role TIDAK diubah; `SidebarStateProvider` dikembalikan saat revisi hybrid — dibutuhkan `TopBar` versi desktop reused).
- **Hapus:** `apps/web/src/app/(guru)/guru-sidebar.tsx` (setelah dikonfirmasi tidak ada lagi yang import).
- **Jangan sentuh:** logic guard role di layout, endpoint `/users/me`, halaman `guru/wali-kelas/*` dan `guru/ekstrakurikuler/*` (isinya, hanya pintu masuk yang pindah).

## Acceptance Criteria
- [x] Halaman `(guru)/*` menampilkan bottom navbar 5 item (Jadwal, Presensi, Jurnal, Nilai, Riwayat) dengan highlight item aktif sesuai halaman yang dibuka.
- [x] Top bar tampil judul halaman (kiri) + avatar (kanan) di semua halaman guru.
- [x] Klik avatar membuka drawer berisi info profil + menu kondisional (Wali Kelas/Ekstrakurikuler sesuai status user) + Logout.
- [x] Guru TANPA wali kelas/ekstrakurikuler — drawer tetap berfungsi normal tanpa 2 menu itu (diverifikasi live: "Tidak ada peran tambahan").
- [x] Logout dari drawer berfungsi sama seperti sebelumnya (redirect ke login, token dibersihkan) — diverifikasi live browser.
- [x] `guru-sidebar.tsx` terhapus, tidak ada import mati/build error.
- [x] Styling ikuti `DESIGN.md`/vault design-system (beige+oranye tunggal, radius besar, dll) — TIDAK ada aksen warna kedua.
- [x] Build + type-check `apps/web` hijau (dan `apps/api` — sempat merah karena bug tidak terkait T168, lihat Status Eksekusi).
- [x] **REVISI 2026-08-15**: Di layar >=768px, tampil sidebar desktop (bukan bottom-nav) — 5 item sama + grup kondisional Wali Kelas/Ekstrakurikuler, dropdown avatar (bukan drawer Sheet) untuk logout — diverifikasi live viewport 1440x900.
- [x] **REVISI 2026-08-15**: Di layar <768px tetap shell mobile-app awal (bottom-nav+drawer) — diverifikasi live viewport 390x844 SETELAH perubahan hybrid, tidak ada regresi dari versi awal.

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA import lain yang masih merujuk `guru-sidebar.tsx` sebelum menghapusnya — grep bersih (hanya 2 komentar di file lain menyebut nama komponen, bukan import).
- [x] Konfirmasi styling drawer/bottom-nav mengikuti vault design-system — nav item radius-lg (16px) + active bg `--color-primary` dari `03-components.md` §Sidebar (tidak ada preseden bottom-tab-bar eksplisit, diadaptasi dari nav-item spec yang ada); avatar pill dari §Top Bar; drawer reuse `Sheet` primitif (`03-components.md` §"Form Input Panjang" — slide dari kanan).
- [x] Konfirmasi path `/guru/wali-kelas` dan `/guru/ekstrakurikuler/*` (termasuk sub-halaman) TIDAK berubah sama sekali — hanya cara masuknya (drawer, bukan sidebar section) yang berubah. Tidak ada file di bawah kedua path itu yang disentuh.

## Status Eksekusi (2026-08-15)
Selesai. Ringkasan implementasi lengkap di `STATUS.md` (baris T168).

**Keputusan implementasi**: avatar SELALU inisial nama (bukan foto) — `GET /users/me` tidak expose `Teacher.foto` saat ini, dan menambahkannya di luar scope task ini sesuai instruksi spec ("JANGAN tambah field foto baru di luar scope"). `SidebarStateProvider` dilepas dari layout guru (konsep off-canvas/collapse tidak relevan lagi untuk shell bottom-nav) — provider itu sendiri TIDAK dihapus dari codebase, masih dipakai route group lain (admin/piket/admin-jurnal/pembina-ekstra).

**Live-verify**: dibuat 1 akun test guru sementara (`Teacher`+`User` baru via docker exec ke dev DB) untuk login di browser (viewport mobile 390x844), cek shell/drawer/logout/highlight-aktif, lalu akun+data terkait dihapus bersih setelahnya (termasuk baris `activity_log` yang tercipta dari percobaan login).

**Temuan sampingan (bukan bug T168)**: saat mencoba login untuk verifikasi, API dev (port 3101) ternyata mati total — root cause `TeachingSessionsModule` (diedit sesi T158 paralel) tidak meng-import `JamPelajaranModule` yang dibutuhkan `TeachingSessionsService`, NestJS gagal boot. Dikonfirmasi ke user dan diperbaiki (1 baris tambah import module, pola sama seperti fix `ImportModule` di T160) karena ini memblokir SELURUH API untuk semua orang, bukan spesifik ke pekerjaan T168.

## Status Eksekusi — Revisi Hybrid (2026-08-15)

User eksplisit minta REVISI dari keputusan awal ("shell mobile-app tetap dipakai di desktop") jadi HYBRID responsif: sidebar desktop di >=768px, shell mobile-app di bawahnya — breakpoint tunggal `md` (Tailwind default, konsisten dgn `Sidebar` admin/`useSidebarState`).

**Perubahan**:
- `guru-sidebar-desktop.tsx` (BARU) — statis tanpa collapse/off-canvas (beda dari `Sidebar` admin yang collapsible), pola identik `guru-sidebar.tsx` lama (accordion grup, `groupHasActiveItem` auto-expand) + 3 menu baru (Jadwal/Presensi/Nilai) supaya konsisten 5 item bottom-nav mobile. Hanya dirender via CSS `hidden md:flex` di root elemen — TIDAK ADA logic JS render-kondisional berbasis window width (hindari hydration mismatch SSR/CSR).
- `mobile-shell.tsx` di-rename `guru-shell.tsx`, isinya jadi hybrid: render KEDUA shell sekaligus di DOM (mobile top-bar+bottom-nav dibungkus `md:hidden`, desktop sidebar+`TopBarWithTitle` dibungkus kebalikannya) — bukan conditional mount, konsisten pola CSS-only show/hide yang sudah dipakai `Sidebar` admin untuk mobileOpen off-canvas.
- Top bar desktop REUSE `TopBarWithTitle`/`TopBar` (`components/shell/`) APA ADANYA (dropdown avatar+logout, bukan drawer Sheet) — komponen ini sudah context-gate fitur khusus piket (`PiketNotificationBell`/`RiwayatAktivitasMenuItem` return null di luar role piket), aman dipakai lintas role tanpa modifikasi.
- `SidebarStateProvider` DIKEMBALIKAN ke layout guru (sempat dilepas di eksekusi awal) — `TopBar` reused butuh context itu untuk tombol hamburger (yang sendiri tidak pernah tampil karena `TopBar` desktop guru cuma dirender di viewport >=768px).

**Live-verify** (akun test guru sementara kedua, dibuat+dihapus bersih pola sama seperti sebelumnya):
- Viewport 1440x900: sidebar desktop tampil dengan benar (grup "Guru" 5 item + ikon, accordion auto-collapse karena path `/guru/jurnal` lama tidak match ke item baru manapun — expected, bukan bug, sama seperti perilaku accordion lama), dropdown avatar top-right berfungsi (Keluar).
- Viewport 390x844 (re-test SETELAH perubahan hybrid): shell mobile-app awal utuh tanpa regresi — top-bar+bottom-nav+drawer semua tetap seperti sebelum revisi.
- `tsc --noEmit` `apps/web` bersih pasca-revisi.

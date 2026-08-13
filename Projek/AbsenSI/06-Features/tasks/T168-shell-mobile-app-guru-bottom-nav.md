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
- Halaman guru dibuka di layar desktop lebar (bukan mobile) — shell mobile-app ini TETAP dipakai (bukan hybrid responsive ke sidebar desktop, sesuai gambaran user "meskipun style mobile app tetap masih ada menu sidebar" — DIBACA sebagai "ada menu tambahan via drawer", BUKAN "sidebar desktop biasa muncul di layar lebar"). Kalau user MAU versi desktop beda, itu di luar scope task ini — TANYAKAN dulu kalau muncul keraguan besar saat implementasi, JANGAN putuskan sendiri diam-diam.
- Konten halaman yang panjang (misal kalender presensi T170) — pastikan padding bawah cukup supaya konten TERAKHIR tidak ketutup bottom nav (masalah klasik fixed-bottom-nav).
- User TIDAK punya `kelasIdWali` DAN TIDAK `isPembinaEkstra` — drawer tetap muncul, cukup tanpa 2 menu kondisional itu, langsung Header→Logout (TIDAK boleh drawer kosong/error).

## Files
- **Buat:** `apps/web/src/app/(guru)/components/bottom-nav.tsx`, `apps/web/src/app/(guru)/components/top-bar.tsx`, `apps/web/src/app/(guru)/components/mobile-shell.tsx` (atau nama serupa, penggabung top-bar+children+bottom-nav).
- **Modifikasi:** `apps/web/src/app/(guru)/layout.tsx` (render shell baru, guard role TIDAK diubah).
- **Hapus:** `apps/web/src/app/(guru)/guru-sidebar.tsx` (setelah dikonfirmasi tidak ada lagi yang import).
- **Jangan sentuh:** logic guard role di layout, endpoint `/users/me`, halaman `guru/wali-kelas/*` dan `guru/ekstrakurikuler/*` (isinya, hanya pintu masuk yang pindah).

## Acceptance Criteria
- [ ] Halaman `(guru)/*` menampilkan bottom navbar 5 item (Jadwal, Presensi, Jurnal, Nilai, Riwayat) dengan highlight item aktif sesuai halaman yang dibuka.
- [ ] Top bar tampil judul halaman (kiri) + avatar (kanan) di semua halaman guru.
- [ ] Klik avatar membuka drawer berisi info profil + menu kondisional (Wali Kelas/Ekstrakurikuler sesuai status user) + Logout.
- [ ] Guru TANPA wali kelas/ekstrakurikuler — drawer tetap berfungsi normal tanpa 2 menu itu.
- [ ] Logout dari drawer berfungsi sama seperti sebelumnya (redirect ke login, token dibersihkan).
- [ ] `guru-sidebar.tsx` terhapus, tidak ada import mati/build error.
- [ ] Styling ikuti `DESIGN.md`/vault design-system (beige+oranye tunggal, radius besar, dll) — TIDAK ada aksen warna kedua.
- [ ] Build + type-check `apps/web` hijau.

## Validasi Claudian
- [ ] Konfirmasi TIDAK ADA import lain yang masih merujuk `guru-sidebar.tsx` sebelum menghapusnya.
- [ ] Konfirmasi styling drawer/bottom-nav mengikuti vault design-system (kutip file/section yang dirujuk).
- [ ] Konfirmasi path `/guru/wali-kelas` dan `/guru/ekstrakurikuler/*` (termasuk sub-halaman) TIDAK berubah sama sekali — hanya cara masuknya (drawer, bukan sidebar section) yang berubah.

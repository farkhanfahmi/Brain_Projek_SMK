# T230 — Web: Navigasi Mobile Guru — Bottom Nav Hanya di Menu Guru + Sidebar Mobile untuk Wali Kelas/Ekstrakurikuler

## Depends on
Tidak ada dependency teknis. Independen, murni frontend shell `(guru)/`.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-20)

**Bug 1 — Bottom nav mobile tampil di SEMUA halaman `(guru)/*`, tanpa syarat**: `BottomNav` (`apps/web/src/app/(guru)/components/bottom-nav.tsx`) di-mount TANPA syarat pathname di `guru-shell.tsx:53-55` (`<div className="md:hidden"><BottomNav /></div>`) — visibilitasnya MURNI CSS breakpoint (`md:hidden`), BUKAN kondisi halaman. Wali Kelas dan Ekstrakurikuler adalah sub-route **DALAM route group `(guru)` yang SAMA** (`/guru/wali-kelas`, `/guru/ekstrakurikuler`) — bukan route group terpisah — sehingga otomatis pakai `GuruShell` yang sama, dan `BottomNav` ikut muncul di sana juga, padahal isinya cuma 5 item menu Guru (Jadwal/Presensi/Jurnal/Nilai/Riwayat) yang tidak relevan di konteks Wali Kelas/Ekstrakurikuler.

**Bug 2 — Tidak ada sidebar mobile sama sekali untuk grup Wali Kelas/Ekstrakurikuler**: Sidebar DESKTOP (`guru-sidebar-desktop.tsx`) SUDAH lengkap 3 grup (`BASE_GROUP`, `WALI_KELAS_GROUP`, `EKSTRAKURIKULER_GROUP`, kondisional `isWaliKelas`/`isPembinaEkstra`) — TAPI hanya render `hidden md:flex`, TIDAK PERNAH tampil di mobile. `BottomNav` mobile punya array `NAV_ITEMS` **TERPISAH dan LEBIH SEMPIT** (cuma setara `BASE_GROUP`, tidak ada referensi Wali Kelas/Ekstrakurikuler sama sekali — bukan data yang sama disembunyikan, tapi memang array data berbeda). Satu-satunya jalur ke Wali Kelas/Ekstrakurikuler di mobile adalah `ProfileDrawer` (dibuka lewat avatar), BUKAN navigasi utama.

**Arsitektur switching**: `guru-shell.tsx` me-mount KEDUA versi shell (mobile+desktop) SEKALIGUS di DOM, visibilitas murni CSS `hidden`/`md:hidden` — TIDAK ADA deteksi JS/pathname sama sekali untuk switching konten navigasi.

## Keputusan Diminta User (2026-08-20)

1. Wali Kelas dan Ekstrakurikuler harus bisa diakses lewat **sidebar** (bukan cuma drawer profil) di mobile — SAMA seperti desktop.
2. **Bottom navbar HANYA muncul di menu Guru** (Jadwal/Presensi/Jurnal/Nilai/Riwayat) — TIDAK muncul saat berada di halaman Wali Kelas atau Ekstrakurikuler.

## Spec Detail

### 1. Deteksi konteks menu aktif berdasar pathname

- Tambah helper (client component, `usePathname()` dari `next/navigation`) untuk menentukan grup menu AKTIF saat ini: `"guru"` (path `/guru/jadwal`, `/guru/presensi`, `/guru/jurnal`, `/guru/nilai`, `/riwayat`), `"wali-kelas"` (path `/guru/wali-kelas`), `"ekstrakurikuler"` (path `/guru/ekstrakurikuler`).
- **VERIFIKASI SAAT IMPLEMENTASI**: apakah lebih rapi taruh logic ini di `guru-shell.tsx` (1 tempat, diteruskan sebagai prop) atau di `BottomNav`/sidebar mobile masing-masing baca `usePathname()` sendiri — REKOMENDASI: 1 sumber kebenaran di `guru-shell.tsx`, diteruskan sebagai prop `activeGroup` ke komponen turunan, supaya konsisten dan tidak ada 2 logic pathname-matching berbeda yang bisa drift.

### 2. `BottomNav` — HANYA render saat grup aktif = "guru"

- `apps/web/src/app/(guru)/components/bottom-nav.tsx` — TAMBAH kondisi: kalau `activeGroup !== "guru"`, return `null` (component tidak render apa pun) — BUKAN cuma disembunyikan CSS, benar-benar tidak di-mount supaya tidak ada elemen kosong menyita ruang di halaman Wali Kelas/Ekstrakurikuler.
- `guru-shell.tsx` — teruskan `activeGroup` sebagai prop ke `BottomNav`.

### 3. Sidebar mobile baru — off-canvas drawer, REPLIKASI pola `Sidebar` admin

- Proyek ini SUDAH punya pola sidebar mobile off-canvas untuk role LAIN (`apps/web/src/components/shell/sidebar.tsx`, dengan `mobileOpen` state) — **REPLIKASI pola itu untuk guru**, BUKAN desain baru dari nol. Guru shell saat ini TIDAK reuse komponen `Sidebar` generik itu sama sekali (punya `GuruSidebarDesktop` sendiri, desktop-only) — VERIFIKASI SAAT IMPLEMENTASI apakah `Sidebar` generik itu CUKUP FLEKSIBEL untuk menerima daftar grup guru (`BASE_GROUP`/`WALI_KELAS_GROUP`/`EKSTRAKURIKULER_GROUP` yang SUDAH ada di `guru-sidebar-desktop.tsx`) via props, atau perlu adaptasi ringan — REKOMENDASI KUAT: REUSE data grup yang SUDAH ada di `guru-sidebar-desktop.tsx` (jangan duplikasi definisi `NavGroup` guru sekali lagi di komponen baru).
- Trigger buka sidebar mobile: ikon hamburger baru di top-bar mobile guru (`apps/web/src/app/(guru)/components/top-bar.tsx`) — KONSISTEN pola trigger sidebar admin kalau ada polanya.
- Sidebar mobile ini MENAMPILKAN SEMUA 3 grup (Guru/Wali Kelas/Ekstrakurikuler, kondisional `isWaliKelas`/`isPembinaEkstra` SAMA seperti desktop) — supaya user bisa pindah ke Wali Kelas/Ekstrakurikuler LANGSUNG dari sini, TIDAK PERLU lagi lewat drawer profil untuk navigasi (drawer profil TETAP ADA untuk fungsinya semula — lihat edge case).

### 4. `BASE_GROUP`/`WALI_KELAS_GROUP`/`EKSTRAKURIKULER_GROUP` — satu sumber, dipakai desktop DAN mobile

- **VERIFIKASI SAAT IMPLEMENTASI**: pindahkan definisi 3 `NavGroup` dari `guru-sidebar-desktop.tsx` ke file shared (misal `apps/web/src/app/(guru)/components/nav-groups.ts`) yang di-import KEDUA `GuruSidebarDesktop` (desktop) DAN sidebar mobile baru (poin 3) — MENCEGAH drift 2 definisi menu yang sama tapi terpisah (persis masalah `NAV_ITEMS` mobile vs desktop yang SEKARANG sudah terjadi, task ini memperbaikinya sekalian supaya tidak terulang).

## Update 2026-08-21 — Refinement Diminta User Setelah Implementasi Awal

Setelah implementasi awal (sidebar mobile + bottom-nav conditional), user minta 2 penyesuaian
lanjutan:

1. **"Riwayat Kehadiran" dihapus dari bottom nav** (tetap ada di sidebar) — `BASE_GROUP` tetap
   1 sumber (dipakai sidebar), tapi `bottom-nav.tsx` filter item `/riwayat` keluar dari
   `NAV_ITEMS`-nya sendiri sebelum render. Bottom nav sekarang 4 item (Jadwal/Presensi/
   Jurnal/Nilai), sidebar tetap 5 item penuh.
2. **Link Wali Kelas/Ekstrakurikuler dihapus dari drawer profil** (dianggap redundan — sudah
   ada di `SidebarMobile`) — `ProfileDrawer` props `isWaliKelas`/`isPembinaEkstra` dan seluruh
   section link navigasi dihapus, drawer sekarang murni avatar+nama+role+tombol Keluar. Prop
   ini juga dihapus dari `TopBar` (guru mobile) karena satu-satunya pemakainya adalah
   `ProfileDrawer` yang sudah tidak butuh lagi.

tsc web bersih setelah kedua revisi ini, tidak ada prop/import yatim tersisa (diverifikasi
grep manual `isWaliKelas`/`isPembinaEkstra` di seluruh folder `(guru)/components/`).

## Pertanyaan Terpisah User (2026-08-21) — Dijawab TANPA Ubah Kode

User tanya: "karena fitur ini nanti akan dibuka di device pribadi guru, saya ingin akun tidak
otomatis logout, agar setiap hari guru tidak login ulang." Riset kode (`auth.service.ts`,
`session.ts`, `middleware.ts`) mengonfirmasi **fitur ini SUDAH ADA sejak T087** — bukan
kekurangan yang perlu di-task-kan:

- Guru dapat refresh token **1 tahun** (`REFRESH_TOKEN_TTL_LONG_SECONDS`, `auth.types.ts:30`),
  BUKAN 7 hari seperti role user biasa.
- Access token cuma 15 menit, TAPI `middleware.ts` otomatis refresh diam-diam setiap request
  kalau `isJwtExpiringSoon()` — guru tidak pernah sadar ini terjadi.
- Tidak ada mekanisme idle-timeout, single-session-lock, atau device-fingerprint yang
  ditemukan — satu-satunya jalan logout adalah tombol "Keluar" manual (blacklist Redis
  per-jti saat itu) atau refresh token benar-benar tidak dipakai >1 tahun berturut-turut.

User konfirmasi jawaban ini cukup, tidak ada perubahan kode diminta.

## Edge Cases

- **Drawer profil (`ProfileDrawer`) tetap ada** — task ini TIDAK menghapusnya, HANYA menambah jalur navigasi baru (sidebar mobile) yang lebih sesuai fungsi navigasi utama. Link Wali Kelas/Ekstrakurikuler di drawer profil BOLEH tetap ada sebagai shortcut tambahan (bukan wajib dihapus), TAPI VERIFIKASI SAAT IMPLEMENTASI apakah ada duplikasi UX yang membingungkan (2 jalur ke tempat yang sama) — kalau user merasa redundan, PERTIMBANGKAN hapus link navigasi dari drawer setelah sidebar mobile ada (drawer profil fokus ke akun/logout saja).
- **User BUKAN wali kelas ATAU pembina ekstra** (guru biasa) — sidebar mobile HANYA tampilkan `BASE_GROUP` (Guru), KONSISTEN behavior desktop yang sudah benar.
- **Buka sidebar mobile dari halaman Wali Kelas** — grup "Wali Kelas" di sidebar HARUS ter-highlight sebagai aktif (KONSISTEN pola `isItemActive()` yang sudah ada di `bottom-nav.tsx`/`guru-sidebar-desktop.tsx`).

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/components/guru-shell.tsx` (deteksi `activeGroup`, render sidebar mobile baru), `bottom-nav.tsx` (kondisi render), `top-bar.tsx` (trigger hamburger), `guru-sidebar-desktop.tsx` (extract definisi grup ke file shared).
- **Buat:** `apps/web/src/app/(guru)/components/sidebar-mobile.tsx` (atau nama serupa, REPLIKASI pola `components/shell/sidebar.tsx` off-canvas), `apps/web/src/app/(guru)/components/nav-groups.ts` (definisi shared 3 grup, dipakai desktop+mobile).
- **Jangan sentuh:** `apps/web/src/components/shell/sidebar.tsx` (komponen admin generik, HANYA dijadikan REFERENSI pola, TIDAK diubah — kalau perlu reuse langsung, VERIFIKASI dulu tidak merusak role lain yang sudah memakainya).

## Acceptance Criteria
- [x] Bottom navbar mobile — HANYA tampil di halaman Jadwal/Presensi/Jurnal/Nilai (Riwayat dihapus per revisi user), TIDAK tampil di Wali Kelas atau Ekstrakurikuler.
- [x] Sidebar mobile baru (off-canvas/hamburger) — bisa pindah ke Guru/Wali Kelas/Ekstrakurikuler langsung, KONSISTEN kondisional role yang sama seperti desktop.
- [x] Definisi grup menu (nama, href, icon) — SATU sumber shared (`nav-groups.ts`), dipakai desktop DAN mobile (tidak ada 2 definisi terpisah lagi).
- [x] Guru biasa (bukan wali kelas/pembina ekstra) — sidebar mobile HANYA tampilkan grup Guru, konsisten desktop.
- [x] Item aktif ter-highlight benar di sidebar mobile sesuai halaman saat ini (reuse `isItemActive()` shared).
- [x] Drawer profil TIDAK regresi — TAPI scope-nya diperkecil per revisi user (link navigasi Wali Kelas/Ekstrakurikuler dihapus, murni akun/logout sekarang).
- [x] Build + type-check hijau (`tsc --noEmit -p apps/web/tsconfig.json` bersih setelah implementasi awal DAN kedua revisi).

## Validasi Claudian
- [x] Konfirmasi `BottomNav` benar-benar TIDAK di-mount (return null) di luar konteks Guru, bukan cuma disembunyikan CSS.
- [x] Konfirmasi definisi `NavGroup` desktop dan mobile SATU sumber shared setelah task ini — `guru-sidebar-desktop.tsx` import dari `nav-groups.ts`, tidak ada lagi 2 array menu terpisah yang bisa drift.
- [x] Konfirmasi sidebar mobile REPLIKASI pola `components/shell/sidebar.tsx` yang sudah ada (overlay backdrop + `translate-x` transition + `useSidebarState()` yang sama, bukan komponen off-canvas baru yang berbeda gaya).

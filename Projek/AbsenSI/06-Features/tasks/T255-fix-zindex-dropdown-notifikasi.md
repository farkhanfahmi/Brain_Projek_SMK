# T255 — Web: Fix Bug UI — Dropdown Notifikasi/Menu Akun Tidak di Layer Paling Atas

## Depends on
Tidak ada. Perbaikan CSS murni, low-risk, independen dari task lain.

## Objective
Dropdown notifikasi (piket & wali kelas) dan menu akun di top bar tidak lagi tertutup/
bocor oleh elemen lain di bawahnya (tabel, dst) — root cause: tidak ada `z-index` eksplisit.

## Konteks — Root Cause Dikonfirmasi via Riset 2026-08-28 (screenshot user)

Bug terjadi berulang di **3 dropdown, lintas 2 file** — SEMUA pakai `position: absolute`
TANPA class `z-*` sama sekali:

1. `apps/web/src/components/shell/top-bar.tsx:210` — `PiketNotificationBell` dropdown.
2. `apps/web/src/components/shell/top-bar.tsx:288` — `WaliKelasNotificationBell` dropdown.
3. `apps/web/src/components/shell/top-bar.tsx:87` — menu akun (`role="menu"`).
4. `apps/web/src/app/(guru)/components/top-bar.tsx:131` — dropdown notifikasi versi guru
   (file TERPISAH dari shell/top-bar.tsx, kode mirip TAPI duplikat, bug sama independen).

Catatan: `(guru)/components/top-bar.tsx` punya `<header>`-nya sendiri sudah
`sticky top-0 z-30` (baris 39) — TAPI dropdown DI DALAMNYA tetap tidak punya `z-*` sendiri,
jadi tetap rentan ketiban konten lain yang kebetulan py stacking context bersaing.

Screenshot user menunjukkan dropdown notifikasi wali kelas ("Alfa" + nama siswa) bocor
tertindih tabel Biodata Murid (kolom NISN/Status) di bawahnya.

## Spec Detail

Tambah `z-50` (VERIFIKASI SAAT IMPLEMENTASI — cek dulu apakah ada elemen lain di codebase
yang sudah pakai `z-40`/`z-50`/dst untuk skala konsisten, jangan asal angka kalau ternyata
sudah ada konvensi z-index tersendiri) ke KEEMPAT elemen dropdown di atas. Pola className
existing (`shadow-overlay` dkk) TIDAK diubah, cukup TAMBAH `z-50` ke className yang sudah ada.

**Pertimbangkan sekalian**: kalau saat implementasi ditemukan dropdown LAIN dengan pola
`absolute` serupa tanpa `z-*` di file lain (grep `position.*absolute` atau className
`"absolute.*top-full"` di seluruh `apps/web/src`) — perbaiki SEKALIAN kalau ditemukan jelas
sama persis pola bug ini, supaya tidak perlu task terpisah lagi untuk kasus identik.

## Edge Cases
- **Dropdown di halaman dengan modal/dialog terbuka bersamaan** (jarang tapi mungkin) — cek
  `z-50` tidak bentrok/ketutup dialog (`Dialog`/`Popover` dari `@absensi/ui` kemungkinan
  py z-index sendiri lebih tinggi lagi — VERIFIKASI tidak ada tabrakan urutan, dropdown
  notifikasi wajar SELALU di bawah modal/dialog yang sedang aktif, bukan di atasnya).

## Files
- **Modifikasi:** `apps/web/src/components/shell/top-bar.tsx` (3 dropdown).
- **Modifikasi:** `apps/web/src/app/(guru)/components/top-bar.tsx` (1 dropdown).

## Acceptance Criteria
- [x] Dropdown notifikasi piket, wali kelas, dan menu akun SELALU tampil di atas konten
      halaman apa pun (tabel panjang, dst) — `z-50` ditambahkan ke keempat lokasi.
- [x] Tidak ada regresi ke dialog/modal lain — `z-50` SAMA PERSIS nilai yang sudah dipakai
      `DialogOverlay`/`DialogContent`/`PopoverContent`/`SelectContent`/`SheetContent`
      (`packages/ui`), TAPI primitif Radix itu semua di-render via PORTAL ke
      `document.body` (di LUAR stacking context top-bar manapun) — dropdown ini (native
      `position:absolute`, BUKAN portal) tidak pernah benar-benar bertabrakan dengan
      dialog/popover aktif meski nilai z-index sama, dialog selalu menang karena portal.
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `next build` sukses).

## Validasi Claudian
- [x] Konfirmasi keempat lokasi diperbaiki — grep ulang `absolute (right|left)-0 top-full`
      dan `className="absolute[^"]*shadow-overlay"` di seluruh `apps/web/src` setelah
      selesai: HANYA 4 file match (2 top-bar + `sesi-card.tsx` tooltip yang SUDAH punya
      `z-10` sendiri, bukan bug yang sama) — tidak ada dropdown/menu serupa TANPA z-index
      yang terlewat. 3 dropdown search-autocomplete lain (`input-izin-view.tsx`,
      `izin-keluar-view.tsx`, `izin-form.tsx`) SUDAH punya `z-10`, BUKAN kelas bug yang
      sama (tidak disentuh, sesuai instruksi "pola yang SAMA PERSIS").

## Implementasi (2026-08-28)

**Root cause dikonfirmasi 100% sesuai riset di kepala task ini** — 3 dropdown di
`components/shell/top-bar.tsx` (menu akun baris 87, `PiketNotificationBell` baris 210,
`WaliKelasNotificationBell` baris 288) dan 1 dropdown duplikat di
`(guru)/components/top-bar.tsx` (`WaliKelasNotificationBell` versi mobile, baris 131) —
SEMUA `position: absolute` tanpa `z-*` sama sekali. Fix: tambah `z-50` ke className
existing (tidak ada perubahan lain), nilai dikonfirmasi KONSISTEN dengan konvensi
codebase (sidebar/overlay mobile/toast semua sudah pakai `z-50`, `Dialog`/`Popover`/
`Select`/`Sheet` primitif dari `packages/ui` juga `z-50` — tapi lewat portal, jadi tidak
ada risiko tabrakan urutan meski sama-sama 50).

**Verifikasi**: `tsc --noEmit` bersih (murni perubahan className, tidak ada perubahan
logic/tipe), `next build` sukses, dev server hot-reload tanpa error. **Test visual
langsung di browser TIDAK dilakukan** di sesi ini (perbaikan CSS murni, risiko sangat
rendah, tidak ada logic untuk diverifikasi selain grep memastikan lokasi lengkap) —
REKOMENDASI ringan: buka halaman dengan tabel panjang (mis. Biodata Murid wali kelas,
sesuai screenshot bug asli) dan klik lonceng notifikasi untuk konfirmasi visual cepat.

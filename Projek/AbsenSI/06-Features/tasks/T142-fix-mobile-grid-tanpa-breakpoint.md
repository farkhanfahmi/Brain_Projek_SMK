# T142 — Web: Fix Grid Tanpa Breakpoint Mobile (Rusak/Sempit di Layar HP)

## Depends on
Tidak ada dependency teknis ke task lain. Independen, bisa dikerjakan kapan saja.

## Objective
Audit menyeluruh sudah dijalankan (2026-08-08, daftar di bawah FINAL, jangan riset ulang). Task ini memperbaiki 5 lokasi grid dengan kolom TETAP (`grid-cols-N` tanpa breakpoint responsive) yang berisiko sempit/rusak/terpotong di layar HP kecil (<375px) — sesuai aturan permanen baru: **UI dirancang mobile-first (base class = tampilan mobile), baru ditambah prefix `sm:`/`md:`/`lg:` untuk override desktop.**

## Context
- Breakpoint yang berlaku: default Tailwind (`sm:640px`, `md:768px`, `lg:1024px`, `xl:1280px`) — TIDAK ADA override custom di `apps/web/tailwind.config.ts`, pakai ini apa adanya.
- **Fakta penting yang SUDAH BENAR, JANGAN diutak-atik**: SEMUA tabel data (31 dari 32 file yang punya tabel) memakai komponen `Table` (`packages/ui/src/components/ui/table.tsx`) yang SUDAH otomatis dibungkus `<div className="relative w-full overflow-auto">` — scroll horizontal untuk tabel lebar SUDAH tertangani sistemik lewat design system, TIDAK PERLU perubahan apa pun untuk tabel. Task ini HANYA soal grid kartu/statistik/layout, BUKAN tabel.
- Shell/navigasi (sidebar, top bar) SUDAH mobile-first solid (drawer pattern `md:` dipakai konsisten) — TIDAK termasuk scope, sudah benar.

## 5 Lokasi WAJIB Diperbaiki (Final, dari Riset 2026-08-08)

| # | File | Baris | Masalah | Perbaikan |
|---|---|---|---|---|
| 1 | `apps/web/src/app/(admin)/jadwal-piket/jadwal-piket-view.tsx` | ±55 | `grid grid-cols-6 gap-3` — grid piket mingguan (Senin-Sabtu), 6 kolom fixed di SEMUA ukuran layar. **PALING BURUK** — di HP kecil, 6 kolom (hari + nama guru piket per kolom) jadi sangat sempit/tidak terbaca. | Ubah jadi mobile-first: HP tampilkan sebagai daftar VERTIKAL bertumpuk (1 kartu per hari, `grid-cols-1`), baru di layar besar jadi grid 6 kolom (`md:grid-cols-6` atau `lg:grid-cols-6`, pilih breakpoint yang membuat 6 kolom masih cukup lega — kemungkinan `lg:` lebih aman daripada `md:` karena 6 kolom butuh ruang lebih dari 768px, PUTUSKAN saat implementasi berdasar test visual langsung, bukan tebakan). |
| 2 | `apps/web/src/app/(admin)/ekstra-monitoring/ekstra-monitoring-view.tsx` | ±190 | `grid grid-cols-3 gap-4` — 3 SummaryCard (Total Siswa/Sudah Mengisi/Belum Mengisi), `text-display` (32px bold) di dalamnya rawan terpotong di HP kecil. | `grid-cols-1 gap-4 sm:grid-cols-3` (1 kolom mobile bertumpuk, 3 kolom mulai `sm:`/640px — verifikasi visual bahwa 3 angka besar `text-display` tetap muat nyaman di 640px, kalau masih sempit pakai `md:grid-cols-3` sebagai gantinya). |
| 3 | `apps/web/src/app/(admin)/import/import-view.tsx` | ±199 | `grid grid-cols-3 gap-4` — 3 StatCard (Total Baris/Berhasil/Gagal), masalah identik #2. | Perbaikan sama seperti #2: `grid-cols-1 gap-4 sm:grid-cols-3` (atau `md:` kalau `sm:` masih sempit setelah dites visual). |
| 4 | `apps/web/src/app/(admin)/kalender/kalender-view.tsx` | ±182, ±190 | `grid grid-cols-7` (×2 lokasi) — grid kalender bulanan 7 hari, TANPA penyesuaian ukuran font/padding untuk layar sangat kecil. | **BEDA dari #1-3** — kalender SECARA UX MEMANG butuh 7 kolom tetap (tidak masuk akal direflow jadi 1 kolom di HP, itu akan merusak konsep kalender). Perbaikan yang TEPAT di sini BUKAN mengubah jumlah kolom, tapi memastikan kolom-kolom itu tetap TERBACA di layar sempit: kecilkan font/padding sel tanggal di mobile (`text-xs p-1 sm:text-sm sm:p-2` atau sejenis, pola mobile-first: kecil dulu di base, besar di breakpoint), dan PASTIKAN container kalender punya `overflow-x-auto` sebagai jaring pengaman kalau tetap sempit di layar sangat kecil (<360px). JANGAN ubah jadi `grid-cols-1` — itu salah untuk kasus ini. |

**Catatan urutan No pada tabel di atas**: Grid Kalender (baris ±182, ±190) dihitung 1 lokasi (2 titik kode, 1 masalah, 1 solusi) — jadi total 4 baris tapi mencakup 5 titik kode sesuai riset awal.

## Spec Detail — Prinsip Umum (berlaku semua lokasi di atas)
- **Base class (tanpa prefix) = tampilan MOBILE**, breakpoint (`sm:`/`md:`/`lg:`) = override untuk layar LEBIH BESAR. Ini urutan wajib (mobile-first), BUKAN sebaliknya (JANGAN tulis base class desktop lalu override kecil pakai media query mobile — itu pola lama yang salah, walau hasil visual akhirnya sama, urutan penulisan class mencerminkan prioritas desain).
- Setelah perubahan, WAJIB dites visual di lebar viewport sempit (gunakan Playwright/browser dev tools resize ke ~360px-390px, ukuran HP umum) — bukan cuma "build berhasil tanpa error", tapi benar-benar dilihat tidak terpotong/tidak sempit.
- **JANGAN mengubah 5 lokasi form 2-kolom (`grid-cols-2 gap-4` tanpa breakpoint) yang disebut riset** (`guru-view.tsx`, `siswa-view.tsx`, `jadwal-view.tsx`, `kalender-view.tsx` baris ±117, `academic-years-section.tsx`, `holiday-form-dialog.tsx`, `izin-keluar-view.tsx`) — ini SEMUA ada di dalam `DialogContent`/`SheetContent` yang sudah full-width mobile, risikonya rendah, DI LUAR SCOPE task ini (kalau nanti ternyata bermasalah di testing HP sungguhan, itu task terpisah, jangan expand scope diam-diam).

## Edge Cases
- Grid Kalender di HP SANGAT kecil (<360px, jarang tapi ada) — kalau setelah pengecilan font/padding TETAP terasa sempit, `overflow-x-auto` jadi jaring pengaman terakhir (scroll horizontal sedikit), BUKAN kegagalan, itu solusi yang wajar diterima untuk kasus grid 7-kolom yang memang tidak bisa direflow.
- Grid Piket (#1) — pastikan urutan hari (Senin→Sabtu) tetap benar saat direflow jadi 1 kolom vertikal di mobile (jangan sampai urutan kebalik atau berubah karena perubahan struktur grid).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal-piket/jadwal-piket-view.tsx`, `apps/web/src/app/(admin)/ekstra-monitoring/ekstra-monitoring-view.tsx`, `apps/web/src/app/(admin)/import/import-view.tsx`, `apps/web/src/app/(admin)/kalender/kalender-view.tsx`.
- **Jangan sentuh:** tabel data mana pun (sudah tertangani `overflow-auto` sistemik), form 2-kolom dalam dialog/sheet (di luar scope, lihat Spec Detail), shell/sidebar/top-bar (sudah mobile-first benar).

## Acceptance Criteria
- [x] Grid Piket (`jadwal-piket-view.tsx`): 1 kolom mobile → progresif 2/3/6 kolom (`sm:`/`md:`/`lg:`) — urutan hari (Senin→Sabtu) TIDAK berubah (struktur `HARI_LABEL.map` tidak disentuh, murni class CSS).
- [x] SummaryCard Ekstra Monitoring: `grid-cols-1 gap-4 sm:grid-cols-3`.
- [x] StatCard Import: sama, `grid-cols-1 gap-4 sm:grid-cols-3`.
- [x] Grid Kalender: TETAP 7 kolom, font 9px→10px + button 20px→24px di breakpoint `sm:`, `min-w-[196px]`+`overflow-x-auto` sebagai jaring pengaman.
- [x] Diverifikasi — **TIDAK bisa live browser** (kredensial test ditolak 2x, dihentikan demi hindari lockout, user konfirmasi skip), diverifikasi via review kode manual (class Tailwind benar, struktur JSX tidak berubah) + `tsc`/`next build` hijau.
- [x] Build + type-check `apps/web` hijau.

## Validasi Claudian
- [x] **TIDAK mengubah Grid Kalender jadi 1 kolom** — tetap `grid-cols-7`, cuma font/padding+safety net yang berubah.
- [x] **TIDAK expand scope** ke 5 lokasi form 2-kolom (termasuk `kalender-view.tsx` baris ~123 grid 12-bulan `grid-cols-2`, dikonfirmasi ulang saat baca kode adalah lokasi yang disebut riset sebagai luar-scope) — tidak disentuh.
- [x] Breakpoint dipilih berdasar analisis lebar-kolom (bukan tebakan buta): Jadwal Piket progresif 4-tahap (1→2→3→6) supaya tidak ada lompatan drastis; SummaryCard/StatCard `sm:` (640px cukup lega untuk 3 kartu `text-display`); Kalender pakai `min-w-[196px]` (28px/kolom×7) sebagai batas aman minimum, bukan breakpoint reflow. Live-verify visual TIDAK terlaksana (kredensial gagal), keputusan breakpoint berbasis perhitungan lebar bukan observasi visual langsung — dicatat sebagai keterbatasan verifikasi.

## Catatan Implementasi (2026-08-16)

- **Re-audit sebelum eksekusi** (diminta user, task ini dibuat 2026-08-08): `git log` dikonfirmasi TIDAK ADA commit menyentuh 4 file target sejak dibuat — task MASIH RELEVAN, tidak stale/usang, tetap perlu dikerjakan.
- Kalender: grid 12-bulan (`grid-cols-2`, baris ~123, di LUAR scope task ini) berarti tiap `MiniMonth` cuma dapat ~50% lebar viewport di mobile — jadi grid-7-kolom DI DALAM `MiniMonth` genuinely sempit di 375px, penyesuaian font/padding relevan (bukan optimasi berlebihan).
- **Insiden dev server 500 (kedua kali dalam sesi ini)**: `next build` (dipakai untuk verifikasi produksi) menimpa folder `.next` milik `next dev` yang jalan bersamaan di port 3100 → corrupt webpack manifest → blank page. Dikonfirmasi user, di-restart bersih (kill proses+`rm -rf .next`+start ulang) — SAMA seperti insiden sebelumnya di sesi ini, pola berulang yang perlu diwaspadai kalau `next build` dijalankan sambil dev server aktif.
- **Live-verify gagal**: 2 percobaan password untuk `adminSU` ditolak backend — dihentikan (bukan terus dicoba) untuk hindari rate-limit/lockout akun, dikonfirmasi user untuk lanjut tanpa verifikasi visual browser, andalkan review kode+build saja.

Verifikasi: `tsc --noEmit` bersih, `next build` sukses (semua 4 halaman muncul di output build tanpa error).

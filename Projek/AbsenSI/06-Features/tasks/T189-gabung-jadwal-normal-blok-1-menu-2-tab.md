# T189 — Web: Gabung Menu "Jadwal Mengajar" + "Jadwal Blok" jadi 1 Halaman 2-Tab

## Depends on
Tidak ada dependency teknis. Independen. **REKOMENDASI kerjakan di KEDUA tempat sekaligus** (admin-jurnal DAN admin biasa T157) dalam 1 task supaya konsisten, sesuai konfirmasi user.

## Objective
Halaman "Jadwal Mengajar" (normal) dan "Jadwal Blok" (SAAT INI 2 menu/halaman terpisah, di KEDUA tempat — admin-jurnal DAN duplikat admin T157) — digabung jadi **1 halaman dengan 2 tab**: "Jadwal Normal" dan "Jadwal Blok".

## Context — Temuan Riset (2026-08-15) & Keputusan User

- `(admin-jurnal)/admin-jurnal/jadwal/` = kelola `Schedule` type `jam_mengajar` per semester.
- `(admin-jurnal)/admin-jurnal/jadwal-blok/` = kelola `BlockWeekRange` (kalender Minggu A/B global).
- Duplikat T157: `(admin)/jadwal-mengajar/` dan `(admin)/jadwal-blok/` — REUSE komponen yang SAMA dengan admin-jurnal.
- **PERINGATAN PENTING dari riset**: `(admin)/jadwal/` (TANPA "-mengajar") adalah domain **BEDA TOTAL** — "Jam Masuk Sekolah 3-Lapis" (T145), BUKAN bagian dari penggabungan ini — JANGAN SENTUH atau CAMPUR dengan halaman itu, namanya mirip tapi konsepnya berbeda sama sekali.
- User konfirmasi: gabung di **KEDUA tempat** (admin-jurnal dan admin biasa).

## Spec Detail

### 1. Frontend — halaman gabungan dengan 2 tab

- Path FINAL — REKOMENDASI: pertahankan path `jadwal-mengajar` sebagai halaman UTAMA (KARENA nama ini paling jelas menggambarkan domain "jadwal mengajar" secara keseluruhan, termasuk mode blok-nya), tab di dalamnya "Normal" dan "Blok" (route SUB-path opsional `?tab=normal|blok` ATAU state lokal — PUTUSKAN saat implementasi, REKOMENDASI query-param supaya bisa di-bookmark/share link langsung ke tab tertentu).
- Path `jadwal-blok` LAMA — REDIRECT ke path gabungan dengan tab blok terpilih (`jadwal-mengajar?tab=blok`) — SUPAYA link/bookmark lama yang mungkin sudah ada tidak mati total, ATAU cukup HAPUS kalau dikonfirmasi tidak ada dependency eksternal ke URL lama (PUTUSKAN saat implementasi, REKOMENDASI redirect untuk aman).
- Komponen isi masing-masing tab — REUSE `JadwalView` (normal) dan `JadwalBlokView` (blok) yang SUDAH ADA APA ADANYA (import langsung sebagai child dalam struktur Tab baru, JANGAN refactor logic internal keduanya — task ini MURNI penggabungan navigasi/layout, bukan perubahan fungsional).
- Lakukan di KEDUA tempat: `(admin-jurnal)/admin-jurnal/jadwal-mengajar/` (path final gabungan, GANTI dari `jadwal/` LAMA — REDIRECT serupa) dan `(admin)/jadwal-mengajar/` (T157 duplikasi).
- Sidebar admin-jurnal DAN admin biasa — HAPUS 1 entry menu terpisah ("Jadwal Blok"), SISAKAN 1 entry "Jadwal Mengajar" (yang sekarang berisi 2 tab).

## Edge Cases
- Bookmark/link lama ke path `jadwal-blok` — TANGANI via redirect (lihat Spec Detail) SUPAYA tidak 404 tiba-tiba untuk admin yang sudah terbiasa URL lama.
- `(admin)/jadwal/` (Jam Masuk Sekolah 3-Lapis) — TIDAK TERSENTUH SAMA SEKALI, WAJIB diverifikasi TIDAK ada perubahan apa pun di halaman itu meski namanya mirip.

## Files
- **Modifikasi/Buat:** `(admin-jurnal)/admin-jurnal/jadwal-mengajar/page.tsx` (baru, 2-tab, reuse `JadwalView`+`JadwalBlokView`), `(admin)/jadwal-mengajar/page.tsx` (update jadi 2-tab), sidebar admin-jurnal+admin (hapus 1 entry menu).
- **Redirect/Hapus:** path lama `jadwal/` dan `jadwal-blok/` di kedua tempat.
- **Jangan sentuh:** `(admin)/jadwal/` (Jam Masuk Sekolah, DOMAIN BEDA TOTAL), logic internal `JadwalView`/`JadwalBlokView` (REUSE apa adanya).

## Acceptance Criteria
- [x] 1 halaman "Jadwal Mengajar" dengan 2 tab (Normal/Blok) di admin-jurnal.
- [x] 1 halaman serupa di admin biasa (T157 duplikasi).
- [x] Sidebar kedua tempat cuma 1 entry menu (bukan 2 terpisah lagi).
- [x] Link lama ke path jadwal-blok redirect dengan benar (tidak 404).
- [x] `(admin)/jadwal/` (Jam Masuk Sekolah) TIDAK berubah sama sekali — verified diff kosong.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] **WAJIB verifikasi** `(admin)/jadwal/` (tanpa "-mengajar") TIDAK tersentuh — cek diff file itu 0 perubahan.
- [x] Konfirmasi logic internal `JadwalView`/`JadwalBlokView` TIDAK direfactor — task ini murni navigasi/layout.

## Catatan Implementasi (2026-08-16)

- **Path final**: `(admin-jurnal)/admin-jurnal/jadwal-mengajar/` dan `(admin)/jadwal-mengajar/` — komponen baru `jadwal-mengajar-tabs-view.tsx` (di folder admin-jurnal, diimport oleh page.tsx admin sesuai pola T157 REUSE).
- **Tab state**: query-param `?tab=normal|blok` (default `normal` kalau tidak ada param), pakai `useRouter().replace()` + `useSearchParams()` — bisa di-bookmark/share link langsung ke tab tertentu.
- **Komponen Tabs**: reuse `Tabs`/`TabsList`/`TabsTrigger`/`TabsContent` dari `@absensi/ui` (Radix, sudah ada, dipakai juga di `akun-view.tsx`) — style pill sudah sesuai design system, tidak perlu styling baru.
- **`usePageTitle` double-call**: `JadwalView` dan `JadwalBlokView` sama-sama panggil `usePageTitle` di dalam — AMAN karena Radix `TabsContent` default UNMOUNT konten tab tidak-aktif (tanpa `forceMount`), jadi cuma 1 yang mount+panggil hook di satu waktu. Wrapper tabs-view juga panggil `usePageTitle("Jadwal Mengajar")` sebagai title default, child akan override sesuai tab aktif.
- **Redirect path lama**: `jadwal/page.tsx` (admin-jurnal) → redirect ke `/admin-jurnal/jadwal-mengajar`; `jadwal-blok/page.tsx` (admin-jurnal) → redirect ke `/admin-jurnal/jadwal-mengajar?tab=blok`; `jadwal-blok/page.tsx` (admin) → redirect ke `/jadwal-mengajar?tab=blok`. File `jadwal-view.tsx`/`jadwal-blok-view.tsx` lama TETAP ADA (diimport oleh tabs-view, bukan dihapus).
- **Sidebar**: `nav-items.ts` (admin) dan `admin-jurnal-sidebar.tsx` — entry "Jadwal Blok Minggu" dihapus, entry "Jadwal Mengajar" tetap 1, href admin-jurnal diupdate dari `/admin-jurnal/jadwal` → `/admin-jurnal/jadwal-mengajar`.
- **Link lain**: `upcoming-gaps-banner.tsx` (link proaktif "lubang jadwal blok mendekat") diupdate dari `/admin-jurnal/jadwal-blok` → `/admin-jurnal/jadwal-mengajar?tab=blok` langsung (hindari hop redirect tambahan).
- **Verifikasi**: `git diff --stat -- "apps/web/src/app/(admin)/jadwal/"` = kosong (domain Jam Masuk Sekolah 3-Lapis benar-benar tidak tersentuh). `tsc --noEmit` web bersih, `next build` sukses (semua route termasuk `/jadwal-mengajar`, `/jadwal-blok` redirect, `/jadwal` untouched muncul di output). 437 test backend tetap lulus (task ini 100% frontend, tidak ada perubahan API).

# T135 — Web: Fix Bug — Halaman Kosong Setelah Login Piket, Baru Muncul Setelah Refresh Manual

## Depends on
Tidak ada dependency teknis. Fix di `login-form.tsx`, satu titik perubahan kecil.

## Objective
Setelah login sebagai `guru_piket`, halaman langsung menampilkan konten yang benar (dashboard normal ATAU form Jurnal Piket kalau ada utang) — TIDAK LAGI kosong/blank yang baru terisi setelah refresh manual.

## Context
- **App:** `apps/web` — fix logic navigasi, tidak ada perubahan API/backend.
- **Bug dilaporkan user 2026-08-08**: login akun piket → halaman kosong total → baru muncul (popup/redirect form Jurnal Piket) setelah refresh manual. Terjadi konsisten, dicoba berkali-kali dengan pola sama.
- **Root cause dikonfirmasi 2026-08-08 (Explore agent, baca kode langsung)** — BUKAN bug di logic Jurnal Piket (T130) sama sekali, ini bug LAMA di alur login yang baru kelihatan efeknya sekarang:
  - `apps/web/src/app/login/login-form.tsx:34-35`:
    ```js
    const next = searchParams.get("next") ?? "/";
    router.push(next);
    router.refresh();
    ```
    Dua pemanggilan navigasi BERUNTUN — `router.push()` memulai transisi client-side (soft navigation) ke halaman tujuan, LALU `router.refresh()` langsung dipanggil lagi SEBELUM transisi pertama selesai, memicu request RSC KEDUA ke rute yang SAMA yang tumpang tindih dengan yang pertama.
  - `apps/web/src/app/(piket)/layout.tsx:43-46` — `PiketLayout` (async Server Component) melakukan `await apiFetch(...)` untuk cek utang jurnal, lalu `redirect()` kalau ada utang. Method `redirect()` Next.js bekerja dengan melempar signal khusus (`NEXT_REDIRECT`) yang harus ditangani bersih oleh 1 siklus render. Ketika 2 request RSC tumpang tindih (dari `push`+`refresh` beruntun), signal redirect dari transisi PERTAMA tidak konsisten diterapkan sebelum transisi KEDUA menimpanya — hasilnya React commit tree yang TERPUTUS (halaman kosong), bukan salah satu dari 2 kemungkinan yang benar (dashboard normal atau form jurnal).
  - **Kenapa refresh manual selalu menyembuhkan**: refresh manual = navigasi FULL-DOCUMENT tunggal (bukan 2 request RSC client-side tumpang tindih) — `PiketLayout` jalan SEKALI dengan bersih, `redirect()` diproses normal, hasil benar langsung tampil.
  - **Kenapa baru kelihatan SEKARANG**: bug navigasi ganda (`push`+`refresh` beruntun) ini sudah ada SEBELUM T130, tapi sebelumnya tidak ada `redirect()` server-side baru di layout piket yang bisa jadi "korban" race ini secara terlihat — T130 menambahkan `redirect()` baru yang pertama kali mengekspos efek bug lama ini ke pengguna.
  - **BUKAN masalah caching/data**: `apiFetch` (`apps/web/src/lib/api.ts:34`) sudah pakai `cache: "no-store"` untuk SEMUA request termasuk cek utang jurnal — dikonfirmasi bukan stale-cache issue. `PiketLayout` juga sudah benar `await` fetch sebelum `redirect()` — tidak ada race DI DALAM layout itu sendiri, race-nya di LEVEL NAVIGASI (2 request RSC tumpang tindih), bukan di dalam 1 render.

## Spec Detail

- `apps/web/src/app/login/login-form.tsx` (baris ±34-35) — HAPUS pemanggilan `router.refresh()` yang redundan setelah `router.push(next)` — TIDAK PERLU dua-duanya, `router.push()` saja sudah cukup untuk navigasi ke halaman baru.
  - **Kalau `router.refresh()` sengaja dipasang untuk alasan tertentu** (misal memaksa refresh data server component lain yang tidak otomatis ter-update oleh `push()` biasa) — cek dulu HISTORI/ALASAN kenapa baris itu ada sebelum menghapus begitu saja. Kalau memang ada alasan valid, PERTIMBANGKAN alternatif yang tidak memicu race: `window.location.href = next` (hard navigation, full-document, SATU request bersih — konsisten dengan perilaku refresh manual yang sudah terbukti benar) — INI REKOMENDASI UTAMA kalau perlu memastikan server component ter-render ulang total, karena ini persis meniru behavior "refresh manual" yang sudah terbukti selalu benar.
- **REKOMENDASI FIX**: ganti `router.push(next); router.refresh();` MENJADI `window.location.href = next;` (hard navigation tunggal) — ini pola yang PALING MENDEKATI behavior refresh manual yang sudah terbukti selalu berhasil, menghindari kompleksitas race antar-transisi client-side Next.js App Router sepenuhnya, dengan trade-off kecil (full-page reload, bukan SPA transition — TAPI ini toh terjadi PERSIS SETELAH login, jeda visual sedikit lebih lama itu wajar dan tidak masalah untuk momen ini).

## Edge Cases
- Pastikan fix ini TIDAK merusak alur login untuk role LAIN (admin, guru, dst) yang TIDAK punya guard `redirect()` di layout mereka — role lain kemungkinan besar tidak terpengaruh bug ini sama sekali (tidak ada `redirect()` server-side serupa di layout mereka), tapi tetap verifikasi login role lain tetap lancar setelah perubahan ini.
- Kalau ada query param `next` yang custom (bukan default `/`) — pastikan `window.location.href = next` tetap menghormati redirect tujuan yang benar, sama seperti `router.push(next)` sebelumnya.

## Files
- **Modifikasi:** `apps/web/src/app/login/login-form.tsx`.
- **Jangan sentuh:** `apps/web/src/app/(piket)/layout.tsx` (logic guard T130 sudah benar, tidak perlu diubah — akar masalahnya di navigasi login, bukan di situ), `apps/web/src/lib/api.ts` (`cache: "no-store"` sudah benar, tidak perlu diubah).

## Acceptance Criteria
- [x] Login sebagai piket YANG PUNYA utang jurnal → langsung tampil form Jurnal Piket tanpa perlu refresh manual.
- [x] Login sebagai piket TANPA utang jurnal → langsung tampil dashboard piket normal tanpa perlu refresh manual.
- [x] Login sebagai role LAIN (admin, guru, dst) tetap berfungsi normal, regresi nol.
- [x] Diulang berkali-kali (minimal 5x percobaan login berturut-turut) untuk memastikan fix ini konsisten, bukan cuma kebetulan berhasil sekali.
- [x] Build + type-check `apps/web` hijau.

## Validasi Claudian
- [x] Cek histori/alasan `router.refresh()` dipasang sebelum menghapusnya — pastikan tidak ada efek samping lain yang bergantung pada baris itu (misal ada state lain yang perlu di-refresh yang tidak otomatis ter-update oleh navigasi biasa).
- [x] Test eksplisit login BERULANG KALI (bukan cuma sekali) untuk memastikan fix benar-benar konsisten — bug ini sifatnya race condition, kadang bisa "kebetulan" tidak muncul di 1 percobaan tapi tetap ada risikonya kalau tidak diperbaiki di akar.

## Status Eksekusi (2026-08-08)

**Selesai, diverifikasi live.**

- `git log --follow -p` pada `login-form.tsx` menunjukkan baris `router.push(next); router.refresh();` sudah ada SEJAK commit pertama file ini dibuat — bukan patch belakangan untuk kasus khusus, jadi aman dihapus sesuai rekomendasi spec.
- Fix: `router.push(next); router.refresh();` → `window.location.href = next;` (hard navigation tunggal). Import `useRouter` dan variabel `router` yang jadi tidak terpakai ikut dihapus.
- Verifikasi live (Playwright, dev server bersih `.next` dihapus dulu):
  - Login `hilma` (piket) TANPA utang jurnal → langsung dashboard piket penuh (tabel siswa dll), tidak blank.
  - Login `hilma` DENGAN utang jurnal (entry T130 dihapus sementara untuk simulasi, lalu direstore persis setelah tes) → **5x percobaan login berturut-turut**, SEMUANYA langsung tampil form Jurnal Piket penuh (tanggal, textarea catatan) tanpa refresh manual — konsisten, tidak ada kejadian blank sekali pun.
  - Login `adminSU` (role tanpa guard `redirect()` piket) → dashboard admin normal, regresi nol.
- `tsc --noEmit` untuk `apps/web` hijau.
- Data uji: entry `piket_journal_entries id=2` (hilma, tanggal 2026-08-06) dihapus sementara lalu di-INSERT ulang persis sama setelah verifikasi — state akhir identik dengan sebelum tes.
- **Belum di-commit** — menunggu instruksi lanjut dari user (sesi masih ada perubahan T134-susulan yang juga belum di-commit).

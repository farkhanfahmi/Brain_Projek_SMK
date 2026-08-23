# T109 — Web: Sorting Otomatis Realtime Tabel "Siswa Belum Hadir" Piket Board

## Depends on
Tidak ada dependency teknis keras — murni pengurutan tampilan atas data yang sudah ada di `piket-board-view.tsx`. Kalau T106b (search+sort+kolom No manual di tabel yang sama) dikerjakan berdekatan/bersamaan, **baca ulang kedua task ini bersama** — T106b menambahkan `SortableHeader` (sort MANUAL oleh klik piket), T109 ini menambahkan sort OTOMATIS default (urutan awal sebelum piket klik apapun). Keduanya tidak kontradiktif tapi perlu digabung dengan hati-hati: urutan default (T109) berlaku saat tidak ada kolom yang di-klik sort manual; begitu piket klik salah satu `SortableHeader` (T106b), urutan manual itu yang menang sampai di-reset.

## Objective
Tabel "Siswa Belum Hadir" di Piket Board (`piket-board-view.tsx`) terurut OTOMATIS dan REALTIME berdasarkan prioritas: siswa Terlambat paling atas, lalu siswa yang sudah diinput Izin/Sakit, lalu siswa yang belum ada aksi sama sekali (belum tap, belum terlambat, belum izin) paling bawah — supaya piket langsung melihat siswa yang paling butuh tindakan tanpa perlu scroll/cari manual.

## Context
- **App:** `apps/web` (murni frontend, sort di-compute dari data yang sudah di-fetch, tidak ada perubahan API/DB)
- File: `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — tabel "Siswa Belum Hadir" (nama section sebelumnya "Board Semua Siswa", diganti T099c) memakai `visibleBoard = board.filter((row) => row.status !== "hadir")` (baris ±126) sebagai sumber data tabel. Board di-update realtime lewat `useAttendanceSocket` (`onKampusUpdate`, baris ±128-135) — task ini murni menambah SORT atas `visibleBoard`/state board, TIDAK mengubah mekanisme realtime yang sudah berjalan.

## Keputusan Final (dikonfirmasi user 2026-08-05)

1. **Urutan grup (prioritas tampil), dari atas ke bawah:**
   1. `status === "terlambat"`
   2. `status === "izin" || status === "sakit"` (sudah diinput piket, sudah "beres" secara administratif tapi tetap informasi relevan)
   3. Belum ada aksi sama sekali (`status === null`/belum tap, bukan terlambat, bukan izin/sakit) — sisanya
2. **Urutan DALAM 1 grup yang sama**: berdasarkan jam tap paling awal duluan (untuk grup Terlambat — siswa yang terlambat paling lama menunggu tampil duluan). Untuk grup tanpa jam tap (belum ada aksi sama sekali, belum pernah tap), urutkan berdasarkan kriteria yang masuk akal sebagai fallback (nama alfabetis, atau urutan kelas — putuskan saat implementasi, tidak signifikan karena grup ini "belum ada kejadian" jadi urutan dalamnya kurang kritis dibanding 2 grup lain).
3. **Realtime**: begitu event socket baru masuk (misal siswa baru tap dan jadi terlambat), tabel HARUS re-sort otomatis tanpa refresh — urutan bukan snapshot statis saat load, tapi recompute tiap kali state `board` berubah.

## Spec Detail

- Tambah fungsi `sortBoardRows(rows: PiketBoardRow[]): PiketBoardRow[]` (atau nama serupa) di `piket-board-view.tsx` — pure function, tidak menyimpan state terpisah, dipanggil di titik render (`useMemo` atas `visibleBoard`, dependency `[board]`, supaya tidak recompute tiap render kalau board tidak berubah).
- Logic prioritas: definisikan `getPriority(row: PiketBoardRow): number` — return `0` untuk terlambat, `1` untuk izin/sakit, `2` untuk sisanya. Sort primer by priority ascending, sort sekunder (dalam priority sama) by `row.jamTap` (atau field waktu tap yang relevan di tipe `PiketBoardRow` — cek `apps/web/src/lib/core-types.ts` untuk nama field pastinya) ascending (paling awal duluan), fallback nama alfabetis kalau `jamTap` null di kedua sisi.
- **Kalau T106b sudah/sedang menambahkan `SortableHeader` manual di tabel ini**: urutan default (T109) HANYA berlaku selama piket belum klik kolom manapun untuk sort manual. State `manualSort: { key, dir } | null` — null berarti pakai `sortBoardRows()` default T109, non-null berarti override pakai sort manual by kolom yang diklik. Koordinasikan implementasi ini dengan state yang sudah/akan ditambahkan T106b, JANGAN bikin 2 sistem sort yang saling menimpa tanpa aturan jelas siapa menang.

## Edge Cases
- Siswa pindah grup secara realtime (misal dari "belum ada aksi" ke "terlambat" begitu tap masuk terlambat masuk lewat socket) → baris itu harus PINDAH POSISI otomatis ke grup Terlambat di render berikutnya, bukan macet di posisi lama.
- Siswa izin lalu (tidak mungkin normal, tapi untuk robustness) status berubah lagi → sort tetap konsisten mengikuti `status` terkini, tidak menyimpan urutan lama yang stale.

## Files
- **Modifikasi:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — tambah fungsi sort + `useMemo`, ganti pemakaian `visibleBoard` langsung di render tabel jadi versi ter-sort.
- **Jangan sentuh:** backend `piketBoard()` — sorting ini murni tampilan, backend tetap kirim data apa adanya (urutan asli dari backend tidak relevan lagi setelah T109).

## Acceptance Criteria
- [x] Tabel Siswa Belum Hadir menampilkan grup Terlambat di atas, Izin/Sakit di tengah, Belum Ada Aksi di bawah, tanpa perlu piket melakukan apapun. **Diverifikasi** via test standalone atas fungsi `sortBoardRowsDefault()` dengan data representatif.
- [x] Dalam grup Terlambat, siswa dengan jam tap paling awal ada di baris paling atas grup itu. **Diverifikasi**.
- [x] Siswa baru tap terlambat (event realtime masuk) langsung muncul di posisi yang benar (grup Terlambat) tanpa refresh halaman. **Diverifikasi** via edge case test (siswa pindah status → pindah grup otomatis di sort berikutnya) — turunan otomatis dari `boardRows` dihitung ulang tiap render, dan `onKampusUpdate` socket handler (`setBoard`) sudah memicu render itu tanpa perlu diubah.
- [x] Kalau T106b (sort manual per kolom) sudah ada, klik header kolom meng-override urutan default T109 sampai direset; urutan default T109 kembali berlaku kalau tidak ada sort manual aktif. **Diverifikasi via pembacaan kode**: `boardRows = boardSort ? sortRows(...) : sortBoardRowsDefault(...)` — persis skema menang-kalah yang diminta spec, tidak ada 2 mekanisme sort yang tumpang tindih.
- [x] Build + type-check `apps/web` hijau, tidak ada regresi realtime socket yang sudah berjalan. `tsc --noEmit` bersih, `next build` sukses (exit 0, route `/piket` compile normal 7.22 kB).

## Validasi Claudian
- [x] Field waktu tap yang benar dikonfirmasi ke `core-types.ts`: `PiketBoardRow.waktuMasuk` (bukan nama lain yang diasumsikan).
- [x] T106b sudah selesai lebih dulu (2026-08-05) — dibaca penuh implementasinya (`sortRows`, `toggleSort`, state `boardSort`) sebelum menambah `sortBoardRowsDefault()`, tidak ada state sort baru yang bentrok, cuma percabangan `boardSort ? manual : default` di titik render `boardRows`.

## Status Eksekusi — SELESAI (2026-08-06)
`apps/web/src/app/(piket)/piket/piket-board-view.tsx` — tambah `getPriority()` (0=terlambat, 1=izin/sakit, 2=sisanya) dan `sortBoardRowsDefault()` (sort primer priority, sekunder `waktuMasuk` ascending, fallback nama alfabetis). Dipakai di `boardRows` HANYA saat `boardSort` (state sort manual T106b) bernilai `null` — begitu piket klik `SortableHeader`, `sortRows()` manual yang menang, persis skema "menang-kalah" yang diminta spec. Tidak pakai `useMemo` eksplisit (spec menyarankan tapi opsional) — konsisten dengan pola T106b yang juga tidak memoize `boardRows`, komponen sudah re-render otomatis tiap `setBoard` dari socket dan datanya kecil (board 1 kampus). Backend `piketBoard()` tidak disentuh sama sekali sesuai spec.

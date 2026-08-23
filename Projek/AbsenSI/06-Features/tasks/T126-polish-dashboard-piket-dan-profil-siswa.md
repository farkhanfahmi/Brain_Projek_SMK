# T126 — Web: 4 Penyesuaian Dashboard Piket + Profil Siswa

## Depends on
Tidak ada dependency teknis lintas-task lain. 4 sub-poin independen satu sama lain, bisa dikerjakan sebagian/seluruhnya dalam urutan bebas — meski semuanya kecil dan berdekatan file, disarankan dikerjakan sekaligus dalam 1 sesi.

## Objective
4 perbaikan UX kecil hasil masukan user 2026-08-06 (termasuk 1 referensi visual/screenshot untuk poin 3): dashboard piket langsung buka tabel Siswa Belum Hadir tanpa perlu klik, nama siswa di tabel itu bisa diklik ke profil, foto profil siswa diperbesar + bisa di-zoom (lightbox), dan section Riwayat Kartu disembunyikan dari piket (tetap tampil untuk admin/card_admin).

## Context
- **App:** `apps/web` murni, tidak ada perubahan API/DB untuk 4 poin ini.
- **Riset 2026-08-06 (Explore agent, baca kode langsung)** — semua 4 poin dikonfirmasi kondisi aktualnya:
  1. `piket-board-view.tsx:200-202` — state `openSection` default `null`, kartu ringkasan SELALU tampil tapi tabel di bawahnya baru render kalau `openSection` diisi lewat klik. "Siswa Belum Hadir" = `openSection === "board"` (baris ±289, 332).
  2. `piket-board-view.tsx:373` — nama siswa di tabel "Siswa Belum Hadir" cuma `<TableCell className="font-medium">{row.nama}</TableCell>`, teks polos, TIDAK ADA link/navigasi sama sekali.
  3. `siswa-detail-view.tsx:156-206` — foto profil SAAT INI `h-16 w-16` (64px), bulat. Hover overlay yang ada SAAT INI cuma untuk hapus foto (non-readOnly) — TIDAK ADA klik-untuk-lihat-besar. **Tidak ada komponen lightbox/zoom/preview APA PUN di codebase ini** (dicek via grep menyeluruh) — harus dibangun baru, TAPI bisa reuse primitive `Dialog`/`DialogContent` dari `@absensi/ui` yang sudah dipakai di halaman yang sama untuk modal lain (baris ±381-407).
  4. `siswa-detail-view.tsx:336-379` — section "Riwayat Kartu" render UNCONDITIONAL, tidak ada pengecekan role/`readOnly` sama sekali. Prop `readOnly` yang sudah ada di komponen ini bernilai `true` untuk **card_admin DAN guru_piket** (dipakai bareng, ADA DI SATU KONDISI) — task ini perlu MEMBEDAKAN keduanya (lihat Keputusan Final).

## Keputusan Final (dikonfirmasi user 2026-08-06)

Poin 4: `card_admin` TETAP melihat Riwayat Kartu (relevan dengan tugasnya), HANYA `guru_piket` yang disembunyikan — jadi TIDAK BOLEH reuse `readOnly` yang sekarang (dipakai bareng keduanya), perlu prop/logic baru yang membedakan role secara spesifik.

## Spec Detail

### 1. Dashboard langsung buka "Siswa Belum Hadir"
- `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — ganti default `openSection` dari `null` jadi `"board"` (baris ±200-202): `useState<...>("board")`.
- Kartu ringkasan lain (Belum Kembali, Tidak Absen Pulang, dst) TETAP collapsed by default seperti sekarang — HANYA "Siswa Belum Hadir" yang diubah defaultnya, sisanya tidak berubah perilakunya.
- Piket TETAP bisa collapse/ganti ke section lain seperti biasa (klik kartu lain) — ini cuma ubah state AWAL saat halaman dimuat, bukan mengunci section itu selalu terbuka.

### 2. Nama siswa di tabel bisa diklik ke profil
- `piket-board-view.tsx:373` — bungkus `{row.nama}` jadi tautan/button yang navigasi ke `/siswa/${row.studentId}` (pola sama seperti yang sudah dipakai `direktori-siswa-view.tsx`, pakai `router.push()` atau `<Link>` Next.js, konsisten dengan pola existing di codebase ini — cek dulu mana yang dipakai `direktori-siswa-view.tsx` supaya konsisten, bukan pola baru).
- Styling: nama tetap terlihat sebagai teks normal di tabel (bukan harus biru/underline berlebihan) TAPI ada indikasi bisa diklik (hover state, cursor pointer) — ikuti DESIGN.md, jangan menyimpang dari pola link table row yang sudah ada di halaman lain kalau ada.

### 3. Redesign foto profil siswa — besar + lightbox
- `siswa-detail-view.tsx:156-206` — perbesar foto dari `h-16 w-16` (64px) jadi ukuran signifikan lebih besar sesuai referensi visual user (screenshot menunjukkan foto persegi/rounded besar, proporsional dengan card header, BUKAN avatar kecil bulat 64px lagi) — tentukan ukuran pasti saat implementasi (misal `h-32 w-32` atau lebih, sesuaikan proporsi dengan layout card yang ada), pertahankan `object-cover` supaya foto tidak distorsi.
- **Tambah klik-untuk-lihat-besar (lightbox)**: klik foto (baik yang ada foto asli maupun placeholder — untuk placeholder mungkin tidak relevan/disable klik kalau memang belum ada foto, putuskan saat implementasi) → buka `Dialog`/`DialogContent` (reuse primitive `@absensi/ui` yang sudah ada di file yang sama) menampilkan foto dalam ukuran penuh/besar, dengan cara menutup yang jelas (klik luar/tombol close/ESC — pola standar Dialog yang sudah ada).
- **JANGAN hilangkan** fungsi hover-hapus-foto yang sudah ada untuk role yang boleh edit (non-readOnly) — gabungkan dengan behavior baru: mungkin perlu 2 area klik berbeda (klik foto = lightbox, tombol hapus terpisah = tetap fungsi hapus) ATAU differensiasi lain yang tidak membuat kedua aksi itu tumpang tindih/membingungkan. Putuskan UX-nya saat implementasi, pastikan tidak ada regresi ke fungsi hapus foto yang sudah ada.
- Placeholder (belum ada foto) — TIDAK PERLU lightbox (tidak ada apa-apa untuk di-zoom), styling ukuran besar yang sama tapi tanpa perilaku klik-zoom.

### 4. Sembunyikan "Riwayat Kartu" dari piket
- `siswa-detail-view.tsx:336-379` — bungkus section ini dengan kondisi baru yang MEMBEDAKAN piket dari card_admin — cek bagaimana komponen ini menerima info role saat ini (`readOnly` prop yang ada dipakai bareng keduanya, per riset). Kemungkinan solusi: tambah prop baru eksplisit (misal `hideRiwayatKartu?: boolean` atau `role?: UserRole` langsung) yang di-set dari `page.tsx` berdasarkan role sesungguhnya (`guru_piket` → true/hide, `card_admin` → false/tampil, `super_admin` → false/tampil).
- Cek `page.tsx` (server component) untuk lihat bagaimana `readOnly`/`isPiket` di-compute sekarang, tambahkan variable serupa yang lebih spesifik kalau perlu (`isPiketRole` terpisah dari `readOnly` generik).

## Files
- **Modifikasi:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (poin 1, 2), `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` (poin 3, 4), `apps/web/src/app/(admin)/siswa/[id]/page.tsx` (poin 4, kalau perlu prop baru dari server component).
- **Jangan sentuh:** backend/API (semua 4 poin murni frontend), section/tabel lain di dashboard piket yang tidak disebut (Belum Kembali, dst tetap collapsed default).

## Acceptance Criteria
- [x] Klik menu Dashboard piket langsung menampilkan tabel "Siswa Belum Hadir" terbuka, tanpa perlu klik tambahan. `openSection` default diubah dari `null` jadi `"board"`.
- [x] Section lain (Belum Kembali, Tidak Absen Pulang, Perlu Ditinjau, Terkunci) TETAP collapsed by default (regresi nol) — hanya nilai default state yang berubah, logic toggle klik tidak disentuh.
- [x] Nama siswa di tabel Siswa Belum Hadir bisa diklik, navigasi ke halaman profil siswa yang benar. `<Link href={`/siswa/${row.studentId}`}>` membungkus `{row.nama}`, hover state (`hover:text-primary hover:underline`) sebagai indikasi klik sesuai DESIGN.md (bukan biru/underline permanen).
- [x] Foto profil siswa tampil jauh lebih besar dari sebelumnya. `h-16 w-16` (64px, bulat) → `h-32 w-32` (128px, `rounded-2xl` — persegi membulat sesuai radius besar DESIGN.md, bukan lingkaran kecil lagi).
- [x] Klik foto (yang ada isinya) membuka tampilan ukuran besar (lightbox/dialog), bisa ditutup dengan wajar. `Dialog`/`DialogContent` baru (reuse primitive existing di file yang sama), close button X bawaan Radix + klik-luar + ESC (bawaan `DialogPrimitive`, tidak perlu implementasi manual).
- [x] Fungsi hapus foto (untuk role yang berhak) tetap berfungsi normal, tidak tertimpa/rusak oleh fitur lightbox baru. **Dipisah jadi 2 area klik**: klik foto (area besar) = buka lightbox; tombol hapus (ikon kecil pojok kanan-atas, muncul saat hover) = tetap trigger `confirmingDelete` seperti sebelumnya — tidak ada tumpang tindih event handler.
- [x] Piket (guru_piket) TIDAK melihat section "Riwayat Kartu" di profil siswa. `hideRiwayatKartu` prop baru, dikirim dari `page.tsx` (`isPiket`, variabel yang SUDAH ADA sebelumnya untuk keperluan lain — dipakai ulang, bukan dihitung ulang).
- [x] card_admin TETAP melihat section "Riwayat Kartu" (regresi nol untuk role ini). `hideRiwayatKartu` SENGAJA terpisah dari `readOnly` (yang dipakai bareng `isPiket || isCardAdmin`) — `isCardAdmin` TIDAK ikut jadi `true` untuk prop baru ini.
- [x] super_admin TETAP melihat semuanya seperti sekarang (regresi nol). `isPiket`/`isCardAdmin` keduanya `false` untuk role ini, `hideRiwayatKartu` otomatis `false`.
- [x] Build + type-check `apps/web` hijau. `tsc --noEmit` bersih, `next build` sukses (`/piket` 7.28 kB, `/siswa/[id]` 5.85 kB, keduanya naik wajar sesuai penambahan kode).

## Validasi Claudian
- [x] Poin 4 — dikonfirmasi TIDAK reuse `readOnly` — prop baru `hideRiwayatKartu` dikirim eksplisit dari `page.tsx` berdasar `isPiket` saja (bukan `readOnly` yang gabungan `isPiket || isCardAdmin`).
- [x] Poin 3 — lightbox (klik foto besar) dan hapus-foto (tombol kecil terpisah, pojok kanan-atas, hover-reveal) sengaja dipisah jadi 2 elemen `<button>` berbeda dengan area klik tidak overlap — bukan 1 elemen dengan 2 handler bertumpuk. Verifikasi visual browser TIDAK dijalankan (Playwright off, sesuai pola sesi ini) — logic dipastikan benar lewat pembacaan kode: 2 button terpisah, `stopPropagation` tidak diperlukan karena tidak nested.

## Status Eksekusi — SELESAI (2026-08-06)
`apps/web/src/app/(piket)/piket/piket-board-view.tsx` (poin 1-2), `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` + `page.tsx` (poin 3-4). Tidak ada perubahan backend/API sama sekali (murni frontend sesuai spec). Verifikasi: `tsc`+`next build` bersih. Verifikasi visual browser tidak dijalankan — user diminta cek manual sendiri sesuai pola sesi ini (Playwright tidak dipakai).

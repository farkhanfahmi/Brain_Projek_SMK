# T256 — Web: Manajemen Akun Murid — Kelompokkan per Kelas + Accordion

## Depends on
Tidak ada. Perbaikan UI di section "Murid" halaman Akun admin (`akun-view.tsx`) yang sudah
ada (dibangun sesi lain sebagai lanjutan T253, disebut "T253-lanjutan" di kode).

## Objective
Section "Murid" di halaman Akun admin — ganti dari tabel flat semua akun jadi: list nama
kelas dulu, klik 1 kelas → accordion expand menampilkan daftar akun murid di kelas itu.

## Konteks — Kondisi Existing Dikonfirmasi 2026-08-28

### Bagian yang SUDAH BENAR, JANGAN diubah
- Akun nonaktif (diganti wali kelas via T248) TETAP muncul di daftar dengan badge
  "Nonaktif" (`akun-view.tsx` baris ~560-564) — **TIDAK dihapus permanen**, konsisten
  aturan `UsersService.remove()` (`users.service.ts:194-208`): hard-delete HANYA kalau akun
  itu belum PERNAH punya `activity_log` — akun murid yang sudah pernah dipakai (hampir pasti
  py log) otomatis DINONAKTIFKAN saja, bukan dihapus. **Dikonfirmasi user (2026-08-28)
  perilaku ini SUDAH SESUAI keinginan — tidak ada perubahan kebijakan hapus/nonaktif di
  task ini**, murni restrukturisasi TAMPILAN.
- Tombol "Buat Akun"+"Edit" sudah disembunyikan khusus section ini (provisioning HANYA
  lewat alur T248) — TIDAK diubah.
- Kolom NISN+nama kelas sudah ditampilkan per baris (baris ~547-552) — data ini nanti
  dipakai sebagai KEY pengelompokan, bukan field baru.

### Yang diubah task ini
Section "Murid" saat ini render `<Table>` SAMA seperti section lain (Admin/Piket/Guru/
Pembina Ekstra) — flat, 1 baris = 1 akun, TIDAK dikelompokkan. User minta restrukturisasi
KHUSUS section ini (section lain TIDAK berubah).

## Spec Detail

### Struktur baru khusus section "Murid"
- **Level 1**: list nama kelas (yang PUNYA minimal 1 akun murid — kelas tanpa akun murid
  TIDAK muncul di list ini sama sekali). Tampilan: card/baris per kelas, tampilkan nama
  kelas + jumlah akun (mis. "X TKJ 1 (2 akun)") — REPLIKASI pola SummaryCard/expand yang
  SUDAH ADA di codebase ini (`piket-board-view.tsx`/`hari-ini-tab.tsx` sudah py pola
  klik-untuk-expand, JANGAN reinvent pola baru).
- **Level 2 (accordion)**: klik nama kelas → expand di bawahnya, tampilkan tabel/list akun
  murid DI KELAS ITU SAJA — kolom SAMA seperti sebelumnya (username, jabatan/role, nama+NISN,
  status, aksi Reset Password/Nonaktifkan). Hanya 1 kelas boleh expand dalam satu waktu
  (VERIFIKASI ke user kalau ternyata mau multi-expand sekaligus — REKOMENDASI: 1 saja,
  konsisten pola existing SummaryCard di file lain).
- **Search/filter existing** (kalau section Murid sudah py search box, VERIFIKASI SAAT
  IMPLEMENTASI) — kalau user search nama/username, REKOMENDASI: auto-expand kelas yang
  match hasil pencarian (supaya search tetap berguna meski di balik accordion), JANGAN
  biarkan search "hilang" tersembunyi di kelas yang collapsed.

## Edge Cases
- **Kelas dengan SEMUA akun muridnya nonaktif** (Ketua+Wakil Ketua sudah diganti wali kelas,
  akun lama semua nonaktif) — kelas ITU TETAP muncul di list Level 1 (akun nonaktif tetap
  "akun yang ada", cuma statusnya beda) — JANGAN disembunyikan dari list kelas.
- **1 kelas dengan akun SANGAT banyak** (kasus tidak wajar tapi mungkin — turnover pengurus
  berkali-kali dalam 1 tahun ajaran, akun nonaktif menumpuk) — accordion tetap scroll biasa,
  TIDAK PERLU pagination internal accordion (skala realistis kecil, maksimal beberapa akun
  per kelas per tahun).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/akun/akun-view.tsx` — struktur render KHUSUS
  cabang `activeSection === "murid"` (section lain TIDAK disentuh, REKOMENDASI extract jadi
  sub-komponen terpisah kalau perubahan struktur cukup besar berbeda dari render tabel biasa
  yang dipakai section lain, supaya tidak mempersulit baca kode section lain yang tidak
  berubah).

## Acceptance Criteria
- [x] Section "Murid" tampil list nama kelas dulu (bukan tabel flat semua akun langsung) —
      komponen baru `MuridSection`, dikelompokkan via `Map` keyed nama kelas.
- [x] Klik nama kelas → accordion expand tampilkan akun murid di kelas itu SAJA — REUSE
      `AccountTable` apa adanya sebagai isi expand (kolom+aksi 100% identik section lain).
- [x] Kelas tanpa akun murid tidak muncul di list — `Map` cuma diisi dari akun yang ADA
      (iterasi `users`, BUKAN iterasi semua `Kelas`), jadi kelas kosong secara struktural
      tidak mungkin muncul.
- [x] Akun nonaktif tetap tampil (dengan badge Nonaktif) di dalam accordion kelasnya — TIDAK
      ADA filter status apa pun di `MuridSection`/`AccountTable`, badge Nonaktif existing
      di `AccountTable` tidak disentuh.
- [x] Section lain (Admin/Piket/Guru/Pembina Ekstra) TIDAK berubah sama sekali — `AccountTable`
      component BYTE-FOR-BYTE tidak diubah, cabang render 4 section itu tetap manggil
      `<AccountTable users={usersBySection[s]} .../>` persis sama seperti sebelumnya.
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `next build` sukses).

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA perubahan ke `UsersService.remove()`/kebijakan hapus-vs-nonaktif
      — dikonfirmasi: TIDAK ADA satu file backend pun disentuh task ini, murni
      `akun-view.tsx` (1 file, penambahan komponen baru + 1 percabangan render).
- [x] Konfirmasi 4 section lain benar-benar tidak terpengaruh — dijamin STRUKTURAL (bukan
      cuma test manual): percabangan `s === "murid" ? <MuridSection/> : <AccountTable/>`
      di titik render TUNGGAL, 4 section lain masuk cabang `else` yang identik kode lama
      persis (tidak ada 1 baris pun diubah di `AccountTable` maupun pemanggilannya utk
      section lain). Test manual pindah tab browser TIDAK dilakukan sesi ini (lihat
      Implementasi) — dijamin lewat argumen struktural di atas, bukan observasi visual.

## Implementasi (2026-08-28)

Komponen baru `MuridSection` disisipkan tepat sebelum `AccountTable` di `akun-view.tsx`.
Level 1 (list kelas): `Map<string, UserAccount[]>` dibangun dari `users` (prop
`usersBySection.murid`), key = `student.kelas?.nama ?? "Tanpa Kelas"` (fallback jaga-jaga,
harusnya tidak pernah kepakai selama invariant T247 studentId->kelasId terjaga), diurutkan
alfabetis. Tiap kelas dirender sebagai baris "Nama Kelas (N akun)" + chevron, klik toggle
expand — state `manualExpanded: string | null`, HANYA 1 kelas boleh terbuka (klik kelas
lain otomatis tutup yang sebelumnya), REPLIKASI pola `expandedKey`/`expandedHari` yang
sudah established di `hari-ini-tab.tsx`/`struktur-piket-tab.tsx`.

**Search level-1** (BEDA dari search internal `AccountTable` per-kelas) — filter SEMUA
akun murid lintas kelas dulu (username/nama siswa/NISN), BARU dikelompokkan dari hasil
filter itu — efeknya kelas yang 0 akunnya cocok otomatis hilang dari list. Selagi search
aktif, SEMUA kelas hasil (bisa >1) di-auto-expand (`searching` override `manualExpanded`,
chevron+klik manual dinonaktifkan sementara) — supaya hasil pencarian TIDAK "hilang" di
balik accordion tertutup, sesuai spec. Search dikosongkan → balik ke mode manual single-
expand seperti semula.

**Level 2 (isi expand)**: REUSE `<AccountTable>` APA ADANYA (nol modifikasi ke komponen
itu) dengan `users` di-scope ke 1 kelas — otomatis dapat search+sort+kolom No+Aksi
(Reset/Set Password/Unlock/Delete) identik section lain TANPA duplikasi logic sama
sekali, sekaligus otomatis comply aturan wajib "semua kolom tabel sortable" project
(warisan langsung dari `AccountTable` yang sudah comply).

**Verifikasi**: `tsc --noEmit` bersih, `next build` web sukses (route `/akun` ter-build
normal). Live browser test (klik akordeon sungguhan, verifikasi search auto-expand
multi-kelas, cek akun nonaktif tetap kelihatan) BELUM dilakukan sesi ini — regresi ke 4
section lain dijamin lewat argumen struktural (kode section lain betul-betul 0 baris
berubah), TAPI perilaku BARU section Murid sendiri (accordion+auto-expand search)
direkomendasikan di-klik-coba manual di browser sebelum dianggap 100% final secara UX.

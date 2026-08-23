# T207 — API+Web: Form Mapel — Tambah Assignment Guru Pengampu (Many-to-Many, Team Teaching)

## Depends on
**WAJIB SETELAH T204** (butuh model `MapelGuru`). Independen dari T203/T206/rangkaian Opsi Jadwal lain secara teknis (bisa dikerjakan paralel), TAPI **WAJIB SELESAI SEBELUM T212** (form input JadwalSlot butuh data `MapelGuru` untuk filter dropdown Guru).

## Objective
Form Tambah/Edit Mapel (SUDAH ADA, T201 — sekarang punya checklist Jurusan many-to-many) — TAMBAH bagian BARU: **checklist/multi-select Guru Pengampu** — 1 Mapel bisa diampu banyak guru (team teaching), 1 guru bisa mengampu banyak mapel.

## Konteks — Keputusan Dikonfirmasi User (2026-08-16)

- Assignment ini **WAJIB diisi** sebelum guru bisa dipilih di form input jadwal (T212) — BUKAN opsional dengan fallback "tampilkan semua guru".
- Layout form Mapel HARUS diubah supaya "fleksibel mendefinisikan beberapa guru" — form SAAT INI (T201, sudah punya checklist Jurusan) perlu diperluas TANPA membuatnya berantakan/terlalu padat.

## Spec Detail

### 1. Backend — endpoint assignment guru di Mapel

- `MapelService.create()`/`update()` — TERIMA parameter BARU `teacherIds: number[]` (BOLEH KOSONG saat create pertama kali, tapi TANDAI di UI bahwa mapel tanpa guru terdaftar TIDAK BISA dipakai di form jadwal sampai diisi — lihat poin UI). REPLACE SELURUH baris `MapelGuru` untuk mapel itu tiap update (pola REPLACE PENUH, KONSISTEN pola `MapelJurusan` T201).
- `MapelController.findAll()`/`findOne()` — response SEKARANG include `teacherIds: number[]` atau `teachers: {id, nama}[]` (PUTUSKAN shape response saat implementasi, REKOMENDASI `teachers: {id, nama, niy}[]` supaya frontend tidak perlu join manual).
- `MapelService.delete()` — VALIDASI referensi (T196, SUDAH ADA — cek Schedule/TeachingSession/GradeAssessment) — TIDAK PERLU tambah cek `MapelGuru` sebagai penghalang hapus (assignment guru bukan data historis yang perlu dilindungi, boleh cascade hapus otomatis via `onDelete: Cascade` di schema T204).

### 2. Frontend — layout form Mapel diperluas

- `mapel-view.tsx` (form Tambah/Edit) — TAMBAH section BARU "Guru Pengampu" SEJAJAR section "Jurusan" yang sudah ada (T201) — checklist/multi-select guru (REKOMENDASI: search-box + checklist untuk daftar guru yang bisa panjang — BEDA dari checklist Jurusan yang cuma beberapa item, jumlah guru bisa puluhan/ratusan, JANGAN pakai checklist polos tanpa search untuk daftar sepanjang ini).
- **Layout keseluruhan form**: PERTIMBANGKAN apakah form Mapel (sekarang: nama, kode, checklist Jurusan, checklist Guru) masih cocok sebagai Dialog kecil, atau SUDAH PERLU jadi Sheet/Full Page (KONSISTEN aturan design-system vault `03-components.md` "Form Input Panjang — Sheet, bukan Dialog kecil untuk >6 field ATAU field yang butuh pengelompokan section bermakna") — dengan 2 checklist (Jurusan+Guru) yang masing-masing bisa panjang, REKOMENDASI KUAT ganti ke Sheet (bukan Dialog kecil lagi), KONSISTEN pola Sheet yang sudah dipakai form Siswa/Guru.
- Tabel daftar Mapel — TAMBAH kolom "Guru Pengampu" (badge multi-guru, atau ringkasan "3 guru" dengan tooltip/expand — PUTUSKAN saat implementasi supaya tabel tidak terlalu lebar, KONSISTEN aturan mobile-first).

## Edge Cases
- Mapel TANPA guru terdaftar sama sekali (baru dibuat, belum diisi) — TIDAK ERROR saat create Mapel itu sendiri (assignment guru MEMANG boleh menyusul), TAPI form input JadwalSlot (T212) untuk Mapel itu HARUS tampilkan pesan jelas ("Mapel ini belum punya guru pengampu terdaftar — daftarkan dulu di menu Mata Pelajaran") BUKAN dropdown kosong tanpa penjelasan.
- Guru yang DIHAPUS assignment-nya dari 1 Mapel (dari update form) TAPI SUDAH ADA `JadwalSlot` yang merujuk guru itu untuk Mapel tersebut — TIDAK ADA validasi block dari task ini (data jadwal lama TETAP ADA, guru itu tetap tercatat mengajar jadwal yang sudah dibuat, HANYA tidak bisa dipilih LAGI untuk jadwal BARU) — INI PERILAKU YANG BENAR, JANGAN cegah unassign guru hanya karena ada histori.

## Files
- **Modifikasi:** `apps/api/src/mapel/mapel.service.ts`+`mapel.controller.ts`+DTO (terima+expose `teacherIds`), `apps/web/.../mapel/mapel-view.tsx` (section Guru Pengampu, PERTIMBANGKAN ganti Dialog→Sheet).
- **Jangan sentuh:** checklist Jurusan (T201, TIDAK diubah logic-nya, HANYA ditata ulang layout kalau form pindah ke Sheet).

## Acceptance Criteria
- [x] Form Mapel punya section "Guru Pengampu" dengan search+checklist.
- [x] 1 Mapel bisa diampu banyak guru, 1 guru bisa mengampu banyak mapel — didukung schema+service (many-to-many `MapelGuru`), verified via test create/update dengan `teacherIds` multi-elemen.
- [x] Tabel Mapel tampilkan ringkasan guru pengampu — badge jumlah (hijau "N guru" / merah "Belum ada guru"), tooltip nama lengkap.
- [x] Mapel tanpa guru terdaftar TETAP bisa dibuat (tidak wajib saat create), ditandai badge merah jelas di tabel.
- [x] Build + type-check hijau, 9 jest baru untuk assignment many-to-many.

## Validasi Claudian
- [x] **Form Mapel PINDAH dari Dialog ke Sheet** — alasan: form sudah 3 field dasar (nama/kode) + 2 checklist yang masing-masing bisa panjang (Jurusan + Guru Pengampu, guru bisa puluhan/ratusan), sesuai aturan design-system "Form Input Panjang — Sheet, bukan Dialog kecil untuk >6 field ATAU butuh pengelompokan section bermakna". Pola SheetHeader/SheetBody/SheetFooter direplikasi dari `guru-view.tsx`.
- [x] Konfirmasi hapus assignment guru dari Mapel TIDAK memblokir/menghapus `JadwalSlot` historis — `MapelGuru.mapel` pakai `onDelete: Cascade` dari sisi Mapel (T204), TIDAK ada FK dari `JadwalSlot`/`JadwalSlotGuru` ke `MapelGuru` sama sekali (relasi independen), jadi unassign guru dari Mapel HANYA menghapus baris `MapelGuru` itu sendiri — data `JadwalSlot`/`JadwalSlotGuru` lama sama sekali tidak tersentuh secara struktural.

## Catatan Implementasi (2026-08-17)

- **Sesi paralel T206**: dikerjakan bersamaan tanpa konflik file — T207 scope murni `apps/api/src/mapel/`, `apps/web/.../mapel/`, `core-types.ts` (interface `Mapel`), T206 scope `opsi-jadwal`/`jadwal-slot` (modul terpisah total). Backend dev-server log sempat menunjukkan compile error transien `flattenJurusan does not exist` saat proses edit berlangsung (normal, watch process recompile mid-edit) — settle bersih 0 error setelah selesai.
- **Backend**: `MapelService.flattenJurusan()` di-generalisasi jadi `flattenRelations()` (handle jurusan+guru sekaligus). Query `findAll`/`findOne` include `guruPengampu: { include: { teacher: true } }`. Pesan error P2003 dibedakan by `error.meta.field_name` (mengandung "teacher" → pesan sebut guru, selain itu → pesan sebut jurusan) — PATUH aturan CLAUDE.md "pesan error sesuai kondisi, bukan generik".
- **Frontend**: `Mapel.teachers: {id,nama,niy}[]` ditambah ke `core-types.ts`. Checklist Guru Pengampu pakai search-filter client-side (`useMemo` filter by nama/NIY) di dalam container `max-h-56 overflow-y-auto` (daftar bisa panjang, checklist polos tanpa scroll+search tidak cocok, beda dari checklist Jurusan yang tetap polos karena item sedikit).
- **2 route diupdate**: `(admin-jurnal)/admin-jurnal/mapel/page.tsx` dan `(admin)/mapel/page.tsx` (T157 duplikasi) — keduanya tambah fetch `GET /teachers`, pass `teacherList` ke `MapelView` yang sama (component generic, tidak ada duplikasi logic).

### Verifikasi

- `tsc --noEmit` bersih 2 app, `nest build`+`next build` sukses.
- `jest apps/api`: 534/534 pass (32/32 suite, naik dari 494 pasca T203+T204 — termasuk suite baru `opsi-jadwal.service.spec.ts` dari sesi T206 paralel, tidak ada file test yang bentrok).
- 9 test baru: create/update dengan `teacherIds` (kosong, terisi, `[]` eksplisit), independensi dari `jurusanIds` (update salah satu tidak mengubah yang lain), response shape `teachers`, pesan error P2003 spesifik guru.
- `git status` dikonfirmasi scope T207 hanya menyentuh file modul `mapel/` + `core-types.ts`, tidak ada overlap dengan file T206.

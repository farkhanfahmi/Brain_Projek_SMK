# T078 — UI: Split Section Kartu (Guru/Siswa) + Filter Kelas & Jurusan + Kolom Jurusan

## Depends on
Tidak ada — murni refactor UI di `kartu-view.tsx`, tidak ada perubahan API/skema.

## Context
- **App:** `apps/web`
- **File:** `apps/web/src/app/(admin)/kartu/kartu-view.tsx`
- **Ref:** Diminta user 2026-07-24 — halaman `/kartu` saat ini 1 tabel gabungan Guru+Siswa dengan filter status saja. User ingin section terpisah per tipe pemilik, plus filter Kelas/Jurusan dan kolom Jurusan (khusus section Siswa — guru tidak punya jurusan di skema, `Teacher` tidak ber-relasi ke `Jurusan`/`Kelas`).

## Spec Detail

### Masalah
`kartu-view.tsx:148-224` — satu `<Table>` menampilkan kartu siswa dan guru bercampur dalam kolom "Pemilik" yang beda format teks (`(Siswa)` / `(Guru)`), filter cuma status aktif/nonaktif.

### Solusi — 2 Section Terpisah
1. **Section "Kartu Guru"** (card terpisah, di atas atau di tab terpisah — pilih **Tabs** dua nilai "Siswa" / "Guru" dalam 1 halaman, pola sama seperti `Tabs` yang sudah dipakai di `GuruForm`) berisi tabel kartu milik `teacher != null`:
   - Kolom: UID, Nama Guru, Status, Diterbitkan, Aksi (Ganti/Nonaktifkan — reuse tombol existing)
   - Filter: hanya Status (Aktif/Nonaktif) — TIDAK ada filter Kelas/Jurusan (guru tidak berelasi ke situ)

2. **Section "Kartu Siswa"** berisi tabel kartu milik `student != null`:
   - Kolom: UID, Nama Siswa, **Jurusan** (baru), Kelas, Status, Diterbitkan, Aksi
   - Filter: Status + **Kelas** (dropdown, dari daftar kelas kampus aktif) + **Jurusan** (dropdown dari `Jurusan.nama`) — filter Kelas dan Jurusan saling terhubung (pilih Jurusan mempersempit opsi Kelas yang muncul, pola sama seperti filter kelas/jurusan di `(admin)/siswa/siswa-view.tsx` — CEK dulu implementasinya, REUSE komponen filter itu kalau strukturnya generik)

### Cara Ambil Data Jurusan Siswa
- `Card.student` (relasi existing) sudah include field `Student` dasar. Perlu extend query/type: `card.student.kelas.jurusan.nama` — cek endpoint `GET /cards` di `apps/api/src/core/cards/` (atau modul terkait), pastikan `include` Prisma-nya sudah menyertakan `kelas.jurusan`, tambahkan kalau belum
- Update type `Card`/`Student` di `apps/web/src/lib/core-types.ts` supaya field jurusan+kelas ikut ter-tipe

### Registrasi Kartu (Dialog existing)
- `RegisterCardForm` (kartu-view.tsx:237-354) — TIDAK perlu diubah signifikan, cukup pastikan dropdown "Tipe Pemilik" (Siswa/Guru) tetap ada di 1 dialog yang sama untuk registrasi baru terlepas dari section mana yang sedang aktif dilihat (tombol "Registrasi Kartu" tetap 1, muncul di kedua tab, bukan diduplikasi jadi 2 tombol berbeda)

## JANGAN
- ❌ JANGAN buat 2 halaman route terpisah (`/kartu/guru`, `/kartu/siswa`) — cukup 1 halaman `/kartu` dengan `Tabs` di dalamnya, konsisten dengan pola Tabs yang sudah ada di codebase
- ❌ JANGAN tambahkan kolom Jurusan di section Guru — guru tidak punya jurusan, jangan paksakan field kosong
- ❌ JANGAN duplikasi state `cards` jadi 2 array terpisah — tetap 1 source of truth (`cards`), filter jadi 2 computed list (`cardsGuru`, `cardsSiswa`) via `.filter()`, sama seperti pola `filteredCards` yang sudah ada

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kartu/kartu-view.tsx` — refactor jadi Tabs 2 section + filter Kelas/Jurusan di section Siswa
- **Modifikasi (cek dulu):** endpoint `GET /cards` di `apps/api/src/core/cards/cards.controller.ts` atau service terkait — pastikan `include: { student: { include: { kelas: { include: { jurusan: true } } } } }` sudah ada
- **Modifikasi:** `apps/web/src/lib/core-types.ts` — extend type `Student`/`Card` untuk field kelas+jurusan bila belum ada

## Acceptance Criteria
- [ ] Halaman `/kartu` punya 2 tab: "Kartu Guru" dan "Kartu Siswa"
- [ ] Tab Kartu Siswa menampilkan kolom Jurusan dan Kelas
- [ ] Tab Kartu Siswa punya filter dropdown Kelas dan Jurusan, keduanya berfungsi dan saling mempersempit
- [ ] Tab Kartu Guru tidak menampilkan kolom/filter Jurusan
- [ ] Tombol Registrasi Kartu, Import Data, Tap-to-Assign tetap berfungsi dari kedua tab

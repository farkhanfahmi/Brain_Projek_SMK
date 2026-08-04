# T079 — UI: Split Section "Foto Tersimpan" (Guru/Siswa) + Kolom Jurusan

## Depends on
Tidak ada — murni refactor UI di `foto-view.tsx`, tidak ada perubahan API/skema (kecuali `include` Prisma bila belum menyertakan jurusan, sama seperti catatan di [[T078-kartu-split-section-filter]]).

## Context
- **App:** `apps/web`
- **File:** `apps/web/src/app/(admin)/foto/foto-view.tsx`
- **Ref:** Diminta user 2026-07-24 — bagian "Foto Tersimpan" (baris 197-245) saat ini 1 tabel gabungan siswa+guru dengan kolom Tipe untuk membedakan. User ingin section terpisah + kolom Jurusan (siswa saja).

## Spec Detail

### Masalah
`foto-view.tsx:56-63` — `hasFotoEntries` menggabungkan `students` dan `teachers` jadi 1 array, di-render 1 tabel dengan kolom "Tipe" untuk membedakan (baris 227-229).

### Solusi
1. **Pisahkan jadi 2 tabel/section** dalam card "Foto Tersimpan" yang sama (tidak perlu Tabs terpisah seperti T078, karena ini cuma list read+delete, lebih ringan — cukup 2 sub-heading "Foto Guru" dan "Foto Siswa" masing-masing dengan tabelnya sendiri, di dalam 1 card, dipisahkan divider atau spacing jelas)
2. **Hapus kolom "Tipe"** dari kedua tabel (sudah implisit dari section-nya)
3. **Section Foto Siswa** — tambah kolom **Jurusan** (dan idealnya Kelas juga, karena keduanya sudah tersedia lewat relasi `student.kelas.jurusan`) setelah kolom NISN
4. **Section Foto Guru** — TIDAK ada kolom Jurusan, tetap: foto, Nama, NIY, Aksi (hapus)

### Cara Ambil Data Jurusan
- `Student` type di `foto-view.tsx` props perlu field kelas+jurusan — cek endpoint yang mem-fetch data untuk halaman `/foto` (`apps/web/src/app/(admin)/foto/page.tsx`), pastikan query API `GET /students` yang dipanggil di situ sudah `include` kelas+jurusan (kemungkinan sama seperti catatan di T078 — cek satu kali, terapkan konsisten di kedua tempat kalau memang butuh perubahan API yang sama)

## JANGAN
- ❌ JANGAN duplikasi logic delete foto (`DeletePhotoForm`) — 1 komponen dipakai untuk kedua section, sama seperti sekarang
- ❌ JANGAN ubah flow upload/assign (bagian atas halaman, baris 1-195) — scope task ini HANYA section "Foto Tersimpan" di bagian bawah
- ❌ JANGAN tambah kolom Jurusan di section Foto Guru

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/foto/foto-view.tsx` — split `hasFotoEntries` jadi 2 array (`fotoGuru`, `fotoSiswa`), render 2 tabel terpisah, tambah kolom Jurusan+Kelas di tabel siswa
- **Modifikasi (bila perlu, cek dulu):** `apps/web/src/app/(admin)/foto/page.tsx` — pastikan fetch `students` sudah include kelas+jurusan

## Acceptance Criteria
- [ ] Section "Foto Tersimpan" terbagi jadi 2 sub-bagian: "Foto Guru" dan "Foto Siswa"
- [ ] Tabel Foto Siswa menampilkan kolom Jurusan dan Kelas
- [ ] Tabel Foto Guru tidak menampilkan kolom Jurusan/Kelas
- [ ] Hapus foto tetap berfungsi dari kedua section
- [ ] Kolom "Tipe" yang lama sudah dihapus dari kedua tabel (redundan dengan section)

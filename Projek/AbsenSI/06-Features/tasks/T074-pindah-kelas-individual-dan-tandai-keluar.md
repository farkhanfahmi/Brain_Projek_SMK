# T074 — UI: Pindah Kelas Individual + Tandai Siswa Keluar (Halaman Detail Siswa)

## Depends on
T063 (field `alasanNonaktif`/`tahunLulus` sudah ada), tidak depend T071-T073 (bagian ini independen, bisa dikerjakan paralel)

## Objective
Tambah 2 aksi ke halaman detail siswa (`siswa-detail-view.tsx`) yang saat ini tidak ada UI-nya sama sekali: (1) dropdown pindah kelas, (2) tombol "Tandai Keluar" dengan alasan Lulus/Mengundurkan Diri/Lainnya.

## Context
- **App:** `apps/web`
- **File existing:** `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` (424 baris — cek isi lengkapnya dulu sebelum edit, jangan asumsi struktur tanpa baca)
- **Ref:** `Projek/AbsenSI/06-Features/plotting-siswa-kelas.md` — bagian "3️⃣ Pindah Kelas Individual" dan "4️⃣ Tandai Siswa Keluar"
- **Backend:** TIDAK PERLU endpoint baru — `PATCH /students/:id` (`UpdateStudentDto extends PartialType(CreateStudentDto)`) sudah mendukung update `kelasId`, `status`, `alasanNonaktif`, `tahunLulus`

## Spec Detail

### Bagian 3 — Dropdown Pindah Kelas
- Tambah field "Kelas" di halaman detail siswa — tampilkan kelas saat ini, dengan tombol edit/dropdown untuk ganti
- Submit → `PATCH /students/:id` dengan `{ kelasId: newId }`
- Update tampilan tanpa reload penuh halaman setelah sukses

### Bagian 4 — Tombol "Tandai Keluar"
- Tombol di halaman detail siswa → buka Dialog kecil (ini KASUS ≤6 field sederhana, sesuai aturan `03-components.md` — Dialog kecil TETAP dipakai, BUKAN Sheet):
  - Dropdown "Alasan": Lulus / Mengundurkan Diri / Lainnya
  - Kalau "Lulus" dipilih → field "Tahun Lulus" (number input) muncul, disembunyikan untuk alasan lain (logic kondisional sama seperti yang dirancang T066)
  - Tombol "Simpan" → `PATCH /students/:id` dengan `{ status: "nonaktif", alasanNonaktif, tahunLulus? }`
- **Reversible**: field Status yang sudah ada di halaman ini (existing, filterable) — pastikan BISA diedit balik ke `aktif` (cek behavior existing, tambahkan kalau belum ada cara ubah balik)

## JANGAN
- ❌ JANGAN buat endpoint API baru — `PATCH /students/:id` sudah cukup untuk kedua aksi ini
- ❌ JANGAN tampilkan field Tahun Lulus untuk alasan selain "Lulus"
- ❌ JANGAN buat aksi "Tandai Keluar" ireversibel — admin harus bisa mengembalikan siswa ke `aktif` kalau salah tandai

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx`

## Acceptance Criteria
- [ ] Ubah kelas siswa dari dropdown → tersimpan, terverifikasi via MySQL MCP
- [ ] Klik "Tandai Keluar" → pilih Lulus → isi Tahun Lulus → simpan → status siswa jadi Nonaktif, `alasanNonaktif: lulus`, `tahunLulus` tersimpan
- [ ] Pilih "Mengundurkan Diri" → field Tahun Lulus tidak muncul/tidak dikirim
- [ ] Siswa yang sudah ditandai Nonaktif bisa diubah balik ke Aktif dari halaman yang sama

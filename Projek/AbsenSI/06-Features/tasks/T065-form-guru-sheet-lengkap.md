# T065 — UI: Form Guru — Migrasi ke Sheet + Field Biodata Lengkap

## Depends on
T064 (primitif Sheet/Tabs), T061 (schema field biodata guru baru sudah ada di backend & tervalidasi bisa insert lewat API)

## Objective
Ganti `GuruForm` existing (popup `Dialog` 3 field: NIY, Nama, No HP) jadi form `Sheet` bertab yang mencakup seluruh field biodata guru baru dari T061 (gelar, status pernikahan, status kepegawaian, agama, tempat/tanggal lahir, jenis kelamin, alamat) — form 3 field yang ada sekarang sudah tidak representatif untuk skema yang sudah bertambah jauh lebih kaya.

## Context
- **App:** `apps/web`
- **File existing:** `apps/web/src/app/(admin)/guru/guru-view.tsx` — baca dulu isi lengkapnya (sudah dibaca saat breakdown task ini, 191 baris, `GuruForm` di baris 110-191)
- **Ref:** `Projek/AbsenSI/06-Features/design-system/03-components.md` bagian "Form Input Panjang" (WAJIB), `Projek/AbsenSI/06-Features/migrasi-database-lama.md` untuk daftar field baru

## Spec Detail

### Ganti kontainer: `Dialog` → `Sheet`
- `GuruForm` sekarang dibungkus `Sheet`/`SheetContent` (dari T064), BUKAN `Dialog`/`DialogContent`
- Trigger tombol "Tambah Guru" tetap sama posisi/stylingnya, hanya target buka Sheet bukan Dialog

### Struktur Tab (3 section)
1. **"Data Pokok"** — NIY, Nama, Status Kepegawaian (dropdown: Guru/Karyawan), Status (Aktif/Nonaktif/Cuti — cek dulu field `status` existing di `Teacher`, `PersonStatus` cuma aktif/nonaktif, KONFIRMASI ke user dulu kalau perlu tambah value "cuti" atau dipetakan ke salah satu existing sebelum implementasi field ini — jangan asumsi sendiri)
2. **"Biodata"** — Gelar Depan, Gelar Belakang, Tempat Lahir, Tanggal Lahir (pakai `DatePicker` existing dari `packages/ui`), Jenis Kelamin (dropdown/radio: Laki-laki/Perempuan), Agama (dropdown: Islam/Kristen/Katolik/Hindu/Buddha/Konghucu), Status Pernikahan (dropdown: Menikah/Belum Menikah/Pernah Menikah)
3. **"Kontak & Alamat"** — No HP, Alamat (textarea, full-width)

### Grid Layout
- Dalam tiap tab, field pendek pakai grid 2 kolom (`grid-cols-2 gap-4`) sesuai spec, field panjang (Alamat) full-width
- Semua field baru (kecuali NIY, Nama) **opsional** — konsisten dengan sifat data lama yang banyak NULL, JANGAN buat wajib diisi

### Migrasi Form Edit (kalau ada)
- Cek apakah ada form EDIT guru existing selain create — kalau ada, terapkan pola Sheet+Tabs yang sama untuk konsistensi (jangan hanya form create yang diperbarui, form edit dibiarkan lama)

## JANGAN
- ❌ JANGAN buat field baru jadi wajib (`required`) — semua data biodata ini historis/opsional, memaksa isi ulang untuk guru yang sudah ada akan menyulitkan alur kerja admin
- ❌ JANGAN hardcode warna/radius sendiri untuk Tabs — pakai komponen `Tabs` dari T064 apa adanya, jangan override stylingnya per halaman
- ❌ JANGAN hapus field NIY/Nama/No HP yang sudah ada dari form lama — task ini MENAMBAH field & mengubah KONTAINER, bukan mengurangi yang sudah berfungsi
- ❌ JANGAN implementasikan value "Cuti" untuk status guru tanpa konfirmasi user dulu — `PersonStatus` existing cuma `aktif`/`nonaktif`, ini butuh keputusan skema terpisah kalau memang dibutuhkan

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/guru/guru-view.tsx` — ganti `Dialog` jadi `Sheet`, tambah field baru dengan struktur Tab

## Acceptance Criteria
- [ ] Klik "Tambah Guru" → Sheet geser dari kanan (bukan Dialog di tengah), lebar 480-560px
- [ ] 3 tab (Data Pokok, Biodata, Kontak & Alamat) berfungsi, berpindah tanpa reload/kehilangan data yang sudah diisi di tab lain
- [ ] Submit form dengan hanya NIY+Nama terisi (field lain kosong) → berhasil tersimpan tanpa error validasi
- [ ] Submit form dengan semua field terisi → data tersimpan lengkap, bisa diverifikasi lewat MySQL MCP (`SELECT * FROM teachers WHERE niy = ...`)
- [ ] Footer (tombol Batal/Simpan) tetap terlihat saat body di-scroll

## Handoff
Pola ini (Sheet+Tabs) jadi rujukan untuk T066 (form Siswa) dan form-form input data panjang lainnya ke depan.

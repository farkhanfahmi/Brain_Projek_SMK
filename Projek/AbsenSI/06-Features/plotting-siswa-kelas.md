---
tags: [absensi, feature, siswa, kelas, admin, final]
status: final
updated: 2026-07-22
---

# Feature — Plotting Siswa ke Kelas (Menu Kelas)

← Index (00-INDEX AbsenSI.md)

> **Status: FINAL, siap breakdown task.** Ditemukan sebagai celah 2026-07-22: siswa cuma bisa di-assign ke kelas SEKALI saat form "Tambah Siswa" dibuat — tidak ada mekanisme pindah kelas (individual maupun massal), tidak ada penempatan siswa baru (SPMB) secara batch, dan tidak ada aksi "Tandai Keluar/Lulus" dari UI meski field skemanya (`status`, `alasanNonaktif` dari T063) sudah ada.

---

## 🎯 Konsep: 3 Mekanisme Terpisah

Ditemukan lewat interview bahwa "plotting siswa ke kelas" sebenarnya 3 kebutuhan berbeda, bukan 1 fitur tunggal:

1. **Penempatan Siswa Baru (SPMB/PPDB)** — paste banyak NISN sekaligus ke 1 kelas, dengan validasi visual per-baris real-time
2. **Kenaikan Kelas Massal** — 1 aksi tombol, semua siswa di kelas asal pindah ke kelas tujuan (mapping manual per kelas), KECUALI siswa yang di-paste terpisah sebagai "Tinggal Kelas"
3. **Pindah Kelas Individual** — dari halaman detail siswa, dropdown sederhana (bukan lewat menu Kelas)

Ditambah 1 kebutuhan terkait yang ditemukan saat diskusi: **Tandai Siswa Keluar** (Lulus/Mengundurkan Diri/Lainnya) — field skemanya sudah ada (T063: `status`, `alasanNonaktif`, `tahunLulus`) tapi belum ada UI aksinya sama sekali.

---

## 1️⃣ Penempatan Siswa Baru (Menu Kelas → Tombol Aksi per Kelas)

### Lokasi
- Menu `Kelas` (existing, `apps/web/.../kelas`) — tabel kelas **tambah kolom "Jumlah Siswa"** (COUNT dari `students.kelasId`)
- Tiap baris kelas punya tombol aksi **"Plot Siswa"** → buka halaman/panel baru (2 kolom: kiri form input, kanan hasil)

### Alur (Kolom Kiri — Input)
1. Textarea besar untuk **paste banyak NISN sekaligus** (1 NISN per baris, atau dipisah baris baru/koma — format fleksibel, di-parse sisi client)
2. Begitu di-paste, sistem **langsung validasi tiap baris secara real-time** (tanpa perlu klik tombol cek terpisah) — panggil API cek eksistensi NISN + status kelas saat ini
3. Preview hasil validasi tampil **di bawah textarea**, per baris NISN diberi warna:
   - **Hijau** — NISN valid, siswa ditemukan, **belum** punya kelas (kelasId null) → siap diinput
   - **Merah, "NISN tidak ditemukan"** — NISN tidak ada di database sama sekali
   - **Merah, "Sudah ada di kelas {nama kelas lain}"** — NISN valid tapi siswa itu **sudah** punya kelas lain (duplikat/pindah tanpa sengaja)
4. Tombol **"Input"** — HANYA memproses baris yang berstatus hijau (valid), baris merah **diabaikan otomatis** (tidak mem-block seluruh proses, sesuai prinsip "input yang valid saja, laporkan yang gagal")
5. Setelah proses, kolom kiri (textarea+preview) **tetap ada** untuk lanjut paste batch berikutnya

### Alur (Kolom Kanan — Hasil)
- List siswa yang **berhasil diinput** ke kelas ini (bertambah tiap kali admin klik "Input"), akumulatif dalam 1 sesi halaman — tidak hilang saat admin paste batch baru di kiri
- Tiap entry: Nama, NISN, waktu diinput

## JANGAN (Bagian 1)
- Halaman ini **hanya untuk siswa yang BELUM punya kelas** — bukan untuk pindah kelas siswa existing (itu Bagian 3, mekanisme terpisah)
- JANGAN otomatis pindahkan siswa yang sudah py kelas lain — tandai merah + keterangan, biarkan admin urus manual lewat Bagian 3 kalau memang itu maksudnya

---

## 2️⃣ Kenaikan Kelas Massal (Menu Tersendiri — REVISI 2026-07-22, baca T073 untuk detail lengkap)

> **Desain diganti total** dari versi awal (pilih 1 kelas asal + 1 kelas tujuan per proses) — sekarang **1 halaman untuk SEMUA kelas sekaligus**. Spec detail lengkap ada di task T073 (06-Features/tasks/T073-kenaikan-kelas-massal.md) (task file itu sumber kebenaran terbaru, ringkasan di bawah).

### Alur (Ringkas)
1. **Menu tersendiri** `/kelas/kenaikan-massal` — admin pilih dulu **"Naik ke Tahun Ajaran"** (dari tahun ajaran yang SUDAH ADA, dibuat lewat menu Kalender Pendidikan — bukan dibuat di sini)
2. Halaman menampilkan **tabel SEMUA kelas existing sekaligus** (Kelas Asal, Jumlah Siswa, dropdown Kelas Tujuan per baris, tombol kecil "Atur NISN Tinggal Kelas" per baris)
3. **Kelas dengan `tingkat: XII`** — dropdown Kelas Tujuan punya opsi tambahan **"Lulus"** (kelas X/XI tidak punya opsi ini — perlu kolom `tingkat` terpisah, lihat T075)
4. 1 tombol **"Proses Kenaikan Kelas"** — proses SEMUA baris yang tujuannya sudah dipilih, dalam 1 transaction (semua berhasil atau semua gagal)

### Implikasi Skema Baru
```prisma
// Kelas — T075
enum Tingkat { X XI XII }
model Kelas {
  // ... existing ...
  tingkat Tingkat  // BARU — dipakai untuk logic "opsi Lulus cuma di kelas XII"
}

// Student — T071
model Student {
  // ... existing ...
  tinggalKelasPada DateTime? @map("tinggal_kelas_pada")
}
```
- **BUKAN bagian dari `AlasanNonaktif`** (T063) — siswa yang tinggal kelas TETAP `status: aktif`, ini konsep terpisah total dari nonaktif/keluar
- `tingkat` (T075) TIDAK mengganti `nama` kelas — `nama` tetap lengkap ("X TKJ 1"), `tingkat` murni kolom tambahan untuk logic

## JANGAN (Bagian 2)
- JANGAN generate/tebak kelas tujuan otomatis dari pola nama string — WAJIB dropdown manual dari kelas yang sudah ada
- JANGAN tampilkan/izinkan opsi "Lulus" untuk kelas yang `tingkat` bukan XII — validasi DI BACKEND, bukan cuma UI
- JANGAN proses tiap kelas sebagai request terpisah — 1 request, 1 transaction untuk semua kelas sekaligus

---

## 3️⃣ Pindah Kelas Individual (Halaman Detail Siswa)

### Alur
- Di `apps/web/.../siswa/[id]/siswa-detail-view.tsx` (existing, saat ini TIDAK ada field kelas yang bisa diedit) — tambah **dropdown Kelas** yang bisa diubah + tombol simpan
- Submit → `PATCH /students/:id` dengan `kelasId` baru (endpoint SUDAH mendukung ini, `UpdateStudentDto` adalah `PartialType(CreateStudentDto)` — tidak perlu perubahan backend, murni UI yang hilang)

## JANGAN (Bagian 3)
- JANGAN buat endpoint API baru untuk ini — `PATCH /students/:id` sudah cukup, cukup tambah UI yang memanggilnya

---

## 4️⃣ Tandai Siswa Keluar (Lulus/Mengundurkan Diri/Lainnya)

> Ditemukan sebagai celah terkait saat diskusi — field skema (`status`, `alasanNonaktif`, `tahunLulus` dari T063) sudah ada, tapi **tidak ada aksi UI untuk mengubahnya** dari halaman manapun.

### Alur
- Di halaman detail siswa (sama tempatnya dengan Bagian 3) — tombol **"Tandai Keluar"** → dialog kecil: pilih Alasan (Lulus/Mengundurkan Diri/Lainnya), kalau Lulus → field Tahun Lulus muncul (logic show/hide sama seperti yang sudah dirancang di T066)
- Submit → `PATCH /students/:id` dengan `status: nonaktif`, `alasanNonaktif`, `tahunLulus` (kalau relevan)
- **Reversible** — kalau admin salah tandai, bisa diubah balik `status: aktif` dari dropdown Status yang sudah ada di halaman ini (existing, sudah bisa difilter tapi cek apakah sudah bisa diedit — task perlu verifikasi ini saat implementasi)

---

## ❓ Belum Diputuskan
- [ ] Histori tinggal kelas 2x (multi-tahun) — field tunggal `tinggalKelasPada` cukup untuk v1 (cuma tahu "terakhir kali"), kalau butuh riwayat lengkap perlu tabel terpisah — ditunda sampai ada kebutuhan nyata
- [ ] Apakah kolom "Jumlah Siswa" di tabel Kelas perlu breakdown (misal per jenis kelamin) atau cukup angka total — v1: cukup angka total

## 🔗 Lihat Juga
- T063 (06-Features/tasks/T063-schema-data-wali-murid.md) — field `alasanNonaktif`/`tahunLulus` yang sudah ada, dipakai Bagian 4
- T066 (06-Features/tasks/T066-form-siswa-sheet-lengkap.md) — form Siswa Sheet+Tabs, tempat status/alasan nonaktif awalnya dirancang (kini sebagian pindah ke halaman detail sesuai Bagian 3/4 di sini)
- 04-Database-Schema (04-Database-Schema.md)

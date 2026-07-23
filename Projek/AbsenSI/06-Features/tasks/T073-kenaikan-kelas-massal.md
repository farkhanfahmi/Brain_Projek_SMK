# T073 — UI+API: Kenaikan Kelas Massal (Menu Tersendiri, Semua Kelas Sekaligus)

## Depends on
T071 (kolom `tinggalKelasPada`), T072 (reuse validasi NISN), T075 (kolom `tingkat` — WAJIB untuk logic dropdown "Lulus")

> **Revisi total 2026-07-22.** Versi sebelumnya (pilih 1 kelas asal + 1 kelas tujuan per proses) DIGANTI TOTAL dengan desain ini — 1 halaman, semua kelas ditampilkan sekaligus, dropdown tujuan per baris. Kalau ada versi task lama yang masih menyebut "pilih kelas asal & tujuan satu-satu", itu SUDAH USANG.

## Objective
Buat menu tersendiri "Kenaikan Kelas" — admin pilih dulu tahun ajaran tujuan, lalu sistem tampilkan SEMUA kelas existing dalam 1 tabel, tiap baris punya dropdown kelas tujuan sendiri (kelas XII otomatis dapat opsi tambahan "Lulus"), submit 1 tombol untuk proses semua sekaligus.

## Context
- **App:** `apps/api` + `apps/web`
- **Route:** `/kelas/kenaikan-massal` (menu tersendiri, terpisah dari tabel Kelas biasa — link dari menu Kelas ke halaman ini)
- **Ref:** `Projek/AbsenSI/06-Features/plotting-siswa-kelas.md` (bagian ini akan direvisi menyusul, task file ini yang jadi sumber kebenaran terbaru dulu)

## Spec Detail

### Langkah 1 — Pilih Tahun Ajaran Tujuan
- Dropdown di atas halaman: **"Naik ke Tahun Ajaran"** — list dari `GET /academic-years` (endpoint existing Fase 1)
- **Tahun ajaran HARUS SUDAH ADA** (dibuat manual duluan lewat menu Kalender Pendidikan existing) — halaman ini TIDAK bisa membuat tahun ajaran baru, murni pilih dari yang sudah ada
- Field ini WAJIB dipilih sebelum tabel kelas di bawah bisa diproses (disabled sampai terisi) — nilainya dipakai sebagai **label/konteks proses** (tercatat di `activity_log`), bukan mengubah logic pemindahan itu sendiri

### Langkah 2 — Tabel Semua Kelas
Tampilkan SEMUA kelas existing (dari `GET /kelas`, sudah ada `jumlahSiswa` dari T071), 1 baris per kelas:

| Kelas Asal | Jumlah Siswa | Kelas Tujuan | NISN Tinggal Kelas |
|---|---|---|---|
| X TKJ 1 | 32 | [dropdown ▾] | [input kecil/link "Atur"] |
| XI RPL 2 | 30 | [dropdown ▾] | [input kecil/link "Atur"] |
| XII TKJ 1 | 28 | **[dropdown ▾ dengan opsi "Lulus" di paling atas]** | [input kecil/link "Atur"] |

- **Kolom "Kelas Tujuan"**: dropdown per baris, isinya SEMUA kelas existing LAIN (dari `GET /kelas`, filter keluar kelas itu sendiri) — **HANYA dari kelas yang sudah ada**, tidak ada opsi "buat kelas baru" di sini (sesuai keputusan: kelas tujuan harus dibuat manual duluan di menu Kelas kalau belum ada)
- **Baris dengan `tingkat: XII`** — dropdown kelas tujuan punya **opsi tambahan "Lulus"** di paling atas daftar (terpisah visual dari daftar kelas, misal dengan divider) — pilih ini berarti SEMUA siswa aktif di kelas itu ditandai keluar (`status: nonaktif`, `alasanNonaktif: lulus`, `tahunLulus` = tahun dari Tahun Ajaran Tujuan yang dipilih di Langkah 1, kalau formatnya bisa diparse jadi angka tahun — atau field terpisah kecil untuk isi tahun lulus eksplisit kalau parsing tidak reliable)
- **Baris kelas X/XI** — dropdown tidak punya opsi "Lulus" (logic filter berdasarkan `tingkat`, BUKAN cuma untuk kelas XII secara visual saja — validasi ini juga ditegakkan backend, bukan cuma disembunyikan di UI)
- **Kolom "NISN Tinggal Kelas"** — link/tombol kecil per baris untuk buka popover/modal kecil berisi textarea paste-NISN (sama pola validasi visual seperti T072: hijau valid, merah tidak ditemukan/bukan siswa kelas ini) — hasil tersimpan sebagai state per-baris sebelum submit final

### Langkah 3 — Submit
- 1 tombol **"Proses Kenaikan Kelas"** di bawah tabel — memproses SEMUA baris yang dropdown Kelas Tujuannya sudah dipilih (baris yang dropdownnya masih kosong DISKIP, tidak memblokir baris lain — beri warning ringan "N kelas belum dipilih tujuannya, akan dilewati")
- **Dialog konfirmasi** sebelum eksekusi final — ringkasan: "X kelas akan diproses, total Y siswa naik, Z siswa lulus, W siswa tinggal kelas"

### API: `POST /kelas/kenaikan-massal`
```json
{
  "tahunAjaranTujuanId": 3,
  "proses": [
    { "kelasAsalId": 5, "kelasTujuanId": 12, "nisnTinggalKelas": ["0044360916"] },
    { "kelasAsalId": 6, "kelasTujuanId": null, "lulus": true, "nisnTinggalKelas": [] }
  ]
}
```
- Item dengan `lulus: true` → semua siswa aktif di `kelasAsalId` (kecuali yang di `nisnTinggalKelas`, kalau ada — meski untuk kelas XII "tinggal kelas" jarang terjadi, tetap didukung untuk fleksibilitas) di-set `status: nonaktif`, `alasanNonaktif: lulus`, `tahunLulus` dari tahun ajaran tujuan
- Item dengan `kelasTujuanId` terisi → sama seperti T073 versi lama (siswa pindah ke kelas tujuan, kecuali yang di `nisnTinggalKelas` yang tetap + `tinggalKelasPada` terisi)
- **Validasi backend**: kalau `lulus: true` dikirim untuk `kelasAsalId` yang `tingkat` BUKAN `XII` → tolak 400 (jangan percaya FE, backend cek ulang)
- Proses SEMUA item dalam 1 **database transaction** — kalau 1 kelas gagal (misal race condition), seluruh batch di-rollback, JANGAN proses sebagian lalu gagal sebagian (supaya tidak ada state kelas yang setengah-jalan naik setengah tidak)
- Response: ringkasan per kelas (`dipindah`, `diluluskan`, `tinggalKelas`, error kalau ada)
- **WAJIB `@LogActivity`** — `action: kelas.kenaikan_massal`, snapshot ringkasan seluruh proses + `tahunAjaranTujuanId`

## JANGAN
- ❌ JANGAN tampilkan opsi "Lulus" di dropdown kelas yang `tingkat` bukan XII — validasi DI BACKEND JUGA, bukan cuma disembunyikan di FE
- ❌ JANGAN proses per-kelas sebagai request terpisah — 1 request dengan semua kelas dalam 1 transaction, supaya konsisten (semua berhasil atau semua gagal)
- ❌ JANGAN buat halaman ini bisa membuat tahun ajaran baru — hanya pilih dari yang sudah ada
- ❌ JANGAN buat opsi "buat kelas baru" di dropdown Kelas Tujuan — hanya dari kelas existing
- ❌ JANGAN lupa `@LogActivity` — operasi paling berdampak luas di seluruh sistem Siswa/Kelas, WAJIB tercatat

## Files
- **Modifikasi:** `apps/api/src/core/kelas/kelas.controller.ts` — tambah `POST /kelas/kenaikan-massal`
- **Buat:** `apps/web/app/(admin)/kelas/kenaikan-massal/page.tsx`
- **Buat:** komponen dropdown-per-baris + popover "Tinggal Kelas" (reuse pola validasi NISN dari T072 kalau memungkinkan)

## Acceptance Criteria
- [ ] Halaman menampilkan semua kelas dalam 1 tabel, dropdown tujuan per baris berisi kelas lain (bukan dirinya sendiri)
- [ ] Baris kelas `tingkat: XII` punya opsi "Lulus" di dropdown; baris `X`/`XI` tidak punya opsi itu
- [ ] Submit dengan campuran: beberapa kelas naik normal, 1 kelas pilih "Lulus", beberapa siswa masuk daftar Tinggal Kelas → semua diproses benar dalam 1 klik, terverifikasi via MySQL MCP
- [ ] Kirim request manual (bypass FE) dengan `lulus: true` untuk kelas `tingkat: X` → 400 ditolak backend
- [ ] 1 kelas gagal diproses (simulasikan error) → SEMUA kelas dalam batch itu rollback, tidak ada yang ke-apply sebagian
- [ ] `action: kelas.kenaikan_massal` tercatat di `activity_log` dengan ringkasan lengkap

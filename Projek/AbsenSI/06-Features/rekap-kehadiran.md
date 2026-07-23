---
tags: [absensi, feature, rekap, laporan, fase-1, planning]
status: planning
updated: 2026-07-03
---

# Feature — Rekap Kehadiran

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> Fitur rekap memberi akses ke ringkasan data kehadiran dalam rentang waktu tertentu, dengan berbagai filter. Didesain bertahap per fase — Fase 1 melayani **Admin Pusat** dulu, Fase 2 menambahkan akses untuk **Wali Kelas** dan **Guru** (riwayat diri sendiri).

---

## 📋 Status

| Item | Detail |
|---|---|
| Phase | Fase 1 (admin), Fase 2 (wali kelas + guru) |
| Status | 🟡 Planning — desain disepakati, belum jadi task |
| Modul terkait | Attendance, Core (Siswa, Kelas, Jurusan) |
| Prasyarat | Data `attendance_records` sudah terisi dari Fase 1 (absensi gerbang berjalan) |

---

## 👤 Aktor per Fase

| Fase | Aktor | Scope Data |
|---|---|---|
| Fase 1 | `super_admin` | Semua siswa, semua kelas, semua jurusan |
| Fase 2 | `kepsek` | Semua siswa (read-only, sama dengan admin) |
| Fase 2 | Wali Kelas (`guru` dengan `kelas_id_wali` terisi — bukan role terpisah, lihat [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]] bagian "Wali Kelas") | Hanya kelas yang dia ampu |
| Fase 2 | `guru` (riwayat pribadi) | Hanya kehadiran diri sendiri |

---

## ⚙️ Fungsi Rekap — Fase 1 (Admin Pusat)

### Filter yang Tersedia

| Filter | Tipe | Keterangan |
|---|---|---|
| **Rentang tanggal** | Date picker (dari – sampai) | Wajib diisi; default: bulan berjalan |
| **Kelas** | Dropdown (opsional) | Pilih 1 kelas spesifik, atau "Semua Kelas" |
| **Jurusan** | Dropdown (opsional) | Pilih 1 jurusan, atau "Semua Jurusan" |

Filter boleh dikombinasikan bebas: misal "Jurusan RPL, bulan April" atau "Kelas XI-A, minggu ini."

### Level Detail Output

Tampilan berupa **tabel ringkasan per siswa** dalam rentang/filter yang dipilih:

| Kolom | Keterangan |
|---|---|
| Nama Siswa | |
| Kelas | |
| Jurusan | |
| Hadir | Jumlah hari status `hadir` + `terlambat` (keduanya dianggap "masuk") |
| Terlambat | Subset dari Hadir — hari masuk tapi terlambat |
| Izin | Jumlah hari `permits.alasan_kategori = izin` |
| Sakit | Jumlah hari `permits.alasan_kategori = sakit` |
| Alfa | Jumlah hari tidak ada tap dan tidak ada `permits` (tidak ada keterangan) |
| Total Hari Aktif | Total hari sekolah dalam rentang yang dipilih (pembagi untuk persentase) |

> **Catatan:** "Hadir" dan "Terlambat" dihitung terpisah supaya laporan bisa membedakan kedisiplinan. Tapi untuk keperluan kehadiran formal (rekapitulasi raport), `hadir + terlambat` dihitung sebagai "masuk."

### Definisi "Alfa" dalam Konteks Sistem

Alfa = tidak ada `attendance_records` hari itu **dan** tidak ada `permits` hari itu, **pada hari wajib** (Senin–Jumat, bukan libur di kalender).

**Tiga kategori hari — penting untuk dipahami saat rekap:**

| Hari | Absen = Alfa? | Keterangan |
|---|---|---|
| Senin–Jumat (bukan libur) | ✅ Ya | Hari wajib — tidak hadir tanpa keterangan = alfa |
| Sabtu | ❌ Tidak | **Hari opsional** — hadir dicatat, tidak hadir tidak dihitung alfa |
| Minggu & hari libur (`school_holidays`) | ❌ Tidak | Libur — tidak ada ekspektasi kehadiran |

> **Implikasi query:** Alfa dihitung dari hari-hari Senin–Jumat dalam `academic_years` aktif yang tidak masuk `school_holidays`, lalu dicek apakah siswa punya record di `attendance_records` atau `permits` untuk hari itu. Sabtu tidak masuk perhitungan alfa sama sekali — tapi kehadiran Sabtu tetap tampil di rekap jika admin memilih filter yang mencakup hari Sabtu (sebagai baris terpisah atau catatan tambahan, diputuskan saat breakdown task).

> **Alfa bukan status tersimpan** — ini adalah kondisi **absennya data** yang dihitung saat query. Tidak ada kolom `status: alfa` di database.

### Interaksi UI

1. Admin buka halaman Rekap → pilih filter → klik "Tampilkan"
2. Tabel hasil muncul di bawah filter
3. **Fase 1: tidak ada export** — hanya tampilan di layar
4. **Fase 2 (opsional):** tambahkan tombol "Export Excel" menggunakan library `exceljs` (effort tambahan, tidak masuk scope Fase 1)

---

## 🔮 Rekap Fase 2 — Wali Kelas & Guru

> Belum didesain detail — hanya outline awal untuk panduan arsitektur.

### Wali Kelas
- Filter yang tersedia: sama seperti admin, tapi scope dibatasi ke kelas yang dia ampu saja (ditegakkan di API, bukan hanya UI)
- Output: sama seperti rekap admin tapi hanya untuk siswa kelasnya

### Guru (Riwayat Kehadiran Pribadi)
- Tidak ada filter kelas/jurusan — hanya rentang tanggal
- Output: tabel riwayat kehadiran guru sendiri (tanggal, jam masuk, jam pulang, status hadir/terlambat)
- Baca dari `attendance_records` dengan `teacher_id` = ID guru yang login

---

## 🗄️ Query & Performa

Index komposit yang relevan untuk query rekap (akan didetailkan saat breakdown task modul Rekap):
- `attendance_records (tanggal, student_id)` — filter rentang tanggal per siswa
- `attendance_records (tanggal, status)` — filter status dalam rentang
- Join ke `students → kelas → jurusan` untuk filter kelas/jurusan
- `permits (tanggal, student_id, alasan_kategori)` — agregasi izin/sakit

Volume estimasi: rekap 1 bulan untuk 2.500 siswa = query ±50.000–75.000 baris (asumsi 20–30 hari sekolah aktif). Dengan index komposit yang tepat di MySQL, ini jauh di bawah batas yang butuh optimasi khusus.

---

## ❓ Open Questions

- [x] **Apakah perlu tabel kalender eksplisit?** → Resolved: **Ya**. Admin input kalender pendidikan (tahun ajaran + daftar libur) via dashboard. Lihat [[Projek/AbsenSI/06-Features/kalender-pendidikan|kalender-pendidikan.md]] untuk spek fitur dan [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] untuk tabel `academic_years` + `school_holidays`.
- [x] **Filter berdasarkan wali kelas?** → Fase 2 — masuk bersamaan dengan akses wali kelas.
- [x] **Export Excel/PDF?** → Fase 2 — tidak masuk scope Fase 1.
- [x] **Rekap guru (kehadiran guru)?** → Fase 2 — admin hanya rekap siswa di Fase 1.

**Status spec:** ✅ Final — siap dipecah jadi task development.

---

## 🔗 Lihat Juga
- [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] — tabel `attendance_records`, `permits`, `students`
- [[Projek/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang (Fase 1)]] — sumber data utama rekap
- [[Projek/AbsenSI/06-Features/dashboard-piket|Dashboard Piket]] — sumber data `permits` (izin/sakit)
- [[Projek/AbsenSI/03-User-Roles|03-User-Roles]] — scope akses per role
- [[Projek/AbsenSI/11-Decisions|ADR-019, ADR-020]]

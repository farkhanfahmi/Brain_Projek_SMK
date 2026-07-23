---
tags: [absensi, feature, kalender, admin, fase-1, planning]
status: planning
updated: 2026-07-03
---

# Feature — Kalender Pendidikan (Admin Dashboard)

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> Fitur pengelolaan kalender akademik sekolah. Dibutuhkan sebagai **fondasi perhitungan alfa** di modul Rekap — tanpa kalender yang terdefinisi, sistem tidak bisa tahu hari mana yang seharusnya ada kehadiran dan hari mana yang libur. Masuk scope **Fase 1** karena rekap tidak bisa akurat tanpa ini.

---

## 📋 Status

| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟡 Planning |
| Modul terkait | Core (Admin), Rekap |
| Prasyarat | Harus diisi **sebelum** data absensi mulai dikumpulkan — kalau kalender belum diisi, perhitungan alfa di rekap tidak bisa diandalkan |

---

## 🎯 Tujuan

1. Mendefinisikan **tahun ajaran aktif** (kapan mulai, kapan selesai)
2. Admin bisa input **libur blok** (libur semester, libur nasional panjang, dsb) sekaligus via range tanggal
3. Admin bisa **tandai hari tertentu sebagai libur mendadak** (misalnya sekolah tiba-tiba libur karena kondisi tertentu) — satu klik, satu hari
4. Memberi **tampilan kalender visual** supaya admin bisa verifikasi mana yang aktif dan mana yang libur sebelum data absensi mulai direkap

---

## 👤 Aktor

| Aktor | Aksi |
|---|---|
| `super_admin` | Full CRUD — buat tahun ajaran, input libur, tandai libur mendadak, hapus/edit entri libur |
| Role lain | Read-only — bisa lihat kalender aktif tapi tidak bisa edit |

---

## ⚙️ Fungsi Utama

### 1. Manajemen Tahun Ajaran

- Admin buat **tahun ajaran** baru dengan mengisi: nama (misal "2025/2026"), tanggal mulai, tanggal selesai.
- Hanya boleh ada **1 tahun ajaran aktif** pada satu waktu (`is_active: true`). Saat tahun ajaran baru diaktifkan, yang lama otomatis tidak aktif.
- Data absensi dan kalender tahun-tahun sebelumnya tetap tersimpan dan bisa diakses untuk rekap historis — tidak dihapus saat tahun ajaran baru dibuat.

### 1b. Semester (Final — Fase 2, 2026-07-22)

> Ditambahkan sebagai perluasan `academic_years` untuk mendukung Dashboard Guru Jurnal ([[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]]) — jadwal mengajar (`schedules`) di-scope PER SEMESTER, bukan cuma per tahun ajaran, karena jadwal riil berubah total antar semester ganjil/genap.

- Setiap `academic_years` punya **2 semester**: Ganjil dan Genap
- **Tanggal mulai/selesai tiap semester diinput MANUAL oleh admin** (sama seperti tahun ajaran) — bukan otomatis dibagi tengah, karena rentang semester riil tidak selalu simetris (libur panjang bisa menggeser)
- **1 semester aktif per waktu** (`is_active: true`, pola identik `academic_years.is_active`) — dipakai job harian (generate `teaching_sessions`) untuk resolve jadwal mana yang berlaku hari ini. Admin **switch manual** saat pergantian semester (bukan otomatis dari tanggal)
- Saat semester baru dibuat, admin_jurnal punya opsi **"Salin jadwal dari semester sebelumnya"** — duplikasi seluruh `schedules` (`type: jam_mengajar`) ke `semester_id` baru, admin tinggal edit yang berubah (lihat [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]] untuk detail alur ini di sisi Jadwal Mengajar)
- **Skema baru:** tabel `semesters` — `id`, `academic_year_id` (FK), `nama` (enum: `ganjil`/`genap`), `tanggal_mulai`, `tanggal_selesai`, `is_active` (boolean, hanya 1 true sekaligus lintas SEMUA tahun ajaran — bukan cuma dalam 1 tahun ajaran), `created_by` (FK users), `created_at`
- **`schedules`** (tipe `jam_mengajar`) tambah kolom **`semester_id`** (FK ke `semesters`, wajib diisi untuk `type: jam_mengajar`) — jadwal gerbang (`type: jam_sekolah`) dan jadwal khusus (`type: jadwal_khusus`) TIDAK perlu `semester_id` (tetap berlaku sepanjang tahun ajaran seperti sebelumnya, tidak berubah)

### 2. Input Libur Blok (Range)

- Admin isi form: **tanggal mulai**, **tanggal selesai**, **jenis libur**, **keterangan** (nama libur).
- Sistem menyimpan range ini — saat menghitung hari aktif, semua tanggal dalam range tersebut dianggap libur.
- Contoh use case: "Libur Semester Ganjil: 16 Desember 2025 – 2 Januari 2026", "Libur Lebaran: 28 Maret – 7 April 2026"
- **Jenis libur yang tersedia:**
  - `libur_nasional` — hari libur nasional (Merah di kalender)
  - `libur_semester` — libur antar semester resmi dari sekolah
  - `libur_sekolah` — libur sekolah lain yang sudah terjadwal (misal persiapan ujian, porseni)
  - `libur_mendadak` — libur tidak terencana, ditetapkan mendadak

### 3. Tandai Libur Mendadak (Single Day — Quick Action)

- Dari tampilan kalender, admin klik tanggal tertentu → muncul popup kecil → isi keterangan singkat → klik **"Tandai Libur Mendadak"**.
- Atau dari form terpisah: isi tanggal (1 hari, bukan range) + keterangan → submit.
- Ini jalur cepat tanpa harus isi form lengkap — untuk kasus seperti "besok libur karena guru piket tidak masuk semua" atau "sekolah libur karena banjir."
- Bisa dihapus kembali oleh admin kalau salah input (tidak immutable — ini data konfigurasi, bukan log).

### 4. Tampilan Kalender Visual

- Halaman kalender bulanan standar (tampilan grid minggu) yang menampilkan:
  - **Hari aktif sekolah** → latar putih/normal
  - **Hari libur** → diberi warna + label singkat jenis libur
  - **Sabtu & Minggu** → abu-abu otomatis (tidak perlu diinput manual — sistem sudah tahu akhir pekan)
  - **Hari ini** → ditandai
- Admin bisa navigasi antar bulan dalam tahun ajaran aktif.
- Klik tanggal libur → bisa edit keterangan atau hapus entri libur tersebut.
- Klik tanggal aktif yang ternyata ingin dijadikan libur → shortcut ke form libur mendadak.

---

## 🗄️ Skema Database

Lihat detail di [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] — bagian Entitas Kalender. Ringkasan:

**`academic_years`** — mendefinisikan tahun ajaran
- `id`, `nama`, `tanggal_mulai`, `tanggal_selesai`, `is_active`, `created_by` (FK users), `created_at`

**`school_holidays`** — mendaftarkan range (atau hari tunggal) yang bukan hari aktif sekolah
- `id`, `academic_year_id` (FK, nullable), `tanggal_mulai`, `tanggal_selesai`, `jenis` (`libur_nasional` / `libur_semester` / `libur_sekolah` / `libur_mendadak`), `keterangan`, `created_by` (FK users), `created_at`, `updated_by` (FK users, nullable), `updated_at`
- Hari tunggal = `tanggal_mulai == tanggal_selesai`

**Logic menentukan "hari aktif sekolah"** untuk query rekap alfa:
```
hari aktif = tanggal berada dalam range academic_years aktif
             AND tanggal bukan Sabtu/Minggu
             AND tanggal TIDAK masuk dalam range manapun di school_holidays
```

---

## 🔗 Hubungan dengan Modul Lain

- **Rekap Kehadiran** — query alfa menggunakan kalender ini sebagai daftar hari yang "seharusnya hadir". Tanpa kalender, alfa tidak bisa dihitung akurat.
- **Dashboard Piket** — opsional: bisa ditampilkan info "hari ini libur" di header dashboard jika admin menandai hari ini sebagai libur (supaya piket tidak bingung kenapa tidak ada siswa tap).
- **Absensi Gerbang** — tap RFID tetap diterima meskipun hari libur (kiosk tidak mati saat libur). Record tetap masuk ke `attendance_records`. Untuk rekap, hari libur tersebut dikecualikan dari perhitungan — tap yang masuk saat hari libur tidak dihitung sebagai "hari aktif."

---

## ✅ Open Questions yang Sudah Resolved

- [x] **Apakah Sabtu termasuk hari aktif wajib?** → **Sabtu adalah hari opsional.** Pelajaran efektif hanya Senin–Jumat. Sabtu tetap aktif (siswa/guru bisa tap dan kehadirannya tercatat), tapi ketidakhadiran di Sabtu **tidak dihitung alfa**. Aturan ini dikode sebagai konstanta di service layer, tidak perlu field konfigurasi di database.

## ✅ Open Questions — Semua Resolved (2026-07-03)
- [x] **Libur nasional hardcoded atau manual?** → **Manual oleh admin** setiap awal tahun ajaran. Fleksibel untuk cuti bersama yang sering berubah. Tidak perlu integrasi API kalender.
- [x] **Notifikasi ke piket saat libur mendadak?** → Tidak di Fase 1. Banner informasi di Dashboard Piket masuk backlog Fase 2 opsional.

**Status spec:** ✅ Final — siap dipecah jadi task development.

---

## 🔗 Lihat Juga
- [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] — tabel `academic_years`, `school_holidays`
- [[Projek/AbsenSI/06-Features/rekap-kehadiran|Rekap Kehadiran]] — konsumen utama kalender ini
- [[Projek/AbsenSI/06-Features/dashboard-piket|Dashboard Piket]] — konteks hari libur mendadak

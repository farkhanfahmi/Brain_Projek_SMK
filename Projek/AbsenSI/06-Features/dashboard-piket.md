---
tags: [absensi, feature, piket, kampus, fase-1b, planning]
status: planning
updated: 2026-06-26
---

# Feature — Dashboard Piket (Fase 1b)

← Index (00-INDEX AbsenSI.md)

> **Fase 1b** — menyusul setelah inti Fase 1 (Absensi Gerbang, Manajemen Kartu, Import Data Master) stabil. Tidak dikerjakan bersamaan dengan inti Fase 1 untuk menghindari overload tim yang sedang belajar stack baru. Lihat ADR-015 s/d ADR-018 (11-Decisions.md) untuk keputusan arsitektur yang melandasi fitur ini.

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1b |
| Status | 🟡 Planning — desain disepakati, belum jadi task |
| Modul terkait | Core (Kampus, Users), Attendance, Permits (baru) |
| Prasyarat | Inti Fase 1 (gerbang + kartu + import) sudah jalan & stabil |

---

## 🏫 Konteks: 2 Kampus

Sekolah punya **2 kampus fisik** (Kampus 1, Kampus 2), masing-masing dengan **guru piket sendiri** yang punya komputer/dashboard sendiri. Setiap kelas terikat ke 1 kampus (`kelas.kampus_id`); siswa mewarisi kampus lewat kelasnya.

**Penting:** Tap RFID di gerbang **tetap diizinkan lintas-kampus** (siswa Kampus 1 boleh tap di kiosk Kampus 2, karena lokasi ruang praktek tidak selalu sama dengan kampus asal siswa). Dashboard Piket men-scope data berdasarkan **kampus asal siswa** (lewat kelas), **bukan** lokasi tap. Lihat ADR-015.

---

## 👤 Aktor

| Aktor | Aksi |
|---|---|
| Guru Piket (per kampus) | Lihat dashboard realtime siswa kampusnya, input status izin/sakit, kelola perizinan keluar + cetak surat, tinjau & kunci/buka-kunci siswa yang tidak kembali dari izin |
| Siswa | Lapor lisan ke piket (tidak masuk atau mau izin keluar) — **tidak ada interaksi sistem langsung** dari sisi siswa |
| Admin Pusat (`super_admin`) | Tetap punya kewenangan koreksi data absensi seperti biasa (lihat 03-User-Roles (03-User-Roles.md)) — relasinya ke kewenangan piket harus dipertegas saat breakdown task (siapa yang menang kalau ada konflik koreksi) |

---

## ⚙️ Fungsi Utama

### 1. Dashboard Realtime Per Kampus
- Daftar **semua siswa di kampus tersebut**, bukan agregat angka seperti Dashboard TV — tiap baris siswa menampilkan status jam hadir (sudah tap/belum, jam tap, terlambat/tidak).
- Update realtime via Socket.IO, channel di-scope per kampus (`attendance:kampus:{id}`), pola sama dengan Dashboard TV tapi granularitasnya per-siswa bukan agregat.

### 2. Input Status Tidak Masuk (Quick Action)
- Tiap baris siswa di daftar punya tombol cepat **[Izin]** dan **[Sakit]**.
- Klik tombol → modal kecil: pilih kategori (sudah terisi dari tombol yang diklik), isi keterangan singkat → submit.
- Submit membuat record `permits` (`jenis: tidak_masuk`) dan otomatis update `attendance_records` hari itu jadi status `izin`/`sakit`.
- **Tidak ada cetak surat** untuk jalur ini — ini cuma pencatatan status, beda dari izin keluar.

### 3. Perizinan Keluar Sekolah (Menu Terpisah)

Ada **dua sub-alur** yang berbeda, keduanya dikelola dari menu Perizinan Keluar:

**Sub-alur A — Izin Keluar Sementara (siswa tidak tap):**
- Siswa lapor lisan ke piket bahwa ingin keluar (urusan keluarga, berobat, dsb).
- Piket isi form: pilih siswa, alasan (`sakit`/`izin`), keterangan, jam keluar, jam kembali yang dijanjikan (opsional).
- Submit → `permits(jenis: keluar)` tersimpan, `status_kembali: belum`, **kode verifikasi** unik digenerate.
- Sistem konstruksi URL ke `print.php` → buka di tab baru → piket print surat izin manual.
- **Siswa langsung pergi tanpa tap** — tidak perlu scan kartu di gerbang.
- Saat kembali: siswa lapor lisan ke piket → piket klik **"Sudah Kembali"** → `status_kembali: sudah`. Siswa **tidak perlu tap** saat kembali juga.
- Kalau siswa tidak kembali sampai akhir jam sekolah: piket klik **"Dianggap Pulang"** → `status_kembali: pulang`, `attendance_records.pulang_via: piket_izin`, `waktu_pulang` diisi `jam_keluar` dari permits.

**Sub-alur B — Izin Pulang Awal (siswa tap dulu):**
- Siswa lapor ke piket bahwa ingin pulang lebih awal (izin resmi).
- Piket setujui → **siswa tap kartu di gerbang terlebih dahulu** → sistem catat `waktu_pulang` dari tap.
- Setelah tap, piket klik konfirmasi **"Tandai sebagai Izin Pulang"** di dashboard → `attendance_records.pulang_via` diubah dari `tap` jadi `tap_izin_pulang`, tercatat di `activity_log`.
- **Fallback** — kalau siswa gagal tap (kartu tertinggal, reader error): piket manual input tanpa tap → `pulang_via: piket_izin`, `waktu_pulang` diisi manual oleh piket.

### 4. Monitoring "Belum Kembali"
- Siswa dari Sub-alur A yang punya `jam_kembali_diharapkan` dan belum ditandai kembali → otomatis muncul di daftar **"Belum Kembali"** saat waktu yang dijanjikan sudah lewat — sinyal visual untuk piket, bukan trigger otomatis apa pun.
- Piket tindak lanjut: hubungi orang tua, atau klik "Dianggap Pulang" kalau memang tidak kembali.

### 5. Antrian Klarifikasi "Tidak Tap Pulang"

- Setiap pagi, Dashboard Piket menampilkan daftar **siswa kampus ini yang kemarin tap masuk tapi tidak tap pulang** (`waktu_pulang = null` pada hari kemarin).
- Siswa datang melapor ke piket → piket resolve dengan salah satu aksi:
  - **"Konfirmasi Pulang Normal"** → piket isi jam pulang perkiraan (atau "tidak diketahui"), `waktu_pulang` diisi dengan catatan `pulang_via: piket_izin`, dicatat di `activity_log`.
  - **"Tandai Izin Keluar Tidak Kembali"** → buat `permits(jenis: keluar, status_kembali: pulang)` retroaktif untuk hari kemarin, sistem update `attendance_records` kemarin.
- Kalau siswa tidak melapor dan tidak ada klarifikasi → record tetap dengan `waktu_pulang = null`. Rekap akan menampilkan ini sebagai "masuk tanpa keterangan pulang" — bukan alfa, tapi data tidak lengkap.
- Daftar ini **per kampus** (guru piket Kampus 1 hanya lihat siswa Kampus 1, sesuai ADR-015).

### 6b. Direktori Siswa — Pencarian & Filter (BARU, 2026-07-22, lihat T076)

- Menu terpisah "Direktori Siswa" — search by nama + filter Kelas/Jurusan, tiap baris tampilkan foto+nama+kelas
- **Pengecualian sadar terhadap scope kampus** (beda dari SEMUA fungsi piket lain di atas): direktori ini **LINTAS KAMPUS**, bukan dibatasi `kampus_id` piket yang login — alasan: piket kadang perlu verifikasi identitas siswa dari kampus lain (siswa titipan, dst)
- Klik siswa → buka halaman detail siswa yang SAMA dipakai admin (foto, biodata, riwayat terlambat/izin/sakit/alfa) — **read-only untuk piket**, tombol edit/upload/hapus disembunyikan untuk role ini
- **Untuk masa depan** (bukan scope T076): fitur guru BK/PDS mencatat pelanggaran tata tertib terpisah dari kehadiran — nanti akan muncul juga di halaman detail siswa yang sama, belum dibangun sekarang

### 6. Tinjauan & Lock/Unlock Siswa
- Di akhir hari, sistem menandai siswa dengan izin "akan kembali" yang masih `status_kembali: belum` sebagai **"Perlu Ditinjau"**.
- Piket review daftar ini besok pagi — kalau memang belum terselesaikan (bukan cuma piket lupa klik), piket **secara manual** mengunci siswa tersebut (isi `locked_reason`, sistem catat `locked_by` + `locked_at`).
- Siswa terkunci yang tap di gerbang → **ditolak** dengan pesan jelas di layar kiosk ("Hubungi Guru Piket"), tetap dicatat sebagai log percobaan (pola sama dengan tap kartu inactive, lihat manajemen-kartu.md (06-Features/manajemen-kartu.md)).
- Proses lanjutan (BK) berjalan **offline**, di luar sistem. Setelah selesai, piket buka kunci + isi `unlock_note` ringkas sebagai catatan audit.

---

## 🖨️ Integrasi Printer (lihat ADR-018)

- **Tidak membangun mekanisme print baru** — pakai ulang `print.php` yang sudah terbukti jalan dengan printer thermal Blueprint ECO 58D (USB, driver terinstal sebagai printer Windows biasa).
- Parameter URL: `petugas`, `tgl`, `nama`, `kls`, `alasan`, `ket`, `jamkembali` (pola sama dengan sistem lama/AppSheet) + parameter baru `kode` (kode verifikasi).
- Server `print.php` (`10.10.10.100:8800`) tetap berdiri independen dari server utama AbsenSI — **tidak perlu dikonsolidasi sekarang** (technical debt opsional, bukan blocker).
- Flow: sistem buka URL `print.php` di tab baru → halaman preview tampil → petugas klik tombol print di browser secara manual (tidak ada auto-print untuk V1).

---

## 🔗 Hubungan dengan `attendance_records`

| Sumber | Efek ke `attendance_records` |
|---|---|
| Tap gerbang masuk (jalur normal) | `waktu_masuk` terisi, status `hadir`/`terlambat` |
| Tap gerbang pulang (jalur normal) | `waktu_pulang` terisi, `pulang_via: tap` |
| `permits` jenis `tidak_masuk` | Status hari itu jadi `izin`/`sakit`, tidak ada `waktu_pulang` |
| `permits` jenis `keluar`, siswa kembali (`status_kembali: sudah`) | Tidak ubah data kehadiran — siswa dianggap hadir normal |
| `permits` jenis `keluar`, siswa tidak kembali (`status_kembali: pulang`) | `waktu_pulang` = `jam_keluar`, `pulang_via: piket_izin` |
| Piket konfirmasi "Tandai Izin Pulang" (setelah tap Sub-alur B) | `pulang_via` diubah dari `tap` → `tap_izin_pulang`, dicatat di `activity_log` |
| Piket manual fallback pulang (tap gagal) | `waktu_pulang` diisi manual, `pulang_via: piket_izin`, dicatat di `activity_log` |
| Siswa terkunci coba tap | Tap **ditolak**, tidak membuat/update `attendance_record`, hanya masuk `tap_events` dengan `result: rejected_locked` |

**Aturan kewenangan (ADR-019):** Hanya `guru_piket` yang bisa membuat `permits` dan mengubah status kehadiran. `super_admin` tidak memiliki akses ke endpoint ini.

---

## ✅ Open Questions yang Sudah Resolved

- [x] **Aturan precedence tap vs permits** → Resolved: domain dipisah bersih. Tap hanya mencatat `waktu_masuk`/`waktu_pulang` fisik. Status kehadiran (`izin`/`sakit`/`alfa`) hanya bisa diubah oleh piket via `permits`. Keduanya tidak saling overwrite — tap tidak mengubah status, permits tidak menghapus data waktu tap.
- [x] **Relasi kewenangan Admin Pusat vs Guru Piket** → Resolved (ADR-019): domain non-overlapping. Admin mengelola data teknis, piket mengelola status kehadiran. Tidak ada konflik karena admin tidak punya akses ke endpoint status kehadiran sama sekali.
- [x] **Field `akan_kembali`** → Dihapus. Semua izin keluar diasumsikan "berpotensi kembali." Kalau tidak kembali, piket klik "Dianggap Pulang" → `status_kembali: pulang`.

## ❓ Open Questions yang Masih Terbuka

- [ ] Apakah guru piket butuh laporan/rekap bulanan perizinan (untuk keperluan BK atau rapat orang tua), atau cukup data mentah di `permits` tanpa fitur rekap khusus di Fase 1b? → **Kemungkinan masuk Fase 2 bersama rekap wali kelas.**
- [ ] UI kiosk gerbang untuk siswa terkunci — perlu desain pesan & tampilan spesifik, beda dari pesan UID tidak terdaftar/inactive
- [x] **Modifikasi `print.php` untuk parameter `kode` baru** → Script ada di `C:\ProjekSMK\print.php` di server lokal `10.10.10.100`. Dikerjakan **sebelum modul Permits di-deploy** — siapapun yang mengerjakan modul Permits bertanggung jawab koordinasi edit `print.php` ini. Perubahan minimal: tambah 1 blok `<?= $kode ?>` di template HTML surat izin.

## 🔗 Lihat Juga
- Absensi Gerbang (Fase 1) (06-Features/absensi-gerbang.md)
- Manajemen Kartu RFID (06-Features/manajemen-kartu.md)
- Dashboard TV Realtime (06-Features/dashboard-tv.md)
- 04-Database-Schema (04-Database-Schema.md) — entitas `kampus`, `users`, `permits`
- ADR-015, ADR-016, ADR-017, ADR-018 (11-Decisions.md)

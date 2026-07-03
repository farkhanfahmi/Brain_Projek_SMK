---
tags:
  - project
  - user-flows
created: 2026-06-04
updated: 2026-06-04
---

# User Flows — E-Berita Acara Ujian

## Flow 1: Pengawas — Hari Ujian (Mode QR)

```
[Pengawas tiba di sekolah]
        ↓
[Scan ID Card di barcode reader TV]
→ POST /login-niy { niy, source: "Barcode Reader TV" }
→ PresensiPengawas.waktu_datang tercatat
        ↓
[Buka frontend/ di smartphone]
[Tap "Scan ID Card Pengawas"]
→ Kamera terbuka (QRScanner)
[Scan QR ID Card sendiri]
→ POST /login-niy → dapat Sanctum token
→ Token disimpan di localStorage
→ Halaman "Selamat Datang" tampil
        ↓
[Tap "Lanjut ke Berita Acara"]
→ GET /get-assignment?ujian_id=X&pengawas_id=Y
→ Jadwal + daftar peserta ruangan dimuat otomatis
        ↓
[Selama ujian berlangsung]
[Tap "Mulai Scan Peserta"]
→ Scan QR card peserta satu per satu
→ POST /scan-peserta → PresensiPeserta.waktu_datang tercatat
→ Peserta pindah dari tabel "Belum Hadir" ke "Hadir"
        ↓
[Ujian selesai]
→ Tab "Berita Acara" dibuka
→ Form diisi: waktu berakhir, catatan, nama yang absen
→ TTD digital digambar di SignaturePad
[Submit]
→ POST /submit-report (multipart/form-data)
→ LaporanUjian tersimpan
        ↓
[Pulang]
[Scan ID Card di barcode reader TV]
→ POST /login-niy → PresensiPengawas.waktu_pulang tercatat
```

---

## Flow 2: Panitia — Monitoring Hari Ujian

```
[Panitia tiba di sekolah]
        ↓
[Scan ID Card di barcode reader TV]
→ POST /login-niy → PresensiPengawas (panitia_id) dicatat
        ↓
[Buka frontend/ di smartphone]
[Scan ID Card sendiri via kamera]
→ POST /login-niy → return { type: 'panitia', user }
→ Data disimpan di localStorage['panitia_session']
→ Halaman "Selamat Datang" tampil (badge "Panitia")
[Tap "Mulai Scan Peserta"]
        ↓
[Tab "Scan Peserta"]
→ GET /peserta-all?ujian_id=X&panitia_id=Y
→ Semua peserta lintas ruang dimuat
[Scan QR peserta di area masuk]
→ POST /scan-peserta { panitia_id }
→ Modal "Catatan Ketidaktertiban" muncul otomatis
   [Jika ada pelanggaran → isi & simpan]
   [Jika tidak ada → tutup modal]
        ↓
[Tab "Dashboard"]
→ GET /panitia-dashboard?ujian_id=X
→ Ringkasan per ruangan: hadir/belum/pengawas
[Klik siswa belum hadir]
→ Modal keterangan absen → input Sakit/Izin/Alfa/upload foto
→ POST /keterangan-absen
        ↓
[Tab "Rekap"]
→ GET /rekap-admin?ujian_id=X
→ Tabel lengkap peserta + pengawas + panitia
→ Tombol export ke Excel
```

---

## Flow 3: Admin — Persiapan Sebelum Ujian

```
[Login frontend-admin/ dengan email + password]
→ POST /login → Sanctum token
        ↓
[Buat Tahun Ajaran baru (jika belum ada)]
→ POST /tahun-ajaran { nama: "2024/2025" }
        ↓
[Buat Event Ujian]
→ POST /ujians { nama_ujian, jenis_presensi, tampilkan_di_tv, ... }
        ↓
[Import/tambah Ruangan]
→ POST /ruang/import (Excel)
        ↓
[Import/tambah Pengawas]
→ POST /pengawas/import (Excel)
        ↓
[Import/tambah Panitia]
→ POST /panitia/import (Excel)
        ↓
[Import Peserta Ujian]
→ POST /peserta-ujian/import (CSV)
        ↓
[Buat Jadwal Ujian]
→ POST /jadwal-ujian/import (Excel)
  — atau buat manual per jadwal
  — assign pengawas ke ruangan + mapel + waktu
        ↓
[Verifikasi di dashboard admin]
→ Cek semua jadwal terisi
→ Cek semua peserta punya ruang
→ Aktifkan ujian (is_active = true)
```

---

## Flow 4: TV Monitor — Real-time Display

```
[frontend-tv/ dibuka di browser TV]
        ↓
[useAttendanceData hook]
→ Polling setiap 10 detik:
  - GET /dashboard/attendance-stats
  - GET /dashboard/attendance-by-campus
  - GET /dashboard/attendance-by-class
  - GET /dashboard/attendance-by-kelas
  - GET /dashboard/attendance-students
  - GET /presensi-pengawas-today
        ↓
[BarcodeListener aktif — keydown listener]
→ Setiap karakter dari barcode reader USB = keystroke
→ Setelah Enter → string terkumpul = NIY
→ POST /login-niy { niy, source: "Barcode Reader TV" }
→ Jika pengawas/panitia belum presensi → catat waktu_datang
→ Jika sudah datang tapi belum pulang → catat waktu_pulang
→ Toast notifikasi tampil di TV
        ↓
[4-panel layout update otomatis]
  Panel 1: Daftar peserta belum hadir
  Panel 2: Kehadiran per kampus & ruang
  Panel 3: Kehadiran per kelas (progress bar + walikelas)
  Panel 4: List pengawas & panitia yang sudah absen
```

---

## Flow 5: Pengawas — Mode Manual (jenis_presensi = 'Manual')

```
[Login + pilih ujian dengan jenis_presensi = Manual]
        ↓
[Tab "Scan Peserta" — tampil beda dari mode QR]
→ Tidak ada tombol kamera
→ Tampil tabel daftar siswa dengan dropdown:
  [Belum | Hadir | Sakit | Izin | Alfa | Lainya]
[Klik "Hadir Semua" → semua jadi Hadir]
  atau set manual per siswa
[Klik "Simpan Daftar Hadir"]
→ POST /manual-presensi-bulk
→ PresensiPeserta/KeteranganKetidakhadiran diupdate dalam transaksi
```

---

## Flow 6: Peserta — Scan Pulang

```
[Peserta selesai ujian, keluar ruangan]
[Scan QR card di barcode reader atau kamera pengawas]
→ POST /scan-peserta { kode_peserta }
→ Sistem cek: PresensiPeserta sudah ada dengan waktu_datang?
  → Ya: isi waktu_pulang
  → Tidak: isi waktu_datang (baru datang)
```

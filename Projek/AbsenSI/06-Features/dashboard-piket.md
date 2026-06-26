---
tags: [absensi, feature, piket, kampus, fase-1b, planning]
status: planning
updated: 2026-06-26
---

# Feature — Dashboard Piket (Fase 1b)

← [[Projek/AbsenSI/00-INDEX|Index]]

> **Fase 1b** — menyusul setelah inti Fase 1 (Absensi Gerbang, Manajemen Kartu, Import Data Master) stabil. Tidak dikerjakan bersamaan dengan inti Fase 1 untuk menghindari overload tim yang sedang belajar stack baru. Lihat [[Projek/AbsenSI/11-Decisions|ADR-015 s/d ADR-018]] untuk keputusan arsitektur yang melandasi fitur ini.

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
| Admin Pusat (`super_admin`) | Tetap punya kewenangan koreksi data absensi seperti biasa (lihat [[Projek/AbsenSI/03-User-Roles|03-User-Roles]]) — relasinya ke kewenangan piket harus dipertegas saat breakdown task (siapa yang menang kalau ada konflik koreksi) |

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
- Form dedicated: pilih siswa, alasan, keterangan, toggle **"Akan kembali hari ini?"**.
  - Kalau **ya** → isi jam kembali yang dijanjikan.
  - Kalau **tidak** → tidak perlu isi jam kembali, `attendance_records` hari itu langsung diset status `izin`/`sakit` untuk sisa hari.
- Submit → record `permits` (`jenis: keluar`) tersimpan, **kode verifikasi** unik digenerate.
- Sistem otomatis konstruksi URL ke `print.php` (server lokal `10.10.10.100:8800`, lihat ADR-018) dengan parameter siswa/alasan/jam/kode, buka di tab baru → petugas piket klik print manual dari preview.
- **Siswa tidak perlu tap** sama sekali — baik saat minta izin maupun saat keluar fisik (keputusan eksplisit, beda dari jalur gerbang normal).

### 4. Monitoring "Belum Kembali"
- Untuk siswa dengan izin keluar yang "akan kembali", begitu siswa lapor balik ke piket secara lisan, piket klik **"Sudah Kembali"** di dashboard (manual, bukan tap) → `permits.status_kembali` jadi `sudah`.
- Kalau `jam_kembali_diharapkan` sudah lewat dan belum ditandai kembali → siswa otomatis muncul di **daftar "Belum Kembali"** sebagai sinyal untuk piket menindaklanjuti (telepon orang tua, dst). Ini **bukan** trigger otomatis ke mekanisme lock — cuma sinyal/peringatan.

### 5. Tinjauan & Lock/Unlock Siswa
- Di akhir hari, sistem menandai siswa dengan izin "akan kembali" yang masih `status_kembali: belum` sebagai **"Perlu Ditinjau"**.
- Piket review daftar ini besok pagi — kalau memang belum terselesaikan (bukan cuma piket lupa klik), piket **secara manual** mengunci siswa tersebut (isi `locked_reason`, sistem catat `locked_by` + `locked_at`).
- Siswa terkunci yang tap di gerbang → **ditolak** dengan pesan jelas di layar kiosk ("Hubungi Guru Piket"), tetap dicatat sebagai log percobaan (pola sama dengan tap kartu inactive, lihat [[Projek/AbsenSI/06-Features/manajemen-kartu|manajemen-kartu.md]]).
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
| Tap gerbang (jalur normal) | `waktu_masuk`/`waktu_pulang` terisi, `pulang_via: tap` |
| `permits` jenis `tidak_masuk` | Status hari itu jadi `izin`/`sakit` |
| `permits` jenis `keluar`, tidak kembali | `waktu_pulang` = `jam_keluar`, `pulang_via: izin_piket`, status sisa hari `izin`/`sakit` |
| `permits` jenis `keluar`, akan kembali & terkonfirmasi | Tidak mengubah status existing — siswa dianggap tetap hadir normal |
| Siswa terkunci coba tap | Tap ditolak, **tidak** membuat `attendance_record`, hanya log percobaan |

**Catatan penting untuk modul Attendance saat breakdown task:** perlu aturan precedence tegas — status dari `permits` tidak boleh ditimpa balik oleh tap yang terjadi setelahnya di hari yang sama (dan sebaliknya). Ini belum didetailkan, lihat Open Questions.

---

## ❓ Open Questions (harus dijawab sebelum spec ini final & siap jadi task)

- [ ] Aturan precedence detail: kalau siswa sudah ditandai `izin` oleh piket pagi-pagi, tapi ternyata tap gerbang siang (kasus aneh tapi mungkin terjadi) — status mana yang menang?
- [ ] Relasi kewenangan koreksi data antara **Admin Pusat** (`super_admin`, sudah ada kewenangannya di `03-User-Roles.md`) dan **Guru Piket** — apakah piket bisa override koreksi admin atau sebaliknya, dan siapa yang jadi log "terakhir mengubah"?
- [ ] Apakah guru piket butuh riwayat/laporan bulanan perizinan (misal untuk keperluan BK atau rapat orang tua), atau cukup data mentah di `permits` tanpa laporan khusus di Fase 1b?
- [ ] UI kiosk gerbang untuk siswa terkunci — perlu desain pesan & tampilan spesifik, beda dari pesan UID tidak terdaftar/inactive
- [ ] Modifikasi `print.php` untuk parameter `kode` baru — siapa yang punya akses edit script ini di tim, dan kapan dikerjakan relatif ke development modul Permits

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang (Fase 1)]]
- [[Projek/AbsenSI/06-Features/manajemen-kartu|Manajemen Kartu RFID]]
- [[Projek/AbsenSI/06-Features/dashboard-tv|Dashboard TV Realtime]]
- [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] — entitas `kampus`, `users`, `permits`
- [[Projek/AbsenSI/11-Decisions|ADR-015, ADR-016, ADR-017, ADR-018]]

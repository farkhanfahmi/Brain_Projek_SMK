---
tags:
  - feature
  - design
created: 2026-06-05
status: approved
---

# Feature-006: Sistem Monitoring Petugas (Keliling + PDS/Team IT)

## Latar Belakang

Saat ujian berlangsung ada 3 peran lapangan yang selama ini tidak tercakup sistem:
1. **Petugas Keliling** — keliling tiap ruang per sesi, cek siswa tidak hadir + verifikasi pengawas
2. **PDS (Penegak Disiplin Sekolah)** — proses siswa terlambat di ruang khusus
3. **Team IT** — tangani siswa dengan device bermasalah, arahkan ke lab

Semua peran ini sudah masuk data **Panitia** di sistem. Yang dibutuhkan: dua flag tambahan di tabel `panitia` + satu kolom atribusi di `presensi_pesertas` + dua halaman/alur baru di `frontend/`.

---

## Scope Fitur

### Bagian 1 — Petugas Keliling (Monitoring Page)

**Login:** Scan NIY yang sama (endpoint `/login-niy`). Backend return flag `is_keliling = true` → frontend redirect ke halaman baru `/keliling`.

**Halaman `/keliling`:**
- Menampilkan daftar ruang yang punya jadwal di **sesi aktif saat ini** (waktu sekarang antara `mulai_ujian` dan `ujian_berakhir`, tanggal hari ini)
- Jika tidak ada sesi aktif → tampilkan pesan "Tidak ada sesi aktif saat ini"
- Petugas pilih ruang → masuk ke halaman detail ruang

**Halaman Detail Ruang:**

*Bagian atas (info ruang):*
- Nama mapel + sesi + jam
- Nama pengawas utama + pengawas pengganti (jika ada, format: "Pengawas: [Nama] | Pengganti: [Nama]")
- Status berita acara: "Belum diisi" / "Sudah disubmit" (dari `laporan_ujians`)

*Bagian bawah (daftar siswa belum terscan):*
- List siswa yang **belum ada presensi** di jadwal tersebut (berapapun yang scan — pengawas, PDS, atau Team IT)
- Tiap baris: nama siswa + nomor peserta + tombol/dropdown keterangan
- Keterangan pilihan: **Alfa**, **Sakit**, **Izin**, **Lainnya** (input teks bebas)
- Tombol simpan per siswa (atau bulk save)
- Saat keterangan disimpan → buat record `presensi_pesertas` dengan status `tidak_hadir` + `keterangan` terisi + `scanned_by_panitia_id` = panitia keliling

**Catatan penting:**
- Jika siswa sudah terscan (apapun yang scan) → tidak muncul di list
- Halaman ini tidak perlu real-time, cukup pull-to-refresh / refresh button
- Atribusi di data presensi tetap "panitia" bukan "petugas keliling" khusus

---

### Bagian 2 — PDS / Team IT Scan dengan Keterangan

**Siapa:** Panitia dengan flag `is_pds = true`

**Alur scan (dari halaman scan yang sudah ada):**
1. PDS scan barcode siswa
2. Backend cek: apakah siswa sudah hadir? → jika sudah → tolak dengan pesan "Sudah terscan"
3. Backend cek: apakah panitia ini `is_pds = true`? → jika ya → return flag `require_keterangan = true`
4. Frontend tampilkan **popup/modal keterangan** dengan pilihan:
   - Terlambat
   - Device Bermasalah
   - Lainnya (input teks)
5. Setelah pilih → konfirmasi → kirim ke backend dengan `keterangan` yang dipilih
6. Backend simpan presensi: status = `hadir`, `scanned_by_panitia_id` = panitia PDS, `scan_keterangan` = keterangan dipilih
7. Presensi masuk ke **jadwal ruang asal siswa** (bukan ruang khusus PDS)

**Di halaman pengawas (saat refresh):**
- Siswa yang `scanned_by_panitia_id` tidak null → tampilkan dengan **warna berbeda** (misal: orange/kuning vs hijau untuk scan normal)
- Di bawah nama siswa ada teks kecil: **"Diabsen oleh PDS/Team IT"** + keterangannya (misal: "Terlambat")

**Di berita acara (laporan_ujians):**
- `catatan_pelaksanaan` tetap diisi bebas oleh pengawas
- Saat laporan **disubmit** atau **ditampilkan**, backend auto-append ke akhir catatan:
  `" | {Nama1} {Keterangan1}, {Nama2} {Keterangan2}"`
- Contoh: `"Ujian berjalan lancar | Ahmad Terlambat, Budi Device Bermasalah"`
- Pendekatan: **computed on read** — stored `catatan_pelaksanaan` tidak diubah, appended info disimpan terpisah dari join presensi

---

## Perubahan Database

### Migration 1: `panitia` — tambah 2 kolom flag

```sql
ALTER TABLE panitia 
ADD COLUMN is_pds TINYINT(1) NOT NULL DEFAULT 0 AFTER can_scan,
ADD COLUMN is_keliling TINYINT(1) NOT NULL DEFAULT 0 AFTER is_pds;
```

**Aturan:** `is_pds` dan `is_keliling` bersifat MUTUALLY EXCLUSIVE dengan satu sama lain (satu panitia tidak bisa sekaligus PDS dan Keliling). Tapi keduanya bisa bersamaan dengan `can_scan`. Validasi di backend.

### Migration 2: `presensi_pesertas` — tambah atribusi + keterangan scan

```sql
ALTER TABLE presensi_pesertas
ADD COLUMN scanned_by_panitia_id INT NULL AFTER panitia_id,
ADD COLUMN scan_keterangan VARCHAR(255) NULL AFTER scanned_by_panitia_id,
ADD CONSTRAINT fk_presensi_scanned_by 
    FOREIGN KEY (scanned_by_panitia_id) REFERENCES panitia(id) ON DELETE SET NULL;
```

**Catatan:** `scanned_by_panitia_id` NULL = scan oleh pengawas sendiri. Tidak NULL = scan oleh panitia (keliling/PDS/Team IT).

---

## Perubahan Backend

### 1. `panitia` Model & Controller
- Tambah `is_pds`, `is_keliling` ke `$fillable`
- `PanitiaController::update()` + `store()`: validasi mutually exclusive (tidak boleh keduanya true)
- `PanitiaController::togglePds()` + `toggleKeliling()`: endpoint baru untuk toggle flag
- Route baru: `PATCH /panitia/{id}/toggle-pds` dan `PATCH /panitia/{id}/toggle-keliling`

### 2. Login (PresensiService / auth endpoint)
- Setelah login NIY berhasil → return tambahan field di response:
  - `is_keliling`, `is_pds`, `can_scan` dari panitia (jika user adalah panitia)
- Frontend pakai ini untuk routing ke halaman yang tepat

### 3. Endpoint baru: Keliling
- `GET /keliling/jadwal-aktif?ujian_id=X` → jadwal aktif saat ini (NOW() between mulai-selesai, hari ini)
- `GET /keliling/students?jadwal_id=X` → siswa belum terscan + info pengawas + status laporan
- `POST /keliling/keterangan` → simpan keterangan tidak hadir (buat presensi + `scanned_by_panitia_id`)

### 4. Modifikasi endpoint scan PDS
- `POST /presensi-peserta` (atau endpoint scan yang sudah ada): cek `is_pds` pada panitia
  - Jika `is_pds = true` → wajib ada field `scan_keterangan` di request body
  - Simpan `scanned_by_panitia_id` + `scan_keterangan` ke presensi
  - Presensi masuk ke jadwal ruang asal siswa (bukan ruang PDS)

### 5. Modifikasi response get-assignment / peserta list (untuk pengawas)
- Sertakan `scanned_by_panitia_id` + `scan_keterangan` di setiap data presensi siswa
- Frontend pengawas pakai ini untuk tampilkan visual berbeda

### 6. Modifikasi laporan (berita acara)
- Di endpoint GET laporan / submit laporan: auto-compute catatan tambahan dari presensi PDS
- Format append: jika ada presensi dengan `scanned_by_panitia_id != null` dan `scan_keterangan` terisi
  → append ke `catatan_pelaksanaan` response: ` | {nama} {keterangan}, ...`
- **Tidak mengubah nilai di database** — ini pure computed dalam response

---

## Perubahan Frontend

### `frontend/` (Pengawas/Panitia App)

**1. Routing setelah login:**
```jsx
if (user.is_keliling) navigate('/keliling');
else if (user.is_pds || user.can_scan) navigate('/scan'); // halaman scan (existing)
else navigate('/pengawas');
```

**2. Halaman baru `Keliling.jsx`:**
- List ruang dengan sesi aktif
- Detail per ruang: info pengawas + status BA + daftar siswa belum scan
- Dropdown keterangan per siswa + tombol simpan

**3. Modifikasi halaman scan (untuk PDS):**
- Setelah scan berhasil, cek `user.is_pds`
- Jika ya → tampilkan modal keterangan sebelum konfirmasi final
- Modal: pilihan Terlambat / Device Bermasalah / Lainnya (input text)

**4. Modifikasi tampilan daftar siswa pengawas:**
- Jika presensi siswa punya `scanned_by_panitia_id != null`:
  - Background/badge warna berbeda (orange/amber)
  - Teks kecil: "Diabsen oleh PDS/Team IT — {scan_keterangan}"

### `frontend-admin/` (Admin App)

**5. Modifikasi `Panitia.jsx`:**
- Tambah dua toggle button per baris panitia: **"PDS"** dan **"Keliling"**
- Styling sama dengan toggle `can_scan` yang sudah ada
- Disable salah satu jika yang lain sudah aktif (mutually exclusive)
- Call endpoint `PATCH /panitia/{id}/toggle-pds` dan `toggle-keliling`

---

## Keputusan Desain

| # | Keputusan | Alasan |
|---|-----------|--------|
| D1 | `is_pds` dan `is_keliling` sebagai boolean terpisah | Fleksibel untuk masa depan (bisa add tipe lain), tidak perlu ubah template import |
| D2 | Catatan PDS di laporan: computed on read, tidak ubah DB | Pengawas bisa edit catatannya sendiri bebas, PDS info tidak overwrite |
| D3 | Presensi keliling masuk sebagai status `tidak_hadir` | Konsisten dengan alur existing — halaman pengawas hanya tampilkan yang `hadir` |
| D4 | Routing frontend berdasarkan flag dari login response | Satu endpoint login, satu codebase — clean tanpa duplicate logic |
| D5 | `scanned_by_panitia_id` FK ke panitia, ON DELETE SET NULL | Jika panitia dihapus, atribusi hilang tapi presensi tetap ada |

---

## Open Question (untuk dikonfirmasi sebelum implementasi)

- [x] Apakah keterangan dari **petugas keliling** (Alfa/Sakit/Izin) juga perlu muncul di berita acara pengawas? → **TIDAK PERLU** — hanya keterangan PDS yang append ke catatan_pelaksanaan
- [ ] Apakah halaman keliling perlu filter per **sesi** (jika pengawas aktif di sesi 1 dan sesi 2 bersamaan di ruang berbeda) atau cukup filter per ruang?

---

## Task Breakdown

| Task | Scope | Dependencies |
|------|-------|--------------|
| [[tasks/task-006-backend-monitoring-foundation]] | Migration + flag panitia + presensi atribusi + semua endpoint baru/modifikasi | — |
| [[tasks/task-007-frontend-keliling]] | Halaman Keliling.jsx + routing login | task-006 selesai |
| [[tasks/task-008-frontend-pds-scan-attribution]] | Modal keterangan PDS + tampilan atribusi di pengawas + toggle admin | task-006 selesai |




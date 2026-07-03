---
tags:
  - project
  - api
created: 2026-06-04
updated: 2026-06-04
---

# API Endpoints — E-Berita Acara Ujian

**Base URL:** `http://[host]:8000/api`
**Auth:** Bearer token di header `Authorization` (Sanctum) — hanya untuk endpoint yang perlu auth
**Format response:** JSON

---

## Public Endpoints (Tidak Perlu Auth)

### Auth
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| POST | `/login` | Login admin (email + password) |
| POST | `/login-niy` | Login pengawas/panitia via scan NIY barcode |

### Template Download (Import)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/pengawas/template-import` | Download template Excel import pengawas |
| GET | `/jadwal-ujian/template` | Download template Excel import jadwal |
| GET | `/peserta-ujian/template` | Download template CSV import peserta |
| GET | `/ruang/template-import` | Download template import ruang |
| GET | `/panitia/template-import` | Download template import panitia |

### Operasi Hari-H (Sengaja Publik — Untuk TV & Dashboard)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/init-data` | Data awal: pengawas, ujians aktif, mata pelajaran, settings |
| POST | `/submit-report` | Submit berita acara (multipart/form-data dengan signature) |
| POST | `/scan-peserta` | Scan QR/barcode peserta — catat presensi |
| POST | `/manual-presensi-bulk` | Presensi manual massal |
| GET | `/presensi-today` | Daftar presensi peserta hari ini |
| GET | `/peserta-all` | Semua peserta satu ujian (panitia view) |
| GET | `/panitia-dashboard` | Data dashboard panitia per ruangan |
| GET | `/rekap-presensi` | Rekap presensi per ujian |
| GET | `/rekap-admin` | Rekap lengkap (peserta + pengawas + panitia) |
| GET | `/rekap-pengawas` | Rekap jadwal pengawas tertentu |
| GET | `/presensi-pengawas-today` | Presensi pengawas & panitia hari ini (untuk TV) |
| GET | `/get-assignment` | Jadwal & peserta untuk pengawas tertentu |

### Dashboard (Public — TV Monitor)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/dashboard/attendance-stats` | Statistik kehadiran global |
| GET | `/dashboard/attendance-by-campus` | Kehadiran per kampus |
| GET | `/dashboard/attendance-by-class` | Kehadiran per ruangan/kelas |
| GET | `/dashboard/attendance-by-kelas` | Kehadiran per kelas (untuk TV kolom 3) |
| GET | `/dashboard/attendance-students` | Detail peserta belum hadir |
| GET | `/dashboard/current-session` | Sesi ujian yang sedang berjalan |

### Keterangan & Pelanggaran (Public)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/keterangan-absen` | List keterangan ketidakhadiran |
| POST | `/keterangan-absen` | Tambah keterangan absen |
| PUT | `/keterangan-absen/{id}` | Update keterangan absen |
| DELETE | `/keterangan-absen/{id}` | Hapus keterangan absen |
| GET | `/catatan-pelanggaran` | List catatan ketidaktertiban |
| POST | `/catatan-pelanggaran` | Tambah catatan pelanggaran |
| PUT | `/catatan-pelanggaran/{id}` | Update catatan pelanggaran |
| DELETE | `/catatan-pelanggaran/{id}` | Hapus catatan pelanggaran |

### Manual Attendance
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| POST | `/manual-attendance` | Presensi manual (dari admin?) |

---

## Protected Endpoints (Auth: Sanctum — Admin)

### User Management
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/user` | Data user yang login |
| POST | `/logout` | Logout |
| GET | `/users` | List semua user |
| POST | `/users` | Buat user baru |
| PUT | `/users/{id}` | Update user |
| DELETE | `/users/{id}` | Hapus user |

### Ujian (Events)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/ujians` | List ujian |
| POST | `/ujians` | Buat ujian |
| GET | `/ujians/{id}` | Detail ujian |
| PUT | `/ujians/{id}` | Update ujian |
| DELETE | `/ujians/{id}` | Hapus ujian |

### Tahun Ajaran
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/tahun-ajaran` | List tahun ajaran |
| POST | `/tahun-ajaran` | Buat tahun ajaran |
| PUT | `/tahun-ajaran/{id}` | Update |
| DELETE | `/tahun-ajaran/{id}` | Hapus |

### Jadwal Ujian
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/jadwal-ujian` | List jadwal |
| POST | `/jadwal-ujian` | Buat jadwal |
| GET | `/jadwal-ujian/{id}` | Detail |
| PUT | `/jadwal-ujian/{id}` | Update |
| DELETE | `/jadwal-ujian/{id}` | Hapus |
| POST | `/jadwal-ujian/import` | Import dari Excel |

### Pengawas
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/pengawas` | List pengawas |
| POST | `/pengawas` | Buat pengawas |
| GET | `/pengawas/{id}` | Detail |
| PUT | `/pengawas/{id}` | Update |
| DELETE | `/pengawas/{id}` | Hapus |
| POST | `/pengawas/import` | Import dari Excel |

### Panitia
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/panitia` | List panitia |
| POST | `/panitia` | Buat panitia |
| GET | `/panitia/{id}` | Detail |
| PUT | `/panitia/{id}` | Update |
| DELETE | `/panitia/{id}` | Hapus |
| POST | `/panitia/import` | Import dari Excel |

### Peserta Ujian
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/peserta-ujian` | List peserta |
| POST | `/peserta-ujian` | Tambah peserta |
| GET | `/peserta-ujian/{id}` | Detail |
| PUT | `/peserta-ujian/{id}` | Update |
| DELETE | `/peserta-ujian/{id}` | Hapus |
| POST | `/peserta-ujian/import` | Import dari CSV |
| GET | `/peserta-ujian-meta` | Metadata peserta (sesi, kelas, dll.) |

### Ruang
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/ruang` | List ruang |
| POST | `/ruang` | Buat ruang |
| PUT | `/ruang/{id}` | Update |
| DELETE | `/ruang/{id}` | Hapus |
| POST | `/ruang/import` | Import ruang |

### Walikelas
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/walikelas` | List kelas + wali kelas |
| PUT | `/walikelas/{id}` | Update wali kelas |

### Laporan (Berita Acara)
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/laporan` | List semua laporan |
| GET | `/laporan/{id}` | Detail laporan |
| DELETE | `/laporan/{id}` | Hapus laporan |

### Settings
| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/settings` | Get semua settings |
| POST | `/settings` | Update/buat setting |

---

## Protected Endpoints (Auth: Pengawas Token)

| Method | Endpoint | Deskripsi |
|--------|----------|-----------|
| GET | `/pengawas-auth/me` | Data pengawas yang login + presensi hari ini |
| POST | `/pengawas-auth/logout` | Logout pengawas |

---

## Format Request/Response Penting

### POST `/login-niy`
```json
// Request
{ "niy": "12345678", "source": "Barcode Reader TV" }

// Response — Pengawas
{ "token": "1|abc...", "user": { "id": 1, "name": "...", "niy": "..." }, "presensi": {...} }

// Response — Panitia
{ "user": { "id": 1, "nama": "...", "jabatan": "...", "niy": "..." }, "type": "panitia" }

// Response — Error
{ "message": "NIY tidak ditemukan" } HTTP 404
```

### POST `/scan-peserta`
```json
// Request
{ "kode_peserta": "2425-001", "ujian_id": 1, "pengawas_id": 3 }
// atau
{ "kode_peserta": "2425-001", "ujian_id": 1, "panitia_id": 2 }

// Response sukses — peserta
{ "message": "Presensi berhasil", "peserta": { "nama": "...", "kelas": "..." } }

// Response sukses — pengawas/panitia (dual-mode)
{ "type": "pengawas", "presensi": { "waktu_datang": "...", "waktu_pulang": null } }
```

### POST `/submit-report` (multipart/form-data)
```
Fields: ujian_id, pengawas_id, mapel_id (nullable), kelas_id (nullable),
        mulai_ujian, ujian_berakhir, total_expected, total_present,
        total_absent, absent_details (nullable), notes (nullable)
File:   signature (image, max 2MB)
```

---

## ⚠️ Catatan Security

Endpoint operasional hari-H (scan, dashboard, rekap) sengaja **tidak dilindungi auth** agar TV display bisa akses tanpa login. Ini adalah keputusan desain yang sudah disepakati — lihat [[11-Decisions]] ADR-002.

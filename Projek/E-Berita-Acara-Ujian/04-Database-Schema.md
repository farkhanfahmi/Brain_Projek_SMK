---
tags:
  - project
  - database
  - schema
created: 2026-06-04
updated: 2026-06-04
---

# Database Schema — E-Berita Acara Ujian

## Diagram Relasi (Ringkas)

```
tahun_ajarans
    └──< ujians >──< jadwal_ujians >──< pengawas
                │           └──< mata_pelajarans >──< kelas
                │           └── ruangs
                │           └── jadwal_peserta (pivot)
                │                       └──< peserta_ujians >── ruangs
                │
                ├──< presensi_pesertas (kode_peserta)
                ├──< presensi_pengawas ──< pengawas / panitia
                ├──< laporan_ujians ──< pengawas
                ├──< keterangan_ketidakhadiran
                ├──< catatan_ketidaktertiban
                └──< panitia

users (admin only — terpisah dari pengawas/panitia)
settings
```

---

## Tabel-Tabel

### `users`
Admin sistem. Login via email + password.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| name | string | |
| email | string unique | |
| password | string | bcrypt |
| role | string | default: 'admin' |
| created_at, updated_at | timestamp | |

---

### `tahun_ajarans`
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| nama | string | contoh: "2024/2025" |
| created_at, updated_at | timestamp | |

---

### `ujians`
Event/jenis ujian utama.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| nama_ujian | string | contoh: "UAS Semester Genap 2025" |
| is_active | boolean | default: true |
| tahun_ajaran_id | FK → tahun_ajarans | nullable |
| jenis_ujian | string | contoh: "Teori", "Praktek" |
| jenis_presensi | enum | `QR` \| `Manual` |
| tampilkan_di_tv | boolean | apakah aktif di TV monitor |
| created_at, updated_at | timestamp | |

---

### `pengawas`
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| name | string | |
| niy | string nullable | Nomor Induk Yayasan — isi di barcode ID card |
| ujian_id | FK → ujians | nullable — pengawas ditugaskan ke ujian tertentu |
| foto | string nullable | path foto profil |
| created_at, updated_at | timestamp | |

---

### `panitia`
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| nama | string | |
| jabatan | string nullable | contoh: "Ketua Panitia", "Sekretaris" |
| niy | string nullable | Nomor Induk Yayasan |
| ujian_id | FK → ujians | cascade delete |
| foto | string nullable | path foto profil |
| can_login | boolean | default: true — izin login scan ID card |
| created_at, updated_at | timestamp | |

---

### `kelas`
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| nama_kelas | string | contoh: "XII TKJ 1" |
| walikelas | string nullable | nama wali kelas (text) |
| walikelas_panitia_id | FK → panitia nullable | wali kelas yang juga panitia |
| created_at, updated_at | timestamp | |

---

### `mata_pelajarans`
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| nama_mapel | string | contoh: "Pemrograman Web" |
| kelas_id | FK → kelas | cascade delete |
| created_at, updated_at | timestamp | |

---

### `ruangs`
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| nama_ruang | string | contoh: "Ruang 101", "Lab Komputer 2" |
| kampus | string nullable | contoh: "Kampus 1", "Kampus 2" |
| ujian_id | FK → ujians | nullable |
| created_at, updated_at | timestamp | |

---

### `jadwal_ujians`
Jadwal per sesi per ruangan per mapel.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| ujian_id | FK → ujians | cascade delete |
| pengawas_id | FK → pengawas | cascade delete |
| pengawas_pengganti_id | FK → pengawas nullable | jika pengawas diganti |
| mapel_id | FK → mata_pelajarans | cascade delete |
| nama_mapel | string nullable | nama mapel override (jika tidak pakai FK) |
| ruang_id | FK → ruangs nullable | |
| mulai_ujian | datetime | |
| ujian_berakhir | datetime | |
| total_siswa | integer | |
| sesi | integer nullable | contoh: 1, 2, 3 |
| created_at, updated_at | timestamp | |

---

### `peserta_ujians`
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| nama | string | |
| nisn | string unique | Nomor Induk Siswa Nasional |
| nomor_peserta | string unique | kode di QR card peserta |
| kelas | string | nama kelas (teks, bukan FK) |
| ruang_id | FK → ruangs nullable | ruang ujian |
| ujian_id | FK → ujians nullable | |
| sesi | integer nullable | sesi ujian |
| waktu_ujian | date nullable | untuk ujian praktek per jadwal individual |
| created_at, updated_at | timestamp | |

---

### `jadwal_peserta` (Pivot)
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| jadwal_ujian_id | FK → jadwal_ujians | |
| peserta_ujian_id | FK → peserta_ujians | |

---

### `presensi_pesertas`
Rekaman kehadiran peserta saat ujian.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| kode_peserta | string | = nomor_peserta |
| ujian_id | FK → ujians nullable | cascade delete |
| waktu_datang | timestamp nullable | |
| waktu_pulang | timestamp nullable | scan keluar |
| scanner_type | string nullable | `qr_camera` \| `barcode_reader` \| `panitia` |
| scanned_by | string nullable | nama panitia yang scan (jika panitia) |
| scan_ruang | string nullable | nama ruang saat scan |
| pengawas_id | FK → pengawas nullable | jika scan oleh pengawas |
| panitia_id | FK → panitia nullable | jika scan oleh panitia |
| created_at, updated_at | timestamp | |
| **UNIQUE** | (ujian_id, kode_peserta) per hari | mencegah race condition |

---

### `presensi_pengawas`
Rekaman kehadiran pengawas dan panitia.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| pengawas_id | FK → pengawas nullable | cascade delete |
| panitia_id | FK → panitia nullable | |
| ujian_id | FK → ujians nullable | |
| waktu_datang | timestamp nullable | |
| waktu_pulang | timestamp nullable | |
| catatan | string nullable | sumber: "Barcode Reader TV" / "Barcode Reader Smartphone" |
| created_at, updated_at | timestamp | |

---

### `laporan_ujians`
Berita Acara yang disubmit pengawas.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| ujian_id | FK → ujians | cascade delete |
| pengawas_id | FK → pengawas | cascade delete |
| mapel_id | FK → mata_pelajarans nullable | |
| kelas_id | FK → kelas nullable | |
| mulai_ujian | datetime | |
| ujian_berakhir | datetime | |
| total_expected | integer | total peserta seharusnya hadir |
| total_present | integer | total hadir |
| total_absent | integer | total absen |
| absent_details | text nullable | nama-nama yang absen |
| notes | text nullable | catatan pengawas |
| signature_path | string nullable | path file PNG tanda tangan |
| created_at, updated_at | timestamp | |
| **UNIQUE** | (ujian_id, pengawas_id, mapel_id, kelas_id, mulai_ujian) | |

---

### `keterangan_ketidakhadiran`
Alasan absensi peserta yang dicatat panitia.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| ujian_id | FK → ujians | cascade delete |
| nomor_peserta | string | |
| keterangan | string(50) | `Sakit` \| `Izin` \| `Alfa` \| `Lainya` |
| catatan | text nullable | keterangan tambahan |
| file_bukti | string nullable | path foto bukti (storage/public/bukti-izin/) |
| panitia_id | FK → panitia nullable | set null on delete |
| tanggal | date nullable | tanggal absen |
| created_at, updated_at | timestamp | |
| **UNIQUE** | (ujian_id, nomor_peserta, tanggal) | |

---

### `catatan_ketidaktertiban`
Catatan pelanggaran peserta yang dicatat panitia saat scan.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| ujian_id | FK → ujians | cascade delete |
| nomor_peserta | string | |
| presensi_id | FK → presensi_pesertas nullable | set null on delete |
| jenis_pelanggaran | enum | `Telat` \| `Atribut` \| `Pelanggaran Lainnya` |
| catatan | text nullable | |
| foto_pelanggaran | string nullable | path foto (storage/public/pelanggaran/) |
| panitia_id | FK → panitia nullable | set null on delete |
| created_at, updated_at | timestamp | |

---

### `settings`
Key-value store konfigurasi aplikasi.
| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| id | bigint PK | |
| key | string unique | contoh: `session_timeout_minutes` |
| value | text | |
| created_at, updated_at | timestamp | |

**Keys yang diketahui:**
- `session_timeout_minutes` — durasi idle sebelum auto-logout (default: 30)

---

## Catatan Penting

1. **Tidak ada soft delete** — semua delete adalah hard delete
2. **`nomor_peserta` vs `kode_peserta`** — dua nama untuk hal yang sama (nomor di QR card peserta). Di `peserta_ujians` disebut `nomor_peserta`, di `presensi_pesertas` disebut `kode_peserta`
3. **`pengawas.niy` bisa duplikat** — seorang pengawas yang mengawas beberapa ujian akan punya beberapa record `pengawas` dengan NIY yang sama. Query selalu cari `WHERE niy = ?` dan ambil semua ID-nya
4. **`presensi_pesertas` unique per hari** — bukan per record saja. Peserta yang scan dua kali di hari yang sama = update waktu_pulang, bukan insert baru

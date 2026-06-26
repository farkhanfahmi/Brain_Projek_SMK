---
tags: [absensi, feature, gerbang, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Absensi Gerbang (Fase 1)

← [[Projek/AbsenSI/00-INDEX|Index]]

> Modul utama fase 1. Siswa & guru tap kartu RFID di gerbang masuk utama sekolah. Status hadir/terlambat dihitung terhadap jam masuk sekolah (ADR-005).

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟡 Draft — belum final, banyak detail belum disepakati |
| Modul terkait | Core (Schedule, jadwal), Attendance, Card |
| Owner | Developer 3 (apps/api) untuk logic; Developer 2 (apps/kiosk) untuk UI tap |

---

## 👤 Aktor

| Aktor | Aksi |
|---|---|
| Siswa | Tap kartu di kiosk gerbang — datang & pulang |
| Guru/Karyawan | Tap kartu di kiosk gerbang — datang & pulang |
| Admin | CRUD jadwal masuk sekolah, lihat & koreksi rekap, kelola kartu |
| Kepala Sekolah | Lihat dashboard TV realtime (read-only) |

---

## ⚙️ Logic Inti

### Tap Pertama vs Tap Berikutnya
- Tap ke-1 di hari itu → status `masuk`, catat `waktu_masuk`
- Tap ke-2+ di hari itu → update jadi status `pulang`, catat `waktu_pulang`
- **Belum disepakati:** apakah tap ke-3+ di hari yang sama diabaikan, atau terus update `waktu_pulang` jadi waktu tap terakhir? → **PERLU DISKUSI LANJUTAN**
- **Belum disepakati:** minimum gap antar-tap untuk mencegah double-scan tidak sengaja (anak tap 2x cepat karena reader lag) dianggap sebagai 1 tap saja → **PERLU DISKUSI LANJUTAN, usulan: debounce 5-10 detik**

### Status Terlambat
- **Siswa:** tap masuk > jam masuk sekolah (dari Schedule, bisa beda per hari/per kondisi ujian) → `terlambat`
- **Guru:** tap masuk > (jam mengajar pertama hari itu − threshold menit yang diset admin) → `terlambat`. Threshold ini per-guru atau global? → **PERLU DISKUSI LANJUTAN**

### Validasi Tap
- UID kartu harus terdaftar & aktif (lihat [[Projek/AbsenSI/06-Features/manajemen-kartu|manajemen-kartu]]) — kalau UID tidak dikenal, kiosk tampilkan pesan error, TIDAK membuat record
- Idempotency: setiap tap dari kiosk dikirim dengan `client_uuid` unik (untuk dukung offline-retry) — server tolak duplikat `client_uuid`

---

## 🔌 Offline-First (Kiosk)

1. Kiosk capture keystroke dari reader HID → bentuk jadi UID
2. Coba POST ke API langsung
3. Kalau gagal (network down) → simpan ke buffer lokal (IndexedDB) dengan `client_uuid` + timestamp lokal
4. Background sync job di kiosk retry buffer setiap interval pendek (misal 10 detik) sampai semua tersinkron
5. Server proses sesuai `client_uuid` (idempotent) — urutan waktu pakai timestamp LOKAL kiosk saat tap (bukan saat sync berhasil), supaya status terlambat akurat meski sync telat

**Belum disepakati:** kalau kiosk down total (mati listrik, bukan cuma network) berapa lama buffer harus survive sebelum dianggap hilang? → **PERLU DISKUSI LANJUTAN**

---

## 📡 Event yang Dipicu

Setiap tap berhasil → dispatch event `attendance.recorded` ke BullMQ (lihat ADR-006). Payload minimal:
```
{ personId, personType: 'siswa'|'guru', tapType: 'masuk'|'pulang', timestamp, kioskId }
```

---

## 🖥️ Dashboard TV

Lihat [[Projek/AbsenSI/06-Features/dashboard-tv|dashboard-tv.md]] untuk detail terpisah.

---

## ❓ Open Questions (harus dijawab sebelum spec ini final & siap jadi task)

- [ ] Aturan tap ke-3+ di hari yang sama
- [ ] Debounce minimum antar-tap
- [ ] Threshold terlambat guru — per-guru atau global default + override
- [ ] Durasi survival buffer offline kiosk saat mati listrik total
- [ ] Apakah siswa yang TIDAK tap pulang (lupa/keburu) perlu auto-close di akhir hari, atau tetap status "masuk" terus?

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/manajemen-kartu|Manajemen Kartu RFID]]
- [[Projek/AbsenSI/06-Features/absensi-kelas-mapel|Absensi Kelas & Mapel (Fase 2)]]
- [[Projek/AbsenSI/11-Decisions|ADR-005]]


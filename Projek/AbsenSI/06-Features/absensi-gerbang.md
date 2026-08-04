---
tags: [absensi, feature, gerbang, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Absensi Gerbang (Fase 1)

← Index (00-INDEX AbsenSI.md)

> Modul utama fase 1. Siswa & guru tap kartu RFID di gerbang masuk utama sekolah. Status hadir/terlambat dihitung terhadap jam masuk sekolah (ADR-005).

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟢 Final — semua Open Questions resolved, siap jadi task |
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
- Tap ke-2 di hari itu → update `waktu_pulang`, catat `pulang_via: tap`
- **Tap ke-3+** di hari yang sama → **terus update `waktu_pulang`** ke waktu tap terakhir (tidak diabaikan). Ini mengakomodasi kasus siswa keluar-masuk sekolah beberapa kali dalam sehari — `waktu_pulang` selalu mencerminkan tap terakhir.
- **Debounce:** tap dari kartu yang sama dalam jarak **< 30 detik** dari tap sebelumnya → **diabaikan** (tidak membuat record baru, tidak mengupdate apa pun). Ini mencegah double-scan tidak sengaja karena reader lambat respons. Dicatat di `tap_events` dengan `result: rejected_duplicate`.

### Status Terlambat
- **Siswa:** tap masuk > jam masuk sekolah (dari `schedules`, bisa beda per hari/kondisi khusus ujian) → status `terlambat`
- **Guru:** tap masuk > (jam mengajar pertama hari itu − **threshold global** yang diset admin di konfigurasi sistem) → status `terlambat`. Threshold berlaku sama untuk semua guru — tidak ada override per-guru.

### Siswa Tidak Tap Pulang
Siswa tap masuk tapi tidak tap pulang sampai akhir hari sekolah → `waktu_pulang` tetap `null`. **Tidak ada auto-close.** Sistem menjalankan job harian di akhir hari untuk mengidentifikasi record `waktu_pulang = null` pada hari itu, lalu memunculkan daftar **"Tidak Tap Pulang Kemarin"** di Dashboard Piket keesokan harinya. Siswa harus klarifikasi ke guru piket → piket resolve manual (lihat dashboard-piket.md (06-Features/dashboard-piket.md) — section Antrian Klarifikasi).

### Validasi Tap
- UID kartu harus terdaftar & aktif (lihat manajemen-kartu (06-Features/manajemen-kartu.md)) — kalau UID tidak dikenal, kiosk tampilkan pesan error, tidak membuat record, dicatat di `tap_events` dengan `result: rejected_unknown`
- Idempotency: setiap tap dari kiosk dikirim dengan `client_uuid` unik (untuk dukung offline-retry) — server tolak duplikat `client_uuid`

---

## 🔌 Offline-First (Kiosk)

1. Kiosk capture keystroke dari reader HID → bentuk jadi UID
2. Coba POST ke API langsung
3. Kalau gagal (network down) → simpan ke buffer lokal (IndexedDB) dengan `client_uuid` + timestamp lokal
4. Background sync job di kiosk retry buffer setiap **5 detik** sampai semua tersinkron
5. Server proses sesuai `client_uuid` (idempotent) — urutan waktu pakai timestamp LOKAL kiosk saat tap (bukan saat sync berhasil), supaya status terlambat akurat meski sync telat

**Kiosk mati total (mati listrik):** IndexedDB tersimpan di disk — data tap yang sudah masuk buffer sebelum mati listrik **survive** saat kiosk restart dan akan tersinkron otomatis begitu nyala kembali. Data yang hilang hanya tap yang terjadi saat kiosk sedang mati. Untuk kasus ini, **admin input manual** setelah kiosk kembali nyala — tidak ada solusi otomatis di Fase 1, dan ini dianggap acceptable untuk konteks sekolah.

---

## 📡 Event yang Dipicu

Setiap tap berhasil → dispatch event `attendance.recorded` ke BullMQ (lihat ADR-006). Payload minimal:
```
{ personId, personType: 'siswa'|'guru', tapType: 'masuk'|'pulang', timestamp, kioskId }
```

---

## 🖥️ Dashboard TV

Lihat dashboard-tv.md (06-Features/dashboard-tv.md) untuk detail terpisah.

---

## ✅ Open Questions — Semua Resolved (2026-07-03)

- [x] **Tap ke-3+** → Update `waktu_pulang` terus ke waktu tap terakhir
- [x] **Debounce** → Tap dari kartu yang sama dalam < 30 detik diabaikan (`rejected_duplicate` di `tap_events`)
- [x] **Threshold terlambat guru** → Global (1 nilai untuk semua guru, diset admin di konfigurasi sistem)
- [x] **Buffer offline kiosk mati total** → IndexedDB survive restart. Tap saat kiosk mati = hilang, admin input manual. Acceptable untuk konteks sekolah. Mitigasi software: sync interval **5 detik** memperkecil window kehilangan data secara drastis. Mitigasi hardware opsional: UPS kecil per kiosk (direkomendasikan tapi tidak blocking Fase 1).
- [x] **Siswa tidak tap pulang** → `waktu_pulang` tetap null, muncul di Dashboard Piket besok pagi sebagai "Tidak Tap Pulang Kemarin", siswa klarifikasi ke piket, piket resolve manual.

**Status spec:** ✅ Final — siap dipecah jadi task development.

## 🔗 Lihat Juga
- Manajemen Kartu RFID (06-Features/manajemen-kartu.md)
- Absensi Kelas & Mapel (Fase 2) (06-Features/absensi-kelas-mapel.md)
- ADR-005 (11-Decisions.md)


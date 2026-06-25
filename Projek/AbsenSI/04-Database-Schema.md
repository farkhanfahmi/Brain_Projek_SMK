---
tags: [absensi, database, schema]
updated: 2026-06-25
---

# 04 — Database Schema

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> **Draft kasar** — belum final, menunggu Open Questions di 06-Features/*.md terjawab. Skema dirancang agar fase 1 (gerbang) dan fase 2 (kelas/mapel) sama-sama didukung tanpa rebuild (ADR-005).

---

## Entitas Inti (rancangan awal, nama tabel & kolom bisa berubah)

### `persons` (basis siswa & guru — pertimbangkan apakah 1 tabel atau 2 tabel terpisah)
- `id`, `type` (siswa/guru), `nama`, `nis_nip`, `kelas_id` (nullable, hanya siswa), `jurusan_id` (nullable), `status` (aktif/nonaktif)

### `cards`
- `id`, `uid` (unique), `person_id` (FK), `status` (active/inactive), `issued_at`, `revoked_at`

### `schedules` (jadwal masuk sekolah + jadwal mengajar guru — generic untuk fase 1 & 2)
- `id`, `type` (jam_sekolah / jam_mengajar / jadwal_khusus), `person_id` (nullable, FK ke guru jika type=jam_mengajar), `kelas_id` (nullable), `mapel_id` (nullable, fase 2), `hari`, `jam_mulai`, `jam_selesai`, `threshold_terlambat_menit`, `tanggal_berlaku_mulai`, `tanggal_berlaku_selesai` (untuk dukung jadwal khusus ujian)

### `attendance_sessions` (generic — gerbang ATAU kelas, lihat catatan di absensi-kelas-mapel.md)
- `id`, `location_type` (gerbang/kelas — fase 1 cuma gerbang), `kelas_id` (nullable), `mapel_id` (nullable, fase 2)

### `attendance_records`
- `id`, `person_id` (FK), `session_id` (FK, nullable di fase 1 kalau session tidak relevan), `tanggal`, `waktu_masuk`, `waktu_pulang` (nullable), `status` (hadir/terlambat/tidak_hadir/bolos — bolos baru relevan fase 2), `client_uuid` (unique, untuk idempotency offline-sync), `kiosk_id` (device asal tap)

## ❓ Open Questions
- [ ] `persons` 1 tabel gabungan atau `students` + `teachers` terpisah? (trade-off: query lebih simpel vs field yang relevan beda jauh antara siswa & guru)
- [ ] Index komposit final untuk filter rekap (kelas, jurusan, tanggal, status) — desain setelah volume data lebih jelas
- [ ] Strategi partitioning tabel `attendance_records` per tahun ajaran (mengingat estimasi ±500rb baris/tahun)

## 🔗 Lihat Juga
- [[30.Projects/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]]
- [[30.Projects/AbsenSI/06-Features/absensi-kelas-mapel|absensi-kelas-mapel.md]]

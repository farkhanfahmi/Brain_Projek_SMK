---
tags: [absensi, overview]
updated: 2026-06-25
---

# 01 — Overview

← [[30.Projects/AbsenSI/00-INDEX|Index]]

---

## Latar Belakang

Sekolah saat ini menjalankan absensi manual berbasis **Google Form + rekap spreadsheet**. Sistem ini akan digantikan dengan absensi **RFID tap-card** yang terintegrasi dengan database terpusat, mendukung pelaporan realtime, dan menjadi **aplikasi perintis** dari ekosistem aplikasi sekolah yang lebih besar (rencana jangka panjang: banyak aplikasi lain — akademik, perpustakaan, dll. — akan dibangun mengikuti pola arsitektur dan stack yang dirintis di proyek ini).

> **Keputusan eksplisit:** data lama dari spreadsheet **tidak** di-import. AbsenSI mulai dari data baru. (ADR-001)

## Visi

Membangun sistem absensi yang:
1. **Reliable** — tidak kehilangan data meski jaringan sekolah putus sementara (offline-first di titik scan)
2. **Realtime** — kepala sekolah bisa lihat rekap kehadiran live di TV tanpa delay berarti
3. **Extensible** — siap berkembang dari absensi gerbang → absensi per kelas/mapel → notifikasi orang tua, tanpa rebuild arsitektur
4. **Jadi blueprint** — pola arsitektur (monorepo, modular monolith, shared types, API kontrak) dipakai ulang untuk aplikasi sekolah berikutnya

## Scope per Fase

### 🟢 Fase 1 — Absensi Gerbang (scope pengembangan SEKARANG)
- Reader RFID (USB-HID) di pintu masuk utama sekolah + mini-PC + monitor
- Siswa & guru tap saat datang dan pulang (tap ke-1 = masuk, tap ke-2+ = update jadi pulang)
- Status hadir/tidak + terlambat/tidak **dihitung terhadap jam masuk sekolah** (belum per jam mengajar — itu butuh reader kelas yang belum ada)
- CRUD data kartu (mapping UID ↔ siswa/guru), termasuk proses ganti kartu hilang oleh admin
- Dashboard TV realtime menampilkan rekap kehadiran live
- Rekap fleksibel dengan filter (kelas, jurusan, hari, dll.)
- Jalur arsitektur untuk notifikasi disiapkan (event-driven), **tapi notifikasi WA TIDAK dibangun di fase ini**

### 🟡 Fase 2 — Absensi Kelas & Mapel (planning, belum dikerjakan)
- Reader RFID di setiap ruang kelas + ruang praktek
- Siswa **wajib tap gerbang dulu** sebelum bisa tap kelas — kalau belum tap gerbang, tap kelas ditolak sistem
- Tap di kelas mencatat kehadiran **per mapel/sesi pelajaran**, dibandingkan jadwal mengajar guru
- Guru dinyatakan terlambat **per sesi mengajar** (bukan cuma terlambat sekolah secara umum) berdasarkan tap kelas vs jadwal
- Siswa yang tap gerbang tapi tidak tap kelas tertentu → tercatat **bolos mapel** itu
- Catatan risiko (lihat [[30.Projects/AbsenSI/13-Backlog|13-Backlog]]): gerbang sekolah **tidak punya penghalang fisik** (bukan turnstile), jadi aturan "wajib tap gerbang dulu" punya potensi false-negative kalau siswa fisik masuk tanpa tap. Perlu keputusan UX (hard block vs flag warning) sebelum fase 2 dimulai.

### 🔵 Fase 3 — Notifikasi & Integrasi Lanjutan (ide masa depan, belum direncanakan detail)
- Notifikasi WA/SMS ke orang tua saat anak tap masuk/keluar
- Kemungkinan integrasi ke sistem akademik/nilai
- Kemungkinan jadi multi-app suite (AbsenSI jadi salah satu modul dari "OS sekolah" yang lebih besar)

## Asumsi yang Disepakati

- 1 sekolah saja (bukan SaaS multi-tenant) — lihat ADR terkait di [[30.Projects/AbsenSI/11-Decisions|11-Decisions]]
- Jaringan sekolah baik & internet merata, tapi tetap pakai offline-first sebagai mitigasi
- RFID reader tipe **USB-HID keyboard emulation** (sudah dikonfirmasi) — bukan SDK/serial proprietary
- Jadwal (jam masuk, jam mengajar, threshold terlambat) **fully configurable** oleh admin — termasuk skenario jadwal khusus (ujian, sesi berbeda)

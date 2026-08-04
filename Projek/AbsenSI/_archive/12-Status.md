---
tags: [absensi, status]
updated: 2026-07-21
---

# 12 — Status

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> Ringkasan progres per modul/app. Detail task-by-task ada di [[Projek/AbsenSI/TASKS-FASE-1|TASKS-FASE-1]], [[Projek/AbsenSI/TASKS-POLISH-1|TASKS-POLISH-1]], [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2]] — dokumen ini cuma snapshot ringkas, bukan sumber kebenaran detail.

---

## 🟦 apps/api (NestJS)
| Modul | Status | Catatan |
|---|---|---|
| Auth (JWT + Redis blacklist) | ✅ Selesai | login/refresh/logout, activity log login sukses+gagal (T034) |
| Core (Kampus/Kelas/Jurusan/Siswa/Guru/Schedule) | ✅ Selesai | — |
| Cards (RFID) | ✅ Selesai | CRUD, import CSV, tap-to-assign |
| Import (CSV students/teachers/cards) | ✅ Selesai | UI terintegrasi ke menu masing-masing (T033) |
| Calendar (academic years, school holidays) | ✅ Selesai | — |
| Attendance (tap, rekap, riwayat catatan) | ✅ Selesai | termasuk lock otomatis 2x terlambat (T037, ADR-025) |
| Attendance PDF export | ⏸️ Ditunda | T035 sengaja dilewati, tunggu diskusi format |
| Permits (izin/sakit/keluar) | ✅ Selesai | — |
| Piket Schedules (jadwal hari piket) | ✅ Selesai | T032, ADR-024 — termasuk `PiketOnDutyGuard` enforcement backend |
| Kiosks (device token, IP whitelist) | ✅ Selesai | ADR-021/022 |
| Photos (upload bulk + upload-by-id) | ✅ Selesai | T036 tambah endpoint upload-by-id |
| Activity Log (filter + pagination) | ✅ Selesai | T034 |
| Realtime (Socket.IO gateway) | ✅ Selesai | — |
| Queue (BullMQ attendance-recorded consumer) | ✅ Selesai | — |

## 🟩 apps/web (Next.js Admin Dashboard)
| Modul | Status | Catatan |
|---|---|---|
| Login + middleware auth | ✅ Selesai | — |
| Dashboard admin (Kampus/Kelas/Jurusan/Siswa/Guru/Kartu) | ✅ Selesai | — |
| Import Data (integrasi tombol per-menu) | ✅ Selesai | T033, menu `/import` lama dihapus dari sidebar (route tetap ada) |
| Upload Foto (bulk + upload-by-id di detail siswa) | ✅ Selesai | T036 |
| Kalender Pendidikan | ✅ Selesai | — |
| Jadwal Piket (grid mingguan admin) | ✅ Selesai | T032, `/jadwal-piket` |
| Log Aktivitas (halaman admin) | ✅ Selesai | T034, `/log` |
| Rekap Kehadiran | ✅ Selesai (tanpa export PDF) | — |
| Dashboard Piket | ✅ Selesai, direstrukturisasi 2x | Sidebar collapsible + 5 kartu ringkas expand-inline (T031, revisi 2026-07-21) |
| Dashboard TV | ✅ Selesai | — |
| Riwayat Guru | ✅ Selesai | — |
| Manajemen Akun | ✅ Selesai | — |
| Cetak Struk Izin | ✅ Selesai | Route handler internal, bukan lagi server PHP eksternal (ADR-018 revisi) |

## 🟨 apps/kiosk (Next.js Kiosk Gerbang)
| Modul | Status | Catatan |
|---|---|---|
| Tap capture (HID keyboard emulation) | ✅ Selesai | — |
| Offline buffer + sync | ✅ Selesai | — |
| Feedback screen (accepted/rejected/terlambat) | ✅ Selesai | — |
| Foto/avatar fallback | ✅ Selesai | Bug fallback foto diperbaiki T029 |
| Varian "Kartu Terkunci" (lock otomatis) | ✅ Selesai | T037, warna+ikon+pesan khusus beda dari accepted/rejected biasa |
| Dashboard guru (5 terbaru) | ✅ Selesai | — |

---

## Ringkasan Milestone

- **Fase 1** ([[Projek/AbsenSI/TASKS-FASE-1|TASKS-FASE-1]]): 31/32 task selesai.
- **Polish Batch 1** ([[Projek/AbsenSI/TASKS-POLISH-1|TASKS-POLISH-1]]): 8/8 task selesai.
- **Polish Batch 2** ([[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2]]): 8/9 task selesai — T035 (Rekap PDF) sengaja dilewati, format belum didiskusikan.
- **Belum dikerjakan:** Fase 2 (Absensi Kelas & Mapel), Fase 3 (Notifikasi Orang Tua) — lihat [[Projek/AbsenSI/13-Backlog|13-Backlog]].
- Belum ada testing hardware fisik menyeluruh dengan reader RFID produksi skala penuh — hanya testing manual dengan kartu dummy + 2 kiosk fisik terverifikasi (lihat catatan sesi di TASKS-POLISH-2.md).

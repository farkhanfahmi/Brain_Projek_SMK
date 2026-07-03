---
tags:
  - project
  - status
created: 2026-06-04
updated: 2026-06-04
---

# Status — E-Berita Acara Ujian

## Sprint Aktif

**Sprint:** Sprint 1 — Setup & Bugfix Infrastructure
**Periode:** 2026-06-04
**Fokus:** Setup aplikasi berjalan di device + fix ZipArchive import error

---

## Status Modul

| Modul | Status | Catatan |
|-------|--------|---------|
| Auth (Pengawas/Panitia) | ✅ Done | Berjalan di produksi |
| Auth Admin | ✅ Done | |
| Presensi Peserta (QR) | ✅ Done | |
| Presensi Peserta (Manual) | ✅ Done | |
| Berita Acara + TTD Digital | ✅ Done | |
| Dashboard Panitia | ✅ Done | |
| Monitor TV Real-time | ✅ Done | |
| Catatan Ketidaktertiban | ✅ Done | |
| Keterangan Absen | ✅ Done | |
| CRUD Admin (semua entitas) | ✅ Done | |
| Import Excel/CSV | ✅ Done | Pengawas, Peserta, Jadwal, Ruang, Panitia |
| Rekap & Export Excel | ✅ Done | |
| Settings Aplikasi | ✅ Done | session_timeout_minutes |
| Pengawas Pengganti | ✅ Done | |
| Barcode Reader TV | ✅ Done | |

---

## Checklist Produksi

- [x] Auth berjalan
- [x] Presensi QR berjalan
- [x] TV monitor berjalan
- [x] Berita acara berjalan
- [x] Admin panel berjalan
- [ ] Rate limiting pada public endpoints
- [ ] SQL backup files dikeluarkan dari repo
- [ ] `ExamReportController` refactor (God class 834 baris)

---

## Prioritas Saat Ini

1. ✅ **task-001** — Setup aplikasi + Fix ZipArchive → [[06-Features/tasks/task-001-setup-dan-fix-ziparchive]]
2. ✅ **task-002** — Fix filter peserta scan pengawas & manual kehadiran → [[06-Features/tasks/task-002-fix-peserta-filter-per-jadwal]]
3. ✅ **task-003** — Fix pivot generation import + template sesi → [[06-Features/tasks/task-003-fix-pivot-generation-import]]
4. ✅ **task-004** — Fitur "Hapus Semua" per menu import → [[06-Features/tasks/task-004-hapus-semua-per-menu-import]]
5. ✅ **task-005** — Fix import jadwal pengawas + edit form + rekap tanggal + template sesi → [[06-Features/tasks/task-005-fix-jadwal-pengawas-dan-rekap]]

---

6. ✅ **task-006** — Backend: migration + panitia flags + presensi atribusi + endpoint keliling → [[06-Features/tasks/task-006-backend-monitoring-foundation]]
7. ✅ **task-007** — Frontend: halaman Petugas Keliling → [[06-Features/tasks/task-007-frontend-keliling]]
8. ✅ **task-008** — Frontend: PDS scan modal + atribusi pengawas + toggle admin → [[06-Features/tasks/task-008-frontend-pds-scan-attribution]]
9. ✅ **task-009** — Fix KelilingController: scan_keterangan + ruang_id + status BA pengganti → [[06-Features/tasks/task-009-fix-keliling-controller]]
10. ✅ **task-010** — Fix login routing: panitia keliling diarahkan ke halaman pengawas → [[06-Features/tasks/task-010-fix-login-routing-panitia]]
11. ⏳ **task-011** — Backend: overview endpoint + migration attachment + file upload + getPresensiToday fix → [[06-Features/tasks/task-011-backend-keliling-redesign]]
12. 🔒 **task-012** — Frontend: rewrite Keliling.jsx one-page + photo upload UI + foto wajib validation *(blocked by task-011)* → [[06-Features/tasks/task-012-frontend-keliling-redesign]]
13. ✅ **task-013** — TV sorting (absent + per ruang) + auto-refresh pengawas 30s → [[06-Features/tasks/task-013-tv-sorting-pds-autorefresh]]
14. ✅ **task-014** — Fix 2 item terlewat: getPresensiToday + attachment conditional required
15. ✅ **task-015** — Post-testing: upload limit + keliling keterangan + PDS attribution + Device Bermasalah + rekapitulasi → [[06-Features/tasks/task-015-post-testing-fixes]]
16. ✅ **task-016** — Fix QRScanner: camera leak + layar hitam + retry button → [[06-Features/tasks/task-016-fix-qr-scanner]]
17. ✅ **task-017** — Fix TV dashboard: `current_sesi` backend + filter 3 panel sesi → [[06-Features/tasks/task-017-tv-sesi-filter-fix]]
18. ✅ **task-018** — Performance: DB index + range query + N+1 fix → [[06-Features/tasks/task-018-performance-index-n1]]
19. ✅ **task-019** — Infrastruktur: Nginx + PHP-FPM setup (ganti artisan serve) → [[06-Features/tasks/task-019-nginx-phpfpm-setup]]

---

### Pekerjaan Tidak Terdokumentasi (dilakukan Claude Code di luar task formal)

- **QRScanner iterasi 2–3**: Processing overlay "Memproses scan…", qrbox dinamis 70%, fps 25, disableFlip, BarcodeDetector native, resolusi native 16:9 width 1920
- **`utils/compressImage.js`**: utility baru resize max 1280px quality 0.7, integrasikan ke App.jsx (PDS), Keliling.jsx, PelanggaranModal.jsx
- **`start-prod.sh`**: script startup production nginx+fpm
- **`chmod o+x /home/anunnaki`**: permission fix untuk nginx read file di /home
- **`.env` optimasi hari-H**: APP_DEBUG=false, CACHE_STORE/SESSION_DRIVER=file, config:cache, route:cache
- **Fix keliling filter + outer presentInRoom/absentInRoom**: hapus kondisi hasPengganti, tambah scanned_by_panitia_nama + keliling_keterangan ke outer scope (2026-06-09)

---

## Feature Aktif

- [[feature-006-monitoring-petugas]] — Sistem Monitoring Petugas (Keliling + PDS/Team IT)

---

## Blockers

_Tidak ada saat ini._

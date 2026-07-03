---
tags:
  - project
  - backlog
created: 2026-06-04
updated: 2026-06-04
---

# Backlog — E-Berita Acara Ujian

## Technical Debt (Harus Diselesaikan)

| ID | Item | Prioritas | Estimasi |
|----|------|-----------|----------|
| TD-001 | Refactor `ExamReportController.php` — God class 834 baris, pisah ke Service/Controller yang lebih kecil | Tinggi | L |
| TD-002 | Hapus/gitignore file `.sql` di root project | Tinggi | XS |
| TD-003 | Frontend-tv hardcode `CAMPUSES = ['Kampus 1', 'Kampus 2']` — perlu dari API atau config | Sedang | S |
| TD-004 | Inkonsistensi nama: `nomor_peserta` (peserta_ujians) vs `kode_peserta` (presensi_pesertas) | Rendah | M |
| TD-005 | Tidak ada rate limiting pada public API endpoints | Tinggi | S |

---

## Fitur Potensial (Future)

| ID | Fitur | Prioritas | Catatan |
|----|-------|-----------|---------|
| F-001 | Notifikasi push ke wali murid jika peserta absen | Rendah | Perlu integrasi WhatsApp/email |
| F-002 | Export PDF Berita Acara dari admin | Sedang | Ada BeritaAcaraDocument.jsx tapi belum digunakan penuh |
| F-003 | Dashboard admin real-time (tanpa reload) | Sedang | Saat ini hanya manual refresh |
| F-004 | Multiple kampus dinamis (dari database, bukan hardcode) | Sedang | Untuk skalabilitas |
| F-005 | Riwayat presensi peserta antar ujian | Rendah | Analitik jangka panjang |
| F-006 | QR code generator untuk kartu peserta | Sedang | Saat ini harus generate manual |
| F-007 | Login admin dengan 2FA | Rendah | Keamanan extra |
| F-008 | Arsip/export rekap per ujian ke PDF/Excel otomatis saat ujian selesai | Sedang | |

---

## MoSCoW Priority

### Must Have (versi berikutnya)
- TD-001: Refactor ExamReportController
- TD-005: Rate limiting public endpoints
- TD-002: Bersihkan SQL backup dari repo

### Should Have
- F-002: Export PDF Berita Acara
- F-003: Dashboard real-time
- TD-003: Kampus dinamis

### Could Have
- F-001: Notifikasi wali murid
- F-006: QR code generator

### Won't Have (sekarang)
- F-007: 2FA admin
- Integrasi nilai/rapor

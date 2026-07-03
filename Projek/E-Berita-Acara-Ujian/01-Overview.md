---
tags:
  - project
  - overview
created: 2026-06-04
updated: 2026-06-04
---

# Overview — E-Berita Acara Ujian

## Latar Belakang

SMK Kartanegara Wates sebelumnya menggunakan form kertas untuk berita acara ujian. Proses ini lambat, rawan kehilangan data, dan tidak memberikan visibilitas real-time kepada panitia. Sistem ini dibuat untuk mendigitalisasi seluruh proses ujian dari presensi hingga laporan akhir.

## Visi

> Sistem ujian terpadu yang memungkinkan presensi real-time via QR code, monitoring lintas ruangan oleh panitia, dan pelaporan digital tanpa kertas.

## Tujuan

1. **Digitalisasi presensi** — peserta, pengawas, dan panitia scan ID card/kartu peserta
2. **Eliminasi berita acara kertas** — pengawas isi dan TTD digital langsung dari smartphone
3. **Visibilitas real-time** — TV monitor di lobby menampilkan kehadiran live
4. **Audit trail** — semua aksi tercatat dengan timestamp dan sumber scanner

## Scope

**In-scope:**
- Presensi peserta ujian (QR scan + manual)
- Presensi pengawas dan panitia (barcode reader TV)
- Berita acara digital dengan tanda tangan digital
- Monitor TV real-time (kehadiran per kampus, per kelas, per ruang)
- Dashboard panitia (monitoring lintas ruang)
- Manajemen data master (admin)
- Catatan ketidaktertiban peserta
- Keterangan ketidakhadiran (Sakit/Izin dengan bukti)

**Out-of-scope:**
- Integrasi dengan sistem akademik lain (rapor, nilai)
- Notifikasi push/email ke wali murid
- Aplikasi mobile native (Android/iOS)
- Manajemen soal ujian

## Asumsi

- Setiap peserta memiliki kartu ujian dengan QR code berisi `nomor_peserta`
- Setiap pengawas/panitia memiliki ID card dengan barcode berisi `NIY`
- Sekolah memiliki 2 kampus: Kampus 1 dan Kampus 2
- TV display terhubung ke komputer/laptop dengan barcode reader USB
- Semua device mengakses via browser (tidak perlu install app)

## Timeline

| Fase | Keterangan | Status |
|------|-----------|--------|
| v1.0 | Fitur inti: presensi QR, berita acara, TV monitor | ✅ Selesai (April 2026) |
| v1.1 | ??? | 🔄 Akan Direncanakan |

## Referensi Kode

```
berita-acara-ujian-baruuuuuuu/
├── backend/          ← Laravel 11 API
├── frontend/         ← React (Pengawas & Panitia)
├── frontend-admin/   ← React (Admin)
└── frontend-tv/      ← React (TV Monitor)
```

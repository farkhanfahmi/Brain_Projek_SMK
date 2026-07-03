# Project Context — E-Berita Acara Ujian

**Stack:** Laravel 11 (API) + React 19 (3 SPA: frontend, frontend-admin, frontend-tv) + TailwindCSS + SQLite/MySQL + Sanctum
**Status:** v1.0 — Sudah berjalan di produksi. Pengembangan lanjutan.
**Docs lengkap:** [[Projek/E-Berita-Acara-Ujian/00-INDEX|00-INDEX]]
**Kode:** `/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/`

## Keputusan Aktif
- Auth pengawas via Sanctum token (JWT-like), panitia via localStorage session — bukan cookie
- Presensi pengawas/panitia HANYA via Barcode Reader TV (frontend-tv), bukan dari frontend utama
- 3 SPA terpisah (frontend, frontend-admin, frontend-tv) — tidak monorepo
- Tidak ada soft delete — semua hard delete
- `jenis_presensi` di ujian menentukan mode: `QR` (scan kamera) atau `Manual` (dropdown)
- `tampilkan_di_tv` flag di ujian untuk menentukan ujian mana yang aktif di TV monitor
- Race condition dicegah dengan unique constraint di `presensi_pesertas`

## Sedang Dikerjakan
_[kosong — isi saat diskusi dimulai]_

## Jangan Lupakan
- SQL backup files ada di root project — JANGAN commit ke git
- Banyak API routes tidak dilindungi auth (disengaja untuk TV display) — dokumentasikan di 11-Decisions
- `ExamReportController.php` = God class 834 baris — kandidat refactor
- `pengawas_pengganti_id` di jadwal_ujians → pengawas pengganti bisa cover ruangan lain
- Frontend-tv hardcode `CAMPUSES = ['Kampus 1', 'Kampus 2']` — perlu diperhatikan jika kampus berubah

---
tags:
  - project
  - decisions
  - adr
created: 2026-06-04
updated: 2026-06-04
---

# Decisions (ADR) — E-Berita Acara Ujian

---

## ADR-001: Dokumentasi Proyek Dipindah ke Obsidian
**Tanggal:** 2026-06-04
**Status:** Accepted
**Konteks:** Aplikasi sudah berjalan di produksi tanpa dokumentasi terpusat. Pengembangan lanjutan membutuhkan source of truth yang jelas.
**Keputusan:** Semua dokumentasi desain, schema, API, dan keputusan arsitektur dikelola di Obsidian vault menggunakan sistem Claudian.
**Alasan:** Obsidian memberikan navigasi lintas file, wikilinks, dan konteks yang mudah di-attach saat sesi diskusi.
**Konsekuensi:** Setiap perubahan desain harus dicatat di vault sebelum dieksekusi di Claude Code.

---

## ADR-002: Endpoint Operasional Hari-H Tidak Dilindungi Auth
**Tanggal:** _tidak diketahui (keputusan lama)_
**Status:** Accepted
**Konteks:** `frontend-tv/` perlu akses ke data dashboard dan presensi tanpa login. TV display harus selalu aktif tanpa perlu maintain session.
**Keputusan:** Endpoint `/dashboard/*`, `/presensi-pengawas-today`, `/scan-peserta`, `/panitia-dashboard`, `/rekap-admin`, `/rekap-pengawas`, dan endpoint keterangan/pelanggaran dibiarkan public (tanpa middleware `auth:sanctum`).
**Alasan:** TV display tidak bisa maintain session dan perlu akses 24/7. Tradeoff keamanan diterima karena data yang diekspos hanya data presensi, bukan data sensitif seperti nilai.
**Konsekuensi:** Siapapun yang tahu URL bisa akses endpoint ini. **Perlu ditambahkan rate limiting** jika diekspos ke internet publik.

---

## ADR-003: Panitia Tidak Mendapat Sanctum Token
**Tanggal:** _tidak diketahui (keputusan lama)_
**Status:** Accepted
**Konteks:** Panitia perlu login tapi tidak perlu auth level sekuat pengawas. Session yang simple sudah cukup.
**Keputusan:** Login NIY panitia tidak menghasilkan Sanctum token. Data panitia disimpan di `localStorage['panitia_session']` sebagai JSON. `panitia_id` dikirim di body request.
**Alasan:** Menghindari overhead Sanctum token management untuk role yang aksesnya lebih terbatas. Session localStorage cukup untuk use case mobile dalam jaringan lokal sekolah.
**Konsekuensi:** Panitia session tidak expire via server-side. Auto-logout hanya via idle timeout di frontend.

---

## ADR-004: 3 SPA Terpisah, Bukan Monorepo
**Tanggal:** _tidak diketahui (keputusan lama)_
**Status:** Accepted
**Konteks:** Ada 3 jenis user dengan UI yang sangat berbeda: pengawas (mobile, simple), admin (desktop, CRUD-heavy), TV display (fullscreen, read-only).
**Keputusan:** Dibuat 3 project React terpisah dengan Vite masing-masing.
**Alasan:** Separation of concerns yang jelas. Pengembangan masing-masing tidak saling mengganggu. Build artifact lebih kecil per unit.
**Konsekuensi:** Duplikasi konfigurasi (vite.config, eslint, tailwind). Tidak ada shared component library. Update dependency perlu dilakukan di 3 tempat.

---

## ADR-005: Presensi Berdasarkan NIY, Bukan User Account
**Tanggal:** _tidak diketahui (keputusan lama)_
**Status:** Accepted
**Konteks:** Pengawas tidak punya akun sistem (email/password). Identifikasi hanya via ID card fisik.
**Keputusan:** Autentikasi pengawas dan panitia menggunakan NIY (Nomor Induk Yayasan) yang di-encode di barcode ID card. Tidak ada password.
**Alasan:** Kemudahan akses — pengawas hanya perlu bawa ID card, tidak perlu ingat password. Cocok untuk use case hari ujian yang hectic.
**Konsekuensi:** Siapapun yang punya ID card bisa login sebagai pengawas/panitia tersebut. Keamanan fisik ID card menjadi penting.

---

## ADR-006: Pengawas Multi-Record per NIY
**Tanggal:** _tidak diketahui (keputusan lama)_
**Status:** Accepted
**Konteks:** Pengawas yang sama bisa mengawas di beberapa ujian berbeda. Schema awal menyimpan `ujian_id` di tabel `pengawas`.
**Keputusan:** Satu pengawas bisa punya multiple record di tabel `pengawas` dengan NIY yang sama (satu record per ujian). Query selalu cari by NIY, bukan by ID tunggal.
**Alasan:** Simplisitas schema — tidak perlu tabel relasi many-to-many antara pengawas dan ujian.
**Konsekuensi:** Query by pengawas harus selalu `WHERE niy = ?` → dapat array ID, bukan single ID. Perlu hati-hati agar tidak bug jika query by ID saja.

---

## ADR-007: `jenis_presensi` Enum di Ujian
**Tanggal:** 2026-04-12
**Status:** Accepted
**Konteks:** Beberapa jenis ujian (praktek) tidak cocok dengan QR scan — presensi lebih mudah dilakukan manual.
**Keputusan:** Tambah kolom `jenis_presensi` di tabel `ujians` dengan nilai `QR` atau `Manual`. Frontend cek nilai ini untuk menentukan mode tampilan.
**Alasan:** Fleksibilitas tanpa perlu membuat aplikasi terpisah. Logika kondisional di frontend dan backend.
**Konsekuensi:** Semua query presensi perlu aware dengan mode ini. Jika `Manual`, `POST /manual-presensi-bulk` yang digunakan, bukan `/scan-peserta`.

---

## ADR-008: Port Backend Dipindah ke 8002
**Tanggal:** 2026-06-04
**Status:** Accepted
**Konteks:** Port 8000 sudah dipakai aplikasi DasiPelajar yang dikelola process manager (s6-supervise, PPID=1) sehingga tidak bisa dihentikan.
**Keputusan:** Backend E-Berita Acara Ujian berjalan di port **8002**.
**Alasan:** Menghindari konflik tanpa mengganggu DasiPelajar yang sudah berjalan di produksi.
**Konsekuensi:** Semua vite.config.js di tiga frontend sudah diupdate proxy ke 8002. `start-dev.sh` juga sudah pakai 8002. Jika server direstart, pastikan artisan serve dijalankan di port 8002.

---

_[tambahkan ADR baru di bawah sini]_

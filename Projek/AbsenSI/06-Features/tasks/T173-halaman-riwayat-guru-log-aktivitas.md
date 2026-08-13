# T173 — API+Web: Halaman "Riwayat" Guru — Log Aktivitas Digroup per Tanggal

## Depends on
**WAJIB setelah T168** (shell mobile-app). **REKOMENDASI dikerjakan TERAKHIR** dari rangkaian T168-T173 — task ini butuh SEMUA endpoint mutasi guru lain (start sesi, jurnal, presensi, nilai) SUDAH terpasang `@LogActivity` supaya riwayatnya lengkap saat diverifikasi live.

## Objective
Ganti/perkaya halaman `/guru/riwayat` (saat ini `GET /attendance/my-history` — HANYA riwayat tap gerbang) jadi log aktivitas lengkap: mulai kelas, membuat/edit jurnal, isi presensi, buat/edit nilai — semua aktivitas guru di dalam aplikasi ini, di-group per tanggal, dengan bahasa yang MUDAH DIBACA guru (bukan audit-log mentah teknis).

## Context — Temuan Riset (2026-08-13)

- `@LogActivity` decorator SUDAH dipakai 87x di seluruh `apps/api/src` — SUDAH ADA endpoint `GET /activity-log/me` (`activity-log.controller.ts:32`) dengan `ListMyActivityLogDto`, di-scope PAKSA ke `actorId` dari JWT sendiri (bukan admin audit mentah — desain existing SUDAH tepat untuk kasus ini, comment eksplisit "piket tidak boleh punya jalur query apa pun untuk mengubah actor" menunjukkan pola scoping yang sama persis dibutuhkan di sini).
- `/guru/riwayat` SAAT INI (`apps/web/src/app/(guru)/riwayat/riwayat-view.tsx`) HANYA tampilkan riwayat kehadiran guru sendiri (tap gerbang, dari `GET /attendance/my-history`) — SEMPIT, tidak mencakup aktivitas jurnal/presensi/nilai.
- **RISIKO UTAMA task ini**: endpoint-endpoint yang perlu tercatat (start sesi T169, create/update journal T171, update presensi existing, create/update grade T172) BELUM TENTU semua sudah pasang `@LogActivity` — riwayat insiden proyek ini (memory: 14/22 controller lupa decorator saat T-sebelumnya) — WAJIB diaudit ulang, bukan diasumsikan.

## Spec Detail

### 1. Audit — pastikan semua endpoint guru sudah pasang `@LogActivity`

Cek SATU PER SATU sebelum lanjut ke UI (kalau BELUM ada, tambahkan sebagai bagian task ini, JANGAN skip):
- `POST /teaching-sessions/:sessionId/start` (mulai kelas).
- Create/update `JournalEntry` (endpoint di `journal.service.ts`/controller).
- `PATCH /teaching-sessions/:id/attendance` (isi presensi per siswa) — VERIFIKASI apakah log per-siswa (berisiko log SANGAT RAMAI kalau per-toggle) atau cukup di level "presensi sesi X ditutup/diupdate" (REKOMENDASI: log level sesi, bukan per-siswa, supaya riwayat tidak banjir entri kecil — putuskan granularitas ini sebelum implementasi UI).
- `POST /grades/assessments` dan `PATCH /grades/assessments/:id/entries` (dari T172, kalau sudah dikerjakan lebih dulu — kalau belum, catat sebagai dependency balik yang perlu dicek ulang saat T173 dieksekusi).

### 2. Backend — endpoint `GET /activity-log/me` sudah cukup, TIDAK PERLU endpoint baru

- REUSE `GET /activity-log/me` apa adanya. VERIFIKASI apakah response-nya sudah cukup deskriptif untuk di-render manusiawi (field `description`/`metadata` per entry — cek struktur `ActivityLog` model) atau perlu sedikit penyesuaian format DESKRIPSI SAJA (bukan struktur/scope query) di titik-titik `@LogActivity` yang relevan supaya teksnya enak dibaca guru (misal "Memulai kelas XI TKR 3 - Matematika" bukan "session.start id=123").

### 3. Frontend — redesign `/guru/riwayat`

- `riwayat-view.tsx` — ganti sumber data dari `GET /attendance/my-history` MURNI jadi `GET /activity-log/me` (atau GABUNGAN keduanya kalau tap-gerbang tetap ingin ditampilkan sebagai salah satu jenis entri — REKOMENDASI: gabungkan, tap gerbang JUGA bagian dari "aktivitas guru hari itu", tampilkan sebagai 1 jenis entri di antara jenis lain).
- **Group by tanggal** — header tanggal (mis. "Rabu, 13 Agustus 2026") lalu list entri di bawahnya urut waktu (terbaru duluan ATAU terlama duluan dalam 1 hari — putuskan yang lebih natural, REKOMENDASI terlama→terbaru dalam 1 hari supaya baca seperti kronologi kejadian).
- Tiap entri: ikon jenis aktivitas (tap gerbang/mulai kelas/isi jurnal/isi presensi/buat nilai — icon berbeda per jenis) + deskripsi singkat manusiawi + jam.
- Pagination/infinite-scroll untuk riwayat panjang (REUSE pola pagination yang sudah ada di proyek kalau ada konvensinya).

## Edge Cases
- Guru baru (belum ada aktivitas sama sekali) — empty state jelas, bukan halaman kosong membingungkan.
- Entri `@LogActivity` yang formatnya masih teknis/tidak manusiawi dari modul LAIN yang kebetulan tercampur (kalau `GET /activity-log/me` scope-nya ternyata lebih luas dari yang diharapkan) — VERIFIKASI response HANYA berisi aktivitas relevan guru (bukan aktivitas admin yang menyentuh data guru itu, misal admin ubah profil guru — itu BUKAN aktivitas guru sendiri, harus TIDAK muncul di riwayat "aktivitas SAYA").

## Files
- **Modifikasi (audit, kemungkinan):** endpoint-endpoint di langkah 1 kalau ditemukan belum pasang `@LogActivity`.
- **Modifikasi:** `apps/web/src/app/(guru)/riwayat/riwayat-view.tsx` (redesign group-by-tanggal, sumber data activity-log).
- **Jangan sentuh:** `GET /activity-log/me` scoping logic (`actorId` dari JWT, sudah benar) — reuse apa adanya, TIDAK melonggarkan scope.

## Acceptance Criteria
- [ ] Semua endpoint aktivitas guru (mulai sesi, jurnal, presensi-level-sesi, nilai) terkonfirmasi pasang `@LogActivity` — verified live (bukan cuma baca kode, coba masing-masing aksi lalu cek muncul di log).
- [ ] `/guru/riwayat` menampilkan aktivitas ter-group per tanggal, urut kronologis dalam 1 hari.
- [ ] Deskripsi tiap entri manusiawi (bukan teknis mentah).
- [ ] Guru A tidak lihat aktivitas guru B (scope `actorId` dari JWT terverifikasi).
- [ ] Empty state untuk guru tanpa aktivitas.
- [ ] Build + type-check hijau, jest existing tetap pass.

## Validasi Claudian
- [ ] Audit eksplisit satu-per-satu endpoint di langkah 1 — laporkan yang SUDAH dan yang BARU ditambahkan `@LogActivity`, jangan asumsi.
- [ ] Konfirmasi granularitas log presensi (per-sesi vs per-siswa) — putuskan dan jelaskan alasannya di laporan implementasi.

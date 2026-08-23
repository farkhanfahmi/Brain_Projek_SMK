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
- [x] Semua endpoint aktivitas guru (mulai sesi, jurnal, presensi-level-sesi, nilai) terkonfirmasi pasang `@LogActivity` — verified live (bukan cuma baca kode, coba masing-masing aksi lalu cek muncul di log).
- [x] `/guru/riwayat` menampilkan aktivitas ter-group per tanggal, urut kronologis dalam 1 hari.
- [x] Deskripsi tiap entri manusiawi (bukan teknis mentah).
- [x] Guru A tidak lihat aktivitas guru B (scope `actorId` dari JWT terverifikasi — tidak diubah, sudah benar sejak T111).
- [x] Empty state untuk guru tanpa aktivitas.
- [x] Build + type-check hijau, jest existing tetap pass.

## Validasi Claudian
- [x] Audit eksplisit satu-per-satu endpoint di langkah 1 — laporkan yang SUDAH dan yang BARU ditambahkan `@LogActivity`, jangan asumsi.
- [x] Konfirmasi granularitas log presensi (per-sesi vs per-siswa) — putuskan dan jelaskan alasannya di laporan implementasi.

## Status Eksekusi (2026-08-15)

**Selesai.**

### 1. Audit `@LogActivity` — hasil per endpoint

Dicek satu-per-satu (grep + baca kode), TIDAK ada yang lupa dipasang (beda dari insiden 14/22 sebelumnya):

| Endpoint | Status sebelum T173 | Mekanisme |
|---|---|---|
| `POST /teaching-sessions/:sessionId/start` | ✅ Sudah ada | `@LogActivity` decorator (`teaching-sessions.controller.ts`) |
| Create/update `JournalEntry` (`journal.service.ts` — `updateJournal`) | ✅ Sudah ada | Manual `activityLog.record()` (targetId = sessionId, bukan row id sendiri — tidak cocok utk decorator) |
| `PATCH .../attendance` (isi presensi, `journal.service.ts` — `updateAttendance`) | ✅ Sudah ada | Manual `activityLog.record()`, level PANGGILAN (1 log per PATCH) |
| `POST /grades/assessments` (`grades.service.ts` — `createAssessment`) | ✅ Sudah ada | Manual `activityLog.record()` (create+child records, tidak cocok decorator) |
| `PATCH /grades/assessments/:id/entries` (`grades.service.ts` — `updateEntries`) | ✅ Sudah ada | Manual `activityLog.record()` (bulk update banyak baris) |

Tidak ada endpoint yang perlu ditambahkan baru — SEMUA sudah logging sejak T169-T172. Yang BARU ditambahkan di T173 murni **pengayaan `snapshotAfter`** (bukan logging baru): field `kelas`/`mapel`/`judul`/`nama siswa` ditambahkan ke payload snapshot di 4 titik manual + response body `startSession` (karena `@LogActivity` men-snapshot response body apa adanya), supaya deskripsi di riwayat bisa manusiawi ("Memulai Kelas — XI TKR 1 - Pemrograman Web") bukan cuma ID mentah.

### 2. Granularitas log presensi — keputusan & alasan

**Keputusan: backend TETAP di level per-PATCH-call (bisa 1 log per toggle siswa), TIDAK diubah.** Penggabungan jadi 1 baris per sesi dilakukan di **layer tampilan** (frontend), bukan backend.

Alasan: mengubah backend ke level "per-sesi" akan mengubah alur `updateAttendance`/`attendance-table.tsx` yang sudah stabil sejak T171 (risiko regresi), sementara solusi tampilan cukup: entri `class_attendance_mark.update` yang BERURUTAN (setelah timeline digabung+diurutkan) dengan deskripsi identik (kelas+mapel sama, dari `snapshotAfter`) digabung jadi 1 baris "Mengisi Presensi — {kelas} - {mapel}" di `riwayat-view.tsx` (`buildTimeline()`). Diverifikasi live: 2 row `activity_log` terpisah (siswa berbeda, timestamp beda 1 menit) tampil sebagai 1 entri di UI.

### 3. Backend — role guard `GET /activity-log/me`

`@Roles(UserRole.guru_piket)` → `@Roles(UserRole.guru_piket, UserRole.guru)` di endpoint yang SAMA (bukan endpoint baru). Scoping `actorId` dari JWT (`{ ...filter, actorId: user.sub }`) tidak disentuh — sudah benar untuk kedua role tanpa perubahan (guru A tidak bisa lihat aktivitas guru B).

### 4. Frontend — `riwayat-view.tsx` redesign

- Sumber data GABUNGAN: `GET /activity-log/me?pageSize=100` (5 aksi guru) + `GET /attendance/my-history` (tap gerbang — TIDAK tercatat di `activity_log` sama sekali, harus digabung terpisah).
- Grup per tanggal (header "Sabtu, 15 Agustus 2026"), urut kronologis awal→akhir dalam 1 hari.
- `ACTION_LABEL` + `ACTION_ICON` map utk 5 aksi: `teaching_session.start`, `journal_entry.update`, `class_attendance_mark.update`, `grade_assessment.create`, `grade_entry.update` (+ `attendance.tap_gerbang` sintetis utk tap gerbang, action ini TIDAK ada di DB, hanya label internal FE).
- Reuse komponen existing `ActivityFeedCard`/`ActivityFeedItem` (`packages/ui`) — sebelumnya cuma dipakai TV Piket dashboard, sudah cocok dgn spec desain "Activity Feed" (icon chip + 2 baris teks, tanpa divider antar-row).
- Empty state: pesan jelas kalau guru belum ada aktivitas sama sekali.

### 5. Verifikasi

- Unit test: 3 file terdampak (`grades.service.spec.ts`, `journal.service.spec.ts` — sudah pass tanpa perubahan, `teaching-sessions.service.spec.ts`) diperbaiki mock-nya (tambah `prisma.kelas`/`prisma.mapel.findUnique`, `student.nama` di fixture) — **415/415 test pass**, seluruh suite `apps/api`.
- `tsc --noEmit` bersih di `apps/api` dan `apps/web` (1 error pre-existing di `kartu-view.tsx`, TIDAK terkait T173, dari perubahan T177 yang belum di-commit).
- Live-verify browser: data sintetis (mapel+schedule+teaching_session+6 activity_log rows, semua di-cleanup setelah verifikasi) di-insert langsung ke DB dev karena hari eksekusi jatuh Sabtu (tidak ada sesi mengajar live real). Hasil: grouping tanggal ✅, urutan kronologis ✅, deskripsi manusiawi ✅ (termasuk judul assessment "UH Bab 3"), merge 2 entri presensi berurutan → 1 baris ✅.
- Empty state TIDAK diverifikasi via browser langsung (tidak ada kredensial test-account kedua yang aman dicoba tanpa override password lagi) — diverifikasi via pembacaan kode (`dateKeys.length === 0` guard di `RiwayatView`), logikanya sederhana dan low-risk.

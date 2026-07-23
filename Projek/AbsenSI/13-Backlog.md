---
tags: [absensi, backlog, roadmap]
updated: 2026-07-21
---

# 13 — Backlog & Roadmap Fase

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> Roadmap Fase 2/3 di bawah masih berupa rencana (belum dikerjakan) — bagian Fase 1/1b sudah diupdate mencerminkan status selesai.

---

## 🟢 Plotting Siswa ke Kelas (✅ final, siap eksekusi — T071-T074, ditemukan 2026-07-22)
Celah: siswa cuma bisa di-assign ke kelas SEKALI saat form "Tambah Siswa" dibuat — tidak ada mekanisme pindah kelas maupun penempatan massal (SPMB/kenaikan kelas). Detail lengkap: [[Projek/AbsenSI/06-Features/plotting-siswa-kelas|plotting-siswa-kelas.md]]. Task breakdown: [[Projek/AbsenSI/TASKS-FASE-2-JURNAL|TASKS-FASE-2-JURNAL]] Blok 14.

## 🟡 Migrasi Data dari Aplikasi Lama (planning, ditemukan 2026-07-22)
AbsenSI adalah rebuild dari aplikasi absensi lama (Laravel+Spatie). Dump lama: `/media/anunnaki/DataNvme/sql_absensi_smk.sql`. Audit struktur menemukan beberapa field biodata hilang saat rebuild (gelar, status pernikahan/kepegawaian guru, data wali murid siswa) — sudah diputuskan ditambahkan kembali. Detail lengkap: [[Projek/AbsenSI/06-Features/migrasi-database-lama|migrasi-database-lama.md]]. **Task siap eksekusi:** T061 (schema biodata guru), T063 (schema wali murid siswa), T062 (ETL migrasi data, WAJIB dry-run dulu sebelum insert nyata — lihat task untuk detail keamanan proses ini).

## ✅ Fase 1 — Absensi Gerbang (SELESAI, 31/32 task)
Lihat [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]] untuk spek awal, [[Projek/AbsenSI/TASKS-FASE-1|TASKS-FASE-1]] untuk detail eksekusi task-by-task.

**Modul yang sudah selesai:**
- Absensi gerbang (tap RFID) — termasuk offline buffer, debounce, idempotency
- Manajemen kartu (CRUD, import CSV, tap-to-assign)
- Import data master (ADR-009) — sekarang terintegrasi ke tombol per-menu (T033), bukan lagi menu terpisah
- **Kalender Pendidikan** — [[Projek/AbsenSI/06-Features/kalender-pendidikan|kalender-pendidikan.md]]
- **Rekap kehadiran untuk Admin Pusat** — [[Projek/AbsenSI/06-Features/rekap-kehadiran|rekap-kehadiran.md]] (export PDF/T035 sengaja ditunda, belum ada di Fase 1 maupun Polish Batch 2)

## ✅ Fase 1b — Dashboard Piket (SELESAI, plus Polish Batch 2)
Lihat [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]] dan [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2]] (T029–T037). Selesai dengan tambahan signifikan di luar rencana awal: jadwal hari piket per akun (T032, ADR-024), activity log lengkap (T034), lock otomatis 2x terlambat (T037, ADR-025), dan restrukturisasi navigasi 2x (T031 + revisi) jadi sidebar collapsible dengan 5 kartu ringkas expand-inline.

## 🟡 Fase 2 — Prioritas #1: Dashboard Guru — Jurnal Mengajar & Absensi Mapel (planning-interview)
Lihat [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]]. **Menggantikan** pendekatan lama tap-RFID-kelas (lihat arsip di bawah) — absensi mapel sekarang diturunkan dari data tap gerbang + koreksi manual guru, bukan reader fisik baru. Termasuk fitur jadwal Mode Blok (Minggu A/B, toggle-able) vs Mode Normal, gating geofence+toleransi keterlambatan untuk "Mulai Mengajar", alur izin guru (role baru `admin_jurnal`), dan role Wali Kelas (belum final). Status: masih tahap interview, banyak open question belum diputuskan (lihat dokumen). **Task breakdown siap eksekusi:** [[Projek/AbsenSI/TASKS-FASE-2-JURNAL|TASKS-FASE-2-JURNAL]] (T038–T051, 14 task).

## 🟡 Fase 2 — Prioritas #2: Dashboard Petugas Kartu (✅ final, siap task)
Lihat [[Projek/AbsenSI/06-Features/dashboard-petugas-kartu|dashboard-petugas-kartu.md]]. Clone menu manajemen kartu (sudah final Fase 1) ke dashboard dedicated untuk role `card_admin` yang sudah ada tapi belum punya UI sendiri — tidak ada perubahan wewenang/skema, murni kerja UI+routing. Bisa dikerjakan kapan saja, tidak bergantung fitur lain.

## 🟢 Fase 2 — Prioritas #3: TV Piket (✅ final, siap eksekusi — T068-T070)
Lihat [[Projek/AbsenSI/06-Features/tv-piket|tv-piket.md]]. Layar publik di lorong piket (per kampus), bento grid 1 layar: persentase hadir/izin/alfa siswa, nama siswa tidak hadir+keterangan, guru belum mulai mengajar, guru izin. Layout difinalisasi 2026-07-22 (tanpa menunggu sketsa — dari referensi EzMart + komponen generik T058/T059). Auth token tanpa expiry + revoke manual. Task breakdown: [[Projek/AbsenSI/TASKS-FASE-2-JURNAL|TASKS-FASE-2-JURNAL]] Blok 13.

## 🟡 Fase 2 — Fitur Lanjutan Lain (planning, dikerjakan setelah prioritas di atas)
- **Rekap untuk Wali Kelas** — scope dibatasi ke kelas yang diampu (lihat [[Projek/AbsenSI/06-Features/rekap-kehadiran|rekap-kehadiran.md]])
- **Riwayat kehadiran Guru** — guru lihat riwayat diri sendiri
- **Rekap perizinan untuk Guru Piket** — laporan bulanan permits per kampus (untuk BK/rapat orang tua)
- **Export Excel** rekap (jika diputuskan butuh)

## ⚰️ Fase 2 (ARSIP, digantikan) — Absensi Kelas & Mapel via tap RFID
Lihat [[Projek/AbsenSI/06-Features/absensi-kelas-mapel|absensi-kelas-mapel.md]]. **Tidak lagi dilanjutkan** — digantikan [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]] karena blocker gerbang-tanpa-penghalang-fisik tidak pernah terpecahkan, dan sekolah lebih butuh jurnal mengajar + fleksibilitas jadwal blok A/B daripada tap fisik di tiap kelas.

## 🔵 Fase 3 — Ide Masa Depan (belum direncanakan detail)
- Notifikasi WA/SMS ke orang tua (jalur sudah disiapkan, lihat ADR-006 + notifikasi-ortu.md)
- Import data historis dari spreadsheet lama (kalau ternyata dibutuhkan — ADR-001 saat ini menolaknya)
- Integrasi ke sistem akademik/nilai
- Bulk import kartu via CSV untuk rollout awal 2.500 kartu (kemungkinan ini harus naik prioritas ke fase 1 — lihat Open Question di manajemen-kartu.md)
- AbsenSI jadi modul dari "OS sekolah" yang lebih besar — kemungkinan modul Core (siswa/guru/jadwal) dipecah jadi service terpisah yang dikonsumsi app lain

## Technical Debt / Risiko yang Diketahui
- Gerbang tanpa penghalang fisik — risiko false-negative di fase 2 (lihat absensi-kelas-mapel.md)
- Belum ada keputusan soal threshold terlambat guru per-guru vs global
- Belum ada keputusan retensi data buffer offline kiosk saat mati listrik total


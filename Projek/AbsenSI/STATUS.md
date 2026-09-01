---
tags: [absensi, status]
updated: 2026-08-31
---

# STATUS.md — Satu-satunya tempat cek status AbsenSI

> **[2026-08-31] File ini dipecah** — sebelumnya 939KB (86% isinya task SUDAH selesai).
> Detail lengkap 154 task selesai dipindah ke `_archive/STATUS-Arsip-Selesai.md`.
> File ini sekarang HANYA berisi: ringkasan fase + task AKTIF/belum dikerjakan.
> Baca ini duluan tiap sesi baru — jauh lebih ringan dari sebelumnya.

---

## Ringkasan Fase

| Fase | Status | Detail |
|---|---|---|
| Fase 1 — Absensi Gerbang | ✅ Selesai (31/32, T035 PDF sengaja ditunda) | `_archive/TASKS-FASE-1.md` |
| Polish Batch 1 | ✅ Selesai (8/8) | `_archive/TASKS-POLISH-1.md` |
| Polish Batch 2 | ✅ Selesai (8/9, T035 masih ditunda) | `_archive/TASKS-POLISH-2.md` |
| Fase 2 — Dashboard Guru Jurnal + turunannya (T038-T089) | ✅ Selesai (51/52 — hanya T060 belum) | `_archive/TASKS-FASE-2-JURNAL.md` |
| Polish Batch 3 — Dashboard Piket (T090-T095) | ✅ Selesai semua | `_archive/TASKS-POLISH-3.md` |
| Ekstrakurikuler + turunannya (T096-T105) | ✅ Selesai semua | lihat `06-Features/tasks/T096-T105` |
| **154 task lain (T106-T262 mayoritas)** | ✅ Selesai — **detail lengkap di `_archive/STATUS-Arsip-Selesai.md`** | lihat arsip |
| T263 — Duplikasi Data Kelas X GAME DEV (screenshot dev) | ✅ Selesai (dieksekusi Claude Code, 2026-08-31) | `06-Features/tasks/T263-*.md` |

---

## 🔴 Task Aktif / Belum Dikerjakan (WAJIB baca sebelum eksekusi apapun)

> Dipisah dari tabel besar 2026-08-31 (klasifikasi diverifikasi manual per-baris,
> bukan otomatis — tabel asli tidak konsisten strukturnya, sudah terverifikasi hati-hati
> untuk 172 baris awal, lihat `_archive/STATUS-Arsip-Selesai.md` untuk yang sudah selesai).

| Task                                                                     | Status           | Prioritas                             | Deskripsi Singkat                                                                                                                                               | Task Terbuat    | Task Tereksekusi | Ref                                                                               |
| ------------------------------------------------------------------------ | ---------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ---------------- | --------------------------------------------------------------------------------- |
| T060 — Audit Visual Menyeluruh Dashboard Guru/Admin Jurnal               | BELUM DIKERJAKAN | Sedang                                | Audit visual sistematis 11 halaman dashboard Guru/Admin Jurnal — belum ada laporan/artefak audit sama sekali.                                                   | Tidak diketahui | —                | `06-Features/tasks/T060` — belum ada file, cek TASKS-FASE-2-JURNAL arsip Blok 9   |
| T088 — Migrasi UUID Tabel Sensitif                                       | BELUM DIKERJAKAN | Rendah                                | Migrasi PK Int→UUID untuk tabel sensitif (Student/Teacher/Card) — SENGAJA ditunda, audit keamanan membuktikan tidak ada celah IDOR nyata.                       | 2026-07-26      | —                | `06-Features/tasks/T088-uuid-tabel-sensitif.md`                                   |
| T098 — Auto-Lock Izin Tidak Kembali                                      | BELUM DIKERJAKAN | Tinggi (risiko)                       | Auto-lock siswa izin keluar tidak kembali + hapus section "Perlu Ditinjau" — perlu didiskusikan ulang, dashboard piket sudah banyak berubah sejak spec ditulis. | 2026-07-30      | —                | `06-Features/tasks/T098-auto-lock-izin-tidak-kembali.md`                          |
| T101 — Validasi Jam Pulang Sesuai Jadwal Kelas                           | BELUM DIKERJAKAN | Ditunda tanpa batas                   | Validasi jam pulang sesuai jadwal kelas — BLOCKED, data jadwal jam_mengajar riil belum lengkap (masih dummy).                                                   | 2026-07-30      | —                | `06-Features/tasks/T101-validasi-jam-pulang-jadwal-kelas.md`                      |
| T114 — Modul Setting Kop Surat (Letterhead, Reusable)                    | BELUM DIKERJAKAN | Tinggi (blocker T115/T116)            | Modul Setting Kop Surat (Letterhead) — blocker untuk T115/T116, model singleton belum dibangun.                                                                 | 2026-08-06      | —                | `06-Features/tasks/T114-modul-setting-kop-surat.md`                               |
| T116 — Rekap Kehadiran Guru (Fitur Baru dari Nol)                        | BELUM DIKERJAKAN | SUPERSEDED oleh T176                  | Rekap Kehadiran Guru dari nol — SUPERSEDED oleh T176, JANGAN dieksekusi.                                                                                        | 2026-08-06      | —                | `06-Features/tasks/T116-rekap-kehadiran-guru-baru.md`                             |
| T138 — Rekap Kehadiran Ekstrakurikuler (Per Kelas/Per Ekstra/Semua)      | BELUM DIKERJAKAN | Sedang                                | Rekap Kehadiran Ekstrakurikuler per Kelas/Ekstra/Semua — fitur baru total, saat ini nol rekap untuk ekstrakurikuler.                                            | 2026-08-08      | —                | `06-Features/tasks/T138-rekap-kehadiran-ekstrakurikuler.md`                       |
| T215 — Hapus Model & UI Lama (Schedule jam_mengajar, BlockWeekRange dkk) | BELUM DIKERJAKAN | Tinggi (destruktif)                   | Hapus model & UI lama (Schedule/BlockWeekRange dkk) — DESTRUKTIF, wajib konfirmasi eksplisit user + backup sebelum eksekusi.                                    | 2026-08-17      | —                | `06-Features/tasks/T215-hapus-model-lama-schedule-blockweekrange-jampelajaran.md` |
| T227 — Migrasi Data: Bersihkan kelasId Siswa Nonaktif Lama (Retroaktif)  | BELUM DIKERJAKAN | Tinggi (bug nyata, destruktif-update) | Migrasi retroaktif kelasId siswa nonaktif lama + filter pengaman rekap — DESTRUKTIF-UPDATE production, wajib dry-run+backup dulu.                               | 2026-08-20      | —                | `06-Features/tasks/T227-migrasi-retroaktif-siswa-nonaktif-lama.md`                |
| T244 — "Biodata Murid" Wali Kelas: Tampilkan Foto Siswa                  | BELUM DIKERJAKAN | Sedang                                | Tampilkan foto siswa di Biodata Murid Wali Kelas — bug 3 lapis dikonfirmasi (backend/tipe/frontend semua belum handle field foto).                              | 2026-08-25      | —                | `06-Features/tasks/T244-biodata-murid-wali-kelas-foto.md`                         |

---

## Status per App (kode aktual, ringkas)

### apps/api (NestJS)
Semua modul Fase 1 + Fase 2 + Ekstrakurikuler CRUD sudah selesai: Auth, Core (Kampus/Kelas/Jurusan/Siswa/Guru/Schedule), Cards, Import, Calendar, Attendance (+ lock otomatis), Permits, Piket Schedules, Kiosks, Photos, Activity Log, Realtime, Queue, Teaching Sessions/Semesters/Block-Week-Ranges/Teacher-Permits/Schedule-Resolver (Fase 2), TV Piket, Ekstra-Absensi (sesi/kelompok/presensi CRUD lengkap).

Yang belum: T088 (UUID, sengaja ditunda), T098 (auto-lock izin tidak kembali).

### apps/web (Next.js Admin)
Semua menu utama selesai: Login, Dashboard Kampus/Kelas/Jurusan/Siswa/Guru/Kartu, Import, Upload Foto (kini split Siswa/Guru — T103), Kalender, Jadwal Piket, Log Aktivitas, Rekap Kehadiran, Dashboard Piket (sidebar berkelompok — T099a), Dashboard TV, Manajemen Akun (4 section — T104), Cetak Struk Izin (route handler internal), Dashboard Guru Jurnal + Admin Jurnal + Wali Kelas (Fase 2), Sidebar Admin 6 grup accordion (T103), Ekstrakurikuler (pendaftaran, monitoring, dashboard pembina lengkap dengan CRUD sesi/kelompok).

Yang belum: T060 (audit visual belum pernah dijalankan penuh).

### apps/kiosk (Next.js Gerbang)
Semua selesai: tap capture, offline buffer+sync, feedback screen, foto/avatar fallback, lock otomatis, dashboard guru 5 terbaru, redesign tampilan + watermark (T089).

---

## Keputusan Arsitektur Terbaru (belum masuk 11-Decisions.md ADR formal)

- **Dev/Production dipisah total secara fisik** (2 folder, 2 DB Docker, 2 set port) — lihat `10-Environment.md` untuk topologi lengkap. Auto-deploy via git post-commit hook dari `dev` branch ke folder production.
- **CRUD sesi/kelompok/presensi ekstrakurikuler** (2026-08-04): sesi hanya bisa dihapus/ganti-kelompok kalau belum ada absen tersimpan (`sudahAdaAbsen: false`); ganti kelompok mereset jam + hapus-buat-ulang semua baris absen; kelompok hanya bisa dihapus kalau tidak dipakai sesi apapun.

---

## Insiden & Pelajaran Penting (JANGAN diulang)

- **2026-07-30 — Database production wipe total.** Tidak ada backup, direkonstruksi dari CSV+dump legacy. Backup 3-lapis sudah dipasang sejak itu. **WAJIB baca memory `feedback_insiden_database_wipe_2026-07-30` sebelum operasi destruktif apapun ke DB.**
- **Wikilink Obsidian (format `[[path|text]]`) terbukti rapuh** (2x broken-link massal, lihat `_archive/_REPAIR-LOG.md`) — vault ini sekarang TIDAK memakai wikilink sama sekali, semua referensi antar file pakai path teks biasa.

---

## Dokumen Referensi (jarang berubah, tetap aktif di root)

| Dokumen | Isi |
|---|---|
| `01-Overview.md` | Latar belakang, visi, skala target |
| `02-Tech-Stack.md` | Stack teknis lengkap |
| `03-User-Roles.md` | Role & permission |
| `04-Database-Schema.md` | Skema database (cross-check ke `apps/api/prisma/schema.prisma` untuk detail terkini) |
| `09-Conventions.md` | Konvensi kode & git |
| `10-Environment.md` | Environment & infra, termasuk topologi dev/production |
| `11-Decisions.md` | ADR-001 s/d ADR-025+, keputusan arsitektur mengikat |
| `06-Features/design-system/MASTER.md` | Brief visual (warna, tipografi, komponen) — WAJIB dibaca sebelum kerja UI apapun |
| `06-Features/tasks/T0xx-*.md` | Detail task individual, dibaca sesuai kebutuhan (bukan semua sekaligus) |

**Sumber kebenaran teknis paling akurat selalu kode aktual** (`/home/anunnaki/Documents/APP SMK/AbsenSI`), bukan dokumen manapun di vault ini — dokumen 05/07/08 masih draft pra-coding yang belum disinkronkan.

---
tags: [absensi, status]
updated: 2026-08-04
---

# STATUS.md — Satu-satunya tempat cek status AbsenSI

> File ini MENGGANTIKAN 12-Status.md, 13-Backlog.md, TASKS-FASE-1.md, TASKS-FASE-2-JURNAL.md,
> TASKS-POLISH-1/2/3.md, dan S.md — semuanya sudah diarsipkan ke `_archive/` (isi lengkap
> narasi implementasi tiap task masih ada di sana, tidak dihapus, cuma tidak lagi jadi
> file aktif yang perlu dibaca tiap sesi).
>
> Cara pakai file ini: cek tabel "Task Aktif / Belum Dikerjakan" dulu untuk tahu apa yang
> masih perlu dikerjakan. Task yang sudah ✅ selesai TIDAK didetailkan di sini lagi — kalau
> perlu histori implementasi lengkapnya, baca file arsip yang dirujuk di kolom "Sumber".

---

## Ringkasan Fase

| Fase | Status | Detail |
|---|---|---|
| Fase 1 — Absensi Gerbang | ✅ Selesai (31/32, T035 PDF sengaja ditunda) | `_archive/TASKS-FASE-1.md` |
| Polish Batch 1 | ✅ Selesai (8/8) | `_archive/TASKS-POLISH-1.md` |
| Polish Batch 2 | ✅ Selesai (8/9, T035 masih ditunda) | `_archive/TASKS-POLISH-2.md` |
| Fase 2 — Dashboard Guru Jurnal + turunannya (T038-T089) | ✅ Selesai (51/52 — hanya T060 belum, lihat tabel di bawah) | `_archive/TASKS-FASE-2-JURNAL.md` |
| Polish Batch 3 — Dashboard Piket (T090-T095) | ✅ **Selesai semua, terverifikasi ke kode 2026-08-04** (T090-T095 dicek satu-satu langsung ke source, semua ada implementasinya) | `_archive/TASKS-POLISH-3.md` |
| Ekstrakurikuler + turunannya (T096-T105) | ✅ Selesai semua termasuk CRUD sesi/kelompok/presensi (2026-08-04) | lihat `06-Features/tasks/T096-T105` + histori chat |

**Kode:** semua fase di atas SUDAH ter-commit ke git (`/home/anunnaki/Documents/APP SMK/AbsenSI`, branch `dev`) dan sudah di-deploy ke production (lihat `10-Environment.md` untuk topologi dev/production). Working tree bersih per 2026-08-04.

---

## 🔴 Task Aktif / Belum Dikerjakan (WAJIB baca sebelum eksekusi apapun)

> **Audit menyeluruh dilakukan 2026-08-04** — setiap task di tabel ini diverifikasi LANGSUNG ke kode (baca file, cek implementasi nyata), bukan cuma percaya dokumen lama. T090-T095 dan T055 yang sebelumnya ditulis "belum dikerjakan" di draf STATUS.md pertama TERBUKTI SALAH — keduanya sudah selesai. Tabel ini sudah dikoreksi.

| Task | Prioritas | Status sebenarnya (terverifikasi ke kode) | Ref |
|---|---|---|---|
| T060 — Audit Visual Menyeluruh Dashboard Guru/Admin Jurnal | Sedang | **Belum dikerjakan** (terkonfirmasi tidak ada laporan/artefak audit sistematis 11 halaman) | `06-Features/tasks/T060` — **belum ada file, cek TASKS-FASE-2-JURNAL arsip Blok 9** |
| T088 — Migrasi UUID Tabel Sensitif | Rendah | **Belum dikerjakan, SENGAJA ditunda** — terkonfirmasi semua model masih `Int autoincrement`, tidak ada kolom UUID. Tunggu instruksi eksplisit user | `06-Features/tasks/T088-uuid-tabel-sensitif.md` |
| T098 — Auto-Lock Izin Tidak Kembali | Tinggi (risiko) | **Belum dikerjakan** — terkonfirmasi: tidak ada job auto-lock baru, enum `StatusKembali` masih `belum/sudah/pulang` (belum ada `tidak_kembali`), dan section "Perlu Ditinjau" MASIH ADA PENUH di kode (bertentangan langsung dengan spec task yang minta section itu dihapus total). Amandemen KEDUA ADR-017, mengubah RBAC lock — butuh konfirmasi eksplisit sebelum eksekusi | `06-Features/tasks/T098-auto-lock-izin-tidak-kembali.md` |
| T101 — Validasi Jam Pulang Sesuai Jadwal Kelas | Blocked | 🔴 **BLOCKED, terkonfirmasi** — `tap()` di `attendance.service.ts` tidak ada validasi jadwal kelas sama sekali, tidak ada enum `TapResult` baru untuk ini. Data jadwal `jam_mengajar` belum lengkap. JANGAN eksekusi sebelum prasyarat terpenuhi | `06-Features/tasks/T101-validasi-jam-pulang-jadwal-kelas.md` |

**T055 (Loopback Rekap) dan T090-T095 (Polish Batch 3) SEMUANYA SUDAH SELESAI** — dihapus dari tabel task aktif di atas setelah verifikasi langsung ke kode 2026-08-04:
- T055: `attendance-report.service.ts` (`resolveDateRange()`) DAN `apps/web/.../rekap/rekap-view.tsx` (dropdown Tahun Ajaran/Semester + validasi rentang tanggal) keduanya sudah lengkap, backend maupun frontend.
- T090-T095: dicek satu-satu ke source (komentar kode eksplisit menyebut nomor task-nya) — Riwayat Izin dengan filter rentang, exclude PKL dari board, menu Input Izin/Sakit mandiri, filter Kelas-mengikuti-Jurusan, fix bug highlight sidebar, form modern — semua ada.

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

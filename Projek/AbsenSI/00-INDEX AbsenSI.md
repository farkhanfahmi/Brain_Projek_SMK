---
tags: [absensi, index, rfid, sekolah]
status: in-progress
updated: 2026-07-21
---

# AbsenSI — Index Proyek

> Sistem Absensi RFID untuk Siswa & Guru SMK. Proyek perintis dari ekosistem aplikasi sekolah yang lebih besar. Dieksekusi via Claude Code, dirancang via Claudian di Obsidian.

---

## 📋 Status Singkat

| Item | Detail |
|---|---|
| Phase | **Fase 1 selesai** (31/32 task) + **Polish Batch 1 selesai** (8/8) + **Polish Batch 2 hampir selesai** (8/9, PDF sengaja dilewati) |
| Repo | Monorepo Turborepo — `/home/anunnaki/Documents/APP SMK/AbsenSI` |
| Stack | TypeScript full-stack — lihat [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack]] |
| Skala target | ±2.500 siswa, 100+ guru/karyawan, 1 sekolah |

---

## 🗂️ Navigasi

### Dokumen Inti
- [[Projek/AbsenSI/01-Overview|01 — Overview]] — latar belakang, visi, scope, fase pengembangan
- [[Projek/AbsenSI/02-Tech-Stack|02 — Tech Stack]] — stack, arsitektur monorepo, alasan tiap pilihan
- [[Projek/AbsenSI/03-User-Roles|03 — User Roles]] — role pengguna aplikasi (bukan tim dev)
- [[Projek/AbsenSI/04-Database-Schema|04 — Database Schema]]
- [[Projek/AbsenSI/05-API-Endpoints|05 — API Endpoints]] — draft awal pra-coding, endpoint aktual lebih lengkap (lihat kode `apps/api/src/*/​*.controller.ts`)

### Fitur (`06-Features/`)
> Catatan: dokumen fitur di bawah ini sebagian besar masih draft awal dari fase perencanaan (sebelum coding). Implementasi aktual sudah berjalan dan kadang berbeda detail — rujuk `TASKS-FASE-1.md`/`TASKS-POLISH-2.md` untuk status & detail implementasi terkini, dokumen di bawah untuk konteks desain awal.
- [[Projek/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang]] (Fase 1 — implementasi selesai)
- [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|Dashboard Guru: Jurnal Mengajar & Absensi Mapel]] (Fase 2 — prioritas #1, masih tahap interview)
- [[Projek/AbsenSI/06-Features/dashboard-petugas-kartu|Dashboard Petugas Kartu]] (Fase 2 — prioritas #2, ✅ final)
- [[Projek/AbsenSI/06-Features/tv-piket|TV Piket]] (Fase 2 — prioritas #3, ✅ final, siap eksekusi T068-T070)
- [[Projek/AbsenSI/06-Features/absensi-kelas-mapel|Absensi Kelas & Mapel]] (⚰️ ARSIP — digantikan dashboard-guru-jurnal.md)
- [[Projek/AbsenSI/06-Features/manajemen-kartu|Manajemen Kartu RFID]] (selesai)
- [[Projek/AbsenSI/06-Features/akun-guru|Akun & Riwayat Kehadiran Guru]] (selesai)
- [[Projek/AbsenSI/06-Features/dashboard-tv|Dashboard TV Realtime]] (selesai)
- [[Projek/AbsenSI/06-Features/notifikasi-ortu|Notifikasi Orang Tua]] (Fase 3 — jalur disiapkan, belum dibangun)
- [[Projek/AbsenSI/06-Features/dashboard-piket|Dashboard Piket]] (selesai, direstrukturisasi ulang di T031)
- [[Projek/AbsenSI/06-Features/rekap-kehadiran|Rekap Kehadiran]] (Fase 1 admin selesai; export PDF/T035 sengaja ditunda)
- [[Projek/AbsenSI/06-Features/kalender-pendidikan|Kalender Pendidikan]] (selesai)
- [[Projek/AbsenSI/06-Features/design-system/MASTER|Design System]] — brief visual EzMart-style (warm beige/oranye) dipakai `apps/web` & `apps/kiosk`
- [[Projek/AbsenSI/06-Features/migrasi-database-lama|Migrasi Database Lama]] — perbandingan struktur lama (Laravel) vs baru, field yang hilang & keputusan migrasi (T061-T063)
- [[Projek/AbsenSI/06-Features/plotting-siswa-kelas|Plotting Siswa ke Kelas]] — penempatan siswa baru (paste-NISN), kenaikan kelas massal, pindah individual, tandai keluar (T071-T074)
- [[Projek/AbsenSI/06-Features/ekstrakurikuler|Ekstrakurikuler]] — pendaftaran publik siswa (selesai & live) + monitoring admin (selesai); absensi oleh pembina (T096 model dasar + T102 amandemen besar, keduanya selesai 2026-07-31 — lihat T102)
- [[Projek/AbsenSI/06-Features/tasks/T097-sidebar-guru-berkelompok|T097 — Sidebar Guru Berkelompok]] — ✅ **selesai & diverifikasi 2026-07-31** (accordion Guru/Ekstrakurikuler + link tunggal Wali Kelas, live di production). **Piket TIDAK termasuk** — didiskusikan tapi sengaja ditunda
- [[Projek/AbsenSI/06-Features/tasks/T098-auto-lock-izin-tidak-kembali|T098 — Auto-Lock Izin Tidak Kembali]] — amandemen KEDUA ADR-017 (job terjadwal tengah malam), hapus section Perlu Ditinjau, tombol "Tidak Kembali" manual — belum dikerjakan, risiko tinggi (ubah RBAC lock)
- [[Projek/AbsenSI/06-Features/tasks/T099-polish-ui-dashboard-piket|T099 — Polish UI Dashboard Piket]] — ✅ **selesai & live di production 2026-07-31** (sidebar grup, fix filter Direktori Siswa, board rename "Siswa Belum Hadir", tombol verifikasi Riwayat Izin, hapus badge dobel)
- [[Projek/AbsenSI/06-Features/tasks/T100-rename-tidak-tap-pulang|T100 — Rename "Tidak Tap Pulang" → "Tidak Absen Pulang"]] — ✅ **selesai & diverifikasi 2026-07-30** (rename menyeluruh UI+kode+endpoint, lihat detail di file task)
- [[Projek/AbsenSI/06-Features/tasks/T101-validasi-jam-pulang-jadwal-kelas|T101 — Validasi Tap Pulang Sesuai Jadwal Kelas]] — 🔴 BLOCKED, butuh data jadwal jam_mengajar lengkap dulu (baru 6 baris dummy) + banyak pertanyaan desain belum terjawab, JANGAN eksekusi sebelum prasyarat terpenuhi
- [[Projek/AbsenSI/06-Features/tasks/T102-ekstra-kelompok-sesi-jadwal|T102 — Dashboard Pembina Ekstra Lengkap]] — ✅ **selesai & diverifikasi 2026-07-31** (amandemen besar T096: kelompok/sesi paralel opsional, jadwal hari+jam, auto-generate EkstraSesi harian, 3 submenu Daftar Peserta/Presensi/Setting, sekalian tutup gap route group `(pembina-ekstra)/` yang belum pernah dibuat T096 — lihat detail di file task)
- [[Projek/AbsenSI/06-Features/tasks/T103-sidebar-admin-berkelompok|T103 — Sidebar Admin Berkelompok]] — 6 grup accordion (konsisten pola T097), pisah Upload Foto jadi Foto Siswa/Foto Guru terpisah — belum dikerjakan
- [[Projek/AbsenSI/06-Features/tasks/T104-akun-section-berbasis-role|T104 — Manajemen Akun Berbasis Section]] — ✅ **selesai & diverifikasi 2026-07-31** (4 tab Admin/Piket/Guru/Pembina Ekstra, dropdown Role cuma di tab Admin, live di production)
- [[Projek/AbsenSI/06-Features/tasks/T105-workflow-dev-production-terpisah|T105 — Pisahkan Dev/Production (1 Mesin)]] — 🔴 PRIORITAS TINGGI pasca-insiden wipe: 136 file (T072-T104) belum di-commit ke GitHub sama sekali, DB+folder+branch dev harus dipisah dari production — belum dikerjakan

### 🎯 Task Execution
- [[Projek/AbsenSI/TASKS-FASE-1|TASKS-FASE-1]] — 32 task bertahap (T001–T028e), **31/32 selesai**
- [[Projek/AbsenSI/TASKS-POLISH-1|TASKS-POLISH-1]] — 8 task polish batch 1 (P001–P008), **semua selesai**
- [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2]] — 9 task polish batch 2 (T029–T037), **8/9 selesai** (T035/Rekap PDF sengaja dilewati, tunggu diskusi format sebelum dikerjakan)
- [[Projek/AbsenSI/TASKS-POLISH-3|TASKS-POLISH-3]] — 6 task perbaikan dashboard piket batch 3 (T090–T095), **0/6 belum dikerjakan** — dieksekusi kapan pun sesuai kebutuhan
- [[Projek/AbsenSI/TASKS-FASE-2-JURNAL|TASKS-FASE-2-JURNAL]] — 14 task Dashboard Guru Jurnal (T038–T051), **0/14**, siap eksekusi

### Proses & Keputusan
- [[Projek/AbsenSI/07-User-Flows|07 — User Flows]] — draft awal, belum diperbarui pasca-implementasi
- [[Projek/AbsenSI/08-UI-UX-Guidelines|08 — UI/UX Guidelines]] — draft awal; brief visual final ada di `06-Features/design-system/`
- [[Projek/AbsenSI/09-Conventions|09 — Conventions]] — draft awal
- [[Projek/AbsenSI/10-Environment|10 — Environment]] — draft awal
- [[Projek/AbsenSI/11-Decisions|11 — Decisions (ADR)]] — **terpelihara aktif**, ADR-001 s/d ADR-025
- [[Projek/AbsenSI/12-Status|12 — Status]] — ringkasan progres per modul
- [[Projek/AbsenSI/13-Backlog|13 — Backlog & Roadmap Fase]]
- [[Projek/AbsenSI/14-Debug-Log|14 — Debug Log]] — belum dipakai aktif, bug/insiden dicatat inline di TASKS-*.md
- [[Projek/AbsenSI/15-Deployment-Guide|15 — Deployment Guide]] — draft awal, belum ada deployment nyata

### Riwayat & Catatan Lepas
- [[Projek/AbsenSI/Catatan Spontan|Catatan Spontan]] — catatan bebas non-terstruktur
- [[Projek/AbsenSI/_VAULT-STRUCTURE|_VAULT-STRUCTURE]] — peta struktur file vault ini
- [[Projek/AbsenSI/_REPAIR-LOG|_REPAIR-LOG]] — riwayat perbaikan struktur/link vault

### Khusus Tim (`_claudian/`)
> Dokumen ini menggambarkan rencana kerja tim 3 developer paralel dari fase perencanaan awal. Eksekusi aktual sejauh ini berjalan sebagai sesi tunggal Claude Code (lihat TASKS-*.md), bukan multi-developer paralel — anggap dokumen ini sebagai referensi workflow yang belum tentu dipakai persis begini.
- [[Projek/AbsenSI/_claudian/team|team.md]] — pembagian modul, kontak, area tanggung jawab
- [[Projek/AbsenSI/_claudian/project-context|project-context.md]] — quick-attach awal sesi
- [[Projek/AbsenSI/_claudian/discussion-log|discussion-log.md]] — buffer keputusan sebelum masuk ADR
- [[Projek/AbsenSI/_claudian/workflow-multi-dev|workflow-multi-dev.md]] — workflow Claudian khusus tim 3 orang

---

## ⚠️ Catatan Penting

Proyek ini **masih dirancang di vault pribadi**. Saat siap kolaborasi, seluruh folder ini akan dipindah ke vault khusus tim + repo GitHub terpisah. Jangan buat keputusan final yang terlalu vault-spesifik sampai migrasi itu terjadi.

Sumber kebenaran teknis untuk detail implementasi terkini adalah **kode aktual** (`/home/anunnaki/Documents/APP SMK/AbsenSI`) dan **TASKS-FASE-1.md / TASKS-POLISH-1.md / TASKS-POLISH-2.md** — dokumen 05-10 & 06-Features/ di bawah ini sebagian besar draft pra-coding yang belum disinkronkan detail per baris.

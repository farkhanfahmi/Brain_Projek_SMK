---
tags: [absensi, environment]
updated: 2026-08-04
---

# 10 — Environment

← lihat STATUS.md untuk ringkasan progres, 11-Decisions.md untuk ADR

> **Sudah LIVE**, bukan lagi rencana. Server sekolah fisik BELUM ada — 1 mesin dev
> (`anunnaki`, workstation ini) berperan sebagai dev DAN production sementara sampai
> server sekolah sungguhan siap (lihat bagian "Migrasi ke server sekolah" di bawah).

## Topologi Dev/Production (T105, sejak 2026-07-31)

2 lingkungan terpisah total di MESIN YANG SAMA:

### DEV
- Path: `/home/anunnaki/Documents/APP SMK/AbsenSI` — branch `dev`.
- Port: web 3100, api 3101, kiosk 3102.
- Database: Docker `absensi-mysql-1` (host port 3307) — boleh direset/rusak bebas, TIDAK ada data sekolah asli.
- Redis: Docker `absensi-redis-1` (port 6379).
- Jalankan: `./scripts/dev-start.sh all` (atau `web`/`api`/`kiosk` individual).

### PRODUCTION
- Path: `/home/anunnaki/Documents/APP SMK/AbsenSI-production` — branch `main` SELALU.
- Port: web 3000, api 3001, kiosk 3002 (tidak berubah dari sebelum T105 — kiosk fisik gerbang & bookmark staff tidak perlu dikonfigurasi ulang). Diakses via `http://10.10.10.198:3000` dst.
- Database: Docker `absensi-mysql-prod` (host port 3309) — **data sekolah ASLI**, dimulai kosong pasca insiden wipe 2026-07-30.
- Redis: Docker `absensi-redis-prod` (port host 6380).
- Jalankan: `./scripts/start-production.sh` (install → migrate deploy → prisma generate → build → restart. Build gagal = exit 1 sebelum service lama dimatikan, production tidak pernah mati gara-gara deploy baru gagal build).

### Auto-deploy (git hook)
- `.git/hooks/post-commit` di folder DEV: commit ke branch `dev` otomatis → push ke GitHub (`origin`) + push ke `production` remote (`dev:main`, path lokal `file://`) → trigger `start-production.sh` di background.
- Folder production: `git config receive.denyCurrentBranch updateInstead` — working tree ter-update otomatis saat menerima push.
- **Trade-off disengaja:** restart production full otomatis tanpa jeda cek manual — risiko kode belum teruji langsung live, dipilih demi kemudahan workflow "push dari dev".

## Database Engine — MySQL 8 (ADR-011)

Docker (`mysql:8`, BUKAN MariaDB). Host tidak punya binary `mysqldump` — semua akses dump lewat `docker exec <container> mysqldump ...`.

## Backup — 3 Lapis (grandfather-father-son), sejak 2026-07-31 pasca insiden wipe

- **Harian**: `/home/anunnaki/scripts/backup-absensi.sh` → `absensi-mysql-prod` → NVMe `/media/anunnaki/DataNvme/backups/absensi/`, retensi 30 hari, cron 02:00.
- **Mingguan/Bulanan**: `/home/anunnaki/scripts/backup-absensi-weekly-monthly.sh` → menyalin dump harian terbaru (bukan dump ulang) ke HDD `/media/anunnaki/New Volume/backups/absensi/{weekly,monthly}/`, retensi 84/365 hari, cron 02:05.
- 3 disk fisik terpisah (sistem, NVMe, HDD) sengaja dipakai supaya kegagalan 1 disk tidak menghapus backup bersama sumber data.
- **Cara restore** (belum pernah diuji end-to-end):
  ```bash
  gunzip -c /media/anunnaki/DataNvme/backups/absensi/absensi_TANGGAL.sql.gz | \
    docker exec -i absensi-mysql-prod mysql -u root -ppassword absensi_db
  ```

## Migrasi ke Server Sekolah (nanti, belum diputuskan jadwalnya)

Server sekolah `git clone` branch `main` dari GitHub, restore dump terbaru dari salah satu lokasi backup di atas, set `.env` production baru sesuai infra server itu. Setelah stabil, folder `AbsenSI-production` di mesin ini bisa dihentikan (**JANGAN dihapus** sebelum server baru dikonfirmasi jalan baik oleh user).

## Master Data (Core) — Tetap di Dalam AbsenSI untuk Sekarang (ADR-014)

Data siswa/guru/jadwal tetap modul internal AbsenSI, belum diekstrak jadi servis terpisah — ditunda sampai aplikasi ekosistem ke-2 benar-benar konkret.

## ❓ Open Questions (belum diputuskan)

- [ ] Spesifikasi hardware server sekolah fisik (RAM/CPU/storage) — sizing menyusul saat provisioning benar-benar dimulai
- [ ] Domain/subdomain untuk akses dari luar jaringan sekolah (kalau dibutuhkan)
- [ ] Prosedur akses darurat (kredensial, SSH key, runbook) — saat ini cuma 1 orang (Fahmi) yang pegang akses, tidak ada backup person

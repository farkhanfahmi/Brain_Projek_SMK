---
tags: [absensi, environment]
updated: 2026-08-25
---

# 10 — Environment

← lihat STATUS.md untuk ringkasan progres, 11-Decisions.md untuk ADR

> **Sudah LIVE**, bukan lagi rencana. Sejak 2026-08-25 topologi BERUBAH: dev dan
> production TIDAK lagi di mesin yang sama.
>
> - **DEV → Windows** (mesin ini): `C:\ProjekSMK\AbsenSI`, branch `dev`.
>   Docker: `absensi-mysql-1` (root/password, host port 3307) + `absensi-redis-1` (6379).
>   Database dev BARU (seed testing saja) — dibuat kosong saat migrasi ke Windows,
>   sesuai prinsip "dev boleh rusak bebas".
> - **PRODUCTION → Linux** (`anunnaki`): path `/home/anunnaki/Documents/APP SMK/AbsenSI-production`,
>   branch `main`, port web 3000 / api 3001 / kiosk 3002, MySQL prod port 3309, Redis 6380,
>   dikelola systemd (T117). Diakses dari Windows via **VPN + SSH** — semua operasi production
>   dari Windows WAJIB lewat SSH (`systemctl status` dulu sebelum restart apa pun).
>
> Riwayat topologi lama (dev+production 1 mesin Linux, auto-deploy post-commit hook)
> tetap relevan untuk memahami insiden & aturan di bawah — baca dengan konteks itu.

## Topologi Dev/Production

### DEV — Windows (mesin ini)
- Path: `C:\ProjekSMK\AbsenSI` — branch `dev`. Node + pnpm native Windows.
- Port: web 3000, api 3001, kiosk 3002 (standar repo; tidak ada folder produksi terpisah di Windows).
- Infrastruktur: Docker Desktop (WSL2), compose file di root repo — `docker compose up -d`.
- Jalankan: `pnpm turbo run dev` (script .sh lama butuh adaptasi bash/git-bash).

### PRODUCTION — Linux (`anunnaki`)
- Path: `/home/anunnaki/Documents/APP SMK/AbsenSI-production` — branch `main` SELALU.
- Port: web 3000, api 3001, kiosk 3002 (tidak berubah dari sebelum T105 — kiosk fisik gerbang & bookmark staff tidak perlu dikonfigurasi ulang). Diakses via `http://10.10.10.198:3000` dst.
- Database: Docker `absensi-mysql-prod` (host port 3309) — **data sekolah ASLI**, dimulai kosong pasca insiden wipe 2026-07-30.
- Redis: Docker `absensi-redis-prod` (port host 6380).
- Jalankan: `./scripts/start-production.sh` (install → migrate deploy → prisma generate → build → restart via systemd, lihat bagian "Process Management" di bawah — T117. Build gagal = exit 1 sebelum service lama dimatikan, production tidak pernah mati gara-gara deploy baru gagal build).

### Auto-deploy (git hook) — ⚠️ TIDAK LAGI BEKERJA DARI WINDOWS (sejak migrasi topologi 2026-08-25)

- `.git/hooks/post-commit` di folder DEV: commit ke branch `dev` otomatis → push ke GitHub (`origin`) + push ke `production` remote (`dev:main`, path lokal `file://`) → trigger `start-production.sh` di background.
- Folder production: `git config receive.denyCurrentBranch updateInstead` — working tree ter-update otomatis saat menerima push.
- **Trade-off disengaja (saat masih 1 mesin):** restart production full otomatis tanpa jeda cek manual — risiko kode belum teruji langsung live, dipilih demi kemudahan workflow "push dari dev".

**Kondisi SEKARANG (dev di Windows, production di Linux, 2 mesin terpisah)**: remote
`production` di hook itu adalah path `file://` lokal (`/home/anunnaki/Documents/APP SMK/AbsenSI-production`)
— **Windows tidak punya akses filesystem ke path itu**, jadi baris `git push production dev:main`
di hook **GAGAL DIAM-DIAM atau tidak pernah ter-trigger sama sekali** kalau hook itu masih ada
salinannya di clone Windows. Auto-deploy dev→production **PUTUS TOTAL** sejak topologi 2 mesin
ini berlaku — commit dari Windows `git push origin dev` HANYA sampai ke GitHub, **TIDAK PERNAH**
mencapai production secara otomatis. Insiden nyata: 2026-08-29, 5 commit (T238–T262) dari Windows
sudah beberapa hari di GitHub tapi production masih menjalankan kode lama sampai ketahuan manual.

#### 🔁 Prosedur WAJIB Sinkronisasi dev(Windows) → production(Linux) — Sampai Ada Otomatisasi Baru

**Setelah commit+push dari Windows (`git push origin dev`), JANGAN anggap production ikut ter-update.**
Sesi Claude Code manapun (Windows atau Linux) yang baru saja push/pull perubahan `dev` WAJIB
jalankan urutan ini dari mesin Linux (`anunnaki`) — via SSH kalau dikerjakan dari sesi Windows:

1. Di folder dev Linux (`/home/anunnaki/Documents/APP SMK/AbsenSI`, kalau ada) atau lewat SSH:
   ```bash
   git fetch origin && git merge --ff-only origin/dev   # pastikan fast-forward, BUKAN divergent
   ```
   Kalau bukan fast-forward (ada commit lokal Linux yang belum di-push) — **STOP**, selesaikan
   divergence dulu (lihat sesi mana yang seharusnya jadi sumber kebenaran), jangan force apa pun.
2. Cek migration baru sejak deploy production terakhir — **WAJIB** untuk migration destruktif
   (`DROP TABLE`/`DROP COLUMN`/`TRUNCATE`), ikuti protokol backup manual di bagian
   "🔒 Aturan WAJIB Sebelum Commit di Dev yang Mengandung Migration DROP/ALTER Destruktif" di bawah
   SEBELUM lanjut ke langkah 3 — additif (`ADD COLUMN`/`ADD TABLE`) aman langsung lanjut.
3. Push working tree dev ke folder production (memicu `receive.denyCurrentBranch updateInstead`):
   ```bash
   git push production dev:main
   ```
4. Trigger build+migrate+restart (skrip yang SAMA dipakai hook lama):
   ```bash
   cd "/home/anunnaki/Documents/APP SMK/AbsenSI-production" && ./scripts/start-production.sh
   ```
5. **VERIFIKASI eksplisit, jangan asumsikan skrip berhasil dari exit code semata** — skrip ini
   memanggil `sudo -n systemctl restart absensi-prod-{api,web,kiosk}`, yang BISA gagal diam-diam
   kalau dijalankan dari sesi/proses yang tidak punya akses ke sudoers NOPASSWD yang sama
   (`/etc/sudoers.d/absensi-systemd`) — terbukti terjadi 2026-08-29 (skrip lapor "confirmed active"
   tapi ternyata itu log LAMA dari deploy sebelumnya, service sebenarnya BELUM restart). Cek:
   ```bash
   systemctl status absensi-prod-api absensi-prod-web absensi-prod-kiosk | grep "Active:"
   ```
   Bandingkan timestamp "active since" dengan waktu SEKARANG — kalau bukan barusan, restart
   BELUM benar-benar terjadi meski log skrip terlihat sukses. Kalau butuh restart manual (sudo
   interaktif), **itu aksi yang harus dikonfirmasi/dijalankan user sendiri**, bukan dijalankan
   sepihak oleh agent tanpa otorisasi eksplisit di sesi itu.

**Untuk agent yang bekerja DARI Windows**: kalian tidak bisa menjalankan langkah 3-5 di atas
langsung (tidak ada akses filesystem/systemd ke mesin Linux). Setelah `git push origin dev`,
**WAJIB beri tahu user secara eksplisit** bahwa production BELUM ter-update dan langkah sinkronisasi
manual di atas perlu dijalankan dari/lewat sesi di mesin Linux — JANGAN diam-diam menganggap
tugas selesai hanya karena push ke GitHub berhasil.

## Process Management — Systemd (T117, sejak 2026-08-06)

**Insiden pemicu**: API production mati sejak reboot mesin 2026-08-06 (proses Node
dijalankan via `nohup`/`setsid` manual, tidak ada watchdog) — 646 tap kiosk menumpuk
di buffer offline berjam-jam sebelum ketahuan.

- **Production SAJA** yang didaftarkan systemd (`absensi-prod-api`, `absensi-prod-web`,
  `absensi-prod-kiosk`) — dev SENGAJA tetap manual via `dev-start.sh`/`dev-stop.sh`
  (keputusan 2026-08-06: auto-restart saat development bisa menyembunyikan crash yang
  harusnya jadi sinyal bug ke developer).
- Unit file template ada di repo: `scripts/systemd/*.service` — salinan untuk provisioning
  ulang server, sumber kebenaran AKTIF tetap `/etc/systemd/system/` di mesin ini.
- `Restart=on-failure` (bukan `always`) — stop manual (`systemctl stop`, misal maintenance)
  TIDAK langsung dihidupkan ulang systemd. Auto-restart hanya untuk crash/exit tidak normal.
- `StartLimitBurst=5` dalam `StartLimitIntervalSec=300` — crash 5x dalam 5 menit = systemd
  berhenti mencoba restart (butuh `systemctl reset-failed` manual), mencegah crash-loop
  tanpa henti membebani CPU tanpa sinyal jelas ada masalah serius.
- **Instalasi lengkap (langkah demi langkah, semua butuh sudo)**: `scripts/systemd/README.md`.

### Diagnosis Cepat

```bash
sudo systemctl status absensi-prod-api absensi-prod-web absensi-prod-kiosk
sudo journalctl -u absensi-prod-api -n 50 -f    # -f = live tail
sudo systemctl restart absensi-prod-api          # restart manual 1 service
```

### Auto-deploy + sudoers (T105 x T117)

`scripts/start-production.sh` sekarang restart service lewat `sudo -n systemctl restart
absensi-prod-{api,web,kiosk}` (bukan lagi `setsid nohup` manual) — dipanggil TANPA
TTY oleh git hook, jadi butuh baris sudoers NOPASSWD ter-scope KETAT hanya untuk
`systemctl restart/status/start/stop` 3 unit ini (`scripts/systemd/absensi-systemd.sudoers`,
install ke `/etc/sudoers.d/absensi-systemd`). Kalau sudoers belum terpasang, auto-deploy
gagal JELAS di log (`/tmp/absensi-post-commit.log`) dengan pesan sudo — bukan macet
diam-diam menunggu password.

## ⚠️ Insiden T215 Auto-Deploy Migrasi Destruktif ke Production (2026-08-19)

**Kejadian**: commit di `dev` yang berisi migration T215 (`DROP TABLE` 5 tabel jadwal lama:
`block_week_ranges`, `jam_pelajaran_aktivasi`, `jam_pelajaran_option_tingkat`,
`jam_pelajaran_options`, `jam_pelajaran_slots`, plus drop 4 kolom di `kelas`/`schedules`/
`semesters`) otomatis ter-deploy ke **production** lewat hook auto-deploy (baris 32-35 di
atas) — TANPA gerbang konfirmasi, ~4 detik setelah `git commit` di dev. Sesi yang commit
BELUM sempat cek row count/backup sebelum migrasi jalan (baru cek SETELAH selesai — urutan
terbalik dari protokol wajib pasca-insiden wipe 2026-07-30). Ini terjadi WALAUPUN keputusan
eksplisit sebelumnya (sesi diskusi terpisah) sudah bilang "T215 dijalankan di dev dulu saja"
— keputusan itu tidak pernah benar-benar mencegah apa pun karena TIDAK ADA mekanisme yang
menegakkannya secara teknis, hanya niat verbal.

**Dampak**: data inti (siswa/guru/kelas/tap_events/permits/ekstrakurikuler) 100% AMAN, tidak
tersentuh migrasi ini sama sekali. Yang hilang PERMANEN: 1 tabel `jam_pelajaran_options`
berisi 68 baris slot jam pelajaran (fitur "Jam Pelajaran" lama, T158/T159) — dikonfirmasi
TIDAK ADA di backup manapun (harian NVMe terakhir 2026-08-05, backup manual pre-deploy
2026-08-10 & 2026-08-13 — SEMUA dicek langsung isinya, 0 baris `INSERT INTO jam_pelajaran_options`
di semuanya) karena fitur itu baru dibuat SETELAH tanggal backup terakhir yang valid. **Tidak
bisa dipulihkan — perlu input ulang manual** lewat menu Alokasi Waktu yang baru (T203+).

**Temuan kedua (independen dari insiden di atas)**: backup harian NVMe (`backup-absensi.sh`,
cron 02:00) TERNYATA sudah gagal diam-diam selama 14 hari (2026-08-05 s.d. 2026-08-19) —
cron TERBUKTI jalan tiap hari (ada di `journalctl`), tapi TIDAK ADA file baru tercipta dan
`backup.log` juga berhenti tercatat di tanggal yang sama. Dijalankan manual (2026-08-19)
berhasil sempurna (3.1MB, jauh lebih besar dari backup 08-05 yang 378KB) — jadi SCRIPT-nya
sendiri TIDAK rusak. Akar penyebab kegagalan cron 14 hari itu TIDAK BERHASIL diidentifikasi
(semua kondisi teknis — permission, PATH, container, mount — tampak sehat saat diperiksa;
tidak ada MTA terpasang jadi error asli cron tidak pernah tercatat ke mana pun). **Artinya:
selama 14 hari itu, KALAU terjadi insiden apa pun ke production, tidak ada jaring pengaman
backup harian sama sekali** — baru ketahuan karena investigasi insiden T215 ini, bukan
karena ada yang memantau backup secara rutin.

### 🔒 Aturan WAJIB Sebelum Commit di Dev yang Mengandung Migration DROP/ALTER Destruktif

Karena auto-deploy dev→production TIDAK PUNYA gerbang teknis (baris 32-35, trade-off sadar
demi kemudahan), satu-satunya pertahanan HANYA disiplin manual SEBELUM `git commit`:

1. **WAJIB baca isi file migration** (`apps/api/prisma/migrations/<nama>/migration.sql`)
   SEBELUM commit — cari kata `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`. Kalau ADA salah satu
   itu, STOP, jangan commit dulu — lanjut ke langkah 2.
2. **WAJIB backup manual production SEBELUM commit** (bukan andalkan cron harian — insiden
   di atas membuktikan cron bisa gagal diam-diam tanpa siapa pun tahu):
   ```bash
   bash /home/anunnaki/scripts/backup-absensi.sh
   # VERIFIKASI file baru benar-benar muncul dan ukurannya wajar (bandingkan ke backup
   # sebelumnya) SEBELUM lanjut commit — jangan asumsikan sukses dari exit code saja.
   ls -lh /media/anunnaki/DataNvme/backups/absensi/ | tail -3
   ```
3. **WAJIB cek row count tabel yang akan ter-DROP, SEBELUM commit** (lewat DB dev yang
   sudah diverifikasi identik strukturnya, ATAU langsung ke production kalau punya akses):
   ```bash
   docker exec absensi-mysql-prod mysql -u root -ppassword absensi_db \
     -e "SELECT COUNT(*) FROM jam_pelajaran_options;"  # ganti nama tabel sesuai migration
   ```
   Kalau row count > 0 DAN tabel itu belum dipastikan datanya sudah dipindah/tidak
   dibutuhkan lagi — JANGAN commit dulu, konfirmasi ke user dulu secara eksplisit (bukan
   asumsi "kemungkinan sudah tergantikan").
4. **Commit HANYA setelah 1-3 di atas beres** — begitu `git commit` dijalankan di dev,
   TIDAK ADA jeda untuk membatalkan (hook jalan otomatis ~4 detik kemudian, background).
5. **SEGERA setelah commit** (bukan nanti) — cek `/tmp/absensi-post-commit.log` untuk
   konfirmasi migrasi berhasil TANPA error, dan cek row count SEKALI LAGI di production
   untuk konfirmasi hasil sesuai ekspektasi.

**Kalau ragu apakah sebuah migration "aman" atau "destruktif"** — anggap destruktif dan
ikuti langkah di atas. Salah menganggap aman padahal destruktif jauh lebih mahal (data
hilang permanen) daripada backup manual yang ternyata tidak terpakai (biaya beberapa detik).

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

## Monitoring Server Production + Backup Lapis ke-4 Google Drive (T223, sejak 2026-08-19)

Dipicu insiden T215 auto-deploy (section di atas) — 2 celah pemantauan yang ditemukan
lewat insiden itu (backup gagal diam-diam 14 hari, tidak ada gambaran kondisi server)
sekarang tertutup lewat modul `apps/api/src/server-health/`.

- **Halaman monitoring**: `(admin)/server-kesehatan/` — CPU load average, RAM/disk
  usedPercent, status 3 unit systemd (`absensi-prod-{api,web,kiosk}`), info backup
  terakhir (tanggal/ukuran/umur), riwayat alert. Fetch snapshot saat buka halaman +
  tombol Refresh manual (BUKAN Socket.IO — data berubah lambat, beda dari kiosk).
- **Cron cek aktif** — BullMQ job tiap 20 menit (`server-health-check-20min`),
  bandingkan snapshot ke threshold (disk >85%, memory >90%, service manapun bukan
  "active", backup bukan "ok") → insert `ServerHealthAlert` (insert-only). Log
  eksplisit tiap kali job jalan (bahkan semua sehat) — supaya "job ini sendiri
  berhenti jalan" (skenario PERSIS backup-absensi.sh yang memicu task ini) bisa
  dideteksi dari ketiadaan log terbaru, bukan diam-diam gagal lagi 14 hari.
- **Backup manual 1-klik** — tombol "Backup Sekarang" di halaman monitoring, `POST
  /server-health/backup/trigger`, REUSE `backup-absensi.sh` apa adanya via
  `child_process` (JANGAN logic mysqldump ditulis ulang di TypeScript). Async +
  gate anti-dobel-jalan (`BackupTriggerService`, in-memory) — mencegah 2 mysqldump
  jalan bersamaan (klik 2x, atau bentrok cron 02:00).
- **Backup ke Google Drive (lapis ke-4)** — TAMBAHAN di luar 3 lapis existing di atas
  (harian NVMe/mingguan+bulanan HDD), BUKAN pengganti. OAuth scope SEMPIT
  `drive.file` (hanya file yang dibuat aplikasi ini sendiri). Refresh token
  disimpan TERENKRIPSI (AES-256-GCM, `apps/api/src/common/token-encryption.ts`) di
  `GoogleDriveBackupConfig` (singleton, 1 akun untuk seluruh sistem).

### ⚙️ Setup Prasyarat Google OAuth (WAJIB dilakukan manual SEKALI oleh developer)

Fitur backup ke Google Drive TIDAK BISA dites end-to-end sampai langkah ini selesai
— kodenya sudah lengkap (T223), tapi butuh kredensial OAuth sungguhan:

1. Buka [Google Cloud Console](https://console.cloud.google.com/) → buat/pilih project.
2. **APIs & Services → Library** → aktifkan **Google Drive API**.
3. **APIs & Services → Credentials** → **Create Credentials → OAuth client ID**:
   - Application type: **Web application**
   - Authorized redirect URIs: isi PERSIS sama dengan `GOOGLE_DRIVE_REDIRECT_URI`
     di `.env` production (default: `https://<domain-production>/api/server-health/google-drive/callback`)
4. **APIs & Services → OAuth consent screen** — scope yang diminta HARUS
   `https://www.googleapis.com/auth/drive.file` SAJA (least-privilege, bukan `drive`
   penuh — data backup sekolah sensitif).
5. Salin **Client ID** dan **Client Secret** ke `.env` production:
   ```
   GOOGLE_CLIENT_ID="..."
   GOOGLE_CLIENT_SECRET="..."
   GOOGLE_DRIVE_REDIRECT_URI="https://<domain-production>/api/server-health/google-drive/callback"
   SERVER_HEALTH_ENCRYPTION_KEY="$(openssl rand -hex 32)"   # WAJIB, key enkripsi refresh token
   ```
6. Restart service `absensi-prod-api` supaya `.env` baru terbaca.
7. Di halaman `(admin)/server-kesehatan/`, super_admin klik "Hubungkan Google Drive"
   → login akun Google yang akan dipakai → approve consent → tempel link folder
   Drive tujuan (folder harus sudah dibuat manual di Drive akun itu dulu).

**Sampai langkah 5-6 di atas dikerjakan** (isi `.env` production), tombol "Hubungkan
Google Drive" akan gagal dengan error dari `ConfigService.getOrThrow()` — ini
DIHARAPKAN, bukan bug, sampai kredensial sungguhan terisi.

## Migrasi ke Server Sekolah (nanti, belum diputuskan jadwalnya)

Server sekolah `git clone` branch `main` dari GitHub, restore dump terbaru dari salah satu lokasi backup di atas, set `.env` production baru sesuai infra server itu. Setelah stabil, folder `AbsenSI-production` di mesin ini bisa dihentikan (**JANGAN dihapus** sebelum server baru dikonfirmasi jalan baik oleh user).

## Master Data (Core) — Tetap di Dalam AbsenSI untuk Sekarang (ADR-014)

Data siswa/guru/jadwal tetap modul internal AbsenSI, belum diekstrak jadi servis terpisah — ditunda sampai aplikasi ekosistem ke-2 benar-benar konkret.

## ❓ Open Questions (belum diputuskan)

- [ ] Spesifikasi hardware server sekolah fisik (RAM/CPU/storage) — sizing menyusul saat provisioning benar-benar dimulai
- [ ] Domain/subdomain untuk akses dari luar jaringan sekolah (kalau dibutuhkan)
- [ ] Prosedur akses darurat (kredensial, SSH key, runbook) — saat ini cuma 1 orang (Fahmi) yang pegang akses, tidak ada backup person

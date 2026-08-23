# T117 — Infra: Daftarkan api/web/kiosk (dev+production) sebagai systemd Service — Auto-Restart

## Depends on
Tidak ada dependency kode. Ini task infrastruktur murni (server config), tidak menyentuh `apps/*/src`.

## Objective
Semua 6 proses aplikasi (api/web/kiosk × dev+production) otomatis restart sendiri kalau crash, dan otomatis hidup lagi setelah reboot mesin — supaya insiden seperti 2026-08-06 (API production mati sejak reboot semalam, 646 tap kiosk menumpuk berjam-jam sebelum ketahuan) tidak terulang tanpa terdeteksi.

## Context
- **Insiden pemicu (2026-08-06)**: API production (port 3001) mati sejak ~22:50 WIB (kemungkinan besar bersamaan reboot mesin), sementara web (3000) dan kiosk (3002) production tetap hidup. Kiosk gerbang otomatis buffer semua tap ke IndexedDB lokal (fitur by-design, `apps/kiosk/src/lib/offline-buffer.ts`) karena tidak bisa mencapai API — user baru sadar setelah kiosk menampilkan "Offline — 646 tap tersimpan lokal". Root cause: proses Node dijalankan via `nohup`/`setsid` manual (`scripts/start-production.sh`, `scripts/dev-start.sh`) — TIDAK ADA watchdog/process-manager yang mendeteksi proses mati dan menghidupkannya lagi.
- **Topologi existing (T105)**: 2 folder terpisah total — `AbsenSI` (dev, port web:3100/api:3101/kiosk:3102) dan `AbsenSI-production` (production, port 3000/3001/3002). Docker container (MySQL, Redis) SUDAH auto-restart (`restart: always` di compose, terbukti dari insiden reboot sebelumnya di mana Docker container hidup normal tapi Node apps tidak) — **HANYA proses Node yang belum punya watchdog**, task ini fokus di situ saja.
- **Keputusan user 2026-08-06**: pakai **systemd** (bukan PM2) — alasan: standar Linux, tidak perlu tool tambahan, auto-start saat boot tanpa lapisan ekstra (PM2 sendiri masih perlu didaftarkan ke systemd juga supaya PM2-nya sendiri hidup lagi setelah reboot — systemd langsung lebih sederhana).
- Script existing yang jadi acuan start/stop tiap service: `scripts/start-production.sh` (production, dipanggil git post-commit hook), `scripts/dev-start.sh`/`dev-stop.sh` (dev, manual). **Task ini TIDAK mengganti isi script-script itu** — systemd unit akan MEMANGGIL perintah yang sama (`pnpm --filter @absensi/xxx start`/`dev`), bukan reinvent cara start service.

## Spec Detail

### 6 systemd unit baru (di `/etc/systemd/system/`)
Pola nama: `absensi-{env}-{app}.service` — contoh: `absensi-prod-api.service`, `absensi-prod-web.service`, `absensi-prod-kiosk.service`, `absensi-dev-api.service`, `absensi-dev-web.service`, `absensi-dev-kiosk.service`.

Tiap unit minimal:
```ini
[Unit]
Description=AbsenSI {env} {app}
After=network.target docker.service
Requires=docker.service

[Service]
Type=simple
User=anunnaki
WorkingDirectory=/home/anunnaki/Documents/APP SMK/AbsenSI{-production jika prod}
ExecStart=/usr/bin/pnpm --filter @absensi/{app} {start untuk prod, dev untuk dev}
Restart=on-failure
RestartSec=5
StandardOutput=append:/var/log/absensi/{env}-{app}.log
StandardError=append:/var/log/absensi/{env}-{app}.log

[Install]
WantedBy=multi-user.target
```
- **`Restart=on-failure`** (bukan `always`) — supaya kalau service SENGAJA di-stop manual (misal untuk maintenance), systemd tidak langsung menghidupkannya lagi secara paksa; tapi kalau proses crash/exit tidak normal, auto-restart. Diskusikan/verifikasi dengan user saat implementasi apakah preferensinya justru `always` (auto-restart bahkan setelah stop manual) — defaultkan ke `on-failure` sebagai lebih aman, tapi ini keputusan kecil yang boleh disesuaikan.
- **`RestartSec=5`** — jeda sebelum restart, hindari crash-loop yang membebani CPU kalau ada bug yang bikin proses selalu langsung mati.
- Path `WorkingDirectory` punya spasi (`APP SMK`) — pastikan quoting benar di unit file (systemd unit file TIDAK butuh escape shell seperti bash, tapi tetap perlu path lengkap yang valid).
- Buat folder log `/var/log/absensi/` dulu (`mkdir -p`, `chown` ke user `anunnaki`) sebelum unit pertama dijalankan.

### Migrasi dari proses manual existing
- **Matikan dulu proses `nohup` manual yang sedang jalan** (PID dari `/tmp/absensi-pids/*.pid`) SEBELUM start via systemd — supaya tidak ada 2 proses rebutan port yang sama. Urutan aman: `stop_one` semua service manual dulu (pola sama seperti di `start-production.sh`), BARU `systemctl start` semua unit baru.
- **`scripts/start-production.sh` (dipanggil git post-commit hook) PERLU DIUBAH** — bagian `start_one()` yang sekarang pakai `setsid nohup ... &` diganti jadi `systemctl restart absensi-prod-{name}` (lebih simpel, systemd yang urus proses lama/PID). **INI PERUBAHAN KODE** (bukan cuma infra), harus hati-hati supaya alur deploy otomatis (T105) tetap berfungsi identik dari sudut pandang "commit ke dev → auto-deploy ke production" — hanya CARA restart proses yang berubah, bukan alur deploy-nya.
- `scripts/dev-start.sh`/`dev-stop.sh` — pertimbangkan apakah TETAP dipertahankan sebagai cara manual (untuk kerja develop sehari-hari, kadang restart cepat tanpa systemctl) ATAU diganti total ke systemd juga — **klarifikasi ke user saat implementasi**: dev mungkin JUSTRU tidak butuh auto-restart-on-crash (crash saat development itu sinyal berguna untuk developer, auto-restart bisa menyembunyikan bug), pertimbangkan apakah systemd HANYA untuk production, dev tetap manual seperti sekarang.

### Enable saat boot
- `systemctl enable` semua unit yang dipilih (minimal production — lihat poin dev di atas) supaya otomatis start saat mesin boot, tanpa perlu login manual dan jalankan script.

## Catatan untuk Fase Berikutnya (BUKAN scope task ini — dicatat sesuai permintaan user)

**Notifikasi aktif saat service down** — user eksplisit ingin ini TAPI ditunda ("kita setup nanti, masukkan ke catatan"), bukan bagian dari eksekusi T117. Kalau dikerjakan nanti, kemungkinan pendekatan: systemd `OnFailure=` directive memanggil script kecil yang kirim notifikasi (WhatsApp API/Telegram bot/email) — atau health-check cron terpisah yang polling tiap beberapa menit. **Jangan mulai kerjakan ini sebelum user eksplisit minta** — cukup ada sebagai referensi kalau topik ini diangkat lagi di sesi mendatang.

## Edge Cases
- Systemd unit gagal start (misal port sudah dipakai proses lain, atau `pnpm` path salah) → pastikan ada cara mudah verifikasi (`systemctl status absensi-prod-api`, `journalctl -u absensi-prod-api -n 50`) — dokumentasikan command ini di `10-Environment.md` vault setelah selesai, supaya user tahu cara diagnosis kalau ada masalah serupa nanti.
- Auto-restart yang terlalu agresif kalau ada bug yang bikin proses selalu crash on-startup → `RestartSec=5` + pertimbangkan `StartLimitBurst`/`StartLimitIntervalSec` (systemd punya rate-limit restart bawaan) supaya tidak crash-loop tanpa henti — set nilai wajar (misal maks 5x restart dalam 5 menit, lalu berhenti dan butuh intervensi manual) supaya ADA sinyal jelas kalau memang ada masalah serius yang bukan sekadar hiccup sesaat.

## Files
- **Buat:** 6 file unit systemd baru di `/etc/systemd/system/` (di luar repo git, ini konfigurasi server — pertimbangkan simpan SALINAN template-nya di repo, misal `scripts/systemd/*.service.template`, supaya kalau server di-provision ulang polanya tidak hilang/perlu diingat ulang dari nol).
- **Modifikasi:** `scripts/start-production.sh` (ganti `start_one()` pakai `systemctl restart`, HATI-HATI ini dipanggil otomatis oleh git hook — test dulu manual sebelum yakin aman dipakai alur auto-deploy).
- **Dokumentasi:** update `Projek/AbsenSI/10-Environment.md` di vault dengan cara diagnosis/restart via systemd (command `systemctl status`/`journalctl`), supaya ini bukan pengetahuan yang cuma ada di kepala 1 sesi.

## Acceptance Criteria
- [x] 3 systemd unit dibuat (`absensi-prod-{api,web,kiosk}.service`) — **hanya production**, dev sengaja dikecualikan (lihat Validasi Claudian).
- [x] `systemctl status` semua unit menunjukkan `active (running)` — **terverifikasi 2026-08-06**, ketiganya `active (running)` + `enabled`.
- [x] Test nyata: `kill -9` salah satu proses API production secara manual → proses otomatis hidup lagi dalam <10 detik tanpa intervensi manual — **LOLOS 2026-08-06**: kill 11:01:31 → active lagi 11:01:36 (~5 detik, sesuai `RestartSec=5`), health-check konfirmasi responsif normal.
- [x] Test nyata: reboot mesin (dengan konfirmasi user dulu sebelum reboot sungguhan) → service otomatis hidup kembali — **LOLOS 2026-08-06**: reboot atas izin eksplisit user, mesin boot 11:02:51 → ketiga service `active (running)` di 11:03:12 (21 detik pasca boot) TANPA login manual/jalankan script apa pun, health-check konfirmasi web/kiosk 200, api 401 (normal, butuh auth).
- [x] `scripts/start-production.sh` diubah pakai `sudo -n systemctl restart absensi-prod-{name}` (bukan lagi `setsid nohup`), syntax divalidasi bersih (`bash -n`). Restart manual via systemd sudah terbukti bekerja (`reset-failed`+`start` dipakai saat instalasi), belum diuji lewat jalur auto-deploy git hook.
- [x] `10-Environment.md` diupdate dengan cara diagnosis via systemd (`systemctl status`/`journalctl`), section "Process Management — Systemd (T117)".

### Instalasi 2026-08-06 — catatan insiden kecil saat pasang
Saat instalasi awal, `absensi-prod-web` & `absensi-prod-kiosk` gagal start berulang (`EADDRINUSE` port 3000/3002) sampai kena `StartLimitBurst` (systemd berhenti coba lagi) — root cause: 2 proses Node manual lama (PID 315825 & 315839, dari sebelum migrasi ke systemd) masih menempel di port itu, tidak berhasil dimatikan langkah "matikan proses manual" di README karena PID file sudah tidak sinkron dengan proses aktual. Fix: `kill -9` kedua PID tersebut (dikonfirmasi ke user dulu), lalu `systemctl reset-failed` + `systemctl start` kedua unit — langsung `active (running)` normal setelahnya. `absensi-prod-api` sendiri sukses start bersih dari awal tanpa masalah ini. Diverifikasi akhir via `health-check.sh`: web 200, kiosk 200, api 401 (normal, endpoint butuh auth).

## Validasi Claudian
- [x] **WAJIB konfirmasi ke user sebelum reboot mesin sungguhan** untuk testing — dikonfirmasi eksplisit via AskUserQuestion 2026-08-06 sebelum reboot dijalankan.
- [x] Klarifikasi dev vs production: **diputuskan hanya production** yang didaftarkan systemd — dev tetap manual via `dev-start.sh`/`dev-stop.sh` (auto-restart saat development bisa menyembunyikan crash yang harusnya jadi sinyal bug ke developer).
- [x] `Restart=on-failure` dipakai (bukan `always`) — sesuai default yang direkomendasikan, tidak ada keberatan user.
- [x] Bagian notifikasi TIDAK dikerjakan — dicatat di `scripts/systemd/README.md` bagian "Belum Dikerjakan" sebagai catatan fase berikutnya.

## Status Eksekusi — SELESAI (2026-08-06)
Instalasi lengkap dijalankan user mengikuti `scripts/systemd/README.md`, sempat ada insiden kecil (2 proses manual lama menyumbat port, lihat catatan instalasi di atas) yang langsung diperbaiki. Kedua test wajib (crash-recovery + reboot) **LOLOS** dengan hasil terverifikasi (lihat Acceptance Criteria di atas). T117 dianggap **selesai penuh** — satu-satunya item yang sengaja ditunda adalah notifikasi aktif saat service down (fase berikutnya, bukan scope task ini).

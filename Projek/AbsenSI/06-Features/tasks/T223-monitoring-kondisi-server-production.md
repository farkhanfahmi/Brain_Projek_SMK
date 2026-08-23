# T223 — API+Web: Monitoring Kondisi Server Production (CPU/RAM/Disk/Service Health) + Backup Manual & Google Drive

## Depends on
Tidak ada dependency ke rangkaian lain. Independen. **Latar belakang eksplisit**: insiden T215 auto-deploy 2026-08-19 (lihat `Projek/AbsenSI/10-Environment.md` section insiden) mengungkap 2 celah pemantauan yang sudah lama tidak kelihatan: (1) tidak ada yang tahu backup harian gagal diam-diam selama 14 hari, (2) tidak ada gambaran kondisi server (disk penuh, CPU tinggi, dst) di luar cek manual. `scripts/systemd/README.md` baris 134-139 SUDAH mencatat "notifikasi aktif saat service down" sebagai fitur yang ditunda — task ini melanjutkan rencana itu, diperluas ke metrik sistem+backup.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-19)

Proyek ini SUDAH punya pola monitoring **kiosk** yang matang, dan task ini MEREPLIKASI pola arsitektur yang sama (bukan desain baru dari nol) untuk **server**, bukan kiosk:

- **Monitoring kiosk existing**: kiosk push heartbeat tiap 15 detik via Socket.IO (`apps/kiosk/src/lib/use-kiosk-health-report.ts`) ke gateway tunggal `apps/api/src/realtime/attendance.gateway.ts` (event `kiosk:health-report`), disimpan in-memory (`kioskHealthMap`), di-broadcast ke room `admin:kiosk-health`, ditampilkan realtime (bukan polling) di `apps/web/src/app/(admin)/kiosk-kesehatan/`.
- **`scripts/health-check.sh`** — HANYA cek HTTP status code 3 service (web/api/kiosk) via curl, TIDAK ada metrik CPU/RAM/disk sama sekali.
- **Tidak ada library system metrics terpasang** di `apps/api/package.json` — cuma Node.js `os` module built-in tersedia (perlu tambah dependency baru, misal `systeminformation`, ATAU cukup pakai `os`/`fs.statvfs` bawaan Node untuk kebutuhan dasar — PUTUSKAN saat implementasi mana yang cukup).
- **Hanya 1 Socket.IO gateway** di seluruh backend (`RealtimeModule`) — semua fitur realtime multiplex lewat room berbeda di gateway yang sama, POLA INI HARUS DIIKUTI (bukan bikin gateway kedua).
- **Insiden yang memicu task ini**: backup harian NVMe gagal diam-diam 14 hari (cron jalan tapi 0 file baru tercipta, log tidak tercatat, tidak ada MTA jadi error asli tidak pernah terlihat) — baru ketahuan lewat investigasi manual, BUKAN karena ada yang memantau.

## Spec Detail

### 1. Backend — endpoint metrik server (REST, snapshot on-demand)

- Modul baru `apps/api/src/server-health/` (`server-health.controller.ts`, `server-health.service.ts`, `server-health.module.ts`) — REPLIKASI struktur modul `kiosks/` sebagai referensi organisasi file.
- `GET /server-health` — return snapshot SAAT DIPANGGIL (bukan cached lama):
  ```ts
  interface ServerHealthSnapshot {
    cpu: { loadAvg1m: number; loadAvg5m: number; loadAvg15m: number; coreCount: number };
    memory: { totalMB: number; usedMB: number; freeMB: number; usedPercent: number };
    disk: { path: string; totalGB: number; usedGB: number; freeGB: number; usedPercent: number }[]; // per mount point relevan
    services: { name: string; status: "active" | "inactive" | "failed" | "unknown" }[]; // systemd 3 unit production
    backupTerakhir: { tanggal: string | null; ukuranMB: number | null; umurJam: number | null; status: "ok" | "terlambat" | "tidak_ada" };
    uptime: { serverUptimeDetik: number };
    checkedAt: string; // ISO timestamp snapshot ini diambil
  }
  ```
- **CPU/Memory** — Node.js `os` module built-in (`os.loadavg()`, `os.totalmem()`/`os.freemem()`) — CUKUP, tidak perlu dependency baru untuk ini.
- **Disk** — VERIFIKASI SAAT IMPLEMENTASI: Node.js tidak punya `statvfs` built-in langsung berguna lintas-platform, cek apakah perlu `child_process.exec("df -h ...")` (parsing output `df`, REKOMENDASI karena server ini pasti Linux, tidak perlu dependency tambahan) ATAU tambah 1 dependency ringan (`check-disk-space` npm package, kecil, tanpa native binding) — PILIH yang paling minim dependency baru.
- **Service status** — `child_process.exec("systemctl is-active absensi-prod-api absensi-prod-web absensi-prod-kiosk")` (3 unit dari T117) — service ini PASTI jalan sebagai user yang sama dengan API production (cek izin, MUNGKIN perlu sudoers scope tambahan mirip pola `absensi-systemd.sudoers` T105/T117 kalau `systemctl status` butuh privilege — REKOMENDASI: `systemctl is-active` biasanya bisa dipanggil user biasa tanpa sudo untuk READ status, beda dari `restart` yang butuh sudo — VERIFIKASI saat implementasi).
- **Backup terakhir** — baca `ls -la /media/anunnaki/DataNvme/backups/absensi/*.sql.gz` (file terbaru), hitung selisih waktu dari sekarang. Status `"terlambat"` kalau umur > 26 jam (cron harusnya tiap 24 jam, beri toleransi wajar), `"tidak_ada"` kalau folder kosong/tidak bisa diakses.
- **Endpoint dibatasi `@Roles(UserRole.super_admin)`** (informasi infrastruktur sensitif, konsisten pola `AttendanceLockConfig`/`SystemLiveConfig`).

### 2. Backend — cron internal cek berkala + alert (BARU, bukan cuma on-demand)

- Ini BEDA dari monitoring kiosk (yang murni pasif nunggu heartbeat) — server perlu **cek AKTIF berkala** karena tidak ada "server melapor sendiri" secara natural seperti kiosk.
- Tambah scheduled job (BullMQ, KONSISTEN stack existing yang sudah dipakai proyek untuk job terjadwal — cek pola `apps/api/src/teaching-sessions/generate-sessions.processor.ts` sebagai referensi cron job existing) — jalan tiap 15-30 menit, ambil snapshot `ServerHealthSnapshot`, BANDINGKAN ke threshold:
  - Disk `usedPercent > 85` → alert
  - Memory `usedPercent > 90` → alert
  - Service manapun `status !== "active"` → alert (KRITIKAL, ini yang bikin API/web/kiosk production DOWN)
  - `backupTerakhir.status !== "ok"` → alert (INI YANG TERJADI DI INSIDEN 2026-08-19 — kalau job ini sudah ada sebelumnya, 14 hari kegagalan backup akan ketahuan di hari PERTAMA, bukan 14 hari kemudian)
- **Kanal notifikasi**: REKOMENDASI mulai dari YANG PALING MURAH dulu — simpan ke tabel `ServerHealthAlert` baru (mirip pola `ActivityLog`, insert-only) + tampilkan badge/banner di halaman admin manapun yang dibuka super_admin (BUKAN langsung WhatsApp/Telegram/email yang butuh setup eksternal, PUTUSKAN kanal eksternal sebagai fase 2 terpisah kalau user memang mau — jangan overbuild fase 1).
- **VERIFIKASI SAAT IMPLEMENTASI**: apakah proses cron job ini SENDIRI rentan gagal diam-diam sama seperti backup script (insiden yang memicu task ini) — TAMBAH log eksplisit tiap kali job jalan (bahkan kalau semua sehat), supaya "job berhenti jalan" itu sendiri bisa dideteksi dari ketiadaan log terbaru (pola self-monitoring).

### 3. Frontend — halaman baru `(admin)/server-kesehatan/`

- REPLIKASI struktur `(admin)/kiosk-kesehatan/` (page.tsx server component + view.tsx client component).
- Tampilkan: gauge/angka CPU load, RAM usedPercent, disk usedPercent (per mount), badge status 3 service (hijau/merah), info backup terakhir (tanggal + umur + status, badge merah kalau terlambat/tidak ada).
- **Realtime vs polling**: BEDA dari kiosk (yang push tiap 15 detik terus-menerus) — server tidak perlu SEREALTIME itu. REKOMENDASI: `GET /server-health` di-fetch ulang tiap buka halaman + tombol "Refresh" manual, TIDAK PERLU Socket.IO tambahan untuk ini (overkill untuk data yang berubah lambat seperti disk usage) — KECUALI user secara eksplisit mau live-update, PUTUSKAN saat implementasi berdasar preferensi user kalau ditanya.
- Riwayat alert (dari tabel `ServerHealthAlert`) — tabel sederhana list alert historis, KONSISTEN aturan tabel wajib (sortable, search, kolom No).

### 4. Data model baru

```prisma
model ServerHealthAlert {
  id         Int      @id @default(autoincrement())
  kategori   String   // "disk" | "memory" | "service_down" | "backup_gagal"
  pesan      String   // actionable, sebut angka/nama spesifik
  createdAt  DateTime @default(now())

  @@map("server_health_alerts")
}
```
- Insert-only (KONSISTEN pola `tap_events`/`activity_log`), tidak ada endpoint UPDATE/DELETE.

### 5. Backend+Web — Trigger Backup Manual dari UI

Permintaan tambahan user (2026-08-19): admin bisa klik tombol di halaman monitoring untuk jalankan backup SEKARANG (bukan tunggu cron 02:00).

- `POST /server-health/backup/trigger` — `@Roles(UserRole.super_admin)`, `@LogActivity` (mutasi, WAJIB dicatat sesuai aturan CLAUDE.md).
- Backend jalankan `bash /home/anunnaki/scripts/backup-absensi.sh` via `child_process.exec()` — **JANGAN duplikasi logic backup di TypeScript**, REUSE script shell yang sudah ada dan sudah teruji (single source of truth untuk cara backup dilakukan).
- **Async, JANGAN blocking request** — backup `mysqldump` bisa makan waktu (lihat ukuran 3.1MB s.d. bisa lebih besar seiring data bertambah) — response endpoint LANGSUNG balik `{ status: "started", jobId }` sementara proses jalan di background, FE poll status ATAU tunggu event (REKOMENDASI: simpan status ke tabel `ServerHealthAlert`-serupa atau field in-memory sederhana `{ status: "running"|"success"|"failed", startedAt, finishedAt, outputFile }`, FE poll `GET /server-health/backup/status` tiap beberapa detik selama `running`).
- **Cegah backup manual dobel bersamaan** (klik tombol 2x, atau bentrok dengan cron 02:00 yang kebetulan jalan barengan) — cek status `running` sebelum mulai baru, tolak dengan pesan jelas ("Backup sedang berjalan, tunggu sampai selesai") kalau sudah ada proses aktif.
- FE: tombol "Backup Sekarang" di halaman `(admin)/server-kesehatan/`, sejajar info "Backup Terakhir" (poin 3) — disable+spinner saat status `running`, toast sukses/gagal.

### 6. Backend+Web — Backup ke Google Drive (OAuth Akun Pribadi + Link Folder Tujuan)

Permintaan tambahan user (2026-08-19): admin hubungkan akun Google (OAuth), tempel link folder Drive tujuan, lalu backup bisa diupload ke Drive dengan 1 klik — sebagai LAPISAN KE-4 selain 3 lapis existing (harian NVMe/mingguan+bulanan New Volume), BUKAN pengganti lapis yang sudah ada.

**Setup awal (dilakukan SEKALI oleh developer, di luar scope kode task ini, WAJIB didokumentasikan sebagai prasyarat)**:
- Buat OAuth Client ID di Google Cloud Console (scope `https://www.googleapis.com/auth/drive.file` — SEMPIT, HANYA akses file yang dibuat aplikasi ini sendiri, BUKAN `drive` scope penuh yang bisa baca semua file akun Google admin — prinsip least-privilege WAJIB, ini data backup sekolah yang sensitif).
- Simpan `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` di `.env` production (BUKAN hardcode, BUKAN commit ke git — KONSISTEN pola `.env` existing).

**Backend — OAuth flow & penyimpanan token**:
- Model baru:
  ```prisma
  model GoogleDriveBackupConfig {
    id                Int      @id @default(1)
    refreshTokenEnc   String   @db.Text // WAJIB terenkripsi, JANGAN plaintext (kredensial akses akun Google admin)
    driveFolderId     String   // diekstrak dari link folder yang admin tempel
    driveFolderNama   String?  // nama folder utk ditampilkan di UI (dari Drive API), bukan wajib
    connectedEmail    String?  // email akun Google yang terhubung, ditampilkan biar admin tahu akun mana
    updatedById       Int      @map("updated_by")
    updatedAt         DateTime @updatedAt

    updatedBy User @relation(fields: [updatedById], references: [id])

    @@map("google_drive_backup_config")
  }
  ```
  - **`refreshTokenEnc` WAJIB dienkripsi** sebelum simpan ke DB (AES atau setara, key dari `.env` terpisah dari `DATABASE_URL` — VERIFIKASI SAAT IMPLEMENTASI apakah proyek sudah punya utilitas enkripsi serupa di tempat lain untuk direuse, atau perlu buat baru — JANGAN simpan token akses Google Drive dalam bentuk yang bisa dibaca langsung dari dump SQL).
  - Singleton (`id @default(1)`) — KONSISTEN pola `AttendanceLockConfig`/`KampusTapConfig` (T216), HANYA 1 akun Drive terhubung untuk seluruh sistem.
- `POST /server-health/google-drive/connect` — mulai OAuth flow (redirect ke halaman consent Google), `GET /server-health/google-drive/callback` — terima authorization code, tukar jadi refresh token, simpan (terenkripsi) + `connectedEmail`.
- `PATCH /server-health/google-drive/folder` — terima `{ folderLink: string }` dari body, EKSTRAK folder ID dari URL (format link Drive: `https://drive.google.com/drive/folders/{FOLDER_ID}` — parse dengan regex, TOLAK dengan pesan jelas kalau format link tidak dikenali, JANGAN diam-diam gagal). VALIDASI folder itu BENAR ADA dan bisa diakses (panggil Drive API `files.get` dengan folder ID itu) SEBELUM simpan — kalau gagal (folder tidak ada/tidak ada akses), pesan actionable ("Folder tidak ditemukan atau akun belum diberi akses — pastikan link benar dan folder bisa diakses akun yang terhubung").
- `DELETE /server-health/google-drive/disconnect` — hapus config, putus koneksi (admin ganti akun/berhenti pakai fitur ini).
- `POST /server-health/google-drive/backup-now` — 1-klik: (1) jalankan backup lokal dulu (REUSE endpoint poin 5, JANGAN duplikasi logic backup), (2) upload file `.sql.gz` hasilnya ke `driveFolderId` via Google Drive API (`googleapis` npm package — dependency baru, VERIFIKASI belum ada duplikat sebelum install), (3) update status.
- **SEMUA endpoint Google Drive `@Roles(UserRole.super_admin)`** (kredensial + akses cloud eksternal, paling sensitif dari semua fitur T223).

**Frontend — section baru di `(admin)/server-kesehatan/`**:
- Kalau BELUM terhubung: tombol "Hubungkan Google Drive" → redirect OAuth.
- Kalau SUDAH terhubung: tampilkan email akun terhubung, input/tampil link folder tujuan (dengan tombol "Ubah" untuk ganti link), tombol "Backup ke Google Drive Sekarang", tombol "Putuskan Koneksi".
- **Riwayat upload Drive** — tampilkan status upload terakhir (sukses/gagal, timestamp) — REKOMENDASI: field ditambahkan ke tabel status backup manual (poin 5) atau tabel `ServerHealthAlert` kalau upload gagal (kategori baru `"gdrive_upload_gagal"`).

## Edge Cases

- **Endpoint `GET /server-health` dipanggil saat salah satu sub-check gagal** (misal `df` command error) — JANGAN gagalkan seluruh response, tampilkan field itu sebagai `null`/`"unknown"` dengan indikasi jelas, bagian lain tetap tampil.
- **Cron job monitoring ITU SENDIRI berhenti jalan** (skenario yang PERSIS terjadi ke backup script) — MITIGASI: log eksplisit tiap run (poin 2), supaya "tidak ada log terbaru" jadi sinyal sendiri yang bisa dicek manual kalau dicurigai.
- **Server benar-benar restart/reboot** — uptime reset ke 0, ini NORMAL bukan alert, JANGAN false-positive alert untuk ini.
- **Refresh token Google Drive expired/dicabut manual dari sisi Google** (misal admin cabut akses lewat myaccount.google.com) — upload akan gagal dengan error auth spesifik dari Google API, TANGKAP dan tampilkan pesan jelas ("Koneksi Google Drive terputus, hubungkan ulang") BUKAN 500 generik, JANGAN retry otomatis tanpa batas.
- **Backup manual (poin 5) dan Backup ke Drive (poin 6) diklik BERSAMAAN, atau bersamaan dengan cron 02:00** — SEMUA jalur backup lokal WAJIB melewati gate "cek status running" yang SAMA (poin 5), supaya tidak ada 2 `mysqldump` jalan bersamaan ke container yang sama.
- **Link folder Drive yang ditempel BUKAN folder tapi file, atau link folder yang SALAH FORMAT sama sekali** (bukan URL Drive) — validasi regex + `files.get` (poin 6) WAJIB menangkap ini sebelum disimpan sebagai config valid.

## Files
- **Buat:** `apps/api/src/server-health/` (controller/service/module/dto, TERMASUK sub-modul/service terpisah untuk Google Drive kalau lebih rapi — PUTUSKAN saat implementasi apakah 1 modul atau `server-health/` + `google-drive-backup/` terpisah), migration Prisma baru (`ServerHealthAlert`, `GoogleDriveBackupConfig`), `apps/web/src/app/(admin)/server-kesehatan/` (page.tsx + view.tsx), scheduled job baru (BullMQ processor, REPLIKASI pola `generate-sessions.processor.ts`), utilitas enkripsi token (baru atau reuse existing).
- **Jangan sentuh:** `apps/api/src/kiosks/`, `apps/api/src/realtime/attendance.gateway.ts` (monitoring kiosk TIDAK diubah, task ini paralel bukan pengganti), `scripts/health-check.sh` DAN `scripts/backup-absensi.sh` (backup manual REUSE script ini apa adanya via child_process, TIDAK menulis ulang logic backup di TypeScript — satu-satunya sumber kebenaran cara backup tetap script shell yang sudah ada).

## Acceptance Criteria
- [x] `GET /server-health` return snapshot CPU/RAM/disk/service/backup, dibatasi `super_admin`.
- [x] Halaman `(admin)/server-kesehatan/` tampilkan semua metrik dengan indikator visual jelas (hijau/kuning/merah sesuai threshold).
- [x] Cron job berkala (15-30 menit, dipilih 20 menit) cek threshold, insert `ServerHealthAlert` kalau melanggar.
- [x] Backup terlambat/gagal (skenario PERSIS insiden 2026-08-19) — terdeteksi dalam 1 siklus cron (bukan 14 hari kemudian). (test eksplisit di `server-health-check.processor.spec.ts`)
- [x] Service production down (skenario T117) — terdeteksi otomatis, bukan cuma dari `systemctl status` manual.
- [x] Riwayat alert tampil di tabel terpisah, sortable+search, kolom No.
- [x] Tombol "Backup Sekarang" di UI — trigger `backup-absensi.sh`, async, status jelas (running/success/failed), tidak bisa dobel-klik jalan bersamaan.
- [x] Google Drive — admin bisa hubungkan akun (OAuth), tempel link folder, dan validasi folder benar sebelum disimpan. (kode lengkap + test, BELUM diverifikasi live — butuh `GOOGLE_CLIENT_ID`/`SECRET` sungguhan, lihat prasyarat di `10-Environment.md`)
- [ ] Google Drive — tombol 1-klik backup+upload berhasil taruh file `.sql.gz` di folder Drive yang dikonfigurasi. **BELUM diverifikasi live** — kode lengkap (`backupNow()`/`runUploadAfterBackup()`) dan 13 test dengan Drive API di-mock semua lolos, tapi verifikasi end-to-end butuh setup OAuth Google Cloud Console manual (prasyarat, di luar scope kode) — checklist ini baru bisa dicentang setelah user setup kredensial dan uji coba nyata.
- [x] Refresh token Google Drive tersimpan TERENKRIPSI di DB, bukan plaintext. (AES-256-GCM, `common/token-encryption.ts`, 6 test unit)
- [x] Scope OAuth Google SEMPIT (`drive.file`, bukan akses penuh `drive`).
- [x] Build + type-check hijau, jest baru untuk `server-health.service.ts` (mock threshold breach) DAN untuk logic Google Drive (mock API, jangan panggil Drive API sungguhan di test).

## Validasi Claudian
- [x] Konfirmasi cron job monitoring log eksplisit tiap run (mitigasi supaya job ini sendiri tidak bisa gagal diam-diam seperti backup script yang memicu task ini). (`ServerHealthCheckProcessor` worker callback: `this.logger.log` tiap run, sukses/gagal)
- [x] Konfirmasi TIDAK menambah dependency besar (native binding dsb) untuk cek disk — pakai `df` via child_process atau dependency ringan murni JS. (`df -BM / --output=...` via child_process, TIDAK ada dependency baru untuk ini — hanya `googleapis` untuk Drive API poin 6)
- [x] Konfirmasi endpoint dibatasi `super_admin` (informasi infrastruktur, bukan untuk role lain). (guard per-method, satu-satunya pengecualian `google-drive/callback` yang PUBLIC by design — redirect murni dari Google, actorId dibawa lewat `state` param OAuth)
- [x] Konfirmasi refresh token Google Drive dienkripsi sebelum disimpan, key enkripsi terpisah dari `DATABASE_URL`. (`SERVER_HEALTH_ENCRYPTION_KEY` env var terpisah)
- [x] Konfirmasi backup manual (poin 5) dan backup-ke-Drive (poin 6) REUSE script shell yang sama, tidak duplikasi logic mysqldump di 2 tempat berbeda. (`GoogleDriveBackupService.backupNow()` panggil `BackupTriggerService.triggerBackup()` yang sama)
- [x] Konfirmasi gate anti-dobel-jalan mencegah backup lokal manual + cron + Drive-trigger bertabrakan menjalankan mysqldump bersamaan. (state in-memory tunggal di `BackupTriggerService`, `ConflictException` kalau sudah running — test eksplisit)

## Catatan Deploy (ditambahkan pasca-eksekusi, 2026-08-19)

Migration `20260819195731_t223_server_health_gdrive_backup` murni additive (2
`CREATE TABLE`, 1 `ADD FOREIGN KEY`) — TIDAK ada `DROP`/`ALTER` destruktif, aman
commit langsung sesuai aturan wajib CLAUDE.md (checked SEBELUM commit). Fitur
Google Drive TIDAK BISA dipakai sampai `.env` production diisi
`GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`/`GOOGLE_DRIVE_REDIRECT_URI`/
`SERVER_HEALTH_ENCRYPTION_KEY` sungguhan — lihat langkah setup lengkap di
`10-Environment.md` section "Setup Prasyarat Google OAuth". Fitur monitoring
CPU/RAM/disk/service/backup manual TIDAK butuh setup tambahan, langsung aktif
begitu ter-deploy.

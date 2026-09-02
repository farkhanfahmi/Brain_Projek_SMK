# Task-INFRA-002: Migrasi Dev Windows dari Docker ke MySQL+Redis Native

> Modul prefix: INFRA (infrastruktur/environment, bukan kode aplikasi).
> Dieksekusi langsung oleh user di Windows (instalasi software) — BUKAN oleh Hermes
> (Hermes dilarang instalasi/eksekusi sistem), BUKAN oleh Claude Code (bukan perubahan kode,
> murni environment). Bagian yang MELIBATKAN kode (`.env`, `docker-compose.yml`) boleh
> didiskusikan dengan Claude Code untuk verifikasi, tapi instalasi MySQL/Redis native tetap
> aksi manual user.

---

## 1. Info Eksekusi

**Rekomendasi Model:** Tidak relevan — ini task administrasi environment (instalasi software + edit config), bukan task coding kompleks.
**Tingkat Effort:** Tidak relevan.
**Alasan pemilihan:** Instalasi software Windows + copy config sederhana, tidak butuh reasoning model AI.

## 2. Konteks & Tujuan Utama

Device dev (Windows, RAM 8GB total, ~3.4GB free saat idle) tidak kuat menjalankan Docker Desktop (WSL2 VM overhead) bersamaan dengan 3 dev server (api/web/kiosk) + Chrome + tools lain — sering RAM habis, `node --max-old-space-size` OOM saat compile.

**Keputusan (2026-09-02, dikonfirmasi user)**: pindah dev MySQL+Redis dari Docker ke instalasi native Windows, MENERIMA risiko kecil versi berbeda dari production (yang tetap pakai Docker `mysql:8`+`redis:7`). User berkomitmen disiplin menjaga `.env` tetap benar supaya tidak ada kebingungan config antara dev-native dan production-Docker.

**Prinsip**: kode aplikasi (Prisma/NestJS) TIDAK berubah — DATABASE_URL/REDIS_URL tetap format standar, cuma host/port yang mengarah ke instalasi native, bukan container. Push ke GitHub TIDAK terpengaruh sama sekali (git cuma bawa source code).

**Depends on:** Tidak ada — task independen, bisa dikerjakan kapan saja.

## 3. Langkah Eksekusi Detail

### A. Backup dulu data dev existing (opsional tapi disarankan — dev boleh rusak bebas, tapi mempercepat kalau mau lanjutkan data lama)
```cmd
docker exec absensi-mysql-1 mysqldump -u root -ppassword absensi_db > C:\ProjekSMK\backup-dev-sebelum-native.sql
```

### B. Install MySQL Community Server 8.x native Windows
1. Download MySQL Installer dari https://dev.mysql.com/downloads/installer/ — pilih versi **8.0.x** (BUKAN 8.4/9.x) supaya paling dekat dengan `mysql:8` image production.
2. Saat instalasi: pilih "Server only" (tidak perlu Workbench/tools lain kalau sudah ada).
3. Set root password — **catat**, akan dipakai di `.env`.
4. **Port**: biarkan default 3306 (BUKAN 3307 seperti Docker lama) — lebih sederhana, tidak ada konflik karena Docker MySQL akan dimatikan.
5. Setelah install, buat database:
   ```sql
   CREATE DATABASE absensi_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```
6. Set MySQL service jadi auto-start (`services.msc` → MySQL80 → Startup type: Automatic) — supaya tidak perlu manual start tiap kali nyalakan PC.

### C. Install Redis alternatif untuk Windows
> Redis resmi TIDAK punya rilis Windows. Pilih salah satu:
- **Opsi 1 (disarankan): Memurai** (https://www.memurai.com/) — fork Redis untuk Windows, kompatibel API, ada versi Developer gratis.
- **Opsi 2**: Redis via WSL2 TANPA Docker Desktop (`wsl --install`, lalu `sudo apt install redis-server` di dalam WSL) — lebih ringan dari Docker Desktop penuh, tapi tetap butuh WSL2 aktif.

Rekomendasi: **Opsi 1 (Memurai)** — paling sederhana, tidak butuh WSL2 sama sekali, port default 6379 (sama seperti sebelumnya).

### D. Update `.env` dev (`apps/api/.env`)
Ganti baris:
```
DATABASE_URL="mysql://root:PASSWORD_BARU@localhost:3306/absensi_db"
REDIS_URL="redis://localhost:6379"
```
(Port MySQL berubah dari 3307→3306 karena tidak ada lagi konflik dengan container Docker.)

### E. Jalankan migration Prisma ke database native yang masih kosong
```cmd
cd C:\ProjekSMK\AbsenSI
pnpm --filter @absensi/api exec prisma migrate deploy
pnpm --filter @absensi/api exec prisma generate
```

### F. (Opsional) Restore data dev lama kalau ada backup dari langkah A
```cmd
mysql -u root -p absensi_db < C:\ProjekSMK\backup-dev-sebelum-native.sql
```

### G. Matikan Docker Desktop sepenuhnya (bukan cuma stop container)
- Settings → General → uncheck "Start Docker Desktop when you log in"
- Quit Docker Desktop dari system tray
- Verifikasi RAM bebas: `wmic OS get FreePhysicalMemory` sebelum vs sesudah Docker Desktop off — seharusnya naik signifikan (WSL2 VM overhead hilang).

### H. Update dokumentasi `docker-compose.yml` — JANGAN dihapus
File `docker-compose.yml` di root repo TETAP DIPERTAHANKAN apa adanya (dipakai kontributor lain / kalau suatu saat kembali ke Docker) — cukup tambahkan komentar di bagian atas:
```yaml
# CATATAN: Dev Windows (device utama) migrasi ke MySQL+Redis native sejak 2026-09-02
# (task-INFRA-002) karena keterbatasan RAM 8GB. File ini tetap valid untuk siapa pun
# yang mau pakai Docker (device lain, kontributor baru). Lihat 10-Environment.md.
```

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/.env` (DATABASE_URL, REDIS_URL) — TIDAK ADA perubahan kode aplikasi.
- **Jangan sentuh:** `docker-compose.yml` (dipertahankan, cuma tambah komentar), skema Prisma, kode NestJS/Next.js apapun.

**Dilarang dilakukan:**
- Jangan ubah `DATABASE_URL`/`REDIS_URL` di `.env` PRODUCTION — task ini SCOPE DEV WINDOWS SAJA.
- Jangan hapus `docker-compose.yml` — masih relevan untuk device/kontributor lain.
- Jangan install MySQL versi 9.x — pilih 8.0.x supaya dekat dengan production.

**Skenario kegagalan yang WAJIB ditangani:**
- **Port 3306 sudah dipakai proses lain** (jarang, tapi cek `netstat -ano | grep 3306` dulu sebelum install) → kalau bentrok, pakai port lain saat instalasi MySQL, sesuaikan `DATABASE_URL`.
- **Redis Memurai gagal start sebagai service** → cek Windows Event Viewer, biasanya masalah permission folder data.
- **Migration Prisma gagal karena collation/charset beda** → MySQL native installer defaultnya `utf8mb4_0900_ai_ci` (MySQL 8.0+), sedangkan image Docker mungkin `utf8mb4_unicode_ci` — kalau ada error terkait ini saat `migrate deploy`, cek `schema.prisma` apakah ada assumption collation eksplisit.
- **Lupa disiplin `.env`** (risiko yang sudah disadari user) → pertimbangkan tambahkan komentar jelas di `.env` dev: `# NATIVE (bukan Docker) sejak 2026-09-02 — port 3306, bukan 3307`.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] MySQL 8.0.x native terinstall, service auto-start, `absensi_db` dibuat dengan charset `utf8mb4`.
- [ ] Redis (Memurai atau WSL native) terinstall, service jalan di port 6379.
- [ ] `apps/api/.env` diupdate ke `localhost:3306` (MySQL) dan `localhost:6379` (Redis).
- [ ] `prisma migrate deploy` berhasil tanpa error ke database native.
- [ ] `pnpm turbo run dev` (api+web+kiosk) jalan normal, bisa login, bisa tap simulasi.
- [ ] Docker Desktop dimatikan total (tidak auto-start), RAM free meningkat terverifikasi (`wmic OS get FreePhysicalMemory` sebelum vs sesudah).
- [ ] `docker-compose.yml` tetap ada di repo dengan komentar penjelasan, TIDAK dihapus.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup
- [ ] Scope tidak melebar ke perubahan production
- [ ] Dependency: tidak ada

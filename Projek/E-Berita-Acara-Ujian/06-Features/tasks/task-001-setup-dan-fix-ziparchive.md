---
tags:
  - task
  - setup
  - bugfix
created: 2026-06-04
status: ready
---

# Task-001: Setup Aplikasi + Fix ZipArchive Import Error

## Objective
Setup semua service aplikasi agar berjalan dengan benar di device ini, lalu perbaiki error `Class "ZipArchive" not found` yang mencegah fitur import XLSX berfungsi di admin panel.

## Context
- **Modul:** Setup & Infrastructure + Admin Import
- **DB:** MySQL — database `berita_acara_ujian_baru`
- **Backend:** Laravel 11, port 8000
- **Frontend Pengawas/Panitia:** React (Vite), port **5173** (HTTPS via basicSsl)
- **Frontend Admin:** React (Vite), port **5174**
- **Frontend TV:** React (Vite), port **5175**

## Root Cause (Sudah Dianalisis)

Proses `php artisan serve` yang berjalan saat ini (PID 1673) menggunakan **binary PHP lama yang sudah dihapus** (`/usr/bin/php8.4 (deleted)`). Binary lama ini tidak punya extension `php8.4-zip`. Sementara PHP 8.4.11 yang baru (built May 25 2026) sudah ada ZipArchive.

**Bukti:**
```
/proc/1673/exe -> /usr/bin/php8.4 (deleted)
```

Solusinya: kill proses lama, start baru dengan PHP terkini.

---

## Part 1 — Verifikasi Environment

Sebelum mengeksekusi apapun, lakukan verifikasi:

**1a. Cek PHP dan ZipArchive**
```bash
php --version
php -r "echo class_exists('ZipArchive') ? 'ZipArchive: OK' : 'ZipArchive: MISSING';"
```
Hasil yang diharapkan: PHP 8.4.11 + `ZipArchive: OK`

**1b. Cek koneksi MySQL**
```bash
cd backend/
php artisan db:show 2>&1 | head -20
```

**1c. Cek apakah storage link sudah ada**
```bash
ls -la backend/public/storage 2>/dev/null && echo "Link ada" || echo "Link BELUM ada"
```

**1d. Cek apakah npm dependencies sudah diinstall**
```bash
ls frontend/node_modules 2>/dev/null && echo "frontend: OK" || echo "frontend: BELUM install"
ls frontend-admin/node_modules 2>/dev/null && echo "frontend-admin: OK" || echo "frontend-admin: BELUM install"
ls frontend-tv/node_modules 2>/dev/null && echo "frontend-tv: OK" || echo "frontend-tv: BELUM install"
```

---

## Part 2 — Setup (Jalankan hanya jika diperlukan)

Jalankan bagian ini **hanya jika verifikasi di Part 1 menemukan masalah**.

**2a. Install PHP dependencies (jika belum)**
```bash
cd backend/
composer install
```

**2b. Buat storage link (jika belum ada)**
```bash
cd backend/
php artisan storage:link
```

**2c. Install npm dependencies (hanya yang belum ada)**
```bash
# Jalankan hanya untuk folder yang node_modules-nya belum ada
cd frontend/ && npm install
cd frontend-admin/ && npm install
cd frontend-tv/ && npm install
```

**2d. Verifikasi .env backend sudah benar**
- File `backend/.env` harus ada (bukan hanya `.env.example`)
- Jika belum ada: `cp backend/.env.example backend/.env && php artisan key:generate`

---

## Part 3 — Fix ZipArchive: Restart Artisan Serve

**WAJIB DILAKUKAN — ini inti dari task ini.**

```bash
# 1. Kill proses artisan serve yang lama (binary deleted)
kill 1673

# 2. Tunggu proses benar-benar berhenti
sleep 2

# 3. Verifikasi proses sudah mati
ps aux | grep "artisan serve" | grep -v grep

# 4. Start ulang artisan serve dengan PHP baru
cd "/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/backend"
nohup php artisan serve --host=0.0.0.0 --port=8000 > /tmp/artisan-serve.log 2>&1 &

# 5. Tunggu sebentar lalu verifikasi
sleep 3
curl -s http://localhost:8000/api/health-check
```

Hasil yang diharapkan dari health-check: response JSON status OK.

---

## Part 4 — Buat Script Startup

Buat file `start-dev.sh` di root project untuk mempermudah menjalankan semua service:

**File:** `/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/start-dev.sh`

```bash
#!/bin/bash
# E-Berita Acara Ujian — Dev Startup Script
# Usage: ./start-dev.sh

ROOT_DIR="$(cd "$(dirname "$0")" && pwd)"

echo "=== E-Berita Acara Ujian — Starting Services ==="

# Kill proses lama jika masih jalan
echo "[1/5] Stopping old processes..."
pkill -f "artisan serve" 2>/dev/null
sleep 1

# Backend
echo "[2/5] Starting Backend (port 8000)..."
cd "$ROOT_DIR/backend"
nohup php artisan serve --host=0.0.0.0 --port=8000 > /tmp/log-backend.log 2>&1 &
BACKEND_PID=$!
echo "      Backend PID: $BACKEND_PID"

# Tunggu backend siap
sleep 2

# Frontend Pengawas/Panitia
echo "[3/5] Starting Frontend Pengawas/Panitia (port 5173)..."
cd "$ROOT_DIR/frontend"
nohup npm run dev > /tmp/log-frontend.log 2>&1 &
echo "      PID: $!"

# Frontend Admin
echo "[4/5] Starting Frontend Admin (port 5174)..."
cd "$ROOT_DIR/frontend-admin"
nohup npm run dev > /tmp/log-frontend-admin.log 2>&1 &
echo "      PID: $!"

# Frontend TV
echo "[5/5] Starting Frontend TV (port 5175)..."
cd "$ROOT_DIR/frontend-tv"
nohup npm run dev > /tmp/log-frontend-tv.log 2>&1 &
echo "      PID: $!"

echo ""
echo "=== Semua service dimulai ==="
echo "Backend API    : http://localhost:8000"
echo "Frontend (PKS) : https://localhost:5173"
echo "Frontend Admin : http://localhost:5174"
echo "Frontend TV    : http://localhost:5175"
echo ""
echo "Log files: /tmp/log-backend.log, /tmp/log-frontend*.log"
echo "Stop all : pkill -f 'artisan serve'; pkill -f 'vite'"
```

Chmod agar bisa dieksekusi:
```bash
chmod +x "/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/start-dev.sh"
```

---

## Part 5 — Verifikasi Fix

Setelah restart artisan serve, verifikasi bahwa ZipArchive sudah berjalan:

```bash
# Test langsung via PHP
/usr/bin/php -r "
\$s = new \PhpOffice\PhpSpreadsheet\Spreadsheet();
echo 'PhpSpreadsheet: OK' . PHP_EOL;
" 2>&1

# Test health check API
curl -s http://localhost:8000/api/health-check

# Cek log untuk memastikan tidak ada error
tail -20 /tmp/artisan-serve.log
```

---

## Files

- **Buat:** `start-dev.sh` di root project
- **Modifikasi:** Tidak ada kode yang diubah
- **Jangan sentuh:** Semua file kode aplikasi — ini murni infrastructure fix

## Acceptance Criteria

- [ ] `php -r "echo class_exists('ZipArchive') ? 'OK' : 'FAIL';"` → `OK`
- [ ] `curl http://localhost:8000/api/health-check` → response HTTP 200
- [ ] Import XLSX di admin panel (menu Pengawas) tidak lagi error "ZipArchive not found"
- [ ] Semua 3 frontend bisa diakses di port masing-masing
- [ ] File `start-dev.sh` sudah ada dan bisa dijalankan
- [ ] CONTEXT.md diupdate dengan hasil pekerjaan ini

## Next Action Setelah Selesai

Update CONTEXT.md bagian "Status Terakhir" dan "Next Action" dengan:
- Konfirmasi ZipArchive fixed
- PID baru artisan serve
- Status setiap frontend
- Apakah ada masalah yang ditemukan selama setup

## Validasi Claudian
- [x] Tidak ada ambiguitas dalam spec ini
- [x] Semua edge case sudah tercakup (verifikasi per-step)
- [x] Scope tidak terlalu besar (tidak ada perubahan kode)
- [x] Tidak ada konflik dengan keputusan di 11-Decisions.md

---
tags:
  - project
  - environment
created: 2026-06-04
updated: 2026-06-04
---

# Environment — E-Berita Acara Ujian

## Development Setup

### Prerequisites
- PHP 8.2+
- Composer
- Node.js 20+
- SQLite (dev) atau MySQL 8+

### Backend Setup
```bash
cd backend/
cp .env.example .env
composer install
php artisan key:generate
php artisan migrate
php artisan db:seed
php artisan storage:link
php artisan serve  # port 8000
```

### Frontend Setup (masing-masing)
```bash
# Frontend Pengawas/Panitia
cd frontend/ && npm install && npm run dev  # port 5174

# Frontend Admin
cd frontend-admin/ && npm install && npm run dev  # port 5175

# Frontend TV
cd frontend-tv/ && npm install && npm run dev  # port 5176
```

### Proxy Config (Vite)
Setiap frontend sudah dikonfigurasi proxy `/api` ke `http://localhost:8000` di `vite.config.js`.

---

## Environment Variables Backend (`.env`)

| Key | Dev Value | Prod Value | Keterangan |
|-----|-----------|------------|-----------|
| `APP_ENV` | `local` | `production` | |
| `APP_DEBUG` | `true` | `false` | WAJIB false di prod |
| `APP_URL` | `http://localhost:8000` | URL server | |
| `DB_CONNECTION` | `sqlite` | `mysql` | |
| `DB_DATABASE` | path SQLite | nama DB MySQL | |
| `DB_HOST` | - | IP MySQL | |
| `DB_USERNAME` | - | mysql user | |
| `DB_PASSWORD` | - | mysql pass | |
| `FILESYSTEM_DISK` | `local` | `local` | Untuk storage signature |

---

## Docker (Production)

File tersedia: `backend/Dockerfile` + `backend/docker-compose.yml`

```bash
cd backend/
docker-compose up -d
```

---

## Port Convention

| Service | Port |
|---------|------|
| Backend API | 8000 |
| Frontend Pengawas/Panitia | 5174 |
| Frontend Admin | 5175 |
| Frontend TV | 5176 |

---

## Storage

File upload disimpan di:
```
backend/storage/app/public/
├── signatures/       ← PNG tanda tangan dari berita acara
├── bukti-izin/       ← Foto bukti keterangan absen
└── pelanggaran/      ← Foto catatan ketidaktertiban
```

Symlink wajib dibuat: `php artisan storage:link`
→ Akses via: `http://[host]:8000/storage/[path]`

---

## Run All (Development)

Script `run-all.bat` di root project (Windows) — menjalankan semua service sekaligus.

---

## Catatan Produksi

- File `.sql` backup ada di root project — **JANGAN commit ke git**
- `APP_DEBUG=false` WAJIB di produksi
- Pastikan `php artisan storage:link` sudah dijalankan
- Nginx/Apache perlu dikonfigurasi untuk serve Laravel dengan `public/` sebagai document root

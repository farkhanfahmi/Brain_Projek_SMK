---
tags:
  - task
  - infrastructure
  - backend
  - frontend
created: 2026-06-08
revised: 2026-06-08
status: ready
revision: 2
---

# Task-019 (REV 2): Nginx + PHP-FPM (Backend) + Serve Frontend Build

> **Revisi 2 — alasan:** Setelah verifikasi kondisi server nyata, ditemukan beberapa
> hal yang membuat task v1 berisiko/kurang lengkap. Lihat bagian **"Temuan Server (Verified)"**
> dan **"Perubahan dari v1"** di bawah. Task v1 hanya menangani backend; padahal penyebab
> "loading lama" & "kamera blank saat 1 orang" adalah **frontend masih `npm run dev` (Vite dev)**.
> Rev 2 menambahkan langkah serve hasil **build** frontend + HTTPS (wajib untuk kamera).

---

## Objective
1. Ganti `php artisan serve` (single-threaded → antre) dengan **Nginx + PHP-FPM** (multi-worker concurrent).
2. Serve **frontend hasil `build`** (bukan Vite dev server) lewat Nginx, **dengan HTTPS**
   (kamera `getUserMedia` wajib HTTPS).
3. Target: backend handle 50+ device scan bersamaan tanpa antre; frontend loading cepat
   tanpa kompilasi on-the-fly.

---

## Temuan Server (Verified 2026-06-08)

Diverifikasi langsung di server sebelum revisi ini ditulis:

| Item | Hasil cek | Implikasi |
|---|---|---|
| nginx host (apt) | **BELUM terinstall** (`dpkg`: tidak ada) | Aman di-`apt install`, tidak menimpa nginx host lain |
| nginx yang berjalan sekarang | **Milik Docker** (container `spmb-app`, `agenda-client`, dll) | **JANGAN diutak-atik.** `apt install nginx` host tidak mengganggu nginx dalam container |
| php8.4-fpm | **BELUM terinstall** (hanya `php8.4-cli` 8.4.11) | Perlu `apt install php8.4-fpm` |
| Port terpakai Docker | **8081, 8082, 9000, 9001** | Nginx host JANGAN pakai port ini. Untuk test jangan pakai 8081/8082 |
| Port 8002 / 8003 / 5173 | **bebas** (skrg dipakai artisan serve & vite yg akan dimatikan) | Aman dipakai nginx |
| user `anunnaki` | uid 1000, **anggota grup `docker` & `sudo`** | FPM pool sebaiknya jalan sebagai `anunnaki` (Opsi B) |
| `php artisan storage:link` | **SUDAH terlink** (`public/storage` → `storage/app/public`) | **Skip Step storage:link v1** — tidak perlu diulang |
| `.env` | `APP_ENV=local`, `APP_URL=http://localhost:8002`, `FILESYSTEM_DISK=local` | OK; pertimbangkan ganti `APP_ENV=production` saat go-live |
| `frontend*/dist` | **Ketiganya sudah ADA** (frontend, -admin, -tv) | Build sudah pernah jalan; cukup rebuild fresh sebelum serve |
| Cert HTTPS sekarang | dari `@vitejs/plugin-basic-ssl` (self-signed, in node_modules) | Untuk produksi pakai self-signed sendiri via openssl (lihat Step F2) |

---

## Perubahan dari v1 (ringkas)

1. **TAMBAH bagian Frontend (Step F1–F4):** build + serve static + HTTPS. Ini inti perbaikan
   "loading lama" & "kamera blank saat sendirian" yang v1 lewatkan total.
2. **HATI-HATI Docker:** v1 Step 4 `rm -f /etc/nginx/sites-enabled/default` aman (itu file host,
   bukan container), TAPI ditambahkan peringatan eksplisit jangan stop/restart nginx Docker.
3. **Port test diganti** dari 8003 → tetap **8003** (terverifikasi bebas), TAPI ditegaskan
   **JANGAN pakai 8081/8082/9000/9001** (dipakai Docker).
4. **FPM user = `anunnaki` (Opsi B) dijadikan default**, bukan opsi. Karena file ada di
   `/home/anunnaki/...` yang tidak bisa dibaca `www-data` → menghindari 403 Forbidden.
5. **Pakai symlink `/var/www/berita-acara`** dijadikan WAJIB (bukan opsional), karena path
   asli mengandung spasi (`APP SMK`) yang rawan error di config nginx.
6. **storage:link DIHAPUS dari langkah** (sudah terlink — diverifikasi).
7. **TLS/HTTPS untuk frontend** ditambahkan — tanpa ini kamera mati total.

---

## Rollback Plan (30 detik)
```bash
sudo systemctl stop nginx
cd "/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/backend"
PHPRC=/home/anunnaki/php-upload.ini nohup php artisan serve --host=0.0.0.0 --port=8002 > /tmp/log-backend.log 2>&1 &
# Frontend dev (sementara): jalankan ulang npm run dev di tiap folder frontend
```
Catatan: rollback ini TIDAK menyentuh container Docker — nginx Docker tetap jalan apa adanya.

---

# BAGIAN A — BACKEND (Nginx + PHP-FPM)

### ⚠️ Peraturan Docker (baca dulu)
- Nginx yang muncul di `ps aux` sekarang adalah **proses di dalam container Docker**.
  **JANGAN** `sudo systemctl stop/restart` apapun yang Anda kira "nginx lama" — itu tidak
  akan mengenai container, tapi pastikan jangan `docker stop` container apapun.
- `apt install nginx` menginstall **nginx HOST baru** yang terpisah dari Docker. Aman.
- Port host yang HARAM dipakai (sudah Docker): **8081, 8082, 9000, 9001**.

### Step A1 — Install Nginx + PHP 8.4-FPM
```bash
sudo apt update
sudo apt install -y nginx php8.4-fpm
```
Verifikasi:
```bash
nginx -v
php-fpm8.4 -v
sudo systemctl status php8.4-fpm --no-pager
ls -la /run/php/php8.4-fpm.sock   # konfirmasi socket muncul
```

### Step A2 — Symlink path (WAJIB, karena folder mengandung spasi)
```bash
sudo ln -sfn "/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/backend" /var/www/berita-acara
ls -la /var/www/berita-acara/public/index.php   # harus ada
```

### Step A3 — Konfigurasi PHP-FPM Pool (jalan sebagai user anunnaki)
Edit `/etc/php/8.4/fpm/pool.d/www.conf`:
```bash
sudo nano /etc/php/8.4/fpm/pool.d/www.conf
```
Ganti/sesuaikan:
```ini
; Jalankan FPM sebagai user pemilik file (hindari 403 di /home) — WAJIB
user = anunnaki
group = anunnaki

; CPU 8 thread (i3-12100) → worker cukup banyak
pm = dynamic
pm.max_children = 20
pm.start_servers = 5
pm.min_spare_servers = 3
pm.max_spare_servers = 10
pm.max_requests = 500

; Upload limit (PHPRC tidak lagi dibutuhkan)
php_value[upload_max_filesize] = 10M
php_value[post_max_size] = 15M
```
Restart:
```bash
sudo systemctl restart php8.4-fpm
```

### Step A4 — Nginx Virtual Host Backend
```bash
sudo nano /etc/nginx/sites-available/berita-acara-api
```
Isi (pakai port **8003 dulu** untuk test, nanti diganti 8002):
```nginx
server {
    listen 8003;          # SEMENTARA untuk test; nanti diganti 8002
    server_name _;

    root /var/www/berita-acara/public;   # via symlink, bebas spasi
    index index.php;

    client_max_body_size 15M;
    fastcgi_read_timeout 120;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(env|git) { deny all; }

    access_log /tmp/nginx-berita-api-access.log;
    error_log  /tmp/nginx-berita-api-error.log;
}
```
> Catatan: socket path **`/run/php/php8.4-fpm.sock`** (Ubuntu modern). Jika `ls` di Step A1
> menunjukkan `/var/run/php/...`, keduanya biasanya symlink yang sama — pakai yang muncul di `ls`.

### Step A5 — Aktifkan & Test di port 8003 (artisan serve MASIH jalan di 8002)
```bash
sudo ln -sfn /etc/nginx/sites-available/berita-acara-api /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default   # file HOST, bukan container — aman
sudo nginx -t
sudo systemctl reload nginx

# Test (jangan pakai 8081/8082 — itu Docker)
curl -s -o /dev/null -w "Nginx 8003 init-data: %{http_code}\n" http://localhost:8003/api/init-data
```
Target: **200**. Jika 502 → cek `sudo systemctl status php8.4-fpm` + socket.
Jika 403 → pastikan Step A3 `user = anunnaki` sudah diterapkan & FPM di-restart.

### Step A6 — Switch ke 8002 (matikan artisan serve)
```bash
sudo sed -i 's/listen 8003;/listen 8002;/' /etc/nginx/sites-available/berita-acara-api
pkill -f "artisan serve.*8002"
sleep 1
sudo nginx -t && sudo systemctl reload nginx

curl -s -o /dev/null -w "Final init-data: %{http_code}\n" http://localhost:8002/api/init-data
curl -s -o /dev/null -w "presensi-today: %{http_code}\n" "http://localhost:8002/api/presensi-today?ujian_id=9"
```

---

# BAGIAN B — FRONTEND (Build + Serve + HTTPS)

> Inilah bagian yang HILANG di v1. Tanpa ini, loading tetap lama & kamera tetap blank
> walau backend sudah kencang. Kamera `getUserMedia` **wajib HTTPS**.

### Step F1 — Build ketiga frontend (fresh)
```bash
cd "/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu"
( cd frontend && npm run build )         # Pengawas/Panitia → dist/
( cd frontend-admin && npm run build )   # Admin → dist/
( cd frontend-tv && npm run build )      # TV → dist/
```
> ⚠️ **Cek proxy API:** saat dev, Vite mem-proxy `/api` & `/storage` ke `:8002`.
> Setelah jadi static di nginx, **nginx yang harus mem-proxy `/api` & `/storage`** ke backend
> (lihat Step F3). Pastikan kode frontend memanggil API dengan **path relatif** (`/api/...`),
> bukan `http://localhost:8002` hardcoded. **Verifikasi di kode `src/api*.js` sebelum lanjut.**

### Step F2 — Siapkan sertifikat HTTPS (self-signed untuk IP)
```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 730 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/berita-acara.key \
  -out /etc/nginx/ssl/berita-acara.crt \
  -subj "/CN=202.58.77.96" \
  -addext "subjectAltName=IP:202.58.77.96"
```
> Self-signed → browser akan warning "Not secure" sekali; user klik "lanjutkan". Kamera tetap
> jalan karena konteks tetap HTTPS. (Jika nanti ada domain, ganti dengan Let's Encrypt.)

### Step F3 — Nginx Virtual Host Frontend (HTTPS 5173) + proxy API
Symlink dulu folder dist agar bebas spasi:
```bash
sudo ln -sfn "/home/anunnaki/Documents/APP SMK/berita-acara-ujian-baruuuuuuu/frontend/dist" /var/www/berita-fe
```
```bash
sudo nano /etc/nginx/sites-available/berita-acara-fe
```
```nginx
server {
    listen 5173 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/berita-acara.crt;
    ssl_certificate_key /etc/nginx/ssl/berita-acara.key;

    root /var/www/berita-fe;
    index index.html;

    client_max_body_size 15M;

    # SPA fallback (React Router) — semua route → index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API ke backend (nginx port 8002 dari Bagian A)
    location /api {
        proxy_pass http://127.0.0.1:8002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120;
    }
    location /storage {
        proxy_pass http://127.0.0.1:8002;
        proxy_set_header Host $host;
    }

    access_log /tmp/nginx-berita-fe-access.log;
    error_log  /tmp/nginx-berita-fe-error.log;
}
```
> Admin (5174) & TV (5175) opsional dibuat blok serupa (root → `frontend-admin/dist` /
> `frontend-tv/dist`). TV biasanya di LAN → boleh HTTP, tapi konsisten HTTPS lebih aman.

### Step F4 — Aktifkan & Test Frontend
```bash
sudo ln -sfn /etc/nginx/sites-available/berita-acara-fe /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

curl -sk -o /dev/null -w "FE https 5173: %{http_code}\n" https://localhost:5173/
curl -sk -o /dev/null -w "FE proxy /api: %{http_code}\n" https://localhost:5173/api/init-data
```
Lalu buka di browser `https://202.58.77.96:5173/` → harus instan (tanpa kompilasi),
kamera scan jalan (HTTPS aktif).

---

# BAGIAN C — Finalisasi

### Step C1 — Update start-dev.sh / buat start-prod.sh
Jangan timpa start-dev.sh (masih berguna utk development). Buat `start-prod.sh`:
```bash
#!/bin/bash
# Production: nginx + php-fpm (tidak ada artisan serve / vite dev)
sudo systemctl start php8.4-fpm
sudo systemctl start nginx
sudo systemctl enable php8.4-fpm nginx   # auto-start saat boot
echo "Backend  : http://<ip>:8002 (nginx+fpm)"
echo "Frontend : https://<ip>:5173 (static build)"
```

### Step C2 — Rebuild frontend setiap ada perubahan kode FE
Ingat: static build TIDAK auto-reload. Setiap ubah kode frontend → `npm run build` ulang
di folder ybs, lalu hard-refresh browser. (Tidak perlu reload nginx; symlink menunjuk dist.)

---

## Acceptance Criteria

**Backend**
- [ ] `curl http://localhost:8002/api/init-data` → 200 (via Nginx)
- [ ] `curl "http://localhost:8002/api/presensi-today?ujian_id=9"` → 200
- [ ] Upload foto 5MB → berhasil (FPM limit 10M)
- [ ] Scan barcode → response < 300ms; tidak ada antrian 500ms saat banyak request paralel
- [ ] Tidak ada 502/403
- [ ] `php artisan serve` TIDAK berjalan di 8002 (`ps aux | grep artisan`)
- [ ] `systemctl status nginx` & `php8.4-fpm` → active

**Frontend**
- [ ] `https://localhost:5173/` → 200, loading instan (bukan Vite dev compile)
- [ ] `https://localhost:5173/api/init-data` → 200 (proxy nginx jalan)
- [ ] Buka di device lain via `https://202.58.77.96:5173/` → kamera scan AKTIF (HTTPS ok)
- [ ] Kamera tidak blank saat 1 user (Vite dev sudah tidak dipakai)
- [ ] `npm run dev` / vite TIDAK lagi dipakai untuk produksi (`ps aux | grep vite`)

**Docker (jangan rusak)**
- [ ] Container `spmb-app`, `agenda-client`, `lms-minio` dll TETAP Up (`docker ps`)
- [ ] Port 8081/8082/9000/9001 tidak diganggu

**Dokumentasi**
- [ ] Update CONTEXT.md setelah selesai

---

## Troubleshooting

**502 Bad Gateway:** `sudo systemctl status php8.4-fpm`; cek socket `ls -la /run/php/php8.4-fpm.sock`;
pastikan `fastcgi_pass` di config = path socket yang benar.

**403 Forbidden:** hampir pasti FPM masih jalan sebagai `www-data` yang tak bisa baca `/home`.
Pastikan Step A3 `user = anunnaki` + `sudo systemctl restart php8.4-fpm`.

**Kamera tetap blank di browser:** pastikan diakses via **https://** (bukan http), dan cert
Step F2 ter-load (`sudo nginx -t` tanpa error). Cek console browser apakah `getUserMedia`
error "Only secure origins".

**/api 404 atau CORS:** cek kode frontend pakai path relatif `/api/...`. Jika hardcoded
`http://localhost:8002`, build akan salah saat diakses dari device lain — perbaiki di `src/api*.js`.

**Nginx config error:** `sudo nginx -t` selalu sebelum reload; `sudo tail -20 /tmp/nginx-berita-*-error.log`.

**Docker container ikut mati:** seharusnya tidak terjadi (host nginx terpisah). Jika iya,
Anda tidak sengaja `docker stop` — jalankan `docker start <nama>`.

---
tags:
  - project
  - tech-stack
created: 2026-06-04
updated: 2026-06-04
---

# Tech Stack — E-Berita Acara Ujian

## Stack Lengkap

### Backend
| Komponen | Teknologi | Versi |
|----------|-----------|-------|
| Framework | Laravel | 11 |
| Auth | Laravel Sanctum | - |
| Database (dev) | SQLite | - |
| Database (prod) | MySQL | - |
| HTTP Server | PHP built-in / Nginx + PHP-FPM | - |
| Containerization | Docker + docker-compose | - |

### Frontend — Pengawas & Panitia (`frontend/`)
| Komponen | Teknologi | Versi |
|----------|-----------|-------|
| Framework | React | 19 |
| Build Tool | Vite | 7 |
| CSS | TailwindCSS | 4 |
| HTTP Client | Axios | 1.13 |
| QR Scanner | html5-qrcode | 2.3 |
| Signature | react-signature-canvas | 1.1 |
| Alert | SweetAlert2 | 11 |
| Excel Export | xlsx | 0.18 |

### Frontend — Admin (`frontend-admin/`)
| Komponen | Teknologi | Versi |
|----------|-----------|-------|
| Framework | React | 19 |
| Build Tool | Vite | 7 |
| CSS | TailwindCSS | 3.4 |
| Router | React Router DOM | 7 |
| Charts | Chart.js + react-chartjs-2 | 4.5 |
| Icons | Lucide React | 0.563 |
| HTTP Client | Axios | 1.13 |
| Alert | SweetAlert2 | 11 |

### Frontend — TV Monitor (`frontend-tv/`)
| Komponen | Teknologi | Versi |
|----------|-----------|-------|
| Framework | React | 18 |
| Build Tool | Vite | 5 |
| CSS | Pure CSS (no framework) | - |
| HTTP Client | Axios | 1.7 |
| Alert | SweetAlert2 | 11 |

---

## Arsitektur Sistem

```
┌─────────────────────────────────────────────────────────────────┐
│  BROWSER CLIENTS                                                 │
│                                                                  │
│  [Smartphone Pengawas/Panitia]  [PC Admin]  [TV Monitor + Barcode Reader] │
│         frontend/                frontend-admin/    frontend-tv/ │
│         :5174                    :5175              :5176        │
└──────────────────────┬───────────────────┬──────────┬──────────┘
                       │                   │          │
                       ▼                   ▼          ▼
              ┌──────────────────────────────────────────┐
              │           Laravel API Backend             │
              │           :8000 / /api/*                  │
              │                                           │
              │  Auth:    Sanctum (pengawas)               │
              │           Session localStorage (panitia)  │
              │                                           │
              │  Storage: /storage/app/public/            │
              │           signatures/  → TTD PNG          │
              │           bukti-izin/  → foto bukti izin  │
              │           pelanggaran/ → foto pelanggaran │
              └────────────────────┬─────────────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │  MySQL / SQLite  │
                          └─────────────────┘
```

## Folder Structure Kode

```
berita-acara-ujian-baruuuuuuu/
│
├── backend/
│   ├── app/
│   │   ├── Http/Controllers/    ← API controllers
│   │   ├── Models/              ← Eloquent models
│   │   └── Services/            ← Business logic (PresensiService, AssignmentService)
│   ├── database/
│   │   ├── migrations/          ← Schema migrations
│   │   └── seeders/             ← Initial data
│   ├── routes/
│   │   └── api.php              ← Semua API routes
│   └── storage/app/public/      ← File uploads (signatures, foto)
│
├── frontend/
│   └── src/
│       ├── App.jsx              ← Main app (Pengawas + Panitia mode)
│       ├── PanitiaDashboard.jsx ← Dashboard per ruang untuk panitia
│       ├── RekapData.jsx        ← Rekap kehadiran (panitia)
│       ├── RekapPengawas.jsx    ← Rekap jadwal pengawas
│       ├── PelanggaranModal.jsx ← Modal catatan ketidaktertiban
│       └── components/
│           ├── QRScanner.jsx    ← Kamera QR scanner
│           ├── SignaturePad.jsx ← Tanda tangan digital
│           └── BeritaAcaraDocument.jsx ← Preview dokumen BA
│
├── frontend-admin/
│   └── src/
│       ├── pages/               ← Halaman per fitur (CRUD)
│       ├── components/          ← Layout, Sidebar, ThemeToggle
│       └── context/             ← AuthContext, ThemeContext
│
└── frontend-tv/
    └── src/
        ├── App.jsx              ← Layout utama TV (4 panel)
        ├── components/
        │   ├── AbsenList.jsx    ← Daftar peserta belum hadir
        │   ├── CampusCard.jsx   ← Card per kampus
        │   ├── PengawasList.jsx ← List pengawas & panitia hadir
        │   └── BarcodeListener.jsx ← Listener keyboard barcode reader
        └── hooks/
            └── useAttendanceData.js ← Polling data API
```

## Port Convention (Development)

| Service | Port |
|---------|------|
| Backend Laravel | 8000 |
| Frontend Pengawas/Panitia | 5174 |
| Frontend Admin | 5175 |
| Frontend TV | 5176 |

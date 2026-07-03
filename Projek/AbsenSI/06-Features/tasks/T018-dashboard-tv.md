# T018 — Dashboard TV Realtime (/tv)

## Depends on
T017 (Socket.IO gateway harus sudah broadcast `attendance:today`)

## Objective
Buat halaman `/tv` di `apps/web` yang menampilkan rekap kehadiran hari ini secara realtime untuk ditampilkan di TV ruang kepala sekolah.

## Context
- **App:** `apps/web`
- **Route:** `/tv` (layout terpisah dari admin)
- **Role:** `kepsek`
- **ADR:** keputusan "route di apps/web, bukan app terpisah"
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-tv.md`

## Spec Detail

### API endpoint baru:
`GET /attendance/today-stats` (akses: authenticated — kepsek + super_admin)
- Return:
```json
{
  "tanggal": "2026-07-03",
  "total_siswa": 2500,
  "hadir": 2350,
  "terlambat": 45,
  "belum_hadir": 105,
  "updated_at": "07:45:00"
}
```

### Layout `/tv`:
- Buat `apps/web/app/tv/layout.tsx` — **terpisah** dari `app/(admin)/layout.tsx`
- Fullscreen, background gelap, font besar
- Tidak ada navbar, tidak ada sidebar

### Halaman `apps/web/app/tv/page.tsx`:
```
┌────────────────────────────────────────────┐
│  SMK [NAMA SEKOLAH]          07:45:32      │
│  Kamis, 3 Juli 2026                        │
├──────────────┬──────────────┬──────────────┤
│   HADIR      │  TERLAMBAT   │ BELUM HADIR  │
│              │              │              │
│    2.350     │      45      │     105      │
│              │              │              │
│   dari 2.500 siswa                         │
├────────────────────────────────────────────┤
│  Update terakhir: 07:45:00                 │
└────────────────────────────────────────────┘
```

### Data flow:
1. Page load → `GET /attendance/today-stats` (initial data)
2. Subscribe Socket.IO `attendance:today` → update angka realtime tanpa reload

### Auth & session:
- Akun `kepsek` login sekali, refresh token 30 hari dengan sliding renewal
- Middleware: kalau tidak ada valid session → redirect ke `/login?redirect=/tv`
- Setelah login, redirect kembali ke `/tv`
- `apps/web/app/tv/page.tsx` harus protected (cek session di server component atau middleware)

### Login page (kalau belum ada):
Buat `apps/web/app/login/page.tsx` — form sederhana username + password, POST ke `/auth/login`, simpan token ke httpOnly cookie atau localStorage (pilih satu, konsisten).

## JANGAN
- ❌ JANGAN buat TV dashboard di apps terpisah — ini route `/tv` di `apps/web` (sudah diputuskan)
- ❌ JANGAN tampilkan filter per kelas/jurusan di TV — agregat total saja
- ❌ JANGAN auto-refresh via polling (setInterval fetch) — gunakan Socket.IO yang sudah ada
- ❌ JANGAN buat TV layout memakai komponen dari `app/(admin)/layout.tsx` — buat layout baru terpisah yang benar-benar fullscreen

## Files
- **Buat:** `apps/web/app/tv/layout.tsx` (fullscreen layout)
- **Buat:** `apps/web/app/tv/page.tsx`
- **Buat:** `apps/web/app/login/page.tsx` (kalau belum ada)
- **Buat:** `apps/api/src/attendance/` — tambah endpoint `GET /attendance/today-stats`

## Acceptance Criteria
- [ ] `/tv` menampilkan 3 angka besar: Hadir, Terlambat, Belum Hadir
- [ ] Tap kartu di kiosk → angka di TV berubah dalam < 2 detik
- [ ] Akses `/tv` tanpa login → redirect ke `/login`
- [ ] TV layout fullscreen — tidak ada navbar/sidebar yang terlihat
- [ ] Jam di TV update setiap detik (client-side)

# T001 — Setup Monorepo Turborepo

## Depends on
Tidak ada. Ini task pertama.

## Objective
Inisialisasi struktur monorepo Turborepo yang menjadi fondasi semua apps dan packages proyek AbsenSI.

## Context
- **Repo:** GitHub baru (nama bebas, misal `absensi-smk`)
- **Apps yang dibuat:** `api` (NestJS), `web` (Next.js), `kiosk` (Next.js)
- **Packages yang dibuat:** `types`, `config`, `ui`
- **ADR:** ADR-007 (Monorepo Turborepo)

## Spec Detail

### Struktur folder target:
```
/
├── apps/
│   ├── api/          ← NestJS + Prisma
│   ├── web/          ← Next.js (admin + TV + piket + guru)
│   └── kiosk/        ← Next.js (kiosk gerbang, fullscreen)
├── packages/
│   ├── types/        ← shared TypeScript interfaces
│   ├── config/       ← shared tsconfig, tailwind, eslint
│   └── ui/           ← shadcn/ui components
├── turbo.json
├── package.json      ← root (workspace)
└── .env.example
```

### `packages/config` harus berisi:
- `tsconfig.base.json` — base TypeScript config yang di-extend semua apps
- `tailwind.config.ts` — base Tailwind config (warna sekolah dikosongkan dulu, isi placeholder)
- `eslint.config.js` — shared ESLint rules

### `packages/types` harus berisi:
- File `index.ts` dengan type/interface dasar yang akan dipakai lintas apps:
  - `Role` enum: `super_admin | card_admin | guru | kepsek | guru_piket`
  - `TapResult` enum: `accepted | rejected_inactive | rejected_locked | rejected_unknown | rejected_duplicate`
  - `PulangVia` enum: `tap | piket_izin | tap_izin_pulang`
  - `PermitJenis` enum: `tidak_masuk | keluar`
  - `StatusKembali` enum: `belum | sudah | pulang`

### `packages/ui` harus berisi:
Setup shadcn/ui. Komponen yang wajib di-copy sekarang:
- Button, Input, Label, Badge
- Table, Dialog, Form
- Select, DatePicker (Calendar + Popover)
- Skeleton (untuk loading state)

### `apps/api` — setup awal:
- NestJS baru via `nest new` (pilih npm/pnpm sesuai keputusan workspace)
- Hapus boilerplate yang tidak perlu (AppController hello world, dsb)
- Extend `tsconfig` dari `packages/config`

### `apps/web` dan `apps/kiosk` — setup awal:
- Next.js 14 App Router
- Pasang Tailwind dari `packages/config`
- Import komponen dari `packages/ui`
- Setup path alias `@/` untuk `./src`

### `.env.example` di root:
```
# Database
DATABASE_URL="mysql://root:password@localhost:3306/absensi_db"

# Redis
REDIS_URL="redis://localhost:6379"

# JWT
JWT_SECRET="ganti-dengan-string-random-panjang"
JWT_REFRESH_SECRET="ganti-dengan-string-random-panjang-berbeda"

# Kiosk
KIOSK_DEVICE_TOKEN="ganti-dengan-token-random-256bit"

# Print server
PRINT_SERVER_URL="http://10.10.10.100:8800/print.php"
```

## JANGAN
- ❌ JANGAN install library di luar yang disebutkan di T001 ini — library tambahan (JWT, Prisma, Socket.IO, dsb) dipasang di task masing-masing
- ❌ JANGAN buat schema Prisma sekarang — itu task T002
- ❌ JANGAN buat halaman/route di apps/web atau apps/kiosk selain setup dasar — itu task berikutnya
- ❌ JANGAN gunakan `create-turbo` template yang opinionated — mulai dari yang bersih supaya struktur sesuai rancangan
- ❌ JANGAN buat folder `apps/tv` terpisah — Dashboard TV adalah route `/tv` di dalam `apps/web` (ADR sudah diputuskan)

## Files
- **Buat:** semua folder dan file sesuai struktur di atas
- **Jangan sentuh:** tidak ada (task pertama, belum ada yang bisa salah disentuh)

## Acceptance Criteria
- [ ] `turbo build` dari root berjalan tanpa error (walaupun semua apps masih kosong)
- [ ] `packages/types` bisa di-import dari `apps/api`, `apps/web`, dan `apps/kiosk`
- [ ] `packages/ui` — komponen shadcn/ui bisa di-import di `apps/web`
- [ ] `.env.example` sudah ada di root
- [ ] `.gitignore` sudah include `.env`, `node_modules`, `.next`, `dist`
- [ ] Repo sudah di-push ke GitHub

## Handoff ke T002
T002 akan menambahkan Prisma ke `apps/api`. Pastikan `apps/api` sudah bisa di-run (`nest start`) meskipun belum ada modul apapun.

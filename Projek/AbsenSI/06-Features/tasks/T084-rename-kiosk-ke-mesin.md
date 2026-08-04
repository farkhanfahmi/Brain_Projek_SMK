# T084 — UI: Ganti Istilah "Kiosk" Menjadi "Mesin" di Seluruh Tampilan

## Depends on
Tidak ada — murni perubahan teks/label yang ditampilkan ke user, TIDAK mengubah kode/nama teknis.

## Context
- **App:** `apps/web`
- **File:** `apps/web/src/app/(admin)/kiosk/kiosk-view.tsx`, `apps/web/src/app/(admin)/kiosk/page.tsx`, sidebar navigasi
- **Ref:** Diminta user 2026-07-24 — istilah "Kiosk" di UI diganti "Mesin" (lebih familiar untuk staf sekolah non-teknis).

## Spec Detail

### Yang DIGANTI (teks tampilan ke user)
Semua string yang benar-benar tampil ke user, contoh yang ditemukan di `kiosk-view.tsx`:
- `usePageTitle("Kiosk")` → `usePageTitle("Mesin")`
- "Daftar Kiosk" → "Daftar Mesin"
- "Tambah Kiosk" → "Tambah Mesin"
- "Nama Kiosk" → "Nama Mesin"
- "Tipe Kiosk" → "Tipe Mesin"
- "URL Kiosk — {kiosk.nama}" → "URL Mesin — {kiosk.nama}"
- "Nonaktifkan Kiosk" → "Nonaktifkan Mesin"
- Teks body "Kiosk tidak akan bisa memproses tap..." → "Mesin tidak akan bisa memproses tap..."
- Label menu sidebar "Kiosk" → "Mesin" (cek komponen sidebar admin)
- Cek juga halaman lain yang MUNGKIN reference kiosk secara user-facing: `apps/kiosk` app itu sendiri (halaman fisik yang dipakai di gerbang — cek apakah ada teks "Kiosk" yang tampil ke siswa/operator gerbang, misal di `apps/kiosk/src/app/` halaman utama tap), notifikasi/toast terkait kiosk di halaman lain (misal dashboard admin yang menyebut status kiosk online/offline)

### Yang TIDAK BOLEH DIGANTI (nama teknis, bukan tampilan)
- Nama type TypeScript `Kiosk`, `KioskTipe` (di `core-types.ts`, `@absensi/types`)
- Nama model Prisma `Kiosk` di schema, nama tabel `kiosks`/`kiosk_id` di DB
- Route URL `/kiosk` (path admin) — TIDAK perlu diganti `/mesin`, cukup label yang tampil di sidebar/judul halaman yang berubah, breaking route change tidak diminta dan berisiko rusak bookmark/link existing
- Nama file (`kiosk-view.tsx`, `kiosk.controller.ts`, dst) — TIDAK perlu rename file
- Env var, nama variabel kode (`kioskId`, `KIOSK_DEVICE_TOKEN`, dst di `.env`/`main.ts`) — ini konfigurasi teknis, bukan tampilan

### Cara Verifikasi Lengkap
Jalankan `grep -rn "Kiosk" apps/web/src --include="*.tsx"` SETELAH selesai — hasil yang tersisa HARUSNYA cuma import type (`import type { Kiosk } from ...`) dan reference variable/prop teknis, BUKAN string yang di-render ke JSX teks (`<h2>...Kiosk...</h2>`, label, placeholder, dst).

## JANGAN
- ❌ JANGAN rename type/model/route/nama file — HANYA teks yang tampil ke user (JSX text content, label, title, placeholder, alt text, toast/error message)
- ❌ JANGAN lupa cek `apps/kiosk` (aplikasi terpisah untuk device fisik) kalau ada teks user-facing di sana yang menyebut "Kiosk" — device ini yang dioperasikan langsung oleh siswa/petugas gerbang, konsistensi istilah penting di situ juga

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kiosk/kiosk-view.tsx` — semua string JSX yang mengandung "Kiosk"
- **Modifikasi:** `apps/web/src/app/(admin)/kiosk/page.tsx` — kalau ada teks di situ
- **Modifikasi:** komponen sidebar admin (cari label menu "Kiosk")
- **Cek+modifikasi jika perlu:** `apps/kiosk/src/app/**/*.tsx` — teks user-facing di device fisik

## Acceptance Criteria
- [ ] `grep -rn "Kiosk" apps/web/src --include="*.tsx"` hanya menyisakan import type/nama variabel teknis, tidak ada lagi string JSX yang tampil ke user
- [ ] Sidebar admin menampilkan label "Mesin", bukan "Kiosk"
- [ ] Semua dialog/form terkait (Tambah, URL, Nonaktifkan) menampilkan "Mesin"
- [ ] Route `/kiosk` (URL) TIDAK berubah — hanya label yang berubah
- [ ] `pnpm turbo run build` bersih (pastikan tidak ada typo/broken reference dari find-replace manual)

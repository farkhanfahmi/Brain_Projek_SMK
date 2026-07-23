# T064 — Primitif UI Baru: Sheet & Tabs

## Depends on
Tidak ada — murni komponen shadcn/ui baru, fondasi untuk T065/T066.

## Objective
Tambahkan primitif `Sheet` (panel geser dari sisi kanan) dan `Tabs` (navigasi section pill-style) ke `packages/ui` — dasar untuk pola form input panjang yang menggantikan popup `Dialog` kecil, sesuai spec baru di `03-components.md` bagian "Form Input Panjang".

## Context
- **App:** `packages/ui`
- **Ref:** `Projek/AbsenSI/06-Features/design-system/03-components.md` — bagian "Form Input Panjang (Sheet, bukan Dialog kecil)" (ditambahkan 2026-07-22, baca detail lengkap sebelum mulai)
- **Referensi shadcn/ui asli:** komponen ini standar di ekosistem shadcn — cara paling cepat & konsisten adalah `npx shadcn@latest add sheet tabs` dari root `packages/ui` (menyesuaikan import path/alias yang sudah dipakai proyek ini), BUKAN menulis dari nol

## Spec Detail

### `Sheet` — cek dulu komponen `Dialog` existing (`packages/ui/src/components/ui/dialog.tsx`) untuk pola konsistensi (radix-ui primitive, className convention, forwardRef pattern) sebelum generate
- Varian yang dibutuhkan: `side="right"` (default proyek ini, sesuai spec — form selalu geser dari kanan, TIDAK perlu varian left/top/bottom kecuali ada kebutuhan lain nanti)
- Styling wajib sesuai spec: lebar 480-560px desktop, radius `24px` HANYA di sudut kiri (atas+bawah), shadow `shadow-popover`, backdrop warm-tone (bukan hitam pekat default shadcn)
- Struktur: `SheetTrigger`, `SheetContent`, `SheetHeader`, `SheetTitle`, `SheetDescription`, `SheetFooter` (footer WAJIB sticky di bawah, tidak ikut scroll body)

### `Tabs` — styling KHUSUS proyek ini, BUKAN default shadcn underline-style
- Default shadcn `Tabs` pakai underline-indicator — **GANTI** stylingnya jadi pill-style sesuai `03-components.md`: tab tidak aktif = Secondary Button spec (bg putih, border tipis), tab aktif = Primary Button spec (bg `--color-primary`, teks putih), semua `radius-full`
- Struktur: `Tabs`, `TabsList`, `TabsTrigger`, `TabsContent` (API sama seperti shadcn standar, styling internal yang beda)

## JANGAN
- ❌ JANGAN pakai styling default shadcn untuk `Tabs` (underline) — WAJIB pill-style sesuai design system proyek ini, ini bukan preferensi kosmetik tapi konsistensi dengan seluruh sistem badge/button yang sudah pill-shaped
- ❌ JANGAN buat `Sheet` dengan radius di SEMUA sudut — hanya sisi kiri (yang menghadap konten), sisi kanan menempel ke tepi viewport jadi tidak perlu radius
- ❌ JANGAN buat backdrop hitam pekat — ikuti warna backdrop warm-tone yang sudah dipakai `Dialog` existing untuk konsistensi

## Files
- **Buat:** `packages/ui/src/components/ui/sheet.tsx`
- **Buat:** `packages/ui/src/components/ui/tabs.tsx`
- **Modifikasi:** `packages/ui/src/index.ts` (atau file export utama) — tambah export `Sheet`, `SheetTrigger`, `SheetContent`, dst, dan `Tabs`, `TabsList`, `TabsTrigger`, `TabsContent`

## Acceptance Criteria
- [ ] `Sheet` bisa diimpor dari `@absensi/ui`, render dari sisi kanan, lebar 480-560px, radius hanya sudut kiri
- [ ] `Tabs` bisa diimpor dari `@absensi/ui`, tab aktif berwarna oranye solid + teks putih, tab tidak aktif putih dengan border tipis — TIDAK ada underline indicator
- [ ] Footer di dalam `Sheet` tetap terlihat (sticky) saat body di-scroll dengan konten panjang (uji dengan dummy 20 field)

## Handoff ke T065 & T066
T065 (form Guru) dan T066 (form Siswa) akan memakai `Sheet`+`Tabs` ini sebagai kontainer form baru, menggantikan `Dialog` existing.

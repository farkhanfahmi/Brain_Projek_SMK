# T057 — Tambah Token Warna Status Workflow (Shipped/Processing) ke Tailwind Config

## Depends on
Tidak ada — task ini murni menambah token CSS variable + Tailwind config, tidak bergantung ke task manapun. Bisa dikerjakan kapan saja sebelum T058/T059 (yang MEMAKAI token ini).

## Objective
Tambahkan 2 token warna baru (`status-shipped`, `status-processing`) ke `globals.css`/`tailwind.config.ts` — hasil audit gambar referensi asli EzMart (2026-07-22) yang menemukan status badge di tabel data ternyata pakai 4 kategori warna (amber/violet/hijau/merah), bukan cuma 2 (success/danger) seperti token yang sudah ada.

## Context
- **App:** `packages/config`, `apps/web`
- **Ref:** `Projek/AbsenSI/06-Features/design-system/01-colors.md` (baris token baru "Added 2026-07-22") dan `03-components.md` bagian "Data Table" — baca dulu, ini yang jadi sumber kebenaran token task ini
- **File token existing:** `packages/config/tailwind.config.ts`, dan file CSS variable-nya (kemungkinan `apps/web/src/app/globals.css` — cek definisi `--color-success-bg` dkk untuk pola yang sama)

## Spec Detail

### Token CSS variable baru (tambahkan di tempat yang sama `--color-success-bg` dkk didefinisikan)
```css
--color-status-shipped-bg: /* HSL dari #FDECD1 */;
--color-status-shipped-text: /* HSL dari #B8720A */;
--color-status-processing-bg: /* HSL dari #EDE3F7 */;
--color-status-processing-text: /* HSL dari #7C4FC7 */;
```
- Konversi hex ke HSL mengikuti pola yang SAMA seperti token existing (cek bagaimana `--color-success-bg` dari `#DCF5E3` dikonversi, ikuti format itu persis — biasanya format `H S% L%` tanpa fungsi `hsl()` di definisi variable, dipakai lewat `hsl(var(--x))` di tempat konsumsi)
- **Kalau proyek pakai dark mode** (cek apakah ada blok `:root[data-theme="dark"]` atau `.dark` di file yang sama) — tambahkan varian dark untuk token baru ini juga, konsisten dengan pola existing untuk success/danger

### Tailwind config (`packages/config/tailwind.config.ts`)
Tambahkan ke `theme.extend.colors`, sejajar dengan `success`/`danger` yang sudah ada:
```ts
status: {
  shipped: {
    bg: "hsl(var(--color-status-shipped-bg))",
    text: "hsl(var(--color-status-shipped-text))",
  },
  processing: {
    bg: "hsl(var(--color-status-processing-bg))",
    text: "hsl(var(--color-status-processing-text))",
  },
},
```
- Hasil pemakaian di className nanti: `bg-status-shipped-bg text-status-shipped-text`, dst — pola penamaan HARUS konsisten dengan `bg-success-bg text-success-text` yang sudah dipakai di kode existing (lihat `izin-table.tsx` sebagai referensi pola)

### Perbaiki komentar path yang usang
Baris 6-7 `tailwind.config.ts` menyebut path vault `session Claude/design-system/` — **path ini sudah tidak akurat**, lokasi sebenarnya `06-Features/design-system/`. Perbaiki komentar ini sekalian (celah kecil yang ditemukan saat audit, tidak berhubungan langsung dengan token baru tapi murah untuk diperbaiki sekalian).

## JANGAN
- ❌ JANGAN pakai token baru ini di luar konteks status workflow tabel (chart, KPI delta, toggle A/B seperti kalender blok T056 tetap wajib monokrom oranye atau success/danger) — baca ulang catatan di `01-colors.md`: ini pengecualian SEMPIT, bukan pembuka jalan warna bebas di tempat lain
- ❌ JANGAN buat token warna tambahan lain di luar yang 2 ini tanpa rujukan eksplisit ke gambar/dokumen desain — kalau butuh warna lagi nanti, itu perubahan terpisah yang perlu didiskusikan
- ❌ JANGAN ubah nilai token existing (`--color-success-bg` dkk) — task ini murni menambah, tidak mengubah yang sudah ada dan sudah dipakai luas di kode

## Files
- **Modifikasi:** `packages/config/tailwind.config.ts`
- **Modifikasi:** file CSS variable global (cek `apps/web/src/app/globals.css` atau setara — samakan dengan yang dipakai `apps/kiosk` juga kalau token warna di-share lewat `packages/config`)

## Acceptance Criteria
- [ ] `bg-status-shipped-bg`, `text-status-shipped-text`, `bg-status-processing-bg`, `text-status-processing-text` bisa dipakai sebagai className Tailwind tanpa error build
- [ ] Warna yang tampil visual cocok dengan hex yang ditentukan (`#FDECD1`/`#B8720A`/`#EDE3F7`/`#7C4FC7`) — verifikasi dengan screenshot Playwright di komponen manapun yang memakainya (lihat T058)
- [ ] `pnpm turbo run build` tidak error setelah perubahan config ini
- [ ] Komentar path vault di `tailwind.config.ts` sudah diperbaiki ke `06-Features/design-system/`

## Handoff ke T058
T058 (Data Table component baru) akan memakai token ini untuk status badge "Shipped"/"Processing" — pastikan token ini tersedia dan teruji dulu sebelum T058 dimulai.

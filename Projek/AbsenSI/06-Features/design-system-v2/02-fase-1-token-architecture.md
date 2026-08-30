---
tags: [absensi, design-system-v2, fase-1]
created: 2026-08-28
status: selesai
---

# Fase 1 -- Token Architecture

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2.

## Tujuan Fase

Membangun lapisan token 3-tingkat (Primitive -> Semantic -> Component) dalam format JSON di repo, sebagai source of truth SESUNGGUHNYA -- bukan Figma. AI coding agent membaca file ini langsung, tanpa perlu akses Figma.

## Lokasi Artefak

- `C:\ProjekSMK\AbsenSI\packages\design-tokens\src\primitive.tokens.json`
- `C:\ProjekSMK\AbsenSI\packages\design-tokens\src\semantic.tokens.json`
- `C:\ProjekSMK\AbsenSI\packages\design-tokens\README.md` (aturan wajib untuk AI agent)
- `C:\ProjekSMK\AbsenSI\packages\design-tokens\package.json`

## Warna Inti (Locked, dari Product Owner)

| Token | Hex | Status |
|---|---|---|
| color.terracotta.500 | #C1452C | LOCKED EXACT |
| color.cream.50 | #FFFEE0 | LOCKED EXACT |

Kedua warna ini diverifikasi WCAG 2.1 AA sebelum dikunci:

| Pasangan | Rasio | Verdict |
|---|---|---|
| Teks putih di atas Terracotta (tombol) | 5.05:1 | PASS AA |
| Teks Terracotta di atas Cream (link/aksen) | 4.93:1 | PASS AA |
| Teks Terracotta di atas kartu putih | 5.05:1 | PASS AA |
| Teks gelap #171412 di atas Cream baru | 17.90:1 | PASS AAA |

Catatan: Terracotta baru (hue 10 derajat, arah oranye-merah) berbeda arah hue dari primary v1 lama (#BA5C08, hue 28 derajat, arah oranye-coklat) -- pergeseran identitas visual yang disengaja sesuai niat revamp total.

## Color Ramp Primitif (10 langkah, 50-900)

### Terracotta (anchor 500 = #C1452C exact)

| Step | Hex |
|---|---|
| 50 | #FEF3F1 |
| 100 | #FBE4DF |
| 200 | #F5C6BC |
| 300 | #EB9E8E |
| 400 | #DC6C56 |
| 500 | #C1452C (LOCKED) |
| 600 | #A43B25 |
| 700 | #833120 |
| 800 | #612519 |
| 900 | #3F1A12 |

### Cream (bg-tint scale, hanya 50-400 -- bukan grayscale penuh)

| Step | Hex |
|---|---|
| 50 | #FFFEE0 (LOCKED) |
| 100 | #F5F4D6 |
| 200 | #E6E5C1 |
| 300 | #D4D3AF |
| 400 | #BFBE9B |

### Neutral (warm gray, hue 30 derajat -- harmonis antara cream dan terracotta; step 950 mewarisi nilai teks gelap lama yang sudah terbukti WCAG-aman)

| Step | Hex |
|---|---|
| 50 | #FBFAF8 |
| 100 | #F6F4F1 |
| 200 | #E9E6E2 |
| 300 | #D1CCC7 |
| 400 | #B2ABA4 |
| 500 | #928A81 |
| 600 | #726B65 |
| 700 | #56524D |
| 800 | #3C3834 |
| 900 | #24211E |
| 950 | #171412 (warisan legacy text/primary, WCAG-audited 2026-07-23) |

## Warna Semantik/Feedback (Bebas Diubah -- Sudah Diverifikasi WCAG)

| Token | Hex | Rasio vs bg pasangan | Verdict |
|---|---|---|---|
| success.600 | #187C3C | 4.67:1 (on success.50), 5.27:1 (on white) | PASS AA. Reuse dari v1 -- sudah teraudit sebelumnya. |
| warning.600 | #8A5A08 | 5.21:1 (on warning.50), 5.92:1 (on white) | PASS AA. Baru -- v1 tidak punya token warning terpisah. |
| danger.600 | #B8202F | 5.28:1 (on danger.50), 6.39:1 (on white) | PASS AA. Hue 354 derajat -- sengaja dibuat berbeda jauh dari terracotta (hue 10 derajat) supaya error TIDAK ambigu dengan warna brand. |
| info.600 | #1D4FA8 | 6.51:1 (on info.50), 7.68:1 (on white) | PASS AA. Baru -- v1 tidak punya token info sama sekali. |

## Palet Kategorikal Chart/Data-Viz (BARU -- v1 tidak punya sama sekali)

Dirancang khusus untuk 8 status kehadiran nyata dari kode (`rekap-view.tsx`), menggantikan warna Material Design hardcode yang ditemukan di Fase 0:

| Token | Hex | Status Absensi | Rasio sebagai teks di atas putih |
|---|---|---|---|
| chart.categorical-1 | #288A4C | hadir | 4.35:1 |
| chart.categorical-2 | #BB811B | terlambat | 3.35:1 |
| chart.categorical-3 | #3462B2 | izin | 5.94:1 |
| chart.categorical-4 | #8D43B1 | sakit | 5.88:1 |
| chart.categorical-5 | #308191 | dispen | 4.49:1 |
| chart.categorical-6 | #938C85 | belum_absen | 3.32:1 |
| chart.categorical-7 | #B1437A | tugas_dinas | 5.31:1 |
| chart.categorical-8 | #C1452C | pelatihan (reuse terracotta) | 5.05:1 |

Semua 8 warna dirancang dengan lightness/saturation serupa (agar "berat visual" merata di chart bar/pie) dan hue tersebar merata di roda warna untuk keterbedaan maksimal.

## Struktur Token Non-Warna (Semua BARU -- Tidak Ada di v1)

- **Spacing scale**: 0, 4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 80, 96px (skala geometris konsisten, dulu tidak ada sebagai variable sama sekali).
- **Elevation**: 5 tingkat (0-4) dengan definisi shadow eksplisit untuk resting/hover/overlay/toast.
- **Z-index**: 7 lapis eksplisit (base, sticky, dropdown, overlay, modal, toast, tooltip) -- dulu tidak terdokumentasi sama sekali, resiko konflik layering.
- **Motion**: durasi (fast/base/slow) + easing curve (standard/decelerate/accelerate) -- dulu tidak ada token, cuma hardcode class Tailwind di tiap file.
- **Breakpoint**: 375/768/1024/1440px -- akhirnya terdokumentasi sebagai token resmi (bukan cuma disebut di teks audit).

## Struktur Typography Semantik

8 text style (display, heading-1/2/3, body, body-medium, label, caption) plus 1 BARU: `numeric-table` dengan `font-variant-numeric: tabular-nums` untuk kolom angka di tabel (jam, tanggal, nominal) -- mengisi gap yang ditemukan di audit typography sebelumnya.

Perbaikan dari v1:
- `fontFamily.sans` v2 TIDAK lagi punya fallback "Inter" yang salah (font itu tidak pernah dimuat browser).
- `display` dan `heading-1` sekarang punya letter-spacing negatif kecil (-0.01em, -0.005em) untuk teks besar -- v1 semua 0%.

## Validasi Teknis

Semua 39 referensi alias di `semantic.tokens.json` diverifikasi berhasil resolve ke `primitive.tokens.json` tanpa broken reference (dicek programatis, bukan visual).

## Keputusan Arsitektur

1. Token disimpan di repo (`packages/design-tokens/`), BUKAN di vault Obsidian -- vault berisi dokumentasi naratif/rasional (dokumen ini), repo berisi source of truth mesin-bisa-baca.
2. Semantic layer WAJIB dipakai komponen, primitive TIDAK BOLEH direferensi langsung dari kode aplikasi.
3. Token v1 (lama) tetap hidup berdampingan selama migrasi (coexistence strategy) -- detail di Fase 6.
4. Warna kategorikal chart sengaja dipisah dari warna feedback semantik (success/warning/danger/info) karena keduanya punya tujuan berbeda: feedback = status UI, kategorikal = pembeda visual data.

## Belum Dikerjakan di Fase Ini (menyusul di fase lanjutan)

- `component.tokens.json` (override per komponen) -- menyusul Fase 3.
- Generator script otomatis token JSON -> CSS variables/Tailwind config -> Figma Variables -- menyusul Fase 5-6.
- Dark mode variant -- belum diputuskan scope-nya, perlu klarifikasi product owner.

## Cermin Visual di Figma (Lapis 2)

Page baru terpisah **"Design System v2"** dibuat di file Figma AbsenSI (terpisah dari page "Design System" v1 lama -- tidak menimpa/menghapus).

Isi yang sudah dibangun dan diverifikasi visual (screenshot + vision check):

| Section | Isi | Status |
|---|---|---|
| Cover | Judul + tagline + versi, background Cream, judul Terracotta | ✅ |
| Color Primitives | 42 swatch (Terracotta/Cream/Neutral ramp + success/warning/danger/info + 8 chart) | ✅ |
| Color Semantic | 37 swatch alias (bg/text/border/brand/feedback/chart) | ✅ |
| Spacing Scale | 13 bar progresif 0-96px, width di-bind ke variable | ✅ |
| Radius Scale | 6 kotak none/sm/md/lg/xl/full, corner radius di-bind ke variable | ✅ |
| Elevation Scale | 4 kartu shadow progresif (Resting/Hover/Overlay/Toast) via Effect Styles | ✅ (1 bug clipping ditemukan & diperbaiki: padding bottom kurang untuk shadow besar) |
| Type Ramp | 9 Text Style (termasuk Numeric Table baru) | ✅ |

Variable Collections Figma yang dibuat (semua prefix `v2/` agar tidak bentrok dengan koleksi v1 lama):
- `v2/Color/Primitive` (42 variable)
- `v2/Color/Semantic` (37 variable, alias ke Primitive)
- `v2/Spacing` (13 variable)
- `v2/Radius` (6 variable)
- `v2/Typography` (9 variable: 1 font-family + 8 font-size)
- 4 Effect Styles (`Elevation/1-4`)
- 9 Text Styles (`v2/Display` s.d. `v2/Numeric Table`)

Peran page ini: cermin visual untuk review manusia. TIDAK dipakai AI agent sebagai source of truth saat menulis kode -- itu tetap `packages/design-tokens/*.tokens.json`.

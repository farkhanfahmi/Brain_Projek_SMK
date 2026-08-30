---
tags: [absensi, design-system-v2, fase-2]
created: 2026-08-29
status: selesai
---

# Fase 2 -- Foundation Rules

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2.

## Tujuan Fase

Fase 1 mendefinisikan NILAI token (hex, px, ms). Fase 2 mendefinisikan ATURAN PEMAKAIAN token itu -- kapan pakai spacing yang mana, breakpoint mana untuk device apa, z-index mana untuk layer apa. Tanpa fase ini, token cuma daftar angka tanpa konteks -- AI agent akan menebak cara pakainya, dan tebakan itulah sumber halusinasi yang ingin dihindari.

## Lokasi Artefak

- `C:\ProjekSMK\AbsenSI\packages\design-tokens\src\foundation-rules.json`

Semua referensi di file ini (`{primitive.breakpoint.mobile}`, dst) diverifikasi resolve ke `primitive.tokens.json` tanpa broken link.

## 1. Grid & Breakpoint System

Sistem grid PENUH per breakpoint (bukan cuma angka breakpoint seperti sebelumnya) -- container width, jumlah kolom, gutter, dan padding container semua berbeda per device:

| Breakpoint | Range | Kolom | Gutter | Padding Container |
|---|---|---|---|---|
| mobile | 375-767px | 4 | 12px | 16px |
| tablet | 768-1023px | 8 | 16px | 24px |
| desktop | 1024-1439px | 12 | 24px | 32px |
| wide | 1440px+ | 12 (max-width 1440px, di-center) | 24px | 40px |

Prinsip: mobile-first WAJIB -- breakpoint lain adalah enhancement dari base mobile, bukan sebaliknya. Ini menegaskan ulang preferensi kerja Anda yang sudah lama dipegang.

### Dashboard Widget Span

Karena fokus utama revamp ini data-dense dashboard, ditambahkan aturan span kolom widget (kartu KPI, chart card) per ukuran:

| Ukuran Widget | Kolom Desktop | Kolom Tablet | Kolom Mobile |
|---|---|---|---|
| small | 3 dari 12 | 4 dari 8 | 4 dari 4 (full-width) |
| medium | 6 dari 12 | 8 dari 8 (full-width) | 4 dari 4 |
| large | 8 dari 12 | 8 dari 8 | 4 dari 4 |
| full | 12 dari 12 | 8 dari 8 | 4 dari 4 |

## 2. Table Density Mode

BARU -- v1 tidak punya mode density sama sekali (gap yang ditemukan di audit awal). Sekarang WAJIB pilih salah satu secara eksplisit per tabel:

| Mode | Row Height | Kapan Dipakai |
|---|---|---|
| compact | 36px | Tabel >50 baris/halaman, power-user admin, data historis (rekap, riwayat) |
| comfortable (default) | 48px | Tabel dengan aksi per baris (edit/hapus), <=50 baris/halaman |

## 3. Touch Target Minimum

WAJIB, bukan saran: **44px minimum** (mengikuti Apple HIG, sedikit lebih ketat dari Material Design 48dp) untuk semua elemen interaktif. Pengecualian: ikon inline di teks/tabel padat boleh visual lebih kecil (24px) SELAMA hit-area transparan tetap >=44px.

## 4. Elevation Layering (Z-Index)

Peta z-index eksplisit -- BARU, v1 tidak mendokumentasikan sama sekali (gap kritis dari audit awal, berisiko konflik layering dropdown vs modal vs toast):

| Layer | Z-Index | Contoh Pemakaian |
|---|---|---|
| base | 0 | Konten halaman normal, Card dashboard, Table |
| sticky | 100 | Sticky table header, sticky filter bar |
| dropdown | 1000 | Select content, Popover, Date Picker calendar |
| overlay | 1100 | Dialog/Sheet backdrop |
| modal | 1200 | Dialog/Sheet content |
| toast | 1300 | Toast/Snackbar |
| tooltip | 1400 | SELALU paling atas -- termasuk di atas modal |

Aturan eksplisit: tooltip di dalam modal harus tetap terlihat di atasnya (1400 > 1200) -- ini mencegah bug umum "tooltip ketutup modal" yang sering terjadi kalau z-index tidak direncanakan.

## 5. Motion Usage

BARU -- v1 tidak punya token motion sama sekali. Sekarang ada 4 kategori pemakaian eksplisit:

| Kategori | Durasi | Easing | Contoh |
|---|---|---|---|
| micro_interaction | 120ms | standard | Checkbox toggle, Switch toggle, chevron rotate |
| standard_transition | 200ms | standard | Hover, dropdown open, tab switch |
| surface_transition | 320ms | decelerate | Dialog muncul, Sheet slide-in |
| exit_transition | 200ms | accelerate | Dialog/Sheet menutup, Toast hilang |

Aturan aksesibilitas wajib: SEMUA transisi harus hormati `prefers-reduced-motion: reduce` -- durasi diturunkan ke 0ms atau diganti fade sederhana, tidak boleh diabaikan.

## 6. Kontrak Aksesibilitas

Ini bagian paling penting secara filosofis: kontrak, bukan rekomendasi. AI agent WAJIB verifikasi sebelum menganggap komponen selesai.

### Kontras Minimum
- Teks normal di atas background: 4.5:1 (WCAG AA)
- Teks besar (>=18px bold atau >=24px regular): 3:1
- Border/ikon yang membawa makna: 3:1

### Warna Bukan Satu-Satunya Indikator (WCAG 1.4.1)
Status/makna TIDAK BOLEH hanya dibedakan lewat warna -- wajib disertai bentuk/ikon/teks berbeda.

- Contoh PATUH: StatusBadge v1 sudah benar sejak awal -- tiap varian (success/danger/shipped/processing/neutral) punya ikon Lucide berbeda, bukan cuma warna beda.
- Contoh PELANGGARAN yang harus diperbaiki: chart hardcode warna di `rekap-view.tsx` tanpa legend/label eksplisit -- item aksi konkret untuk Fase 4 (Data-Dense Pattern Library).

### Focus Visible
Setiap elemen interaktif (button, input, link, tab, row yang bisa diklik) WAJIB punya ring focus-visible: width 2px, offset 2px, warna dari `semantic.color.border.focus` (alias ke terracotta.500). Tidak boleh dihilangkan demi estetika -- ini keputusan non-negotiable untuk aksesibilitas keyboard.

### Token Terbatas (Restricted)
`text/tertiary` HANYA boleh dipakai untuk teks besar (>=18px) atau ikon, TIDAK untuk body text/label penting -- rasio kontrasnya di bawah standar AA untuk teks kecil (ini bug v1 yang sudah diperbaiki nilainya di Fase 1, tapi tetap dibatasi pemakaiannya di sini sebagai lapis proteksi kedua).

## Keputusan Arsitektur

1. Foundation Rules dipisah dari Semantic Tokens (file JSON terpisah) karena tujuannya beda: semantic = NILAI, foundation-rules = ATURAN PAKAI. AI agent membaca keduanya bersamaan sebelum menulis kode.
2. Semua rule ditulis sebagai kontrak terverifikasi (angka pasti, bukan deskripsi kualitatif seperti "cukup besar" atau "kira-kira") -- ini yang membedakan design system rigid dari yang longgar.
3. Z-index memakai skala lompatan besar (100 per layer, bukan 1,2,3) supaya ada ruang insert layer baru di masa depan tanpa renumbering semua.

## Cermin Visual di Figma (Lapis 2)

Meski sebagian besar Foundation Rules bersifat kontrak numerik (cocoknya di JSON), 2 aturan yang paling butuh verifikasi visual manusia tetap dibangun di page "Design System v2":

| Section | Isi | Status |
|---|---|---|
| Grid System | 3 demo grid (Mobile 4-kol, Tablet 8-kol, Desktop 12-kol) dengan kolom oranye transparan sesuai gutter/padding asli | ✅ Terverifikasi visual |
| Z-Index Layering | Diagram isometric 7 kartu bertumpuk (base→sticky→dropdown→overlay→modal→toast→tooltip), warna makin gelap ke atas | ✅ Terverifikasi visual (1 bug ditemukan & diperbaiki: offset diagonal awal terlalu kecil sehingga nama layer bawah tertutup kartu di atasnya — diperbaiki dari 40px ke 55px offset) |

Table Density, Touch Target, Motion Usage, dan Kontrak Aksesibilitas **sengaja tidak dibuat versi Figma** — sifatnya aturan/kontrak tekstual (angka minimum, kapan pakai apa), representasi visual tidak menambah kejelasan dibanding tabel di dokumen ini dan JSON. Kalau nanti terasa perlu (mis. demo before/after kontras warna), bisa ditambah di Fase 5.

## Status Akhir Fase 2

Ketiga lapis selesai:
- ✅ Repo: `foundation-rules.json` (7.3KB, semua alias tervalidasi resolve ke primitive)
- ✅ Figma: Grid System + Z-Index diagram, terverifikasi visual
- ✅ Obsidian: dokumen ini

## Belum Dikerjakan (menyusul fase lanjutan)

- Automated lint/checker yang membaca foundation-rules.json untuk validasi kode otomatis -- menyusul Fase 6 (Governance).

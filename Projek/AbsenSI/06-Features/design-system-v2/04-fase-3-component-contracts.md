---
tags: [absensi, design-system-v2, fase-3]
created: 2026-08-29
status: selesai
---

# Fase 3 -- Component API Contracts

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2.

## Tujuan Fase

Fase 1-2 mendefinisikan NILAI dan ATURAN PEMAKAIAN token. Fase 3 mengikat keduanya ke level KOMPONEN -- kontrak presisi yang menjawab pertanyaan paling sering menyebabkan halusinasi AI agent: "props apa yang valid?", "state apa saja yang harus ada?", "token mana yang dipakai di bagian mana?", "kapan komponen ini dipakai vs komponen lain yang mirip?".

## Lokasi Artefak

- `C:\ProjekSMK\AbsenSI\packages\design-tokens\src\component-contracts\_schema.json` -- meta-schema wajib diikuti semua kontrak.
- `C:\ProjekSMK\AbsenSI\packages\design-tokens\src\component-contracts\*.json` -- 8 kontrak komponen.

Semua 24 referensi token (`semantic.*`) di seluruh kontrak diverifikasi resolve ke `semantic.tokens.json` tanpa broken reference.

## Meta-Schema: Struktur Wajib Tiap Kontrak

Setiap file kontrak WAJIB berisi field berikut (didefinisikan di `_schema.json`):

| Field | Isi |
|---|---|
| name, category, status | Identitas dan status (stable/new/deprecated) |
| description | 1-2 kalimat kapan dipakai |
| sourceOfTruthCode | Path file kode React, atau 'BELUM ADA' jika status=new |
| props | Daftar props valid -- AI agent DILARANG mengarang prop di luar ini |
| variantAxes | Axis untuk sinkron ke Figma Component Properties |
| states | Daftar state WAJIB didukung |
| stateRules | Perubahan visual per state, merujuk token semantic |
| tokensUsed | Peta bagian visual ke token semantic |
| responsive | Perubahan per breakpoint (mobile/tablet/desktop) |
| accessibility | Role ARIA, keyboard interaction, kontrak kontras |
| doAndDont | Panduan pemakaian eksplisit |
| relatedComponents | Komponen mirip + kapan pakai yang mana |
| migrationNote | Perubahan dari v1/ad-hoc dan alasannya |

### Perbaikan Naming Convention (dari temuan audit)

Ditemukan di audit v1: inkonsistensi axis varian (sebagian pakai 'State', sebagian 'Variant', sebagian 'Shape' tanpa aturan jelas). v2 mengunci konvensi:

- **Variant** = perbedaan STYLE/warna/tujuan (primary/secondary/destructive)
- **Size** = perbedaan UKURAN (sm/default/lg)
- **State** = HANYA state interaksi (default/hover/focus/active/disabled) -- bukan untuk variant style

## 8 Kontrak Komponen yang Dibangun

### Prioritas berdasarkan data Fase 0 (demand tertinggi)

| Komponen | Status | Alasan Prioritas |
|---|---|---|
| Button | stable | Demand tertinggi (75 file) |
| Input | stable | Demand tinggi (47 file) + isi gap validation state |
| Select | stable | Demand tinggi (31 file) |
| Dialog | stable | Demand tinggi (34-36 file) |
| DataTable | stable | PALING KRITIS -- sumber masalah responsif awal proyek |

### Komponen BARU mengisi gap kritis dari audit

| Komponen | Kenapa Baru |
|---|---|
| SearchInput | Aturan lama proyek "tabel wajib search box" belum pernah jadi kontrak resmi -- selama ini diimplementasikan manual berbeda-beda per halaman. |
| Checkbox | v1 sama sekali tidak punya checkbox asli -- hanya ada "Checkbox Chip" ad-hoc (multi-select pill) yang sebenarnya use-case berbeda. Kode saat ini pakai `<input type='checkbox'>` native tanpa styling konsisten di 7 file. |
| Toast | v1 sama sekali tidak punya sistem notifikasi -- gap kritis untuk feedback aksi (simpan/hapus/error). |

## Highlight Keputusan Penting per Komponen

### Button
- Menambahkan state `active/pressed` dan `loading` yang tidak ada eksplisit di v1.
- Peringatan aksesibilitas eksplisit: `size=sm` (36px) berada DI BAWAH touch target minimum 44px dari Fase 2 -- hanya boleh dipakai di desktop/tablet, TIDAK sebagai CTA utama mobile.

### Input
- Menambahkan state `error` dan `success` -- v1 cuma 3 state (Default/Filled/Disabled), gap kritis karena form validation adalah pola paling umum di aplikasi absensi (banyak form: siswa, guru, akun, jadwal).

### SearchInput (baru)
- Debounce 300ms default -- mencegah request API berlebihan di tabel data besar.
- WAJIB urutan render Search di posisi paling kiri/atas filter bar (Search -> Jurusan -> Kelas), menegaskan ulang aturan filter bertingkat proyek yang sudah lama berlaku.

### Checkbox (baru)
- Eksplisit dipisahkan dari "Checkbox Chip" (ad-hoc v1) -- keduanya punya use-case berbeda: Checkbox = pilih dari list/bulk-select tabel, Checkbox Chip = filter multi-pilih pill-style. Dokumentasi doAndDont menjelaskan kapan pakai yang mana supaya tidak ambigu.
- Hit-area WAJIB diperbesar ke 44x44px via padding transparan meski ukuran visual cuma 20x20px (mengikuti exception rule touch target dari Fase 2).

### Dialog
- Menambahkan 3 ukuran eksplisit (sm/default/lg) -- v1 cuma 1 ukuran tetap.
- Merekomendasikan pembuatan `ConfirmationDialog` terpisah di Fase 4 -- proyek ini punya aturan keras "operasi destruktif DB wajib konfirmasi" yang sebelumnya tidak punya komponen resmi (cuma Dialog generik).

### DataTable (paling kritis)
- `mobileLayout` WAJIB dipilih eksplisit, DEFAULT='card' (bukan 'scroll') -- perubahan paling signifikan dari seluruh Fase 3. Setiap baris data di mobile dirender ulang sebagai card vertikal (key-value list), BUKAN tabel dengan scroll horizontal.
- Ini adalah PERBAIKAN LANGSUNG dari temuan audit paling kritis: primitif Table v1 sebenarnya sudah punya `overflow-auto` wrapper, tapi 8 dari 9 modul tabel di aplikasi tidak memanfaatkannya secara konsisten, dan tidak ada strategi card-mobile sama sekali.
- Menambahkan `loading` state (Skeleton rows) dan `emptyState` (BELUM ADA di v1 -- tabel kosong tanpa penjelasan).

### Toast (baru)
- variant `danger` dengan `duration=0` untuk error kritis yang wajib dibaca user sebelum hilang.
- `aria-live='assertive'` untuk danger vs `'polite'` untuk lainnya -- screen reader membaca error segera tanpa menunggu jeda.

## Komponen Teridentifikasi Tapi Belum Dibuat Kontraknya

Dicatat sebagai kebutuhan masa depan, muncul dari relatedComponents di kontrak yang sudah ada:

- **ConfirmationDialog** (varian Dialog dengan styling destructive) -- direkomendasikan Fase 4.
- **EmptyState** (dipakai DataTable dan kemungkinan komponen list lain) -- direkomendasikan Fase 4.
- **Alert/Banner** (informasi persisten, beda dari Toast yang sementara) -- direkomendasikan Fase 4.
- **Combobox** (Select dengan pencarian untuk opsi >15 item) -- belum prioritas, dicatat saja.
- **MultiSelect** -- belum prioritas, dicatat saja.

## Keputusan Arsitektur

1. Kontrak disimpan sebagai JSON terpisah per komponen (bukan satu file besar) supaya AI agent bisa membaca HANYA kontrak yang relevan dengan tugasnya, hemat context.
2. Setiap kontrak WAJIB referensi balik ke Fase 1 (token) dan Fase 2 (foundation rules) via path eksplisit (`semantic.color.brand.primary`, `foundation-rules.json elevation_layering.modal`) -- bukan duplikasi nilai, supaya kalau token berubah, kontrak otomatis konsisten (tidak perlu update dua tempat).
3. Field `migrationNote` wajib diisi untuk melacak APA yang berubah dari v1 dan MENGAPA -- ini jadi bahan Fase 6 (Governance) untuk keputusan rollout.

## Representasi di Figma

Kontrak komponen ini SENGAJA TIDAK dibuatkan halaman Figma baru terpisah -- isinya murni logika/teks (props, state rules, do's-and-don'ts), representasi visual yang bermakna untuk kontrak ini justru adalah KOMPONEN ITU SENDIRI dengan seluruh variannya. Untuk 5 komponen stable (Button, Input, Select, Dialog, DataTable), representasi visual v1 di page "Design System" (lama) sudah cukup mewakili strukturnya -- akan disinkron ulang dengan token v2 di Fase 5. Untuk 3 komponen baru (SearchInput, Checkbox, Toast), representasi visual dibangun di Fase 5 (Figma Build) bersamaan dengan komponen v2 lainnya, bukan sekarang -- supaya semua komponen dibangun visual sekaligus dalam satu proses yang konsisten.

## Belum Dikerjakan (menyusul fase lanjutan)

- Kontrak untuk komponen sisanya (Textarea, Skeleton, Label, Tabs, Popover, Sheet, Calendar, DatePicker, Form, Badge, StatusBadge, Pagination, ActivityFeed) -- akan dilengkapi bertahap, prioritas berikutnya ditentukan bersamaan dengan Fase 4/5.
- Kontrak untuk 4 komponen teridentifikasi tapi belum dibuat (ConfirmationDialog, EmptyState, Alert/Banner) -- direncanakan di Fase 4 karena berkaitan langsung dengan Data-Dense Pattern Library.
- Script validator otomatis yang membaca seluruh kontrak dan verifikasi implementasi kode React sesuai props/state yang didefinisikan -- menyusul Fase 6 (Governance).

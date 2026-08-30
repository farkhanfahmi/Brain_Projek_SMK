---
tags: [absensi, design-system-v2, fase-5]
created: 2026-08-29
status: selesai
---

# Fase 5 -- Figma Build

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2.

## Tujuan Fase

Membangun representasi VISUAL di Figma untuk seluruh 14 kontrak komponen/pattern dari Fase 3-4, sinkron penuh dengan token v2 (bukan replikasi manual terpisah seperti v1). Setiap komponen visual WAJIB memakai variable binding ke `v2/Color/Semantic` dkk, bukan hex hardcode.

## Progress Checkpoint (sesi ini)

| Komponen | Status | Catatan |
|---|---|---|
| SECTION: Components (divider) | Selesai | Header oranye penanda mulai section komponen |
| Button | Selesai | 21 varian (7 Variant x 3 Size), grid rapi. 1 bug ditemukan & diperbaiki (auto-layout menimpa posisi manual grid). **Radius DIKOREKSI MANUAL oleh product owner** dari primitive.radius.full (pill/kapsul) menjadi radius custom per size (sm=10px, default=12px, lg=14px) -- tetap rounded-rectangle, BUKAN kapsul. Kontrak button.json sudah diupdate mencerminkan koreksi ini. |
| Input | Selesai | 5 state (default/filled/error/success/disabled) -- 2 state BARU (error/success) tervisualisasi jelas dengan border merah/hijau kontras. |
| Select | Selesai | 4 state (placeholder/selected/error/disabled), radius-md (BUKAN pill, sengaja beda dari Input/Button sesuai kontrak). |
| Checkbox | Selesai | 4 state (unchecked/checked/indeterminate/disabled), ikon check/minus putih kontras di atas Terracotta. |
| SearchInput | Selesai | 4 state (empty/typing/loading/has-value) dengan ikon kaca pembesar/spinner/clear. |
| Dialog | Selesai | 3 ukuran (sm/default/lg) di atas backdrop scrim 60%, progresif rapi. 1 bug ditemukan & diperbaiki (frame context terlalu sempit, modal lg terpotong -- fix: perbesar frame). |
| ConfirmationDialog | Selesai | Ikon warning + judul + deskripsi konkret (jumlah data) + tombol Batal/Hapus (merah menonjol). |
| DataTable | Selesai | **Perbandingan visual desktop-table vs mobile-card berdampingan** -- solusi utama temuan audit kritis. 1 bug ditemukan & diperbaiki (status 'Terlambat' salah warna hijau, seharusnya warning/amber). |
| Toast | Selesai | 4 varian (success/danger/warning/info), shadow floating jelas. |
| EmptyState | Selesai | 3 varian (no-data/no-results/error), center-aligned dengan tombol aksi kontekstual. |
| Alert | Selesai | 2 varian (warning/info), border-left 4px aksen jelas membedakan dari Toast. |
| Chart | Selesai | Bar chart 8 kategori dengan warna categorical-1 s.d. -8, label jelas beda. |
| DashboardWidget | Selesai | 3 ukuran (small/medium/large) dengan KPI besar, trend arah+persen, mini bar chart -- struktur ala TeamHub, warna Cream/Terracotta. |
| FilterBar | Selesai | Urutan Search->Jurusan->Kelas->Tingkat(chip) sesuai kontrak, chip aktif menonjol jelas. |

**SEMUA 14 KOMPONEN/PATTERN FASE 5 SELESAI.**

## Bug Ditemukan & Diperbaiki

**Button component set grid**: `combineAsVariants` menghasilkan auto-layout yang menimpa posisi X/Y manual jika di-set SEBELUM `layoutMode = 'NONE'`. Gejala: semua 21 tombol bertumpuk 1 kolom vertikal alih-alih grid 7x3. Fix: urutan operasi dibalik -- matikan layoutMode dulu, baru reposisi manual.

## Keputusan Teknis

1. Warna Button/Input v2 SEMUA di-bind via `figma.variables.setBoundVariableForPaint` ke collection `v2/Color/Semantic` -- tidak ada fill hardcode, konsisten dengan prinsip Fase 1.
2. Setiap komponen diverifikasi visual (screenshot + vision check) SEBELUM lanjut ke komponen berikutnya -- mengulang disiplin yang sama seperti pembangunan Foundations v2 sebelumnya.
3. Komponen dibangun berurutan sesuai kontrak Fase 3/4 (Button -> Input -> Select -> dst mengikuti urutan definisi kontrak), bukan acak.

## Koreksi dari Product Owner (sesi lanjutan)

Setelah review checkpoint sebelumnya, product owner melakukan beberapa koreksi manual di Figma yang kemudian disinkron balik ke kontrak repo:

1. **Button radius**: dikoreksi dari `primitive.radius.full` (kapsul) ke radius custom per size (sm=10px, default=12px, lg=14px) -- rounded-rectangle, bukan kapsul.
2. **Input & SearchInput radius**: menyusul koreksi Button, ikut diperbaiki dari radius 9999 (kapsul) ke 12px rounded-rectangle -- konsistensi visual antar field form.
3. **Toast border**: ditambahkan stroke 1px berwarna sesuai variant (success=hijau, danger=merah, dst) untuk definisi visual lebih tegas di atas background pastel.
4. **DataTable header**: ditambahkan ikon sort (panah dua-arah) pada kolom Nama & Kelas, ikon filter (segitiga kecil warna brand/primary) pada kolom Status -- merepresentasikan kapabilitas sorting & filtering yang sebelumnya cuma implisit di kontrak, sekarang eksplisit terlihat.
5. **EmptyState variant baru**: ditambahkan `coming-soon` (dashed border + badge "Coming Soon") untuk menandai fitur/modul yang direncanakan tapi belum dikembangkan -- beda dari no-data/no-results/error yang menandakan kondisi runtime data.

Semua koreksi ini disinkronkan balik ke `component-contracts/*.json` di repo (button.json, input.json, search-input.json, toast.json, data-table.json, empty-state.json) supaya repo tetap jadi source of truth yang akurat, bukan Figma yang "diam-diam" lebih benar dari dokumennya.

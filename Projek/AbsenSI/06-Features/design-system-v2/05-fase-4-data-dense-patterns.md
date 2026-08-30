---
tags: [absensi, design-system-v2, fase-4]
created: 2026-08-29
status: selesai
---

# Fase 4 -- Data-Dense Pattern Library

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2.

## Tujuan Fase

Fase 3 mendefinisikan kontrak komponen individual (atom/molecule). Fase 4 naik satu level ke PATTERN -- komposisi beberapa komponen untuk kebutuhan spesifik data-dense UI (dashboard analitik, chart, filter kompleks) yang jadi fokus utama batasan revamp ini. Fase ini juga menuntaskan 3 komponen yang teridentifikasi tapi belum dibuat di Fase 3 (ConfirmationDialog, EmptyState, Alert).

## Lokasi Artefak

- `C:\ProjekSMK\AbsenSI\packages\design-tokens\src\component-contracts\chart.json`
- `.../dashboard-widget.json`
- `.../filter-bar.json`
- `.../confirmation-dialog.json`
- `.../empty-state.json`
- `.../alert.json`

Total kontrak sekarang: **15 file** (14 komponen/pattern + 1 _schema.json meta). Semua 29 referensi token unik (`semantic.*`) di seluruh kontrak diverifikasi valid tanpa broken reference.

## 6 Kontrak Baru di Fase Ini

### 1. Chart (pattern)
Membungkus Chart.js yang SUDAH dipakai di kode (`rekap-view.tsx`, `rekap-guru-view.tsx`) dengan token warna kategorikal dari Fase 1, menggantikan hardcode Material Design yang ditemukan di Fase 0. Aturan kunci:
- Maksimal 8 kategori per chart (sesuai jumlah token categorical yang didesain).
- Legend WAJIB true kecuali ruang sangat terbatas -- warna tidak boleh satu-satunya pembeda (WCAG 1.4.1).
- Mobile: sumbu-x dirotasi/disingkat, legend pindah ke bawah untuk pie/donut.
- Aksesibilitas: canvas Chart.js tidak accessible native ke screen reader -- WAJIB aria-label ringkasan data sebagai fallback.

### 2. DashboardWidget (pattern)
Kartu modular untuk dashboard analitik, grid span mengacu ke `foundation-rules.json` dashboard_widget_span (dari Fase 2). Terinspirasi STRUKTUR referensi TeamHub yang didiskusikan user (grid modular, bukan warnanya -- tetap Cream/Terracotta). 4 ukuran (small/medium/large/full) dengan aturan kolom berbeda per breakpoint.

### 3. FilterBar (pattern)
Mengunci urutan filter bertingkat yang sudah lama jadi ATURAN proyek (Search -> Jurusan -> Kelas) sebagai KOMPONEN resmi, plus filter Tingkat X/XI/XII wajib untuk halaman rekap. Perilaku mobile khusus: filter dropdown/chip disembunyikan di balik tombol 'Filter' dengan badge count, TAPI SearchInput tetap selalu visible (tidak ikut disembunyikan).

### 4. ConfirmationDialog (molecule)
Preset terkunci dari Dialog untuk operasi destruktif -- menjawab aturan keras proyek 'operasi destruktif DB wajib konfirmasi'. Detail penting:
- role='alertdialog' (bukan 'dialog' biasa).
- Enter TIDAK langsung trigger confirm -- mencegah konfirmasi tidak sengaja.
- BARU: prop `requireTypedConfirmation` untuk operasi sangat destruktif (bulk delete, hapus tahun ajaran) -- user harus ketik ulang kata kunci sebelum tombol aktif.
- Description WAJIB jelaskan konsekuensi konkret (jumlah data, entitas spesifik), bukan peringatan generik.

### 5. EmptyState (molecule)
Membedakan 3 kondisi yang sering tertukar: `no-data` (memang belum ada entri), `no-results` (ada data tapi filter tidak ketemu), `error` (gagal teknis). Dipakai wajib oleh DataTable dan Chart saat kosong.

### 6. Alert (molecule)
Banner PERSISTEN, dibedakan eksplisit dari Toast (sementara/floating). Alert = inline dalam alur konten, tetap terlihat sampai ditutup manual. Border-left 4px sebagai aksen visual pembeda sekilas pandang dari Toast.

## Keputusan Arsitektur

1. Pattern (Chart, DashboardWidget, FilterBar) dipisah kategori dari component individual -- pattern adalah KOMPOSISI beberapa komponen untuk use-case spesifik, bukan elemen atomik tunggal.
2. ConfirmationDialog sengaja dibuat sebagai PRESET terkunci dari Dialog (bukan komponen independen dari nol) -- konsisten dengan prinsip DRY, styling/copy pattern tidak bisa 'dikustomisasi bebas' oleh developer per halaman, mencegah variasi implementasi konfirmasi yang tidak konsisten.
3. Referensi silang antar kontrak (mis. Chart merujuk ke EmptyState, DataTable merujuk ke SearchInput) sengaja dipertahankan sebagai jaringan relasi eksplisit di field relatedComponents -- AI agent bisa menelusuri komponen terkait tanpa harus tahu semuanya sekaligus.

## Referensi Desain yang Dipertimbangkan (dan Batasannya)

Selama sesi diskusi sebelumnya, referensi produk UI8 'TeamHub' (HR dashboard) dibahas sebagai inspirasi struktur. Keputusan eksplisit: HANYA struktur layout modular (grid widget, komposisi kartu KPI+chart) yang diadopsi ke DashboardWidget -- skema warna TeamHub (mint/teal) TIDAK dipakai karena bertentangan dengan batasan warna inti Cream/Terracotta yang dikunci product owner.

## Representasi di Figma

SAMA seperti Fase 3 -- kontrak pattern/komponen baru ini murni logika/teks, representasi visualnya akan dibangun bersamaan di Fase 5 (Figma Build) bersama seluruh komponen v2 lainnya, bukan sekarang.

## Status Kumulatif Component Contracts (Fase 3 + 4)

| Kategori | Jumlah | Nama |
|---|---|---|
| Atom (stable) | 3 | Button, Input, Select |
| Atom (new) | 1 | Checkbox |
| Molecule (stable) | 1 | Dialog |
| Molecule (new) | 5 | SearchInput, Toast, ConfirmationDialog, EmptyState, Alert |
| Organism (stable) | 1 | DataTable |
| Pattern (new) | 3 | Chart, DashboardWidget, FilterBar |
| **Total** | **14** | (+ 1 _schema.json meta) |

## Belum Dikerjakan (menyusul fase lanjutan)

- Kontrak untuk komponen sisanya yang sudah ada di Figma v1 tapi belum dikontrak presisi (Textarea, Skeleton, Label, Tabs, Popover, Sheet, Calendar, DatePicker, Form, Badge, StatusBadge, Pagination, ActivityFeed, Switch, Avatar, CheckboxChip, Tooltip, Accordion) -- akan dilengkapi bertahap sesuai kebutuhan migrasi modul di Fase 6.
- Representasi visual Figma untuk seluruh 14 kontrak -- menyusul Fase 5.
- Script validator otomatis -- menyusul Fase 6 (Governance).

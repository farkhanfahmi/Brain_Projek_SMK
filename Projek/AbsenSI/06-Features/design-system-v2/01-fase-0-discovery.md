---
tags: [absensi, design-system-v2, fase-0]
created: 2026-08-28
status: selesai
---

# Fase 0 -- Discovery & Alignment

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2.

## Tujuan Fase

Mengumpulkan data kuantitatif nyata dari codebase sebelum membangun token/komponen baru -- supaya keputusan desain (urutan migrasi, prioritas komponen) berbasis bukti, bukan tebakan.

## Metodologi

Audit otomatis (Python script) terhadap seluruh `apps/web/src` dan `apps/kiosk/src`, menghitung:
1. Jumlah file per modul (route group) -- proxy ukuran & risiko migrasi.
2. Frekuensi import tiap komponen shared dari `@absensi/ui` -- proxy demand/dampak breaking change.
3. Pemakaian warna hex hardcode di luar sistem token -- proxy technical debt warna.
4. Konsistensi pemakaian primitif Table -- proxy kesehatan struktural existing.

## Hasil: Ukuran Modul (file count per route group)

| Modul | Jumlah File | Catatan Risiko |
|---|---|---|
| (admin) | 98 | Terbesar -- risiko tinggi, effort tinggi |
| (guru) | 63 | Besar |
| (admin-jurnal) | 28 | Sedang |
| (piket) | 22 | Sedang, TAPI paling kritis dari audit responsif sebelumnya (semua tabel piket tanpa scroll wrapper) |
| (pembina-ekstra) | 7 | Kecil -- kandidat pilot module rollout pertama (risiko rendah) |
| (siswa) | 5 | Kecil |
| tv-piket | 3 | Sangat kecil, display khusus |
| akun-terkunci, daftar-ekstra, ganti-password, login, tv | 2 masing-masing | Halaman tunggal |

## Hasil: Komponen Paling Vital (top demand)

| Komponen | Jumlah File Memakai |
|---|---|
| Button | 75 |
| Table (Header/Body/Cell/Row) | 51-53 |
| Input | 47 |
| Dialog (Content/Header/Title) | 34-36 |
| Select (Content/Item/Trigger/Value) | 31 |
| Label | 30 |
| cn (utility) | 27 |
| StatusBadge | 12 |
| Tabs | 8 |

Implikasi: perubahan breaking pada Button atau Table akan berdampak ke puluhan file sekaligus -- komponen ini harus paling hati-hati saat migrasi, dan idealnya dibuat backward-compatible dulu sebelum breaking change.

## Hasil: Bug Drift Warna Ditemukan (Bukti Konkret)

### Bug 1: Warna primary usang di kampus-map.tsx

File `apps/web/src/app/(admin)/kampus/kampus-map.tsx` masih memakai `#F5841F` untuk radius lingkaran peta geofence. Warna ini adalah primary color versi SANGAT LAMA, sebelum revisi WCAG 2026-07-23 yang mengubahnya ke `#BA5C08`. File ini terlewat saat migrasi warna dulu.

### Bug 2: Chart warna hardcode Material Design generik

`rekap-view.tsx` dan `rekap-guru-view.tsx` mendefinisikan `SINGLE_DAY_STATUS_CHART_COLOR` dengan hex Material Design default (`#4caf50` hijau, `#fb8c00` oranye, `#42a5f5` biru, `#ab47bc` ungu, dst) -- sama sekali tidak terhubung ke design system manapun.

### Total hardcode hex ditemukan

27 kemunculan, 11 warna unik, tersebar di modul admin (peta, rekap, rekap-guru).

## Hasil: Kesehatan Primitif Table

- 0 pemakaian raw `<table>` tanpa wrapper `@absensi/ui` -- BAIK, semua sudah lewat komponen shared.
- 45 file memakai `TableHeader` (komponen shared) secara konsisten.
- Implikasi positif: migrasi struktural Table (misal menambah strategi mobile/scroll) bisa dilakukan di SATU titik pusat komponen dan otomatis berdampak ke 45 file, bukan perlu ubah manual satu-satu.

## Keputusan dari Fase Ini

1. Data ukuran modul dan demand komponen dicatat sebagai referensi, TAPI urutan migrasi final tetap ditentukan product owner secara manual (bukan otomatis dari data ini) -- lihat 07-fase-6-governance-migration.md saat sudah diisi.
2. Bug drift warna (#F5841F di kampus-map.tsx, chart hardcode di rekap) menjadi item konkret yang harus diperbaiki saat migrasi modul terkait tercapai.
3. Table primitif yang sudah sehat secara struktural jadi kandidat baik untuk migrasi awal -- risiko regresi rendah karena sentralisasi sudah ada.

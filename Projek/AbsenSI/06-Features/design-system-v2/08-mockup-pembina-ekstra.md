---
tags: [absensi, design-system-v2, migrasi]
created: 2026-08-29
status: menunggu-approval
---

# Mockup Figma -- Modul Pembina Ekstrakurikuler (Revamp v2)

Kembali ke 00-INDEX.md untuk konteks proyek revamp v2.

## Status: MENUNGGU APPROVAL USER -- Belum Ada Perubahan Kode

Sesuai instruksi eksplisit: "buatkan saja modul pembina ekstra di figma, jangan langsung
rubah kodenya, setelah figma saya setuju maka nanti akan lanjut memperbaiki kode program".
Dokumen ini murni CATATAN MOCKUP, TIDAK ADA satu baris kode pun yang diubah di
`apps/web/src/app/(pembina-ekstra)/` atau `apps/web/src/features/ekstrakurikuler/`.

## Lokasi Figma

Page baru: **"Module - Pembina Ekstra"** di file AbsenSI (fileKey zPRebQuE5Lo77ZFjSIVfJD),
terpisah dari page "Design System v2".

## 5 Screen yang Dibangun (semua terverifikasi visual)

| # | Screen | Sumber kode asli | Perubahan v2 yang diterapkan |
|---|---|---|---|
| 1 | Sidebar navigasi | `pembina-ekstra-sidebar.tsx` | Token v2 (Terracotta baru), tidak ada perubahan struktur |
| 2 | Presensi -- Desktop | `presensi-view.tsx` | Tabel v2, ikon lock/trash utk sesi yang sudah ada absen |
| 2b | Presensi -- Mobile | (baru, tidak ada di kode) | **PERBAIKAN**: mobileLayout=card menggantikan tabel yang akan overflow di layar sempit |
| 2c | Dialog Buat Presensi | `CreateSesiForm` di presensi-view.tsx | Field disabled (Tanggal/Jam otomatis) v2 |
| 2d | ConfirmationDialog Hapus Sesi | `DeleteSesiForm` (pakai `Dialog` generik) | **PERBAIKAN**: representasi ConfirmationDialog resmi (ikon warning, deskripsi konsekuensi eksplisit) menggantikan Dialog generik |
| 3 | Peserta -- Desktop | `peserta-view.tsx` | FilterBar Search->Jurusan->Kelas (urutan sudah benar di kode asli), kolom No + sort icon eksplisit |
| 3b | Peserta -- Mobile | (baru) | **PERBAIKAN**: card layout menggantikan tabel 6 kolom yang overflow |
| 4 | Setting -- Jadwal | `JadwalSection` di setting-view.tsx | Token v2, field disabled saat punyaKelompok |
| 4b | Setting -- Kelompok chip | Chip kelompok di setting-view.tsx | Chip aktif/inactive dengan warna primary baru |
| 5 | Sesi Detail -- Tandai Kehadiran | `sesi-detail-view.tsx` | Badge status v2 (Hadir hijau/Izin kuning/Alfa merah), tombol tandai bulat (centang/kamera/silang), lock icon utk baris yang sudah ditandai |

## Perbaikan yang DITERAPKAN di Mockup (berbeda dari kondisi kode v1 saat ini)

Berbeda dari filosofi "replikasi apa adanya" yang dipakai untuk audit Design System v1
lama, mockup modul ini SENGAJA menerapkan 2 perbaikan konkret dari temuan Fase 6
(karena modul ini adalah PILOT migrasi nyata, bukan dokumentasi kondisi lama):

1. **Mobile card layout untuk kedua tabel** (Presensi & Peserta) -- menjawab temuan
   audit "3 dari 3 tabel tanpa overflow-x-auto" dari hasil uji coba Claude Code CLI
   sebelumnya (lihat workflow-ai-agent-full-hermes.md).
2. **ConfirmationDialog menggantikan Dialog generik** untuk operasi hapus -- menjawab
   temuan "3 operasi destruktif pakai Dialog generik, bukan ConfirmationDialog resmi".

## Yang TIDAK Diubah dari Kode Asli (dipertahankan sengaja)

- Alur kerja "Hadir Semua lokal dulu, baru 1x Save" di Sesi Detail -- logika bisnis
  yang sudah benar, tidak disentuh strukturnya.
- Field jam yang disabled/auto-fill mengikuti jadwal kelompok -- logika bisnis asli.
- Badge "Belum berkelompok" -- istilah dan lokasi sama seperti kode asli.
- Ikon lock utk sesi/absen yang tidak bisa dihapus -- perilaku bisnis asli
  (`disabled={sesi.sudahAdaAbsen}`).

## Langkah Berikutnya

**MENUNGGU keputusan user**: review mockup di Figma page "Module - Pembina Ekstra" ->
kalau disetujui, lanjut ke implementasi kode nyata (kemungkinan via Sesi Eksekusi +
Claude Code CLI sesuai workflow-ai-agent-full-hermes.md) -> update status dokumen ini
jadi "disetujui, siap implementasi" atau catat revisi yang diminta.

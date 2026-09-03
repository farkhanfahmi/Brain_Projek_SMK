---
tags: [absensi, riset, perbandingan, jurnale-pro, jurnal-guru, planning]
status: disimpan-belum-ditindaklanjuti
updated: 2026-09-02
---

# Riset — Perbandingan Fitur JurnalePro vs AbsenSI

← Index (00-INDEX AbsenSI.md)

> **Status: PEMBAHASAN DISIMPAN, BELUM ditindaklanjuti jadi task.** Dibahas 2026-09-02 di sesi diskusi. Keputusan user: fokus dulu ke penyempurnaan Jurnal Guru (kerja aktif), gap-gap di dokumen ini dibahas ulang nanti sambil jalan — BUKAN prioritas sekarang.

---

## Konteks

JurnalePro (`pro.jurnale.id`) adalah aplikasi yang dipakai sekolah ini SEBELUM AbsenSI — jadi jadi pembanding relevan untuk fitur yang sudah terbukti dipakai user riil (guru/kepsek/piket) di lapangan. Riset dilakukan dengan membaca panduan resmi (`pro.jurnale.id/panduan/`, `/panduan/mobile`) via web search snippet (extract langsung gagal, situs SPA + backend extract API bermasalah saat riset).

## ✅ AbsenSI sudah setara atau lebih baik dari JurnalePro

| Fitur JurnalePro | Status AbsenSI |
|---|---|
| Ceklok KBM + GPS | Sudah ada, lebih ketat (geofence radius per kampus + QR scan Ketua Kelas, bukan cuma GPS mentah) |
| Jurnal per sesi | Sudah ada (`JournalEntry`: elemen, CP, materi, tujuan, tugas, catatan) |
| Presensi siswa per jam | Sudah ada (`ClassAttendanceMark`), lebih pintar (default dari tap gerbang, guru cuma koreksi — bukan input manual semua siswa tiap sesi) |
| Mode Blok minggu A/B | Sudah ada — JurnalePro versi yang dipakai sekolah ini dulu justru TIDAK support pola mingguan bergantian (alasan AbsenSI dibangun ulang, lihat `dashboard-guru-jurnal.md`) |

## 🟡 Gap nyata — kandidat untuk dipertimbangkan nanti (belum jadi task)

1. **Cetak/Export Jurnal Mengajar & Presensi (PDF/print)** — JurnalePro punya menu ini baik sisi admin maupun guru. AbsenSI **tidak punya sama sekali** (dikonfirmasi grep, nihil). Kebutuhan administratif riil (akreditasi, laporan Dinas, arsip fisik).
2. **Notifikasi WhatsApp otomatis untuk guru terlambat/belum ceklok** — JurnalePro kirim WA otomatis lewat batas toleransi, variabel dinamis (`[sapaan]`, `[nama]`, `[rombel]`, `[waktu]`). Butuh keputusan biaya (JurnalePro pakai wablas.com berbayar) sebelum diadopsi.
3. **Dashboard Supervisi KBM Real-time** (untuk Kepsek/Waka Kurikulum) — 1 layar live HANYA menampilkan sesi KBM yang SEDANG berlangsung (bukan histori), kolom: Nama Guru, Rombel, Jadwal vs Aktual, Status, tombol kirim notifikasi manual. AbsenSI punya data mentahnya (`TeachingSession` + Socket.IO sudah ada infrastruktur serupa untuk Dashboard TV siswa) tapi belum ada dashboard setara untuk KBM guru — perlu verifikasi ulang apakah role `kepsek` sudah punya pandangan ini.
4. **Peran "Petugas Monitor"** (Guru Piket sebagai supervisor lapangan bergerak, verifikasi manusia independen dari GPS — kolom `Status` sistem vs `Monitor` manusia dibandingkan berdampingan) — AbsenSI murni andalkan GPS+QR, tidak ada lapis verifikasi manusia.

## 🔴 Sengaja TIDAK relevan / ditolak

- Notifikasi WA berbayar pihak ketiga (wablas.com) — keputusan biaya berlangganan, bukan otomatis "ya".
- 4 aplikasi terpisah per peran (Admin/Mobile/Guru/Monitor) — AbsenSI sudah lebih modern (1 web app role-based routing), tidak perlu ditiru.
- Hard-block "wajib presensi siswa dulu baru bisa isi jurnal" — AbsenSI sudah punya pendekatan lebih baik (default hadir dari tap gerbang), mengadopsi hard-block ini justru KEMUNDURAN UX.

---

## 🔗 Lihat Juga
- `06-Features/dashboard-guru-jurnal.md` — spec Jurnal Guru AbsenSI (fokus kerja aktif saat ini)
- `STATUS.md` — daftar task aktif

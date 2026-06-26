---
tags: [absensi, environment]
updated: 2026-06-25
---

# 10 — Environment

← [[Projek/AbsenSI/00-INDEX|Index]]

> Sebagian sudah diputuskan (lihat ADR-011 s/d ADR-013 di [[Projek/AbsenSI/11-Decisions|11-Decisions]]) — arsitektur server & database sudah final, detail provisioning teknis masih menyusul.

## Arsitektur Server (Resolved 2026-06-26)

### Topologi — 1 Server Fisik, Bukan Split-VM (ADR-012)
Production di-hosting **on-premise** (server fisik di sekolah, dikelola tim sendiri) — bukan cloud VPS. Semua aplikasi ekosistem sekolah (AbsenSI dan aplikasi berikutnya) berjalan di **1 server fisik yang sama**, dengan **separasi logis per aplikasi** (1 database/schema MySQL per aplikasi), bukan VM/server virtual terpisah-pisah. Alasan lengkap & trade-off yang dipertimbangkan ada di ADR-012.

**Penanggung jawab infrastruktur:** 1 dari 3 developer jadi penanggung jawab utama (paling berpengalaman mengelola server sekolah), 2 lainnya sebagai backup — bukan single point of failure, tapi tetap perlu didokumentasikan prosedur akses (kredensial, SSH key) supaya backup benar-benar bisa ambil alih kalau perlu.

### Database Engine — MySQL (ADR-011)
MySQL dipilih menggantikan rencana awal PostgreSQL — alasan lengkap di ADR-011. Semua aplikasi ekosistem pakai MySQL sebagai standar.

### Data Warehouse, Arsip, & Backup — 3 Mekanisme Terpisah (ADR-013)
1. **Database global (data warehouse):** ETL satu arah, berkala **bulanan atau per semester**, dari tiap database aplikasi → 1 database global read-only untuk laporan lintas-aplikasi. Pakai tooling replikasi/ETL teruji (built-in MySQL replication atau script cron), bukan logic custom per aplikasi.
2. **Arsip dingin tahunan ("server 1"):** backup historis murni dari database global, di-push manual tiap tahun ajaran baru, tidak dipakai operasional aplikasi apa pun.
3. **Backup operasional rutin:** backup harian/mingguan terjadwal dari tiap database aplikasi, terpisah dari arsip tahunan — **wajib ada sebelum go-live Fase 1**, jangan hanya andalkan arsip tahunan sebagai satu-satunya jaring pengaman (prinsip 3-2-1 backup). Peningkatan opsional: MySQL replication (master-replica) untuk recovery point objective lebih baik.

### Master Data (Core) — Tetap di Dalam AbsenSI untuk Sekarang (ADR-014)
Data siswa/guru/jadwal tetap modul internal AbsenSI, belum diekstrak jadi servis terpisah — ditunda sampai aplikasi ekosistem ke-2 benar-benar konkret. Lihat ADR-014 untuk alasan & kapan ini harus dibuka kembali.

## Rencana Lain (Belum Diputuskan)
- Dev: lokal masing-masing developer, Docker Compose untuk MySQL + Redis
- Mini-PC gerbang: browser kiosk mode menjalankan `apps/kiosk`, konek ke server fisik sekolah lewat jaringan lokal

## ❓ Open Questions
- [ ] Spesifikasi hardware server fisik (RAM/CPU/storage) — perlu cukup untuk menjalankan beberapa database aplikasi sekaligus + database global, sizing detail menyusul saat provisioning
- [ ] Tooling/jadwal pasti untuk ETL bulanan/semester ke database global — script cron sederhana, atau pakai tool ETL yang sudah ada?
- [ ] Domain/subdomain untuk akses dashboard (akses dari luar jaringan sekolah, kalau dibutuhkan kepala sekolah/guru akses dari rumah)
- [ ] Prosedur akses darurat untuk 2 developer backup (dokumentasi kredensial, SSH key, runbook dasar) — supaya backup person benar-benar bisa ambil alih, bukan cuma di atas kertas


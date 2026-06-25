---
tags: [absensi, environment]
updated: 2026-06-25
---

# 10 — Environment

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> Belum disetup — placeholder sampai repo GitHub & infrastruktur dibuat.

## Rencana Awal
- Dev: lokal masing-masing developer, Docker Compose untuk PostgreSQL + Redis
- Mini-PC gerbang: browser kiosk mode menjalankan `apps/kiosk`, perlu koneksi ke server (lokal jaringan sekolah atau cloud — **belum diputuskan** apakah server di-hosting cloud atau on-premise di sekolah)
- Production: **belum diputuskan** — perlu didiskusikan: hosting cloud (VPS) vs server fisik di sekolah, mengingat ini data sensitif (kehadiran siswa/guru) dan butuh uptime tinggi saat jam masuk sekolah

## ❓ Open Questions
- [ ] Hosting production — cloud atau on-premise?
- [ ] Backup strategy database
- [ ] Domain/subdomain untuk akses dashboard

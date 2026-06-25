---
tags: [absensi, feature, notifikasi, fase-3, planning]
status: planning
updated: 2026-06-25
---

# Feature — Notifikasi Orang Tua (Fase 3 — Jalur Disiapkan, Belum Dibangun)

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> **Tidak dikerjakan di fase 1.** Hanya jalur arsitektur (event-driven via BullMQ) yang disiapkan dari fase 1, sesuai ADR-006. Dokumen ini placeholder untuk planning detail nanti.

---

## Konsep (kasar, belum spec)
- Saat siswa tap masuk/pulang, kirim notifikasi WA ke nomor orang tua terdaftar
- Modul Notification (stub di fase 1, hanya logging) akan diisi listener WA gateway di fase ini

## Yang Perlu Diputuskan Nanti
- [ ] Provider WA gateway (resmi WhatsApp Business API vs provider pihak ketiga) — ada biaya per pesan, perlu budget sekolah
- [ ] Data nomor HP orang tua — sumber data dari mana, siapa yang maintain
- [ ] Opt-in/opt-out per orang tua atau wajib semua
- [ ] Format & isi pesan notifikasi
- [ ] Reliability — apa yang terjadi kalau pesan gagal kirim (retry policy)

## 🔗 Lihat Juga
- [[30.Projects/AbsenSI/11-Decisions|ADR-006]]

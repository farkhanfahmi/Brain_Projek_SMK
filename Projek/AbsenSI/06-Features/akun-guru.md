---
tags: [absensi, feature, akun-guru, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Akun & Riwayat Kehadiran Guru

← Index (00-INDEX AbsenSI.md)

> Guru bisa login untuk melihat riwayat kehadirannya sendiri. Scope fase 1 **sengaja sangat sempit** — read-only, tanpa edit apa pun, tanpa akses lihat data kelas/siswa lain. Fitur wali kelas (lihat kehadiran kelas yang diampu) **dibahas di fase 2**.

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟢 Final — siap jadi task |
| Owner | Modul `apps/web` (UI) + `apps/api` (auth & endpoint) |

---

## Fungsi (Fase 1 — Scope Sempit)
- Guru login (kredensial dibuat oleh Admin Pusat saat onboarding — tidak ada self-register)
- Setelah login: lihat tabel riwayat tap kehadiran dirinya sendiri (tanggal, waktu masuk, waktu pulang, status terlambat/tidak)
- **Tidak ada** fitur edit, tidak ada akses ke data siswa, tidak ada akses ke data guru lain

## Eksplisit Bukan Scope Fase 1
- Wali kelas melihat rekap kehadiran kelas yang diampu → **Fase 2**
- Guru mapel melihat siapa yang hadir di kelasnya → **Fase 2** (baru relevan setelah ada reader kelas)

## ✅ Open Questions — Resolved (2026-07-03)

- [x] **Reset password guru** → **Manual oleh admin di Fase 1.** Belum ada jalur email/WA terverifikasi. Admin generate password baru dari dashboard dan sampaikan ke guru via komunikasi langsung. Self-service reset (kirim link via WA) masuk backlog Fase 3 bersama notifikasi orang tua.

**Status spec:** ✅ Final — siap dipecah jadi task development.

## 🔗 Lihat Juga
- 03-User-Roles (03-User-Roles.md)
- ADR-008 (11-Decisions.md)


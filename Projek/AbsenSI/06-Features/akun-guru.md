---
tags: [absensi, feature, akun-guru, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Akun & Riwayat Kehadiran Guru

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> Guru bisa login untuk melihat riwayat kehadirannya sendiri. Scope fase 1 **sengaja sangat sempit** — read-only, tanpa edit apa pun, tanpa akses lihat data kelas/siswa lain. Fitur wali kelas (lihat kehadiran kelas yang diampu) **dibahas di fase 2**.

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟡 Draft |
| Owner | Developer 1 (apps/web) untuk UI; Developer 3 (apps/api) untuk auth & endpoint |

---

## Fungsi (Fase 1 — Scope Sempit)
- Guru login (kredensial dibuat oleh Admin Pusat saat onboarding — tidak ada self-register)
- Setelah login: lihat tabel riwayat tap kehadiran dirinya sendiri (tanggal, waktu masuk, waktu pulang, status terlambat/tidak)
- **Tidak ada** fitur edit, tidak ada akses ke data siswa, tidak ada akses ke data guru lain

## Eksplisit Bukan Scope Fase 1
- Wali kelas melihat rekap kehadiran kelas yang diampu → **Fase 2**
- Guru mapel melihat siapa yang hadir di kelasnya → **Fase 2** (baru relevan setelah ada reader kelas)

## ❓ Open Questions
- [ ] Reset password guru — self-service (lupa password, kirim email/WA) atau harus minta admin reset manual? (Fase 1 mungkin cukup manual mengingat skala awal & belum ada jalur email/WA terverifikasi)

## 🔗 Lihat Juga
- [[30.Projects/AbsenSI/03-User-Roles|03-User-Roles]]
- [[30.Projects/AbsenSI/11-Decisions|ADR-008]]

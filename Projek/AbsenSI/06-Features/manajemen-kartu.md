---
tags: [absensi, feature, kartu, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Manajemen Kartu RFID

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> CRUD mapping UID kartu RFID ↔ siswa/guru. Termasuk proses ganti kartu hilang/rusak — wajib lewat admin (sesuai keputusan diskusi awal).

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟡 Draft |
| Owner | Developer 3 (apps/api — Card module) + Developer 1 (apps/web — UI admin) |

---

## Fungsi Utama

- **Registrasi kartu baru**: admin input UID (scan kartu via reader yang ditancep ke PC admin, atau input manual) + pilih siswa/guru yang dimapping
- **Nonaktifkan kartu**: kartu hilang/rusak ditandai `inactive`, riwayat tap lama tetap tersimpan (tidak dihapus)
- **Ganti kartu**: kartu baru di-assign ke orang yang sama, kartu lama otomatis `inactive`. **Wajib lewat admin** — tidak ada self-service.
- **Riwayat kartu per orang**: 1 siswa/guru bisa punya >1 kartu sepanjang waktu (riwayat ganti), tapi cuma 1 yang `active` di satu waktu

## Aturan
1. 1 UID hanya boleh terhubung ke 1 orang dalam status aktif kapan pun
2. UID yang sudah pernah dipakai (lalu di-nonaktifkan) **tidak boleh** didaftarkan ulang ke orang lain — untuk integritas histori (kalau kartu fisik didaur ulang/diberi ke siswa lain, harus pakai UID kartu fisik baru, bukan reuse UID yang sama meski kartunya beda secara fisik kalau UID collision — catatan: UID kartu RFID umumnya pre-printed dan unik dari pabrik, jadi ini edge case kecil tapi tetap perlu unique constraint di DB)
3. Tap dari UID yang `inactive` → ditolak dengan pesan jelas, dicatat sebagai log percobaan (untuk audit, bukan attendance record)

## Role & Akses
**Resolved (lihat ADR-008 & [[30.Projects/AbsenSI/03-User-Roles|03-User-Roles]]):** Registrasi & CRUD kartu boleh dilakukan oleh **Admin Pusat (`super_admin`)** maupun **Admin Pengelola Kartu (`card_admin`)** — role kedua ini didedikasikan khusus untuk delegasi tugas kartu (misal ke staff TU), tanpa kasih akses ke fitur lain (jadwal, koreksi absensi, kelola akun). Validasi role wajib dicek di backend API, bukan cuma disembunyikan di UI.

## ❓ Open Questions
- [ ] Apakah perlu fitur bulk-import kartu (misal saat awal rollout, daftarkan 2.500 kartu sekaligus via CSV)? — sangat mungkin perlu, mengingat skala awal. **Kemungkinan ini harus naik prioritas ke fase 1**, bukan nice-to-have.

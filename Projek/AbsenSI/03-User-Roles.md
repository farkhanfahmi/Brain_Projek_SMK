---
tags: [absensi, user-roles]
updated: 2026-06-25
---

# 03 — User Roles

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> Ini role PENGGUNA APLIKASI (siswa, guru, admin sekolah) — bukan role tim developer. Untuk pembagian tim developer lihat [[30.Projects/AbsenSI/_claudian/team|team.md]].

---

## Role & Akses (Fase 1 — Disepakati)

> **Keputusan desain:** Role disimpan sebagai field generik di database (`super_admin`, `card_admin`, `guru`), BUKAN diikat ke identitas spesifik orang. "Admin Pusat" untuk sekarang kebetulan dipegang 3 developer, tapi secara sistem itu cuma akun dengan role `super_admin` — siapa pun bisa diberi role ini di masa depan tanpa ubah struktur. Lihat ADR-008.

| Role | Login? | Akses |
|---|---|---|
| **Siswa** | ❌ Tidak login | Tap kartu di kiosk gerbang saja. Tidak ada interaksi sistem selain fisik tap |
| **Guru** | ✅ Login | Tap kartu di kiosk gerbang (sama seperti siswa). **Setelah login ke dashboard:** hanya bisa **melihat riwayat kehadirannya sendiri** (read-only, tidak bisa edit apa pun) |
| **Admin Pusat** (`super_admin`) | ✅ Login | Full CRUD ke **semua** fitur & data — kartu, jadwal, koreksi data absensi, kelola akun, kelola role. Saat ini dipegang 3 developer |
| **Admin Pengelola Kartu** (`card_admin`) | ✅ Login | **Hanya** CRUD data kartu (registrasi, nonaktifkan, ganti kartu). Tidak bisa edit jadwal, tidak bisa edit data absensi, tidak bisa kelola akun lain |
| **Kepala Sekolah** (`kepsek`) | ✅ Login — akun khusus tersendiri | Lihat dashboard TV + rekap (read-only). **Resolved:** TV dashboard tetap butuh auth, bukan akses bebas tanpa login meski di ruang terbatas |

**Catatan:** Wali kelas dengan akses lihat data kelasnya (bukan cuma riwayat sendiri) — **dibahas di fase 2**, belum di-desain di fase 1.

## Aturan Tegas
1. **Hanya Admin Pusat dan Admin Pengelola Kartu yang boleh mengubah data.** Guru read-only mutlak — tidak ada exception.
2. Admin Pengelola Kartu **tidak boleh** akses fitur di luar modul kartu — ini batasan permission di level API, bukan cuma disembunyikan di UI (kalau cuma disembunyikan di frontend, endpoint API tetap bisa diakses langsung — harus dicek role di backend).

## ❓ Open Questions
- [ ] Matriks izin detail per endpoint API menyusul saat modul Web/Core dirancang lebih detail

> **Implikasi teknis dari "TV dashboard tetap butuh auth":** layar TV di ruang kepsek perlu mekanisme login yang tidak ribet diulang tiap hari (TV biasanya nyala terus, tidak ada keyboard/mouse di TV itu sendiri). Opsi yang perlu dipikirkan saat masuk task breakdown: token sesi berumur panjang khusus device TV (mirip "remember this device"), atau login sekali di awal lalu browser kiosk di mini-PC TV tidak pernah logout. Detail teknis ini didesain saat task dashboard-tv dipecah, bukan sekarang.

---
tags: [absensi, feature, kartu, admin, fase-2]
status: final
updated: 2026-07-21
---

# Feature — Dashboard Petugas Kartu (Fase 2)

← Index (00-INDEX AbsenSI.md)

> Dashboard dedicated untuk role `card_admin` (role ini **sudah ada** sejak Fase 1, lihat ADR-008 & manajemen-kartu.md (06-Features/manajemen-kartu.md) — sebelumnya cuma didefinisikan wewenangnya di backend, belum punya UI/dashboard sendiri, praktiknya semua orang pakai akun admin pusat). Fitur ini murni membangun UI dedicated + mengaktifkan akun nyata untuk role yang sudah dirancang.

---

## 🎯 Konsep

- **Clone fitur, bukan role baru.** Menu manajemen kartu (registrasi, nonaktifkan, ganti kartu, riwayat, bulk import CSV, tap-to-assign — lihat manajemen-kartu.md (06-Features/manajemen-kartu.md) untuk daftar lengkap) di-clone ke dashboard `card_admin`, dengan wewenang **identik persis** dengan yang sudah final — tidak dipersempit maupun diperluas
- Menu manajemen kartu **tetap ada juga** di dashboard admin pusat (`super_admin`) — dua entry point paralel ke fungsi & data yang sama, bukan menu eksklusif yang dipindah
- **Input data siswa/guru tetap eksklusif admin pusat.** Petugas kartu tidak bisa create/edit biodata siswa/guru — cuma pilih dari daftar existing saat registrasi/mapping kartu (sesuai alur registrasi kartu yang sudah ada: pilih siswa/guru yang dimapping ke UID)
- Dashboard `card_admin` tidak menampilkan menu lain (kelola akun, jadwal, absensi, rekap, dst) — fokus tunggal ke modul kartu, sesuai batasan yang sudah ditegakkan di ADR-008

## Fungsi yang Di-clone (referensi manajemen-kartu.md (06-Features/manajemen-kartu.md))

- Registrasi kartu baru (scan UID via reader di PC petugas, atau input manual) + pilih siswa/guru existing untuk dimapping
- Nonaktifkan kartu (hilang/rusak)
- Ganti kartu (kartu baru assign ke orang yang sama, kartu lama otomatis inactive)
- Riwayat kartu per orang
- Bulk import CSV (Mode A) & tap-to-assign (Mode B) — ADR-009

## Role & Akses

Tidak ada perubahan dari yang sudah final:
- **Wewenang**: `super_admin` dan `card_admin` — sama persis, ditegakkan di backend API (bukan cuma UI)
- **Pembuatan akun `card_admin`**: tetap wewenang `super_admin` (konsisten pola pembuatan akun role lain)

## ✅ Yang TIDAK Berubah

- Tidak ada perubahan skema database — `users.role = card_admin` sudah ada sejak Fase 1
- Tidak ada perubahan aturan bisnis kartu (1 UID = 1 orang aktif, UID bekas tidak boleh reuse, dst)
- Ini murni pekerjaan UI/UX: layout dashboard baru + routing akses berbasis role yang sudah ada

**Status spec:** ✅ Final — siap dipecah jadi task development. Tidak ada open question tersisa.

## 🔗 Lihat Juga
- manajemen-kartu.md (06-Features/manajemen-kartu.md) — spec lengkap fungsi kartu yang di-clone
- 03-User-Roles (03-User-Roles.md) — definisi role `card_admin`
- ADR-008 (11-Decisions.md) — role generik bukan identitas orang

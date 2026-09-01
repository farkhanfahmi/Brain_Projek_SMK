---
tags: [absensi, user-roles]
updated: 2026-06-25
---

# 03 — User Roles

← Index (00-INDEX AbsenSI.md)

> Ini role PENGGUNA APLIKASI (siswa, guru, admin sekolah) — bukan role tim developer. Untuk pembagian tim developer lihat team.md (_claudian/team.md).

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

**Catatan:** Wali kelas dengan akses lihat data kelasnya (bukan cuma riwayat sendiri) — **Final (2026-07-21):** bukan role baru, extend akun `guru` existing dengan kolom `kelas_id_wali` (pola identik `guru_piket.kampus_id`). Read-only, scope ke kelas yang diampu. Detail lengkap & isi menu di dashboard-guru-jurnal.md (06-Features/dashboard-guru-jurnal.md) bagian "Wali Kelas".

## Role yang Ditambahkan Setelah Fase 1 (SUDAH LIVE, bukan lagi planning)

> **Koreksi 2026-08-04:** bagian ini sebelumnya menyebut role di bawah sebagai "Fase 2 — planning", padahal SEMUA sudah dikerjakan dan live di production. `enum UserRole` aktual di `apps/api/prisma/schema.prisma`: `super_admin`, `card_admin`, `guru`, `kepsek`, `guru_piket`, `admin_jurnal`, `pembina_ekstra`.

| Role | Login? | Akses |
|---|---|---|
| **Guru Piket** (`guru_piket`) | ✅ Login — akun terpisah dari `guru`, di-scope per `kampus_id` | Dashboard Piket: buat/ubah `permits` (izin/sakit/keluar), lock/unlock siswa, konfirmasi kembali. **HANYA role ini** yang boleh mengubah status kehadiran siswa — `super_admin` TIDAK BISA (lihat ADR-019). Enforcement jadwal tugas via `PiketOnDutyGuard` — di luar hari jadwalnya, read-only meski tetap login |
| **Admin Jurnal** (`admin_jurnal`) | ✅ Login | Terkunci ke domain jurnal mengajar: kelola jadwal (termasuk Mode Blok A/B), izin guru, monitor+koreksi jurnal, master data mapel. **Tidak** bisa akses `users`, kartu, kalender pendidikan, atau rekap kehadiran siswa — pola sama seperti `card_admin` (ADR-008), dipisah dari Admin Pusat supaya operasional jurnal harian tidak numpuk ke satu role |
| **Pembina Ekstrakurikuler** (`pembina_ekstra`) | ✅ Login | Akun untuk pembina EKSTERNAL (bukan guru sekolah) — dashboard sesi/presensi/kelompok ekstrakurikuler yang dibinanya saja, scope sempit via `Ekstrakurikuler.pembinaId`. Guru sekolah yang jadi pembina TETAP pakai akun role `guru` biasa (bukan role ini) — role `pembina_ekstra` khusus untuk yang bukan guru |

Sama seperti `card_admin`, pemisahan role-role ini ditegakkan di level API guard, bukan cuma disembunyikan di UI (lihat "Aturan Tegas" di bawah).

## Role Tambahan — Akun Siswa Terbatas (T247, SUDAH LIVE)

> **[2026-08-31] Perubahan arsitektur penting**: sejak T247, pernyataan "Siswa ❌ Tidak login" di atas **tidak lagi berlaku mutlak**. Ditambahkan SEMPIT untuk kasus Ketua/Wakil Ketua Kelas — bukan perubahan ke semua siswa.

| Role | Login? | Akses |
|---|---|---|
| **Ketua Kelas** (`ketua_kelas`) | ✅ Login — akun `User` terhubung ke `Student` via `studentId` (unique FK) | Portal siswa terbatas: QR jadwal, kemungkinan piket kebersihan kelas. Scope sempit ke kelasnya sendiri saja |

**Detail arsitektur penting:**
- `Student` (data siswa, tap RFID) tetap **terpisah** dari `User` (akun login) — `ketua_kelas` MENJEMBATANI keduanya untuk kasus ini saja, BUKAN mengubah pemisahan Student/User secara umum. Mayoritas siswa TIDAK punya akun `User` sama sekali.
- 1 siswa maksimal 1 akun (`User.studentId` unique) — meski jabatan berganti, tidak ada 2 akun untuk 1 `Student` yang sama.
- **Trade-off yang SUDAH DISADARI & DITERIMA user** (bukan bug): kalau Ketua DAN Wakil Ketua sama-sama absen, akun salah satu boleh dipakai/dioper informal oleh siswa lain secara fisik — sistem tidak membangun mekanisme delegasi otomatis, dan tidak pernah tahu PERSIS individu mana yang menekan tombol (cuma tahu akun mana yang dipakai).
- Model pendukung: `KelasPengurus` (struktur pengurus per tahun ajaran: ketua/wakil_ketua/sekretaris/wakil_sekretaris/bendahara/wakil_bendahara — 4 jabatan terakhir TIDAK dapat akun login, hanya `ketua`/`wakil_ketua`).
- Detail lengkap: `06-Features/tasks/T247-schema-struktur-pengurus-piket-akun-siswa.md` s/d T251 (backend QR, portal siswa, scanner guru).

## Aturan Tegas
1. **Hanya Admin Pusat dan Admin Pengelola Kartu yang boleh mengubah data.** Guru read-only mutlak — tidak ada exception.
2. Admin Pengelola Kartu **tidak boleh** akses fitur di luar modul kartu — ini batasan permission di level API, bukan cuma disembunyikan di UI (kalau cuma disembunyikan di frontend, endpoint API tetap bisa diakses langsung — harus dicek role di backend).

## ❓ Open Questions
- [ ] Matriks izin detail per endpoint API menyusul saat modul Web/Core dirancang lebih detail

> **Implikasi teknis dari "TV dashboard tetap butuh auth":** layar TV di ruang kepsek perlu mekanisme login yang tidak ribet diulang tiap hari (TV biasanya nyala terus, tidak ada keyboard/mouse di TV itu sendiri). Opsi yang perlu dipikirkan saat masuk task breakdown: token sesi berumur panjang khusus device TV (mirip "remember this device"), atau login sekali di awal lalu browser kiosk di mini-PC TV tidak pernah logout. Detail teknis ini didesain saat task dashboard-tv dipecah, bukan sekarang.


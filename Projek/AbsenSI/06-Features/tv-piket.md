---
tags: [absensi, feature, tv, piket, realtime, fase-2, final]
status: final
updated: 2026-07-22
---

# Feature — TV Piket (Lorong Piket, Layar Publik)

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> **Status: FINAL (2026-07-22), siap breakdown task.** Dependensi (`teaching_sessions`, `teacher_permits` dari dashboard guru jurnal — T038-T054) sudah ada kode-nya di working tree (belum di-audit visual lewat T060, tapi datanya siap dikonsumsi). Layout dirancang dari referensi EzMart + komponen generik yang sudah dibuat (`DataTableCard`/T058, `ActivityFeedCard`/T059) — tidak perlu menunggu sketsa lagi, semua keputusan sudah final di bawah.

---

## 🎯 Konsep

Selain komputer kerja guru piket (dashboard piket existing, lihat [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]]), lorong piket juga punya **TV layar publik** yang bisa dilihat banyak orang (siswa lewat, orang tua, tamu). Berbeda dari dashboard kerja piket (interaktif, perlu login aktif untuk kerja), TV ini murni tampilan pantauan pasif, read-only, mirip semangat [[Projek/AbsenSI/06-Features/dashboard-tv|dashboard-tv.md]] (TV kepsek) tapi dengan isi & scope berbeda.

**Semua widget tampil dalam 1 layar** (tidak ada navigasi/klik — sesuai maksud "tv" yang dilihat sambil lewat), layout **bento grid, final (2026-07-22)**:

```
┌──────────────────────────────────────────────────────────────────┐
│  BAR PERSENTASE HADIR/IZIN/ALFA (full-width, 1 baris)             │
│  [████████████ Hadir 82% ████][███ Izin 9% ███][██ Alfa 9% ██]    │
└──────────────────────────────────────────────────────────────────┘
┌───────────────────────────────────┐┌─────────────────────────────┐
│  SISWA TIDAK HADIR + KETERANGAN   ││  GURU BELUM MULAI MENGAJAR   │
│  (Data Table, ~60% lebar)         ││  (Activity Feed, ~40% lebar) │
│  auto-scroll internal             ││  scroll internal jika >5     │
└───────────────────────────────────┘└─────────────────────────────┘
┌────────────────────────────────────────────────────────────────────┐
│  GURU IZIN HARI INI (Activity Feed, full-width, baris bawah)        │
└────────────────────────────────────────────────────────────────────┘
```

- **Bar persentase** (widget #1): 1 baris horizontal penuh, PALING ATAS — 3 segmen proporsional terhadap persentase riil, warna dari token status (BUKAN "hijau/kuning/merah" generik lama — lihat revisi warna di bawah), angka persentase ditampilkan di dalam tiap segmen (`text-h3` bold putih, kontras di atas warna solid)
- **Baris tengah, 2 kolom**: Siswa Tidak Hadir (kiri, lebih lebar karena datanya lebih banyak) + Guru Belum Mulai Mengajar (kanan)
- **Baris bawah, full-width**: Guru Izin Hari Ini
- Semua widget pakai **`DataTableCard`**/**`ActivityFeedCard`** generik yang sudah dibuat (`packages/ui/src/components/{data-table,activity-feed}/` — T058/T059), BUKAN komponen custom baru — konsisten radius `rounded-xl`, shadow `shadow-elevated`, `bg-surface`

### Revisi Warna Bar Persentase (Final)
Draft awal menyebut "3 warna hijau/kuning/merah" — setelah `01-colors.md` diperketat (hanya `success`/`danger` untuk delta biner, TIDAK ada warna ketiga "kuning/warning" generik kecuali konteks status workflow tabel), bar ini dipetakan ulang:
- **Hadir** → `--color-success-bg`/`--color-success-text` (hijau, sudah ada token)
- **Alfa** → `--color-danger-bg`/`--color-danger-text` (merah, sudah ada token)
- **Izin/Sakit** → **`--color-status-shipped-bg`/`--color-status-shipped-text`** (amber, token T057) — ini pemakaian PERSIS sesuai keputusan T057 "status workflow kategori tambahan, HANYA untuk konteks tabel/status non-binary" — bar persentase kehadiran dianggap konteks yang sama (3+ kategori non-binary), bukan pelanggaran aturan monokrom

## Scope: Per Kampus

Konsisten dengan `guru_piket` yang sudah di-scope per `kampus_id` ([[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]]) — **setiap kampus TV-nya sendiri**, menampilkan data kampus itu saja. Tidak ada gabungan lintas kampus dalam 1 layar TV.

## Isi / Widget (4 kelompok data — dikoreksi dari draft awal yang menyebut "6", isinya sebenarnya 4)

1. **Bar persentase hadir/izin/alfa** — persentase siswa hari itu, di kampus tsb, dihitung dari `attendance_records` + `permits` dengan logic sama seperti modul Rekap ([[Projek/AbsenSI/06-Features/rekap-kehadiran|rekap-kehadiran.md]]) — **reuse service Rekap yang sudah ada, JANGAN hitung ulang logic alfa dari nol**
2. **Nama siswa tidak hadir + keterangan** — daftar siswa alfa/izin/sakit hari itu berikut keterangannya, pakai `DataTableCard` (kolom: Nama, Kelas, Status via `StatusBadge`, Keterangan)
3. **Guru yang belum absensi/mulai mengajar di kelas** — dari `teaching_sessions` (guru yang jadwalnya sudah lewat jam mulai + toleransi tapi `started_at` masih `null`), pakai `ActivityFeedCard` (icon chip `danger`, teks: nama guru + kelas + mapel + "belum mulai, terlambat X menit")
4. **Guru yang izin** — dari `teacher_permits` (status "Diizinkan" hari itu), pakai `ActivityFeedCard` (icon chip `primary-soft` kalau tugas sudah diisi, `danger` kalau `follow_up_needed: true` — sinyal "Perlu Ditindaklanjuti" dari T046 harus menonjol di sini juga, bukan cuma di dashboard admin_jurnal)

## Keputusan Final (2026-07-22)

- [x] **Layout** — bento grid final di atas (bar 1 baris + 2 kolom tengah + 1 baris bawah), pakai komponen generik `DataTableCard`/`ActivityFeedCard` yang sudah ada
- [x] **Widget siswa tidak hadir (potensi ratusan baris)** → **auto-scroll otomatis** di dalam card (list bergulir sendiri tiap beberapa detik tanpa interaksi apa pun — device TV tidak punya mouse/keyboard). Kecepatan scroll & jeda pause per halaman: nilai default masuk akal (misal scroll 1 layar penuh per 8 detik, jeda 2 detik di posisi awal/akhir) — bisa disesuaikan saat implementasi, bukan angka yang mengunci
- [x] **Realtime** — Socket.IO, channel `attendance:kampus:{id}` (channel yang SAMA dengan yang sudah dipakai Dashboard Piket kerja, `dashboard-piket.md` — TV ini subscribe channel yang sama, tidak perlu channel baru), konsisten pola existing `attendance:today`/`attendance:kampus:{id}`
- [x] **Auth** — akun `guru_piket` kampus tsb, **token TANPA masa berlaku (no expiry)** — beda dari pola TV Kepsek (30 hari) karena TV ini murni read-only (tidak ada aksi tulis dari layar), risiko dibatasi ke kebocoran data-baca kalau device dicuri, BUKAN risiko manipulasi data. **WAJIB ada mekanisme revoke manual** — `super_admin` bisa mencabut sesi/token TV kapan saja dari admin panel (lihat implikasi skema di bawah) kalau device hilang/dicuri, sesuai kebutuhan kontrol keamanan minimal
- [x] **Route** — `/tv-piket/[kampusId]` di `apps/web` (route dengan parameter kampus, BUKAN `/tv-piket` polos — supaya 1 device per kampus bisa dikonfigurasi eksplisit ke kampusnya, konsisten pola kiosk yang device-specific), terpisah dari `/tv` (TV Kepsek) dan terpisah dari `(piket)/piket` (dashboard kerja piket interaktif)

## Implikasi Skema (Baru)

- **Token sesi TV tanpa expiry** butuh mekanisme tersendiri, BUKAN sekadar refresh token JWT biasa (yang selalu punya `exp`) — opsi: tabel baru `tv_sessions` (`id`, `kampusId`, `token` unique, `revokedAt` nullable, `createdAt`) mirip pola `kiosks.deviceToken` yang sudah ada (ADR-021), validasi request TV cek `revokedAt IS NULL`. **Revoke** = set `revokedAt = now()`, request berikutnya dengan token itu ditolak
- Endpoint admin: `POST /tv-sessions` (generate token baru untuk kampus tsb, role `super_admin`), `POST /tv-sessions/:id/revoke` (role `super_admin`)
- **JANGAN** pakai JWT access/refresh token biasa untuk TV ini — mekanismenya sudah punya `exp` bawaan yang bertentangan dengan keputusan "tanpa expiry", perlu token custom seperti pola kiosk

## 🔗 Terkait — Dashboard Kepsek (Fase 3, dicatat singkat)

Pemilik proyek awalnya minta TV Kepsek juga diperbarui isinya (sama seperti TV Piket + tambahan persentase kehadiran mingguan per kelas & persentase kehadiran mingguan guru) — **diputuskan masuk Fase 3, tidak urgent sekarang**. [[Projek/AbsenSI/06-Features/dashboard-tv|dashboard-tv.md]] (Fase 1, route `/tv`) untuk saat ini **tidak berubah**. Catatan untuk nanti dipakai sebagai starting point saat Fase 3 dimulai — jangan mulai desain ulang `/tv` sebelum itu.

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/dashboard-guru-jurnal|dashboard-guru-jurnal.md]] — sumber data guru belum absen kelas & guru izin (dependensi utama)
- [[Projek/AbsenSI/06-Features/dashboard-piket|dashboard-piket.md]] — pola scope per kampus, dashboard kerja piket yang berdampingan dengan TV ini
- [[Projek/AbsenSI/06-Features/dashboard-tv|dashboard-tv.md]] — TV kepsek existing (Fase 1, final, tidak berubah untuk sekarang)
- [[Projek/AbsenSI/06-Features/rekap-kehadiran|rekap-kehadiran.md]] — logic hitung persentase hadir/izin/alfa
- [[Projek/AbsenSI/13-Backlog|13-Backlog]]

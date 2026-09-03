# Mockup Figma — Modul Jurnal Guru (Design System v2)

← Index (00-INDEX AbsenSI.md)

> **Status: MOCKUP SELESAI, MENUNGGU PERSETUJUAN.** Dibuat 2026-09-02 lewat MCP Figma remote. BELUM ada perubahan kode — sesuai workflow "Module Mockup as Approval Gate": replikasi persis fitur existing dengan visual Design System v2, ditunjukkan ke user dulu sebelum implementasi.

## Lokasi

Figma file **"AbsenSI"** (`https://www.figma.com/design/zPRebQuE5Lo77ZFjSIVfJD/AbsenSI`), page baru **"Module - Jurnal Guru"** — page terpisah dari foundations "Design System v2" dan dari mockup modul lain ("Module - Pembina Ekstra"), mengikuti konvensi penamaan yang sudah ada di file ini.

## Cakupan — 6 Screen, Mobile 375px (breakpoint mobile v2)

| # | Screen | Berdasarkan Kode | Isi |
|---|---|---|---|
| 1 | Shell Mobile (referensi) | `guru-shell.tsx`, `top-bar.tsx`, `bottom-nav.tsx` | Top bar (hamburger+judul+bell+avatar) + Bottom nav 4 item (Jadwal/Presensi/Jurnal/Nilai), item aktif chip oranye |
| 2 | Jadwal Mengajar Hari Ini | `jadwal-view.tsx`, `sesi-card.tsx` | Badge Sudah/Belum Tap Gerbang, 2 sesi card (Berlangsung, Belum Waktunya), tombol Mulai Mengajar+Izin, disabled reason text |
| 3 | Presensi — Step 1 Pilih Kelas | `presensi-view.tsx` | List 3 kelas yang diajar (icon chip, nama, kampus) — step kalender/detail tidak dibuat terpisah (representatif, flow 3-step sudah dipahami dari step 1) |
| 4 | Sesi Aktif — Presensi + Jurnal | `attendance-table.tsx`, `journal-form.tsx` | Card Presensi Kelas (badge Hadir hijau/Tidak Ada merah, ikon warning belum tap gerbang) + Card Jurnal Mengajar 6 field urutan PERSIS kode (Elemen→CP→TP→Materi→Tugas→Catatan), indikator "Tersimpan" |
| 5 | Nilai — Daftar Assessment | `nilai-view.tsx` | Tombol Buat Penilaian Baru, list assessment dengan badge progress "X/Y dinilai" |
| 6 | Nilai — Input per Assessment | `assessment-detail-view.tsx` | Tombol kembali, header assessment+cakupan pertemuan, tabel nilai per siswa (input 0-100), tombol Simpan Nilai |

## Prinsip Kerja

- **Copy/label/urutan field PERSIS kode asli** — tidak direka ulang. Contoh: urutan field jurnal (Elemen→CP→TP→Materi→Tugas→Catatan) dicek dari `journal-form.tsx` FIELDS array; label tombol "Mulai Mengajar"/"Izin"/"Simpan Nilai" dicek dari kode.
- **Semua warna/radius/tipografi di-bind ke variable v2** (`v2/Color/Semantic`, dll) via `setBoundVariableForPaint` — bukan hex hardcode. Kalau token berubah di masa depan, mockup ini ikut sinkron.
- **Setiap screen di-screenshot + vision-verify** sebelum lanjut ke screen berikutnya — 1 bug ditemukan & diperbaiki (badge "Belum Waktunya" overflow wrap 2 baris keluar pill, diperbaiki dengan memperlebar pill + right-align).
- Belum termasuk: dialog "Buat Penilaian Baru" (form pilih kelas→centang pertemuan) dan step kalender/detail presensi — bisa ditambah sesi lanjutan kalau user minta lebih detail.

## Langkah Selanjutnya

1. User review mockup di Figma.
2. Diskusi perbaikan fitur (user eksplisit: "setelah berhasil dibuat baru nanti kita perbaiki sesuai perbaikan fitur — sementara sama dulu") — draft ini SENGAJA replikasi 1:1 tampilan lama, perbaikan UX menyusul di iterasi berikutnya.
3. Setelah disetujui final → ditulis sebagai task implementasi (task-WEB-xxx) untuk Claude Code, BUKAN dikerjakan Hermes langsung (aturan proyek: Hermes tidak menulis kode).

## 🔗 Lihat Juga
- `06-Features/design-system-v2/00-INDEX.md` — arsitektur 3 lapis Design System v2
- `06-Features/dashboard-guru-jurnal.md` — spec fungsional modul Jurnal Guru
- `STATUS.md` — daftar task aktif

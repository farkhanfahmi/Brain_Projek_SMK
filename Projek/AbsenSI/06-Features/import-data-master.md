---
tags: [absensi, feature, import, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Import Data Master (Guru, Siswa, Kartu RFID)

← Index (00-INDEX AbsenSI.md)

> **Naik ke scope Fase 1** (awalnya dianggap fase 3/nice-to-have, dinaikkan karena rollout awal 2.500 kartu tidak praktis kalau input manual satu-satu). Mencakup 3 alur: import data siswa, import data guru, dan pemetaan kartu RFID ke orang.

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟢 Final — siap jadi task |
| Owner | Developer 3 (apps/api — logic import & validasi) + Developer 1 (apps/web — UI upload & laporan hasil) |

---

## ⚠️ Urutan Wajib (Dependency)

```
1. Input manual: Kelas & Jurusan (master data, jumlah kecil — puluhan, bukan ribuan)
2. Import CSV: Data Siswa (mapping ke Kelas/Jurusan yang SUDAH ada)
3. Import CSV: Data Guru
4. Pemetaan Kartu RFID ke Siswa/Guru (hybrid — lihat di bawah)
```

**Kelas & Jurusan TIDAK auto-create saat import siswa.** Kalau baris CSV mereferensikan kelas yang belum terdaftar, baris itu **gagal dengan pesan error jelas** (bukan otomatis bikin kelas baru) — mencegah duplikat kelas akibat typo/inkonsistensi penulisan di file sumber data.

---

## Sub-Fitur 1: Import Data Siswa (CSV)

**Kolom yang dibutuhkan (usulan, perlu dikonfirmasi sesuai format data sekolah yang ada):**
```
nisn, nama, kelas, jurusan, (kolom lain opsional: tanggal_lahir, dst.)
```

**Validasi per baris:**
- `nisn` unik (tolak duplikat, baris gagal dicatat)
- `kelas` + `jurusan` harus sudah ada di master data — kalau tidak ketemu, baris gagal
- Baris invalid **tidak menggagalkan seluruh proses** — partial commit: baris valid tetap masuk, baris gagal dilaporkan di akhir dengan nomor baris + alasan

**Output:** laporan hasil import — "X baris berhasil, Y baris gagal" + detail baris gagal & alasannya (downloadable, supaya bisa diperbaiki & re-upload baris yang gagal saja)

## Sub-Fitur 2: Import Data Guru (CSV)

Sama strukturnya dengan import siswa.
```
nip, nama, (mapel yang diajar — opsional, baru relevan penuh di fase 2 saat jadwal mengajar didesain)
```

## Sub-Fitur 3: Pemetaan Kartu RFID — Hybrid (Resolved 2026-06-25)

> Hasil diskusi: UID kartu untuk rollout awal **sudah diketahui** (ada daftar dari vendor/pengadaan), TAPI ada siswa baru yang kartunya belum terekam. Maka dibutuhkan **2 mode**, bukan cuma CSV.

### Mode A — Bulk CSV (rollout awal, UID sudah diketahui)
```
nisn_nip, uid
```
- Prasyarat: `nisn_nip` harus sudah ada di sistem (dari Sub-Fitur 1/2)
- Validasi: `uid` harus unik (tolak duplikat — 1 UID cuma boleh aktif ke 1 orang, sesuai aturan di manajemen-kartu.md (06-Features/manajemen-kartu.md))
- Baris gagal (NISN/NIP tidak ditemukan, atau UID sudah terpakai) dilaporkan terpisah, partial commit sama seperti Sub-Fitur 1

### Mode B — Tap-to-Assign (siswa baru / kartu belum terekam)
- UI: admin pilih nama dari daftar siswa/guru **yang belum punya kartu aktif** (filter otomatis, supaya tidak salah pilih yang sudah punya kartu)
- Admin minta orang itu tap kartu di reader yang tersambung ke PC admin
- Sistem capture UID real-time saat tap terjadi, langsung tampilkan konfirmasi "Kartu berhasil dipasangkan ke [nama]"
- Didesain untuk proses cepat berurutan (next antrian, bukan form terpisah per orang) — cocok dipakai saat hari pembagian kartu

## ✅ Open Questions — Resolved (2026-07-03)

- [x] **Format kolom CSV final** → Dikonfirmasi saat developer mulai breakdown task modul Import — perlu lihat contoh file Dapodik/Excel yang sekolah punya. Kolom usulan di atas (`nisn, nama, kelas, jurusan`) dijadikan default; developer menyesuaikan saat melihat data riil. Ini **tidak blocking** task creation — dicatat sebagai langkah pertama task modul Import.
- [x] **Siapa yang boleh jalankan import** → Sub-Fitur 1 & 2 (siswa & guru): `super_admin` saja. Sub-Fitur 3 (pemetaan kartu): `super_admin` + `card_admin` (sesuai scope role `card_admin` di ADR-008).
- [x] **Fitur "undo" import** → Tidak ada undo. Terlalu kompleks untuk manfaatnya di skala ini. Mitigasi: laporan hasil import mendetail (baris gagal + alasan) supaya admin bisa review sebelum commit, dan proses partial commit (baris valid masuk, baris gagal dilaporkan) sudah meminimalkan kerusakan dari file yang tidak sempurna. Kalau import salah besar, admin koreksi manual atau re-import dengan data yang sudah diperbaiki.

**Status spec:** ✅ Final — siap dipecah jadi task development.

## 🔗 Lihat Juga
- Manajemen Kartu RFID (06-Features/manajemen-kartu.md)
- 03-User-Roles (03-User-Roles.md)
- 13-Backlog (13-Backlog.md)


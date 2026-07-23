---
tags: [absensi, feature, dashboard, realtime, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Dashboard TV Realtime

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> Tampilan realtime rekap kehadiran untuk TV di ruang kepala sekolah. Push via WebSocket, bukan polling.

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟢 Final — siap jadi task |
| Owner | Developer 2 (apps/kiosk atau app terpisah untuk TV — perlu diputuskan apakah TV display bagian dari `apps/kiosk` atau modul Next.js sendiri) |

---

## Fungsi Utama
- Tampilkan jumlah hadir/terlambat/belum hadir hari ini (live, update instan tiap ada tap baru)
- Update via Socket.IO channel `attendance:today`, server push setiap event `attendance.recorded`
- Tampilan ringan, didesain untuk dilihat dari jarak jauh (font besar, kontras tinggi)

## Rekap Fleksibel (juga dipakai di dashboard admin, bukan cuma TV)
Filter yang harus didukung:
- Per kelas
- Per jurusan
- Per mapel (fase 2)
- Per rentang tanggal/hari
- Per status (hadir/terlambat/tidak hadir/bolos — bolos baru relevan fase 2)

**Catatan performa:** dengan data tahunan (2.500 siswa × ±200 hari sekolah/tahun = ±500rb baris/tahun), query filter butuh index komposit yang tepat di kolom `(tanggal, kelas_id)`, `(tanggal, status)`, dst. Desain index final menyusul di [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]].

## Akun & Auth (Resolved)
- TV dashboard **tetap butuh autentikasi** — tidak dianggap "tampilan publik" meski di ruang terbatas (lihat [[Projek/AbsenSI/03-User-Roles|03-User-Roles]])
- Role khusus **Kepala Sekolah (`kepsek`)** — akun login tersendiri, read-only
- **Sesi TV:** akun `kepsek` di TV menggunakan refresh token berumur panjang (**30 hari, sliding renewal**) — setiap kali halaman dimuat atau Socket.IO event masuk, token diperbarui otomatis di background. TV tidak pernah perlu re-login manual selama aktif dipakai. Kalau token expire (TV mati lebih dari 30 hari), admin login ulang sekali dari keyboard/mouse yang tersedia.

## ✅ Open Questions — Resolved

- [x] **TV display: app terpisah atau route di app yang sudah ada?** → **Route `/tv` di `apps/web`** (Next.js admin app), bukan app terpisah dan bukan di `apps/kiosk`.
  - **Alasan:** `apps/web` sudah punya Socket.IO client, auth context, dan semua API client data attendance — tinggal buat route `/tv` dengan layout khusus (fullscreen, font besar, no navbar, no sidebar).
  - `apps/kiosk` tidak tepat karena tugasnya capture tap RFID — bukan display agregat.
  - App terpisah menambah deployment target baru (Nginx config, container, env vars) tanpa manfaat yang proporsional.
  - **Implementasi:** layout `apps/web/app/tv/layout.tsx` terpisah dari layout admin biasa, tidak ada navbar/sidebar, optimasi untuk layar besar dan jarak pandang jauh.

**Status spec:** ✅ Final — siap dipecah jadi task development.

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang]]
- [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack — bagian Realtime]]


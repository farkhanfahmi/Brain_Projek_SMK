---
tags: [absensi, feature, dashboard, realtime, fase-1]
status: draft
updated: 2026-06-25
---

# Feature — Dashboard TV Realtime

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> Tampilan realtime rekap kehadiran untuk TV di ruang kepala sekolah. Push via WebSocket, bukan polling.

---

## 📋 Status
| Item | Detail |
|---|---|
| Phase | Fase 1 |
| Status | 🟡 Draft |
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

**Catatan performa:** dengan data tahunan (2.500 siswa × ±200 hari sekolah/tahun = ±500rb baris/tahun), query filter butuh index komposit yang tepat di kolom `(tanggal, kelas_id)`, `(tanggal, status)`, dst. Desain index final menyusul di [[30.Projects/AbsenSI/04-Database-Schema|04-Database-Schema]].

## Akun & Auth (Resolved)
- TV dashboard **tetap butuh autentikasi** — tidak dianggap "tampilan publik" meski di ruang terbatas (lihat [[30.Projects/AbsenSI/03-User-Roles|03-User-Roles]])
- Role khusus **Kepala Sekolah (`kepsek`)** — akun login tersendiri, read-only
- Tantangan teknis yang perlu didesain saat breakdown task: TV biasanya nyala terus tanpa keyboard/mouse fisik di unit TV — perlu mekanisme sesi login yang tidak butuh re-login harian (session token berumur panjang khusus device TV, atau browser kiosk yang login sekali dan tidak pernah logout otomatis)

## ❓ Open Questions
- [ ] TV display itu app Next.js terpisah, atau cukup route khusus di `apps/kiosk`/`apps/web`? — perlu didiskusikan saat mulai breakdown task

## 🔗 Lihat Juga
- [[30.Projects/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang]]
- [[30.Projects/AbsenSI/02-Tech-Stack|02-Tech-Stack — bagian Realtime]]

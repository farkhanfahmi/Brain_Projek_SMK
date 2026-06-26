---
tags: [absensi, feature, kelas, fase-2, planning]
status: planning
updated: 2026-06-25
---

# Feature — Absensi Kelas & Mapel (Fase 2 — Planning Only)

← [[Projek/AbsenSI/00-INDEX|Index]]

> **Belum dikerjakan.** Dokumen ini hanya mencatat rencana & keputusan awal agar skema database fase 1 sudah kompatibel. Detail spec lengkap baru disusun saat fase 2 dimulai.

---

## Konsep Inti

- Reader RFID dipasang di setiap ruang kelas + ruang praktek
- Siswa **wajib sudah tap gerbang** hari itu sebelum bisa tap kelas — kalau belum, tap kelas ditolak
- Tap kelas mencatat kehadiran **per sesi mapel** (dibandingkan jadwal pelajaran saat itu)
- Guru tap kelas → dibandingkan jadwal mengajarnya → status terlambat per-sesi (bukan per-hari seperti fase 1)
- Siswa tap gerbang tapi TIDAK tap kelas tertentu → status **bolos mapel** itu, bukan tidak hadir sekolah

## ⚠️ Risiko yang Sudah Diidentifikasi (belum diputuskan solusinya)

**Gerbang sekolah tidak punya penghalang fisik (bukan turnstile).** Siswa bisa secara fisik masuk sekolah tanpa tap kartu (lupa, buru-buru, dll). Kalau aturan "wajib tap gerbang dulu" di-enforce sebagai **hard block** di sistem, siswa yang fisik hadir tapi lupa tap gerbang akan ditolak absen kelasnya juga — padahal dia hadir. Ini bisa jadi sumber komplain guru/siswa yang sebenarnya bukan bug software, tapi gap antara desain logis dan realita fisik.

**Opsi yang perlu didiskusikan saat fase 2 dimulai:**
1. Hard block penuh (sesuai rencana awal) — risiko false-negative di atas
2. Soft block — sistem tetap catat tap kelas, tapi beri flag "belum tap gerbang" untuk dicek admin manual
3. Kombinasi — guru kelas punya tombol override manual untuk tandai hadir meski sistem tidak lihat tap gerbang

**Belum diputuskan — wajib dibahas ulang sebelum fase 2 mulai development.**

## Siklus Sesi Kelas — Tap Buka/Tutup oleh Guru (dicatat 2026-06-25)

> Desain dari diskusi: setiap ruang kelas/praktek punya AIO/mini-PC + monitor sendiri. Konsep "sesi terkunci" untuk cegah siswa tap iseng di luar jam pelajaran.

1. Sistem **tidak menerima tap apa pun** dari siswa di luar jam pelajaran terjadwal
2. Saat jam pelajaran mulai, **guru mapel tap dulu** → ini membuka sesi (`session: open`), menandai mulai pembelajaran
3. Selama sesi `open`, siswa maju tap kartu satu per satu untuk presensi kelas
4. **Sebelum guru tap pembuka, tap dari siswa tidak akan terbaca/diproses** — ini mencegah siswa iseng tap saat guru belum di kelas
5. Saat pembelajaran selesai, guru tap lagi → menutup sesi (`session: closed`)
6. Guru mapel berikutnya tap untuk membuka sesi baru di jam berikutnya

## ❓ Open Questions — Siklus Sesi Kelas (untuk dibahas detail saat fase 2 dimulai)
- [ ] **Lupa tutup sesi:** Kalau guru lupa tap tutup di akhir jam pelajaran, dan jam berikutnya kosong/istirahat — apakah sesi otomatis tertutup berdasarkan jadwal (auto-close at scheduled end time), atau tetap `open` sampai ada tap penutup manual? Risiko kalau tidak auto-close: siswa bisa iseng tap saat istirahat karena sesi masih "terbuka".
- [ ] **Guru berikutnya tap buka padahal sesi sebelumnya belum ditutup:** apakah sistem otomatis tutup sesi lama lalu buka sesi baru, atau tolak/beri warning?
- [ ] **Tap ganda siswa dalam 1 sesi:** siswa tap 2x karena bercanda — apakah diabaikan (idempotent, sudah tercatat hadir) atau ada efek lain?
- [ ] **Guru lupa tap pembuka sama sekali, langsung mulai mengajar:** siswa tidak bisa tap presensi sepanjang jam itu — apakah ada override manual (misal guru input manual lewat dashboard kalau lupa tap di awal)?
- [ ] **Identitas sesi vs UID guru:** sesi dibuka oleh UID guru yang tap — berarti guru piket/pengganti yang beda dari guru mapel terjadwal tap, sistem harus tetap terima (guru pengganti) atau validasi ketat harus guru yang sesuai jadwal? Kalau validasi ketat, guru pengganti akan ditolak sistem padahal fisik hadir mengajar.

## Implikasi ke Desain Fase 1

- Skema DB harus sudah punya struktur **jadwal mengajar per guru per kelas per jam** (modul Schedule) dari fase 1, meskipun belum dipakai untuk reader kelas — supaya fase 2 tidak rebuild skema (ADR-005)
- Modul Attendance fase 1 harus dirancang generic untuk "sesi tap" (bisa sesi gerbang ATAU sesi kelas nanti), bukan hardcode hanya untuk gerbang

## 🔗 Lihat Juga
- [[Projek/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang (Fase 1)]]
- [[Projek/AbsenSI/11-Decisions|ADR-005]]
- [[Projek/AbsenSI/13-Backlog|13-Backlog]]


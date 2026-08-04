# T089 — API+UI: Redesign Tampilan Absensi Kiosk (Guru/Staf/Siswa) + Watermark Copyright

## Depends on
Tidak ada — extend `TapResponse` (backend) + redesign `FeedbackScreen`/`IdleScreen` (`apps/kiosk`), tidak mengubah logic tap/debounce/attendance yang sudah ada.

## Context
- **App:** `apps/api` + `apps/kiosk`
- **File:** `apps/api/src/attendance/attendance.service.ts` (atau lokasi yang generate `TapResponse`), `apps/kiosk/src/components/feedback-screen.tsx`, `idle-screen.tsx`, `guru-dashboard.tsx`
- **Ref:** User berikan 4 file sketsa desain 2026-07-26 (`Guru pengajar.svg`, `staf karyawan.svg`, `stanby.svg`, `gambar-100.jpg.jpeg` — versi dengan foto asli+logo sekolah). Sketsa BUKAN untuk ditiru persis (dikonfirmasi user: "tidak harus sama persis, anda bisa perbaiki warna/ukuran"), tapi jadi acuan STRUKTUR layout dan informasi apa yang harus ada.

## Spec Detail

### Keputusan Final (dikonfirmasi user, 2026-07-26)
- **Perbedaan Guru vs Staf/Karyawan HANYA 2 hal**: label status ("Guru Pengajar" vs "Staf/Karyawan") dan section "Jadwal Mengajar Hari Ini" (HANYA muncul untuk guru, TIDAK muncul untuk staf/karyawan). Selebihnya (layout foto, riwayat, dst) SAMA.
- **Panel "Riwayat Datang"/"Riwayat Pulang"** di sketsa = **riwayat KOLEKTIF** (5 orang berbeda yang baru tap kartu di kiosk itu, siapapun) — BUKAN riwayat pribadi 1 orang. Ini SAMA PERSIS dengan yang sudah ada di `GuruDashboard` (`kiosk-recent-table.tsx`, `KioskRecentEntry`) — REUSE komponen itu, jangan bikin ulang.
- **Tampilan kartu personal (foto besar+nama+jam+jadwal) MENGGANTIKAN `FeedbackScreen` yang sudah ada** — bukan dashboard baru terpisah. Tetap tampil sesaat (pola timing yang sama seperti sekarang) lalu kembali ke idle/dashboard.
- **UI Siswa**: TIDAK sama persis dengan guru/staf, dan **TIDAK ADA** section riwayat datang/pulang. Cukup: foto, nama, jam absen — sesuai instruksi eksplisit user ("yang terpenting ada foto, nama, jam absennya").
- **Watermark**: teks copyright "Develop By Bisnis Canter TEKAJE" di bagian bawah SEMUA layar kiosk (idle, feedback guru/staf, feedback siswa) — kecil, tidak mengganggu, posisi footer.

### Backend — Extend `TapResponse` (WAJIB, data belum di-expose sekarang)
`TapResponse` (`packages/types/src/index.ts:110-130`) SAAT INI cuma punya `name`, `status`, `tapType`, `time`, `foto` — TIDAK ADA `niy`/`nisn`, `statusKepegawaian` (guru vs staf), atau jadwal mengajar. Field baru yang perlu ditambahkan (semua opsional, hanya terisi kalau relevan):
```typescript
export interface TapResponse {
  // ...field existing TIDAK diubah...
  /** NIY guru / NISN siswa — ditampilkan di kartu personal kiosk */
  identifier?: string;
  /** Guru vs Staf/Karyawan — HANYA untuk kiosk tipe guru, tentukan label + tampilkan/sembunyikan section jadwal */
  statusKepegawaian?: "guru" | "karyawan";
  /** Jadwal mengajar HARI INI (hanya diisi kalau statusKepegawaian === "guru" DAN ada jadwal type=jam_mengajar untuk teacherId ini di hari ini) */
  jadwalHariIni?: Array<{ kelas: string; jamMulai: string; jamSelesai: string }>;
}
```
- Backend (`attendance.service.ts` atau service yang generate response tap) — setelah tap guru berhasil, query `Teacher.statusKepegawaian` + (kalau `guru`) query `Schedule` `type: jam_mengajar`, `teacherId`, `hari: <hari ini>` untuk isi `jadwalHariIni`
- Untuk tap siswa — isi `identifier` dengan NISN, TIDAK isi `statusKepegawaian`/`jadwalHariIni` (field itu `undefined` untuk siswa)
- **JANGAN bikin query jadwal berat/N+1** — kalau tap terjadi sangat sering (jam masuk sekolah, puluhan tap beruntun), query jadwal HARUS cepat (index `teacherId`+`hari` di `Schedule` sudah ada atau cek dulu)

### Frontend — `apps/kiosk`

#### 1. `FeedbackScreen` — Redesign Total Varian Guru/Staf (Accepted, bukan rejected)
Ganti card polos warna solid (hijau/oranye) jadi layout mirip sketsa:
- **Kolom kiri (utama)**: foto besar bulat (reuse `KioskAvatar`, perbesar ukurannya dari 160 ke ~200-240px sesuai proporsi sketsa), nama, label status ("Guru Pengajar" / "Staf/Karyawan" berdasar `statusKepegawaian`), NIY (`identifier`), "Tercatat Absensi" + timestamp lengkap
- **Section "Jadwal Mengajar Hari Ini"** — HANYA render kalau `statusKepegawaian === "guru"` DAN `jadwalHariIni` tidak kosong, list kelas+jam (kalau kosong array tapi guru, tampilkan state kosong singkat "Tidak ada jadwal mengajar hari ini", JANGAN sembunyikan section-nya sepenuhnya biar konsisten layout)
- **Kolom kanan**: REUSE `KioskRecentTable` x2 (Riwayat Datang + Riwayat Pulang, SAMA PERSIS seperti yang ada di `GuruDashboard` sekarang — komponen sudah ada, sekadar dipindah render-nya ke overlay feedback ini)
- Warna: TETAP pertahankan pembeda status (hijau=hadir, oranye=terlambat) tapi terapkan sebagai AKSEN (border/badge), bukan background solid penuh layar seperti sekarang — supaya layout kompleks di atas tetap terbaca (background solid penuh tidak cocok untuk layout sekompleks ini). Pakai token design system AbsenSI yang sudah ada (`--color-success-*`/`--color-warning-*` kalau ada, JANGAN reinvent warna baru sendiri)
- `justLocked` (kartu terkunci, T037/ADR-025) — TETAP pertahankan varian overlay solid gelap yang sudah ada, TIDAK perlu redesign kompleks (kasus jarang, pesan singkat sudah cukup)

#### 2. `FeedbackScreen` — Varian Siswa (SEDERHANA, beda dari guru/staf)
Sesuai instruksi eksplisit user "jangan samakan persis seperti guru, tidak ada riwayat":
- Foto besar (boleh proporsi lebih kecil dari guru, karena tidak ada panel kanan yang perlu diimbangi), Nama, Jam absen — TITIK, tidak lebih
- TIDAK ADA riwayat datang/pulang di kartu siswa
- TIDAK ADA NIY/jadwal (itu murni konsep guru)

#### 3. `IdleScreen` — Sesuaikan dengan Sketsa "stanby.svg"
Sketsa menunjukkan jam BESAR jadi elemen utama (bukan cuma pelengkap kecil di bawah judul seperti sekarang) — perbesar proporsi jam digital, pertahankan pola "AbsenSI" + "Tempelkan Kartu Untuk Absen" yang sudah ada (kode existing sudah dekat dengan sketsa ini, cuma perlu penyesuaian ukuran/hierarchy visual, BUKAN rombak total)

#### 4. Watermark Copyright (SEMUA Layar Kiosk)
- Teks kecil di footer/bagian bawah: **"© <tahun berjalan> — Develop By Bisnis Canter TEKAJE"** (tahun dinamis dari `new Date().getFullYear()`, bukan hardcode)
- Terapkan di: `IdleScreen`, `FeedbackScreen` (semua varian: accepted guru/staf, accepted siswa, justLocked, rejected), `GuruDashboard` (kalau tidak sudah tertutup elemen lain)
- Styling: kecil (`text-caption`/setara ~12px), warna redup (`text-ink-tertiary` atau opacity rendah), posisi `fixed bottom` atau di dalam flow layout paling bawah — TIDAK mengganggu keterbacaan konten utama

## JANGAN
- ❌ JANGAN tiru sketsa SVG persis (warna oranye/cokelat solid, ukuran, font serif Times) — user eksplisit bilang boleh disesuaikan ke design system AbsenSI yang sudah ada (warna primary oranye EzMart, font sans yang sudah dipakai di seluruh app)
- ❌ JANGAN bikin ulang komponen riwayat datang/pulang dari nol — REUSE `KioskRecentTable`/`KioskRecentEntry` yang sudah ada persis, cukup dipindah lokasi render-nya
- ❌ JANGAN tampilkan riwayat datang/pulang atau NIY/jadwal di kartu SISWA — instruksi eksplisit user, siswa CUMA foto+nama+jam
- ❌ JANGAN query jadwal mengajar dengan cara yang lambat/N+1 — ini di jalur kritis tap kartu (harus responsif, siswa/guru antre tap di gerbang)
- ❌ JANGAN hardcode tahun di watermark copyright — pakai `new Date().getFullYear()`

## Files
- **Modifikasi:** `packages/types/src/index.ts` — extend `TapResponse` (field baru: `identifier`, `statusKepegawaian`, `jadwalHariIni`)
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` (atau file yang generate response tap — cek dulu nama pastinya) — isi field baru saat tap guru/siswa
- **Modifikasi:** `apps/kiosk/src/components/feedback-screen.tsx` — redesign total varian accepted (guru/staf beda dari siswa), pertahankan varian `justLocked`/rejected apa adanya
- **Modifikasi:** `apps/kiosk/src/components/idle-screen.tsx` — perbesar proporsi jam sesuai sketsa
- **Modifikasi (kemungkinan):** `apps/kiosk/src/components/guru-dashboard.tsx` — cek apakah panel riwayat masih relevan di sini juga setelah dipindah ke feedback screen, atau cukup 1 tempat saja (jangan duplikat 2x kalau menyebabkan render ganda tidak perlu)
- **Buat (baru, shared):** komponen kecil `KioskWatermark` di `apps/kiosk/src/components/` — 1 komponen dipakai di semua layar, bukan copy-paste teks yang sama berkali-kali

## Acceptance Criteria
- [ ] Tap kartu guru sukses → kartu personal tampil: foto besar, nama, "Guru Pengajar", NIY, jam tap, jadwal mengajar hari ini (kalau ada), panel riwayat datang+pulang kolektif di sebelahnya
- [ ] Tap kartu staf/karyawan sukses → SAMA seperti guru TAPI label "Staf/Karyawan" dan TANPA section jadwal mengajar
- [ ] Tap kartu siswa sukses → HANYA foto, nama, jam — tidak ada riwayat, tidak ada NIY/jadwal
- [ ] Semua layar kiosk (idle, feedback guru/staf/siswa) menampilkan watermark "Develop By Bisnis Canter TEKAJE" di bagian bawah, tahun otomatis mengikuti tahun berjalan
- [ ] Layar standby (idle) menampilkan jam dengan proporsi besar sesuai sketsa
- [ ] Tap kartu tetap responsif (tidak ada jeda tambahan signifikan dari query jadwal baru)
- [ ] Verifikasi visual langsung di browser/kiosk sungguhan (bukan cuma review kode) — task ini murni UI-facing, WAJIB dicoba tampil nyata sebelum ditandai selesai

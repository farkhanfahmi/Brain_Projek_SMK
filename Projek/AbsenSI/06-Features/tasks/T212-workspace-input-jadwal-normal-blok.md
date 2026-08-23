# T212 — Web: Workspace Input Jadwal (Mode Normal & Blok) — Checklist Hari, Save-per-Hari, Bentrok Real-Time, Date Generator

## Depends on
**WAJIB SETELAH T206, T207, T208, T210** (butuh SEMUA backend: JadwalSlot CRUD+cek-ketersediaan-guru, MapelGuru terisi, Date Generator, halaman Menu Utama yang mengarahkan ke sini). **Task PALING KOMPLEKS di seluruh rangkaian T203-T215** — PERTIMBANGKAN memecah jadi sub-task lebih kecil SAAT implementasi kalau terasa terlalu besar untuk 1 sesi (misal T212a Checklist+Tab Hari, T212b Tabel Input+Dropdown Guru Real-time, T212c Date Generator UI Blok) — TAPI task ini ditulis sebagai 1 kesatuan supaya sesi manapun yang mengerjakan paham GAMBARAN UTUH sebelum pecah sendiri.

## Objective
Halaman Workspace (`/jadwal-pelajaran/[opsiJadwalId]`) — SPA tanpa refresh, 2 section utama (Jadwal per Kelas & Jadwal per Guru, saling sync), dengan:
1. Checklist Hari GLOBAL-PERSISTEN (tidak reset saat ganti dropdown Kelas).
2. Alur input: Pilih Kelas → Tab per Hari (hanya hari tercentang) → Tabel (Jam Ke+Waktu autofill dari Alokasi Waktu, admin isi Mapel+Guru saja) → Save PER TAB HARI → toast sukses → auto-pindah tab berikutnya.
3. Dropdown Guru — **real-time cek bentrok** saat dibuka (endpoint T206), guru bentrok disabled+badge merah.
4. KHUSUS mode Blok: 2 tab besar Minggu A/B, masing-masing punya section Kelas+Guru sendiri, PLUS UI Date Generator (input 2 tanggal mulai, tombol Generate, tampilkan hasil ringkas).

## Konteks — Detail Alur Dikonfirmasi User (2026-08-16)

Alur BUKAN "1x save untuk semua hari sekaligus" — melainkan **Save PER TAB HARI**, dengan auto-advance ke tab berikutnya setelah save berhasil (mempercepat input berurutan tanpa admin perlu klik tab manual tiap kali). Ini detail dari file referensi `dariDev/penjadwalan.md`, DIKONFIRMASI user sebagai alur yang benar (bukan salah tafsir Claude).

## Spec Detail

### 1. Komponen Global — Checklist Hari Persisten

- State React di level Workspace TERTINGGI (BUKAN di dalam komponen section Kelas/Guru masing-masing) — checkbox/pill toggle Senin-Sabtu. Ganti dropdown Kelas TIDAK me-reset state ini.
- Hari yang TERSEDIA di checklist — **HANYA hari yang ADA slot-nya di Alokasi Waktu** yang dirujuk Opsi Jadwal ini (kalau Alokasi Waktu tidak punya Sabtu, opsi Sabtu di checklist DISABLED/tidak muncul — JANGAN tampilkan opsi yang tidak mungkin diisi).

### 2. Section "Jadwal per Kelas"

- Dropdown pilih Kelas (dari SEMUA kelas, ATAU difilter cakupan tingkat Opsi Jadwal ini kalau `tingkatScopes` tidak kosong — VERIFIKASI logic filter ini benar).
- Setelah Kelas dipilih → render **Tab Pill** per hari (HANYA hari yang tercentang di Checklist Global) — KONSISTEN styling Tab pill dari design-system.
- **Di dalam tab 1 hari** — tabel:
  - Baris = tiap `jamKe` dari `AlokasiWaktuSlot` untuk `hari` ini (istirahat DITAMPILKAN sebagai baris non-editable, label "Istirahat", TIDAK ada dropdown Mapel/Guru untuk baris itu).
  - Kolom Jam Ke, Waktu Mulai-Selesai — **AUTO-FILL dari AlokasiWaktuSlot**, TIDAK BISA diedit di sini.
  - Kolom Mapel — dropdown single-select (SEMUA Mapel, ATAU difilter Mapel yang relevan jurusan kelas ini via `MapelJurusan` T201, KONSISTEN filter yang sudah ada).
  - Kolom Guru — **Multi-Select Tagging (badge/chip)** — dropdown BERISI HANYA guru yang terdaftar `MapelGuru` untuk Mapel yang dipilih di baris itu (kalau Mapel belum dipilih, dropdown Guru DISABLED dengan pesan "Pilih Mapel dulu"). **SETIAP KALI dropdown ini DIBUKA** — panggil `GET /jadwal-slot/cek-ketersediaan-guru` (T206) REAL-TIME — guru yang BENTROK ditampilkan **disabled** dengan badge merah `(Mengajar di Kelas [Nama Kelas])`.
  - Baris TANPA Mapel/Guru diisi (kosong) — DIABAIKAN saat save (bukan error, admin BOLEH tidak isi semua jam).
- **Tombol "Save [Nama Hari]"** di bawah tabel — submit HANYA baris hari itu ke `POST /jadwal-slot` (batch untuk semua baris terisi di tab itu) — SUKSES → **Toast Notification hijau 3 detik** → **otomatis pindah fokus ke Tab hari berikutnya** (dalam urutan checklist yang tercentang, kalau ini tab TERAKHIR, tetap di situ atau tampilkan pesan "Semua hari sudah diinput").

### 3. Section "Jadwal per Guru" — Two-Way Sync

- Dropdown pilih Guru (dari SEMUA guru yang terdaftar di `MapelGuru` MANAPUN — supaya relevan).
- Setelah Guru dipilih → tampilkan tabel jadwal guru itu (SEMUA `JadwalSlot` yang punya guru ini di `JadwalSlotGuru`, untuk Opsi Jadwal ini) — grouped by hari.
- **INI BUKAN JALUR INPUT TERPISAH** — ini VIEW dari data `JadwalSlot` YANG SAMA yang diisi dari Section Kelas — TIDAK ADA DUPLIKASI DATA. TAPI user JUGA minta bisa INPUT dari section Guru (poin 6 diskusi: "admin bisa input dari tab session kelas/guru hasilnya terimplementasi di keduanya") — JADI section Guru INI JUGA punya form input (Pilih Kelas di dalam context Guru yang sudah dipilih → sama seperti alur section Kelas) — REKOMENDASI: REUSE komponen tabel input yang SAMA PERSIS antara section Kelas dan section Guru (parameterisasi "siapa yang sudah fixed" — di section Kelas, Kelas sudah fixed pilih Mapel+Guru; di section Guru, Guru sudah fixed pilih Kelas+Mapel) — JANGAN duplikasi 2 komponen tabel terpisah yang gampang divergen.

### 4. KHUSUS Mode Blok — Tab Minggu A/B + Date Generator

- Kalau `OpsiJadwal.mode === blok` — di ATAS Section Kelas/Guru, ada **Tab besar Minggu A / Minggu B** — MEMPENGARUHI SEMUA input di bawahnya (section Kelas dan Guru untuk Minggu A TERPISAH dari Minggu B, meski keduanya dalam 1 Opsi Jadwal yang sama — 4 kombinasi total: A-Kelas, A-Guru, B-Kelas, B-Guru, KONSISTEN pemahaman Tahap 6 diskusi).
- **Date Generator** — panel terpisah (misal di atas Tab Minggu A/B, atau modal terpisah) — 2 input tanggal mulai (Minggu A, Minggu B) + tombol "Generate" → panggil `POST /opsi-jadwal/:id/generate-minggu` (T208) → tampilkan hasil ringkas (jumlah tanggal per minggu, jumlah yang di-skip karena libur). **Kalau Opsi Jadwal ini SUDAH punya JadwalSlot terisi** — tampilkan WARNING sebelum generate ulang (KONSISTEN Edge Case T208).
- `JadwalSlot.minggu` (A/B) diisi OTOMATIS sesuai Tab Minggu mana yang sedang aktif dipilih admin saat submit — admin TIDAK perlu pilih manual per baris.

## Edge Cases
- Guru YANG SAMA muncul di dropdown Section Kelas (mengajar Kelas A) DAN kebetulan sedang dipilih di Section Guru (viewing jadwal Guru itu) BERSAMAAN oleh admin — TIDAK ADA race condition RIIL karena keduanya baca dari SUMBER DATA SAMA (`JadwalSlot`), sinkronisasi terjadi via re-fetch setelah save, BUKAN state lokal terpisah yang bisa divergen.
- Save PER HARI GAGAL (misal validasi bentrok yang LOLOS real-time-check tapi GAGAL saat submit karena race — 2 admin submit bersamaan) — tampilkan pesan error backend APA ADANYA (T206 sudah pesan jelas), JANGAN pindah tab otomatis kalau save GAGAL (auto-advance HANYA terjadi setelah SUKSES).
- Alokasi Waktu TIDAK PUNYA slot sama sekali untuk 1 hari yang tercentang (kondisi aneh, seharusnya dicegah UI poin 1) — tab hari itu tampil KOSONG dengan pesan jelas, bukan tabel error.

## Files
- **Buat:** `apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/page.tsx` + `workspace-view.tsx` + komponen pendukung (`checklist-hari.tsx`, `tab-hari-input.tsx` [REUSE lintas Kelas/Guru], `guru-multiselect-realtime.tsx`, `date-generator-panel.tsx` [khusus blok]).
- **Jangan sentuh:** `JadwalFormModal` LAMA (T157/T189, TIDAK dihapus sampai T215).

## Acceptance Criteria
- [x] Checklist Hari persisten saat ganti dropdown Kelas.
- [x] Tab per hari hanya render untuk hari tercentang.
- [x] Tabel autofill Jam Ke+Waktu dari Alokasi Waktu, admin hanya isi Mapel+Guru.
- [x] Dropdown Guru filter hanya guru terdaftar di Mapel dipilih, real-time disable guru bentrok dengan badge merah — **REVISI: single-select** (bukan "Multi-Select Tagging" seperti draf spec asli di bawah), dikonfirmasi ulang user, KONSISTEN koreksi T209 (1 JadwalSlot = 1 guru).
- [x] Save per hari — toast sukses, auto-advance tab berikutnya.
- [x] Section Kelas dan Guru saling sync (input dari salah satu, muncul di keduanya) — REUSE komponen tabel yang sama.
- [x] Mode Blok — tab Minggu A/B berfungsi, Date Generator UI terhubung ke backend T208, warning sebelum generate ulang kalau sudah ada data.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] **WAJIB konfirmasi** section Kelas dan Guru REUSE komponen tabel input yang SAMA — DIKONFIRMASI: `components/tab-hari-input.tsx` adalah 1 komponen, dipanggil dari `TabsContent value="kelas"` DAN `TabsContent value="guru"` di `workspace-view.tsx` dengan prop `kelas` yang sama-sama diisi (Section Kelas: dari dropdown Kelas langsung; Section Guru: dari dropdown Kelas SETELAH Guru dipilih) — bukan 2 implementasi tabel terpisah.
- [x] Konfirmasi auto-advance tab HANYA terjadi setelah save SUKSES — DIKONFIRMASI: `handleSaved()` di `workspace-view.tsx` (callback `onSaved`) HANYA dipanggil dari dalam blok try SETELAH semua `apiClientFetch` sukses di `tab-hari-input.tsx`; kegagalan lempar ke `catch` yang set `error` state dan TIDAK memanggil `onSaved` — diverifikasi juga via smoke-test manual browser (lihat Status Eksekusi).
- [x] Konfirmasi dropdown Guru benar-benar panggil endpoint real-time SETIAP KALI dibuka — DIKONFIRMASI: `GuruDropdownRealtime`'s `handleOpenChange()` memanggil `apiClientFetch` ke `cek-ketersediaan-guru` setiap `onOpenChange(true)` dari `Popover`, TIDAK ada memoization/cache di luar `options` state yang di-reset (`setOptions(null)`) tiap dibuka — diverifikasi visual di smoke-test (screenshot dropdown terbuka menampilkan 2 guru MapelGuru, tanpa bentrok karena slot pertama).

## Status Eksekusi

**Selesai 2026-08-17 06:15**

Route dibuat 2x (pola T157/T210): `apps/web/src/app/(admin)/jadwal-pelajaran/[opsiJadwalId]/` (page.tsx + workspace-view.tsx + components/) dan duplikasi tipis `apps/web/src/app/(admin-jurnal)/admin-jurnal/jadwal-pelajaran/[opsiJadwalId]/page.tsx` yang REUSE `WorkspaceView` yang sama dengan `basePath` berbeda.

**Koreksi ke T206** (dengan izin eksplisit user, DI LUAR file-list T212 tapi diperlukan supaya kolom Guru konsisten): backend `cekKetersediaanGuru()` (`jadwal-slot.service.ts`) diperluas — sebelumnya cuma return `{teacherIdsTersedia, teacherIdsBentrok}` (ID saja), sekarang tambah `bentrokDetail: {teacherId, kelasNama}[]` supaya UI bisa tampilkan badge "Mengajar di Kelas X" tanpa endpoint terpisah. Diimplementasi via helper baru `findBentrokConflict()` (dipisah dari `ensureNoBentrok()` yang sekarang murni wrapper throw) — REUSE logic yang SAMA, bukan reimplementasi.

**Ditemukan & diperbaiki saat smoke-test 375px**: `TabHariInput` awalnya cuma pakai `<Table>` + `overflow-x-auto` — di layar sempit kolom Guru "hilang" di balik scroll horizontal tanpa affordance jelas (technically ada strategi eksplisit sesuai huruf design-system, tapi gagal secara visual). Diperbaiki jadi card-per-jam (`sm:hidden` vs `hidden sm:block`) — tabel HTML tetap dipakai di ≥sm, TIDAK dihapus.

**Smoke-test dijalankan sungguhan** (bukan cuma build+tsc): login browser asli sebagai `admin_jurnal` via kredensial dev, seed data test lewat API asli (AlokasiWaktu 4 slot termasuk istirahat, OpsiJadwal mode normal, MapelGuru assign 2 guru) — bukan INSERT langsung ke DB, supaya lewat validasi service layer yang sama dengan produksi. Isi Mapel+Guru 1 baris, Save, verifikasi row `jadwal_slots`+`jadwal_slot_guru` di DB benar, buka tab "Selasa" buktikan auto-advance, buka Section Guru buktikan two-way sync (baca data yang sama dari Kelas), screenshot 1440px dan 375px. Data test dihapus bersih setelahnya (`DELETE /jadwal-slot`, `/opsi-jadwal`, `/alokasi-waktu`, kosongkan `MapelGuru`).

**Insiden minor selama smoke-test**: `pnpm build` (dijalankan untuk verifikasi tsc) sempat menimpa folder `.next` yang dipakai proses `next dev` milik sesi lain yang sudah berjalan — dev server sempat 500. Dikonfirmasi ke user sebelum restart (`rm -rf .next` + `next dev` ulang), downtime ~15 detik, dipulihkan bersih. Pelajaran: JANGAN jalankan `pnpm build` produksi di direktori yang sama dengan `next dev` yang sedang aktif dipakai sesi lain.

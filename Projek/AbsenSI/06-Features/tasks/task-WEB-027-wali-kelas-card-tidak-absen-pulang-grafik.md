# Task-WEB-027: Halaman Wali Kelas — Card "Tidak Absen Pulang" di Sub-Menu Grafik

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> **Depends on task-CORE-023** — butuh model data `TidakAbsenPulangCatatan` + strike counter yang didefinisikan task itu.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low-medium
**Alasan pemilihan:** Halaman "Grafik" saat ini kosong (`ComingSoonPlaceholder`), jadi ini bukan sekadar tambahan — tapi scope-nya sempit (1 card, 1 tabel sederhana, data source sudah didefinisikan task-CORE-023).

## 2. Konteks & Tujuan Utama

Halaman `apps/web/src/app/(guru)/guru/wali-kelas/grafik/page.tsx` saat ini murni placeholder ("Sub Menu Graf", `ComingSoonPlaceholder`). Keputusan user (2026-09-03): tambahkan card **"Tidak Absen Pulang"** di halaman ini — daftar siswa (dari kelas milik wali kelas yang login) dengan kejadian "tidak absen pulang" yang **BELUM di-clear** (akumulasi menuju 3x, lihat task-CORE-023 untuk model strike counter).

**Kolom**: No | Nama | Tanggal — sorting default ASC berdasar kolom **Nama**.

**Perilaku data**: daftar ini otomatis KOSONG untuk siswa begitu di-unlock oleh piket (strike counter reset via `tidakAbsenPulangResetAt`, task-CORE-023) — TIDAK menampilkan histori permanen (itu ada di Riwayat Catatan siswa, halaman terpisah), murni "siapa yang sedang menuju 3x sekarang".

**Scope**: HANYA kelas yang wali kelas ini ampu (pola scoping SAMA seperti sub-menu wali kelas lain — cek `hari-ini-tab.tsx`/`daftar-siswa-tab.tsx` untuk pola scoping existing).

**Depends on:** task-CORE-023 (model `TidakAbsenPulangCatatan` + field `Student.tidakAbsenPulangResetAt` harus sudah ada).

## 3. Langkah Eksekusi Detail

### Backend

1. **Cek pola scoping wali kelas existing** — baca `apps/api/src/journal/` (atau modul lain yang serve endpoint wali-kelas, VERIFIKASI SAAT IMPLEMENTASI lokasi persis controller/service untuk fitur wali kelas — kemungkinan `JournalController`/`JournalService` berdasar pola task-CORE-014 yang REUSE controller yang sama) untuk memahami cara scoping "kelas milik wali kelas yang login" (`kelasIdWali` di JWT/User, atau query `Kelas.waliKelasId`).

2. **Endpoint baru** `GET /journal/wali-kelas/tidak-absen-pulang` (atau lokasi konsisten dengan endpoint wali-kelas lain, VERIFIKASI SAAT IMPLEMENTASI) — query `TidakAbsenPulangCatatan` untuk siswa di kelas wali kelas yang login, filter kejadian dalam window aktif (`tanggal >= student.tidakAbsenPulangResetAt OR tidakAbsenPulangResetAt IS NULL`, REPLIKASI logic window yang SAMA dengan `catatTidakAbsenPulangExpired()` di task-CORE-023 — JANGAN tulis ulang logic window, kalau perlu extract ke method shared yang dipanggil dari 2 tempat).
   - Response: array `{ id, studentId, nama, tanggal }`, per SISWA (bukan per baris kejadian — kalau siswa sama muncul 2x kejadian dalam window aktif, VERIFIKASI SAAT IMPLEMENTASI apakah ditampilkan 2 baris terpisah [1 baris = 1 kejadian, sesuai spesifikasi kolom "Tanggal" yang menyiratkan 1 baris = 1 tanggal kejadian] — REKOMENDASI: 1 baris = 1 kejadian, supaya wali kelas bisa lihat SEMUA tanggal kejadian yang berkontribusi ke strike, bukan cuma nama unik).
   - Role guard: sama seperti endpoint wali kelas lain (kemungkinan cek `Kelas.waliKelasId === user.teacherId`, REPLIKASI pola existing).

### Frontend

3. **`apps/web/src/app/(guru)/guru/wali-kelas/grafik/page.tsx`** — ganti dari `ComingSoonPlaceholder` jadi fetch data (server component, REPLIKASI pola `page.tsx` sub-menu wali kelas lain yang sudah fetch data awal) + render view baru.

4. **Komponen baru** `TidakAbsenPulangWaliKelasCard` (atau ditaruh langsung di `grafik/page.tsx` kalau sederhana, VERIFIKASI SAAT IMPLEMENTASI apakah perlu file terpisah) — tabel dengan kolom **No | Nama | Tanggal**, sortable (REPLIKASI `SortableHeader`, WAJIB patuh aturan tabel proyek — kolom No + semua kolom non-No sortable), **default sort ASC by Nama** (state awal `sort = { field: 'nama', dir: 'asc' }`).
   - Empty state: "Tidak ada siswa dengan catatan tidak absen pulang aktif saat ini." (atau serupa, JANGAN generic "Tidak ada data").
   - Card kosong TOTAL kalau tidak ada data (bukan section besar dengan pesan panjang, konsisten pola card ringkas lain di dashboard).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi/Buat:** endpoint backend baru (lokasi VERIFIKASI SAAT IMPLEMENTASI, kemungkinan `apps/api/src/journal/`)
- **Modifikasi:** `apps/web/src/app/(guru)/guru/wali-kelas/grafik/page.tsx` — ganti placeholder
- **Buat:** komponen tabel baru (lokasi VERIFIKASI SAAT IMPLEMENTASI, kemungkinan `apps/web/src/app/(guru)/guru/wali-kelas/components/`)
- **Jangan sentuh:** `riwayat-catatan-table.tsx` (halaman ini BEDA dari Riwayat Catatan — ini list AKTIF/live, bukan histori permanen).

**Dilarang dilakukan:**
- Jangan tampilkan siswa dari kelas LAIN (bukan milik wali kelas yang login) — scope ketat sesuai pola existing.
- Jangan reuse endpoint `TidakAbsenPulangRow`/`tidakAbsenPulangKemarin()` milik dashboard piket — itu domain BEDA (piket = klarifikasi harian AKTIF hari ini/kemarin, ini = akumulasi strike menuju lock, sumber data BEDA tabel).

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: wali kelas belum di-assign ke kelas manapun (`kelasIdWali` null) → endpoint return array kosong ATAU error jelas (VERIFIKASI SAAT IMPLEMENTASI pola existing untuk kondisi serupa di sub-menu wali kelas lain, REPLIKASI persis).
- Kondisi: siswa yang tadinya di kelas wali kelas ini PINDAH kelas SETELAH kejadian tercatat (sebelum di-unlock) → VERIFIKASI SAAT IMPLEMENTASI apakah tetap tampil ke wali kelas LAMA (data historis milik kelas SAAT kejadian) atau langsung hilang (ikut kelas BARU) — REKOMENDASI: ikut kelas siswa SAAT INI (`student.kelasId` live, BUKAN snapshot), konsisten prinsip "wali kelas urus siswa yang SEKARANG jadi tanggung jawabnya".

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Halaman Wali Kelas > Grafik tidak lagi "Coming Soon" — tampilkan card "Tidak Absen Pulang".
- [ ] Kolom No | Nama | Tanggal, default sort ASC by Nama, sortable sesuai aturan wajib tabel.
- [ ] Data HANYA siswa kelas wali kelas yang login.
- [ ] Data otomatis kosong untuk siswa yang sudah di-unlock (strike reset).
- [ ] Empty state jelas, bukan generik.
- [ ] Build + typecheck bersih.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 200 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: task-CORE-023 WAJIB selesai dulu

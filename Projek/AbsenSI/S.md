---
tags: [absensi, tasks, polish, batch3]
updated: 2026-07-30
---

# Task Perbaikan Dashboard Piket — Batch 3

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> 6 usulan perbaikan dashboard piket dari user (Fahmi), disampaikan 2026-07-30. Dianalisis lewat riset kode langsung (Explore + baca file) sebelum ditulis jadi task, sama seperti pola [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2]]. Task ditulis lengkap supaya bisa dieksekusi kapan pun tanpa perlu riset ulang dari nol — tapi tetap baca ulang kode terkait dulu sebelum mulai, kemungkinan ada perubahan kecil sejak 2026-07-30.

---

## 📊 Progress

| Task | Selesai |
|---|---|
| T090 — Menu Riwayat Izin (rentang tanggal) | 0/1 |
| T091 — Exclude siswa PKL dari Board Semua Siswa | 0/1 |
| T092 — Menu khusus Input Izin/Sakit (Tidak Masuk) | 0/1 |
| T093 — Filter Direktori Siswa: Kelas mengikuti Jurusan | 0/1 |
| T094 — Fix bug sidebar piket: highlight Dashboard nempel di semua menu | 0/1 |
| T095 — Redesign field form Perizinan | 0/1 |
| **Total** | **0/6** |

---

## ✅ Sudah Diklarifikasi Lewat Riset Kode (Sebelum Menulis Task)

1. **Endpoint `GET /permits` sudah ada filter tanggal**, tapi HANYA tanggal tunggal (`ListPermitsDto.tanggal?: string`, exact match) — BUKAN rentang. Perlu diperluas jadi rentang (`dari`/`sampai`), bukan dibuat dari nol.
2. **Filter kelas di Direktori Siswa piket (`direktori-siswa-view.tsx`) sudah ada** untuk kelas DAN jurusan, tapi keduanya independen (tidak saling filter). Perbaikannya murni UI (client-side filter opsi kelas berdasar jurusan terpilih), backend `GET /students?kelasId=&jurusanId=` sudah mendukung keduanya sekaligus, tidak perlu ubah API.
3. **Bug sidebar (usulan #5) root cause sudah ditemukan**: `PiketSidebar` (`apps/web/src/app/(piket)/piket-sidebar.tsx` baris 25) pakai `pathname === item.href || pathname.startsWith(item.href + "/")` — untuk item Dashboard (`href: "/piket"`), SEMUA path piket (`/piket/izin-keluar`, `/piket/siswa`, dst) match `startsWith("/piket/")` juga. Fix: butuh exact-match utuh untuk Dashboard, ATAU urutan pengecekan yang tidak tumpang tindih (match path terpanjang dulu).
4. **`PiketBoardView` (`piket-board-view.tsx`) TIDAK punya menu form "izin/sakit tidak masuk" mandiri** — satu-satunya cara input izin/sakit hari ini adalah tombol per-baris di tabel "Board Semua Siswa" (`PermitForm`, dipicu dari kolom Aksi). Usulan #3 minta menu terpisah sebagai cara kedua, form serupa tapi dengan pencarian siswa sendiri (pola sama seperti `IzinKeluarForm` di `izin-keluar-view.tsx`), bukan gantung di baris tabel.
5. **Siswa PKL TETAP `status: aktif`** (lihat komentar model `StudentPkl` di schema: "siswa PKL tetap status: aktif, kelasId TIDAK PERNAH berubah") — jadi filter `status: "aktif"` yang sudah ada di `piketBoard()` TIDAK cukup untuk exclude siswa PKL. Perlu tambahan exclude eksplisit siswa dengan `StudentPkl` aktif (`endedAt: null`).
6. **Field foto siswa nonaktif TIDAK relevan di sini** — pastikan task ini tidak tumpang tindih dengan histori kelas ([[project_pola_penamaan_kelas_2026]]) yang baru saja selesai dikerjakan 2026-07-30; siswa nonaktif SUDAH ter-exclude dari board via `status: "aktif"`, jadi T091 hanya perlu menangani kasus PKL.

---

## T090 — Menu "Riwayat Izin" — Siswa Izin/Sakit dengan Filter Rentang Tanggal

**Prioritas: tinggi** — dasar untuk piket melihat histori izin, bukan cuma hari ini.

**Konteks:** saat ini piket cuma bisa lihat permit HARI INI (lewat board) atau permit yang statusnya belum selesai (Belum Kembali/Perlu Ditinjau). Tidak ada cara melihat "siapa saja yang izin ke piket pada tanggal X" atau rentang tanggal tertentu secara historis.

- [ ] **Backend** — `apps/api/src/permits/dto/list-permits.dto.ts`: ganti/tambah field. Opsi termudah dan konsisten dengan pola filter lain di codebase (lihat `ListStudentsDto`/`AttendanceReportQueryDto` untuk pola `dari`/`sampai`): tambah `dari?: string` dan `sampai?: string` (keduanya `@IsDateString() @IsOptional()`), pertahankan `tanggal` lama untuk backward-compat (dipakai di tempat lain?, cek dulu — grep `ListPermitsDto` dan `findAll(` di seluruh `apps/web` sebelum menghapus).
- [ ] `apps/api/src/permits/permits.service.ts` — `findAll()`: where clause `tanggal` jadi kondisional 3 mode: (a) `filter.tanggal` diisi → exact match seperti sekarang; (b) `filter.dari`/`filter.sampai` diisi → `{ gte: parseDateOnly(dari), lte: parseDateOnly(sampai) }`; (c) tidak ada apapun → default ke HARI INI SAJA (jangan biarkan tanpa filter tanggal sama sekali, supaya tidak ambil seluruh histori tanpa batas di halaman baru). Filter tambahan yang masuk akal untuk menu ini: `jenis` (biarkan default tampilkan SEMUA jenis — baik `tidak_masuk` maupun `keluar` — supaya "izin ke piket" mencakup kedua alur, kecuali user nanti minta dipisah).
- [ ] **Halaman baru** `apps/web/src/app/(piket)/piket/riwayat-izin/page.tsx` + `riwayat-izin-view.tsx` — tabel: Nama, Kelas, Jenis (Tidak Masuk / Keluar), Kategori (Izin/Sakit), Keterangan, Tanggal, (untuk jenis=keluar: Jam Keluar, Status Kembali). Filter di atas tabel: date range picker (dari–sampai, default: hari ini saja), search nama (client-side atau lewat `nama` di endpoint kalau join student memungkinkan — cek apakah `PermitsService.findAll` sudah bisa filter by nama siswa, kalau belum tambahkan `student: { nama: { contains } }` ke where).
- [ ] `apps/web/src/app/(piket)/piket-sidebar.tsx` — tambah menu baru "Riwayat Izin" (icon: `History` dari lucide-react) ke `NAV_ITEMS`, href `/piket/riwayat-izin`.
- [ ] Cek `apps/web/src/lib/core-types.ts` — `Permit` interface, pastikan field yang ditampilkan (jenis, tanggal, dst) semua sudah ada di tipe, tambah kalau kurang.

**Ref:** `apps/api/src/permits/permits.service.ts` (`findAll`), `apps/api/src/permits/dto/list-permits.dto.ts`, `apps/api/src/permits/permits.controller.ts`, `apps/web/src/app/(piket)/piket-sidebar.tsx`.

---

## T091 — Board Semua Siswa: Exclude Siswa PKL

**Prioritas: tinggi** — bug fungsional, piket bingung kenapa siswa yang sedang PKL di luar sekolah tercatat "Belum Hadir".

**Konteks:** `piketBoard()` di `apps/api/src/attendance/attendance.service.ts` (baris ±279) query `status: "aktif"` tapi siswa PKL statusnya TETAP `aktif` (by design, lihat komentar `StudentPkl` di schema). Siswa PKL tidak absen tap di sekolah selama PKL berlangsung, jadi seharusnya tidak muncul di board "Belum Hadir".

- [ ] `apps/api/src/attendance/attendance.service.ts` — `piketBoard()`: tambah exclude siswa dengan record `StudentPkl` yang masih berjalan (`endedAt: null`) pada tanggal `today`. Cara: tambah `where` clause `NOT: { pklRecords: { some: { endedAt: null } } }` pada query `student.findMany`, ATAU (kalau butuh presisi berdasarkan tanggalMulai/tanggalSelesai bukan cuma endedAt) filter `pklRecords: { none: { tanggalMulai: { lte: today }, OR: [{ tanggalSelesai: null }, { tanggalSelesai: { gte: today } }] } }` — pilih sesuai bagaimana PKL sebenarnya "aktif saat ini" dipakai di tempat lain (cek `apps/api/src/core/students/students.service.ts` baris ±133 yang sudah query `pklRecords: { where: { endedAt: null } }` untuk pola serupa, ikuti pola yang sama untuk konsistensi — artinya `endedAt: null` saja kemungkinan sudah cukup dan konsisten dengan kode existing).
- [ ] Pertimbangkan: apakah siswa PKL perlu SAMA SEKALI hilang dari board, atau tetap muncul tapi dengan badge/status khusus "PKL" (mirip kolom PKL yang sudah ada di `/siswa` admin, lihat `pklStudentIdSet` di `siswa-view.tsx`)? **User secara eksplisit bilang "jangan tampilkan siswa yang PKL"** — jadi default: EXCLUDE total dari board, bukan cuma ubah badge. Konfirmasi ke user kalau ternyata ingin tetap terlihat tapi berbeda.
- [ ] Verifikasi: siswa dengan `StudentPkl` aktif tidak muncul di Board Semua Siswa; siswa yang PKL-nya sudah `endedAt` terisi (sudah selesai) muncul normal kembali.

**Ref:** `apps/api/src/attendance/attendance.service.ts` (`piketBoard`), `apps/api/prisma/schema.prisma` (model `StudentPkl`), `apps/api/src/core/students/students.service.ts` (pola query pkl aktif yang sudah ada, baris ±128-133).

---

## T092 — Menu Khusus "Input Izin/Sakit" (Cara Kedua, Terpisah dari Board)

**Prioritas: sedang.**

**Konteks:** saat ini satu-satunya cara menandai siswa izin/sakit (jenis `tidak_masuk`, BUKAN izin keluar sementara) adalah tombol di kolom Aksi tabel "Board Semua Siswa" (dalam dialog `PermitForm`, lihat `piket-board-view.tsx` baris ±646). User minta menu MANDIRI kedua untuk hal yang sama — supaya piket tidak harus buka section Board dulu untuk menemukan barisnya.

- [ ] **Halaman baru** `apps/web/src/app/(piket)/piket/input-izin/page.tsx` + `input-izin-view.tsx` — form dengan pencarian siswa mandiri (contoh pola: `IzinKeluarForm` di `izin-keluar-view.tsx` baris ±153, pakai `PiketBoardRow[]` dari `attendanceService.piketBoard()` sebagai sumber data siswa+status hari ini — supaya form bisa cek siswa yang SUDAH ditandai izin/hadir dan mencegah submit ganda), field: cari nama siswa (autocomplete dari board), kategori (Izin/Sakit), keterangan. Submit ke `POST /permits` dengan `jenis: "tidak_masuk"` — endpoint SUDAH ADA dan sudah benar (`PermitsService.createTidakMasuk`, termasuk validasi "sudah tap tidak bisa ditandai izin").
- [ ] Setelah submit sukses, tampilkan konfirmasi + opsi lanjut input siswa lain (form reset, tetap di halaman yang sama) — beda dari alur dialog di board yang otomatis tertutup.
- [ ] `apps/web/src/app/(piket)/piket-sidebar.tsx` — tambah menu baru "Input Izin/Sakit" (icon: `ClipboardPlus` atau `FileText` dari lucide-react) ke `NAV_ITEMS`, href `/piket/input-izin`.
- [ ] Pastikan TIDAK menghapus tombol izin/sakit yang sudah ada di board (`PermitForm` di board tetap ada sebagai cara pertama) — user eksplisit minta 2 cara, bukan pindah lokasi.
- [ ] Cek server component page.tsx pola existing (`apps/web/src/app/(piket)/piket/izin-keluar/page.tsx`) untuk cara fetch `initialBoard`/`petugas` di server sebelum render client view — ikuti pola yang sama untuk konsistensi data awal (SSR) dan supaya piket-duty-context tetap berfungsi.

**Ref:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (`PermitForm`, baris ±646, untuk direuse/dicontoh), `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` (`IzinKeluarForm`, pola pencarian siswa), `apps/api/src/permits/permits.service.ts` (`createTidakMasuk`, sudah ada, tidak perlu diubah), `apps/web/src/app/(piket)/piket-sidebar.tsx`.

---

## T093 — Direktori Siswa Piket: Filter Kelas Mengikuti Jurusan Terpilih

**Prioritas: rendah** — murni UX, bukan bug data.

**Konteks:** halaman `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx` sudah punya filter Kelas dan Jurusan (dropdown terpisah), tapi keduanya independen — pilih jurusan tidak mempersempit opsi kelas yang tampil. User minta: **kalau jurusan dipilih, dropdown Kelas cuma tampilkan kelas milik jurusan itu; kalau jurusan "Semua", tampilkan semua kelas seperti sekarang.**

- [ ] `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx` — computed `kelasListFiltered = jurusanId === ALL ? kelasList : kelasList.filter((k) => k.jurusanId === Number(jurusanId))` (cek dulu apakah `Kelas` interface di `core-types.ts` punya field `jurusanId` langsung atau cuma nested `jurusan.id` — sesuaikan). Pakai `kelasListFiltered` sebagai sumber `<SelectItem>` di dropdown Kelas (baris ±117), bukan `kelasList` mentah.
- [ ] Kalau user ganti Jurusan sementara Kelas yang sedang dipilih TIDAK ADA di jurusan baru (mis. pilih "XII TKJ 1" lalu ganti Jurusan ke "TKR") — reset `kelasId` balik ke `ALL` otomatis lewat `useEffect` yang watch `jurusanId`, supaya tidak nyangkut di filter kelas yang sudah tidak valid untuk jurusan itu.
- [ ] Terapkan pola yang SAMA di halaman admin `/siswa` (`apps/web/src/app/(admin)/siswa/siswa-view.tsx`) KALAU ADA filter kelas+jurusan serupa di sana juga — user cuma menyebut "data siswa" secara umum, cek dulu apakah maksudnya cuma direktori piket atau juga admin. **Klarifikasi ke user sebelum kerjakan bagian admin** kalau ternyata ada 2 tempat filter serupa dan tidak jelas mana yang dimaksud.

**Ref:** `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx`, `apps/web/src/lib/core-types.ts` (`Kelas`, `Jurusan`).

---

## T094 — Fix Bug Sidebar Piket: Highlight "Dashboard" Menempel di Semua Menu

**Prioritas: tinggi** — bug visual jelas terlihat, salah paham navigasi.

**Konteks:** root cause SUDAH ditemukan lewat baca kode (lihat bagian klarifikasi di atas). `apps/web/src/app/(piket)/piket-sidebar.tsx` baris 25:
```ts
const isActive = pathname === item.href || pathname.startsWith(`${item.href}/`);
```
Untuk item "Dashboard" (`href: "/piket"`), path apapun yang diawali `/piket/` (yaitu SEMUA menu piket lain: `/piket/izin-keluar`, `/piket/siswa`, dan menu baru T090/T092 nanti) ikut match `startsWith("/piket/")`, sehingga "Dashboard" selalu tampil ter-highlight oranye bersamaan dengan menu manapun yang sedang aktif.

- [ ] `apps/web/src/app/(piket)/piket-sidebar.tsx` — perbaiki logic `isActive`. Cara paling aman: exact match untuk href yang JUGA merupakan prefix dari href lain (`/piket`), sedangkan untuk href yang lebih spesifik boleh tetap pakai `startsWith` untuk sub-route (misal `/piket/siswa/123` kalau ada rute detail turunan). Contoh perbaikan:
  ```ts
  const isActive = item.href === "/piket"
    ? pathname === "/piket"
    : pathname === item.href || pathname.startsWith(`${item.href}/`);
  ```
  Atau, lebih robust untuk skala (kalau menu baru T090/T092 nanti juga py sub-route): urutkan `NAV_ITEMS` dari href terpanjang, cari MATCH TERPANJANG sekali saja (bukan tiap item independen mengecek dirinya sendiri) — pertimbangkan pendekatan ini kalau solusi sederhana di atas terasa rapuh setelah menu bertambah jadi 5 (T090+T092 nambah 2 lagi).
- [ ] Terapkan hal yang sama di `apps/web/src/components/shell/sidebar.tsx` (sidebar admin) KALAU pola `isActive` di sana juga punya bug serupa — cek dulu, sidebar admin punya lebih banyak menu jadi risiko sama ada di sana juga.
- [ ] Verifikasi manual: klik tiap menu piket satu-satu, pastikan HANYA menu yang sedang dibuka yang berwarna oranye, "Dashboard" oranye HANYA saat benar-benar di `/piket`.

**Ref:** `apps/web/src/app/(piket)/piket-sidebar.tsx` (`NavLink`, `isActive`), `apps/web/src/components/shell/sidebar.tsx` (cek pola serupa).

---

## T095 — Redesign Field Form Perizinan (Lebih Modern)

**Prioritas: rendah** — polish visual, tidak ada perubahan fungsional.

**Konteks:** form-form perizinan saat ini (`PermitForm`, `IzinKeluarForm`, `LockForm`, `UnlockForm`, `TidakTapPulangForm` di `piket-board-view.tsx` dan `izin-keluar-view.tsx`) pakai styling dasar: `<Input>`/`<select>` native dengan `h-10 rounded-lg`, tanpa ikon, tanpa state visual selain warna border default. User minta desain field yang "lebih modern" — **WAJIB baca [[design_system_absensi.md]] dulu** sebelum menyentuh styling apapun (lihat [[feedback_design_system_wajib_dibaca]] — pelanggaran desain sebelumnya terjadi karena lewati langkah ini).

- [ ] Baca ulang `06-Features/design-system/*.md` di vault — cek warna aksen (harus 1 aksen saja), radius, shadow, spacing yang berlaku untuk form field bergaya "modern" di proyek ini (kemungkinan pola serupa `ganti-password-form.tsx` yang sudah pakai ikon di dalam input + rounded-full + border halus, lihat `apps/web/src/app/ganti-password/ganti-password-form.tsx` sebagai referensi pola field modern yang SUDAH ada di codebase — reuse pola ini alih-alih bikin gaya baru).
- [ ] Terapkan pola field modern (ikon di kiri input, radius konsisten, focus state jelas) ke SEMUA form perizinan yang disebutkan di atas — tapi JANGAN ubah field `<select>` kategori Izin/Sakit jadi native `<select>` polos, ganti ke komponen `Select` dari `@absensi/ui` (sudah dipakai di `IzinKeluarForm`, belum dipakai di `TidakTapPulangForm` yang masih native `<select>`, lihat baris ±621).
- [ ] Konsisten dengan form baru dari T090/T092 (buat dulu form itu dengan style baru ini sejak awal, supaya tidak double-kerjakan).
- [ ] Verifikasi visual (screenshot Playwright/manual) sebelum+sesudah, minta review user karena ini murni preferensi visual — style "modern" bisa ditafsirkan beda-beda, konfirmasi arahnya sudah benar sebelum menyelesaikan semua form.

**Ref:** `apps/web/src/app/ganti-password/ganti-password-form.tsx` (pola field modern existing, dijadikan referensi), `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (semua form di dalamnya), `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` (`IzinKeluarForm`), design-system docs di vault (WAJIB dibaca dulu).

---

## Catatan Umum

- Baca ulang bagian kode terkait sebelum tiap task — deskripsi di sini berdasarkan riset kode per 2026-07-30, mungkin sudah ada perubahan kecil dari task/batch sebelumnya.
- Kerjakan T094 (fix bug sidebar) LEBIH DULU atau paralel dengan T090/T092 — keduanya menambah menu baru ke `NAV_ITEMS`, lebih murah memperbaiki logic `isActive` sebelum menu bertambah jadi 5 daripada sesudahnya.
- T090 dan T092 sama-sama menyentuh alur permit `jenis: tidak_masuk` — pertimbangkan mengerjakan berurutan (T090 dulu untuk lihat data yang sudah ada, baru T092 untuk cara input baru) supaya lebih mudah verifikasi silang (input lewat T092, cek muncul di T090).
- T091 murni backend (query), risiko regresi rendah, bisa dikerjakan kapan saja terpisah dari task lain.
- Setiap task selesai: build (`pnpm turbo run build` atau `./scripts/build.sh` kalau ada) + type-check hijau, verifikasi manual atau Playwright untuk perubahan UI, restart dev/prod server setelah build, update checklist & catatan implementasi di file ini (ikuti format T029-T037 di [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2]] sebagai contoh: tulis "Implementasi:", "Verifikasi:", dan bug yang ditemukan selama pengerjaan kalau ada).

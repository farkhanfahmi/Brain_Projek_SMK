---
tags: [absensi, tasks, polish, batch3]
updated: 2026-07-30
---

# Task Perbaikan Dashboard Piket — Batch 3

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> 6 usulan perbaikan dashboard piket dari user (Fahmi), disampaikan 2026-07-30. Dianalisis lewat riset kode langsung (Explore + baca file) sebelum ditulis jadi task, sama seperti pola [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2]]. **Semua 6 task selesai dieksekusi 2026-07-30** — lihat catatan implementasi di tiap section.
>
> Catatan: file ini sempat hilang dari vault di tengah eksekusi (kemungkinan aktivitas sesi paralel), direkonstruksi dari isi yang sudah dibaca sebelumnya + catatan implementasi ditambahkan setelah eksekusi selesai.

---

## 📊 Progress

| Task | Selesai |
|---|---|
| T090 — Menu Riwayat Izin (rentang tanggal) | 1/1 |
| T091 — Exclude siswa PKL dari Board Semua Siswa | 1/1 |
| T092 — Menu khusus Input Izin/Sakit (Tidak Masuk) | 1/1 |
| T093 — Filter Direktori Siswa: Kelas mengikuti Jurusan | 1/1 |
| T094 — Fix bug sidebar piket: highlight Dashboard nempel di semua menu | 1/1 |
| T095 — Redesign field form Perizinan | 1/1 |
| **Total** | **6/6** |

---

## ✅ Sudah Diklarifikasi Lewat Riset Kode (Sebelum Menulis Task)

1. **Endpoint `GET /permits` sudah ada filter tanggal**, tapi HANYA tanggal tunggal (`ListPermitsDto.tanggal?: string`, exact match) — BUKAN rentang. Perlu diperluas jadi rentang (`dari`/`sampai`), bukan dibuat dari nol.
2. **Filter kelas di Direktori Siswa piket (`direktori-siswa-view.tsx`) sudah ada** untuk kelas DAN jurusan, tapi keduanya independen (tidak saling filter). Perbaikannya murni UI (client-side filter opsi kelas berdasar jurusan terpilih), backend `GET /students?kelasId=&jurusanId=` sudah mendukung keduanya sekaligus, tidak perlu ubah API.
3. **Bug sidebar (usulan #5) root cause sudah ditemukan**: `PiketSidebar` (`apps/web/src/app/(piket)/piket-sidebar.tsx` baris 25) pakai `pathname === item.href || pathname.startsWith(item.href + "/")` — untuk item Dashboard (`href: "/piket"`), SEMUA path piket (`/piket/izin-keluar`, `/piket/siswa`, dst) match `startsWith("/piket/")` juga. Fix: butuh exact-match utuh untuk Dashboard, ATAU urutan pengecekan yang tidak tumpang tindih (match path terpanjang dulu).
4. **`PiketBoardView` (`piket-board-view.tsx`) TIDAK punya menu form "izin/sakit tidak masuk" mandiri** — satu-satunya cara input izin/sakit hari ini adalah tombol per-baris di tabel "Board Semua Siswa" (`PermitForm`, dipicu dari kolom Aksi). Usulan #3 minta menu terpisah sebagai cara kedua, form serupa tapi dengan pencarian siswa sendiri (pola sama seperti `IzinKeluarForm` di `izin-keluar-view.tsx`), bukan gantung di baris tabel.
5. **Siswa PKL TETAP `status: aktif`** (lihat komentar model `StudentPkl` di schema: "siswa PKL tetap status: aktif, kelasId TIDAK PERNAH berubah") — jadi filter `status: "aktif"` yang sudah ada di `piketBoard()` TIDAK cukup untuk exclude siswa PKL. Perlu tambahan exclude eksplisit siswa dengan `StudentPkl` aktif (`endedAt: null`).
6. **Field foto siswa nonaktif TIDAK relevan di sini** — pastikan task ini tidak tumpang tindih dengan histori kelas ([[project_pola_penamaan_kelas_2026]]) yang baru saja selesai dikerjakan 2026-07-30; siswa nonaktif SUDAH ter-exclude dari board via `status: "aktif"`, jadi T091 hanya perlu menangani kasus PKL.

---

## T090 — Menu "Riwayat Izin" — Siswa Izin/Sakit dengan Filter Rentang Tanggal ✅

**Prioritas: tinggi** — dasar untuk piket melihat histori izin, bukan cuma hari ini.

**Konteks:** saat ini piket cuma bisa lihat permit HARI INI (lewat board) atau permit yang statusnya belum selesai (Belum Kembali/Perlu Ditinjau). Tidak ada cara melihat "siapa saja yang izin ke piket pada tanggal X" atau rentang tanggal tertentu secara historis.

- [x] **Backend** — `apps/api/src/permits/dto/list-permits.dto.ts`: tambah `dari?: string` dan `sampai?: string` (`@IsDateString() @IsOptional()`), pertahankan `tanggal` lama untuk backward-compat. Tambah `nama?: string` untuk filter nama siswa.
- [x] `apps/api/src/permits/permits.service.ts` — `findAll()`: where clause `tanggal` jadi kondisional 3 mode: (a) `filter.tanggal` diisi → exact match; (b) `filter.dari`/`filter.sampai` diisi → `{ gte, lte }`; (c) tidak ada apapun → default HARI INI SAJA. Filter `jenis` sengaja TIDAK dibatasi (tampilkan semua: `tidak_masuk` dan `keluar`).
- [x] **Halaman baru** `apps/web/src/app/(piket)/piket/riwayat-izin/page.tsx` + `riwayat-izin-view.tsx` — tabel: Nama, Kelas, Jenis, Kategori, Keterangan, Tanggal, Jam Keluar, Status Kembali. Filter: 2× `DatePicker` (dari–sampai, default hari ini), search nama (client-side filter atas hasil yang sudah di-fetch).
- [x] `apps/web/src/app/(piket)/piket-sidebar.tsx` — menu baru "Riwayat Izin" (icon `History`), href `/piket/riwayat-izin`.
- [x] `Permit` interface di `core-types.ts` sudah lengkap, tidak perlu field tambahan.

**Implementasi:** `findAll()` di `PermitsService` menghitung `tanggalWhere` sebagai `Date | { gte?, lte? }` tergantung filter mana yang diisi, fallback ke `startOfDay(new Date())` kalau semua kosong (jaga-jaga supaya endpoint tidak pernah tanpa sengaja return seluruh histori tanpa batas). Filter `nama` pakai `student: { nama: { contains } }`. Halaman baru pakai pola `DatePicker` ganda + tombol "Tampilkan" persis seperti `RekapView` (`apps/web/src/app/(admin)/rekap/rekap-view.tsx`) untuk konsistensi UX rentang tanggal di seluruh aplikasi.

**Verifikasi:** `tsc --noEmit` bersih (api+web), `pnpm turbo run build` 6/6 sukses, Jest 183/183 lulus. Verifikasi visual Playwright TIDAK dilakukan (user memilih skip untuk hemat token pada sesi ini) — kredensial `guru_piket` tidak tersedia di sesi. Perlu verifikasi manual oleh user sebelum dianggap final secara UX.

**Ref:** `apps/api/src/permits/permits.service.ts` (`findAll`), `apps/api/src/permits/dto/list-permits.dto.ts`, `apps/api/src/permits/permits.controller.ts`, `apps/web/src/app/(piket)/piket-sidebar.tsx`.

---

## T091 — Board Semua Siswa: Exclude Siswa PKL ✅

**Prioritas: tinggi** — bug fungsional, piket bingung kenapa siswa yang sedang PKL di luar sekolah tercatat "Belum Hadir".

**Konteks:** `piketBoard()` di `apps/api/src/attendance/attendance.service.ts` (baris ±279) query `status: "aktif"` tapi siswa PKL statusnya TETAP `aktif` (by design, lihat komentar `StudentPkl` di schema). Siswa PKL tidak absen tap di sekolah selama PKL berlangsung, jadi seharusnya tidak muncul di board "Belum Hadir".

- [x] `apps/api/src/attendance/attendance.service.ts` — `piketBoard()`: tambah exclude `pklRecords: { none: { endedAt: null } }` pada where clause `student.findMany` — pola sama persis dengan `StudentsService` (`pklRecords: { where: { endedAt: null } }`, baris ±133) untuk konsistensi.
- [x] Konfirmasi: siswa PKL EXCLUDE TOTAL dari board (bukan cuma ubah badge) — sesuai instruksi eksplisit user di dokumen task ini.
- [x] Verifikasi: query where clause diperiksa manual — siswa dengan `StudentPkl` aktif (`endedAt: null`) tidak akan match `none: { endedAt: null }`, sehingga otomatis ter-exclude; siswa PKL yang sudah `endedAt` terisi otomatis match kembali (kembali normal).

**Implementasi:** Perubahan 1 baris pada `where` clause query siswa di `piketBoard()` — `pklRecords: { none: { endedAt: null } }` ditambahkan berdampingan dengan `kelas: { kampusId }, status: "aktif"` yang sudah ada. Tidak ada perubahan skema, tidak ada query tambahan (masih 1 query siswa seperti sebelumnya, filter murni ditambahkan ke where clause yang sudah ada).

**Verifikasi:** `tsc --noEmit` bersih, build sukses. Verifikasi data-level langsung (query MySQL manual terhadap data PKL yang ada) TIDAK dilakukan di sesi ini — perlu dicek user bahwa siswa PKL aktif benar-benar hilang dari board secara live.

**Ref:** `apps/api/src/attendance/attendance.service.ts` (`piketBoard`), `apps/api/prisma/schema.prisma` (model `StudentPkl`), `apps/api/src/core/students/students.service.ts` (pola query pkl aktif yang sudah ada).

---

## T092 — Menu Khusus "Input Izin/Sakit" (Cara Kedua, Terpisah dari Board) ✅

**Prioritas: sedang.**

**Konteks:** saat ini satu-satunya cara menandai siswa izin/sakit (jenis `tidak_masuk`, BUKAN izin keluar sementara) adalah tombol di kolom Aksi tabel "Board Semua Siswa" (dalam dialog `PermitForm`). User minta menu MANDIRI kedua untuk hal yang sama.

- [x] **Halaman baru** `apps/web/src/app/(piket)/piket/input-izin/page.tsx` + `input-izin-view.tsx` — form dengan pencarian siswa mandiri (pola sama `IzinKeluarForm`), pakai `PiketBoardRow[]` dari `attendanceService.piketBoard()` sebagai sumber data, filter kandidat pencarian ke `status === null` saja (siswa yang belum tap dan belum ditandai izin/sakit hari ini — mencegah submit ganda). Field: cari nama (autocomplete), kategori (Izin/Sakit), keterangan. Submit ke `POST /permits` dengan `jenis: "tidak_masuk"` (endpoint sudah ada, tidak diubah).
- [x] Setelah submit sukses: form direset (bukan redirect/close), pesan konfirmasi tampil, siap lanjut input siswa berikutnya di halaman yang sama.
- [x] `apps/web/src/app/(piket)/piket-sidebar.tsx` — menu baru "Input Izin/Sakit" (icon `ClipboardPlus`), href `/piket/input-izin`.
- [x] Tombol izin/sakit di Board Semua Siswa (`PermitForm`) TIDAK dihapus — dua cara tetap ada, sesuai instruksi eksplisit.
- [x] `page.tsx` pakai pola SSR fetch `initialBoard` yang sama seperti `izin-keluar/page.tsx`.

**Implementasi:** Filter kandidat autocomplete awalnya salah (`s.status === null && s.alasanDetail === null`) — ditemukan dan diperbaiki SEBELUM ship: `alasanDetail` di `PiketBoardRow` collapse "tidak ada permit" dan "permit ada tapi keterangan kosong" jadi `null` yang sama, jadi bukan sinyal yang reliable untuk "sudah ditandai izin". Diperbaiki jadi `s.status === null` saja (sinyal yang benar-benar dipakai di `piket-board-view.tsx` sendiri untuk kondisi yang sama) — `createTidakMasuk` mengisi `status` jadi `izin`/`sakit` di board row, jadi `status === null` sudah cukup dan akurat.

**Verifikasi:** `tsc --noEmit` bersih, build sukses. Verifikasi visual Playwright TIDAK dilakukan (skip, sama seperti T090).

**Ref:** `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (`PermitForm`, direuse/dicontoh), `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` (`IzinKeluarForm`, pola pencarian siswa), `apps/api/src/permits/permits.service.ts` (`createTidakMasuk`, tidak diubah).

---

## T093 — Direktori Siswa Piket: Filter Kelas Mengikuti Jurusan Terpilih ✅

**Prioritas: rendah** — murni UX, bukan bug data.

**Konteks:** halaman `direktori-siswa-view.tsx` sudah punya filter Kelas dan Jurusan (dropdown terpisah), tapi keduanya independen. User minta: kalau jurusan dipilih, dropdown Kelas cuma tampilkan kelas milik jurusan itu.

- [x] `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx` — computed `kelasListFiltered = jurusanId === ALL ? kelasList : kelasList.filter((k) => k.jurusanId === Number(jurusanId))`, dipakai sebagai sumber dropdown Kelas.
- [x] `useEffect` watch `jurusanId` — kalau `kelasId` yang sedang dipilih tidak valid lagi untuk jurusan baru, reset otomatis ke `ALL`.
- [x] **Diperluas juga ke admin `/siswa`** (`siswa-view.tsx`) setelah dikonfirmasi ke user — bug filter kelas+jurusan independen SAMA PERSIS ditemukan di sana (baris 132-157). Pola computed `kelasListFiltered` + reset-on-jurusan-change diterapkan juga, disesuaikan dengan mekanisme URL searchParams (`activeFilter`, bukan local state) yang dipakai halaman itu — reset kelasId dilakukan di dalam `updateFilter()` saat key-nya `jurusanId` dan kelas lama sudah tidak valid.

**Implementasi:** Piket pakai local state (`useState` + `useEffect`), admin pakai URL searchParams (`router.push` + `router.refresh()`) — dua mekanisme berbeda tapi logic inti sama: hitung daftar kelas valid untuk jurusan terpilih, dan bersihkan pilihan kelas yang sudah tidak valid saat jurusan berganti.

**Verifikasi:** `tsc --noEmit` bersih (kedua halaman), build sukses. Verifikasi visual Playwright TIDAK dilakukan (skip).

**Ref:** `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx`, `apps/web/src/app/(admin)/siswa/siswa-view.tsx`, `apps/web/src/lib/core-types.ts` (`Kelas`, `Jurusan`).

---

## T094 — Fix Bug Sidebar Piket: Highlight "Dashboard" Menempel di Semua Menu ✅

**Prioritas: tinggi** — bug visual jelas terlihat, salah paham navigasi.

**Konteks:** `PiketSidebar` (`apps/web/src/app/(piket)/piket-sidebar.tsx`) pakai `pathname === item.href || pathname.startsWith(item.href + "/")` — untuk item Dashboard (`href: "/piket"`), semua path piket lain ikut match `startsWith("/piket/")`, sehingga "Dashboard" selalu ter-highlight bersamaan dengan menu manapun yang aktif.

- [x] `apps/web/src/app/(piket)/piket-sidebar.tsx` — `isActive` diperbaiki: exact-match kalau `item.href === "/piket"`, else tetap `pathname === item.href || pathname.startsWith(item.href + "/")` untuk sub-route.
- [x] Dicek `apps/web/src/components/shell/sidebar.tsx` (sidebar admin) — SUDAH punya exact-match untuk `/` sejak awal, tidak ada bug serupa, tidak perlu diubah.
- [x] Dicek juga `guru-sidebar.tsx`/`admin-jurnal-sidebar.tsx` — polanya sama (`pathname.startsWith`) tapi TIDAK ada bug nyata di sana karena tidak ada href "akar" yang jadi prefix href lain (`/guru/jurnal`, `/admin-jurnal/toleransi`, dst semua setara, tidak ada `/guru` atau `/admin-jurnal` polos sebagai item terpisah) — dibiarkan, bukan bug yang sama.

**Implementasi:** Perbaikan 5 baris, pola persis seperti yang sudah dipakai `sidebar.tsx` admin untuk kasus `/`.

**Verifikasi:** `tsc --noEmit` bersih. Verifikasi manual visual (klik tiap menu piket satu-satu) TIDAK dilakukan langsung di sesi ini (skip Playwright) — perlu dicek user bahwa highlight sekarang benar per-menu.

**Ref:** `apps/web/src/app/(piket)/piket-sidebar.tsx` (`NavLink`, `isActive`), `apps/web/src/components/shell/sidebar.tsx` (dicek, tidak ada bug).

---

## T095 — Redesign Field Form Perizinan (Lebih Modern) ✅

**Prioritas: rendah** — polish visual, tidak ada perubahan fungsional.

**Konteks:** form-form perizinan (`PermitForm`, `IzinKeluarForm`, `LockForm`, `UnlockForm`, `TidakTapPulangForm`) pakai styling dasar tanpa ikon. User minta desain lebih modern, mengikuti pola yang SUDAH ADA di `ganti-password-form.tsx` (ikon di kiri input, `rounded-full`, border halus).

- [x] Baca ulang design-system vault — 1 aksen oranye saja, konsisten dengan pola `ganti-password-form.tsx` (ikon `text-ink-tertiary`, bukan warna aksen).
- [x] Pola field modern diterapkan ke SEMUA form yang disebutkan: `PermitForm`, `TidakTapPulangForm`, `LockForm`, `UnlockForm` (semua di `piket-board-view.tsx`), `IzinKeluarForm` (`izin-keluar-view.tsx`) — tiap `<Input>` teks/waktu dibungkus `relative` + ikon `absolute left-4 top-1/2 -translate-y-1/2 text-ink-tertiary`, input jadi `h-11 rounded-full border-border bg-surface pl-11 text-body`.
- [x] `<select>` native di `TidakTapPulangForm` (baris ±621) diganti komponen `Select` dari `@absensi/ui`, konsisten dengan `IzinKeluarForm`.
- [x] Form baru dari T090 (`riwayat-izin`, search input)/T092 (`input-izin`, seluruh form) DIBUAT sejak awal dengan pola modern ini, tidak perlu dikerjakan dua kali.
- [x] Pesan error/sukses juga diseragamkan jadi `rounded-lg bg-danger-bg/success-bg px-4 py-2` (pill message box), bukan teks polos tanpa background.

**Implementasi:** Pola diambil 1:1 dari `ganti-password-form.tsx` yang sudah ada — TIDAK menciptakan gaya baru, murni menyalin & menerapkan pola existing secara konsisten ke 7 form (5 form lama + 2 form baru T090/T092). Ikon dipilih sesuai semantik field: `Search` (pencarian nama siswa), `MessageSquare` (keterangan), `Clock` (input jam), `FileText` (alasan/catatan lock-unlock).

**Verifikasi:** `tsc --noEmit` bersih, `pnpm turbo run build` 6/6 sukses (build juga menjalankan ESLint, tidak ada error lint). **Verifikasi visual Playwright TIDAK dilakukan** — user secara eksplisit memilih skip demi hemat token pada sesi eksekusi ini, dan kredensial `guru_piket` untuk login tidak tersedia. **Task ini secara eksplisit meminta review visual user sebelum dianggap final** (preferensi visual, bisa ditafsirkan beda-beda) — WAJIB diverifikasi manual oleh user secepatnya, bukan dianggap selesai murni dari checklist ini.

**Ref:** `apps/web/src/app/ganti-password/ganti-password-form.tsx` (pola field modern existing, dijadikan referensi), `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (semua form di dalamnya), `apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx` (`IzinKeluarForm`).

---

## Catatan Umum

- Baca ulang bagian kode terkait sebelum tiap task — deskripsi di sini berdasarkan riset kode per 2026-07-30, mungkin sudah ada perubahan kecil dari task/batch sebelumnya.
- T094 (fix bug sidebar) dikerjakan lebih dulu, sebelum T090/T092 menambah menu baru ke `NAV_ITEMS` — sesuai urutan yang disarankan.
- T091 dikerjakan sebagai perubahan backend terisolasi, risiko regresi rendah.
- **Blocker sempat terjadi**: build backend (`nest build`) sempat gagal karena file TIDAK TERKAIT (`ekstra-publik/ekstra-kurikuler.controller.ts`, kerjaan T096 dari sesi paralel yang belum selesai saat itu) — diselesaikan setelah T096 rampung di sesi lain, backend berhasil di-restart dengan seluruh perubahan T090/T091/T092 aktif.
- **PENTING — belum ada verifikasi visual/Playwright/manual sama sekali** untuk seluruh 6 task di batch ini (user memilih skip demi hemat token). Semua "selesai" di atas berarti KODE SUDAH DITULIS + type-check & build hijau, BUKAN berarti sudah diverifikasi berjalan benar secara fungsional/visual. **Prioritas berikutnya: verifikasi manual oleh user**, terutama T095 (murni preferensi visual) dan T090/T091/T092 (alur data baru).

# T166 — Web+API: Rombak Halaman Kartu — 1 Baris Per Orang, Klik untuk Lihat Riwayat Kartu

## Depends on
**REKOMENDASI kerjakan SETELAH T165** (tombol Aktifkan Kembali) — supaya saat merombak tampilan riwayat kartu di task ini, tombol Aktifkan Kembali yang relevan sudah tersedia untuk langsung dipasang di riwayat itu. TIDAK WAJIB menunggu kalau ingin dikerjakan paralel, TAPI lebih rapi berurutan.

## Objective
Halaman admin Kartu (`(admin)/kartu`) yang SAAT INI menampilkan **1 baris per KARTU** (jadi 1 orang dengan banyak kartu histori muncul berkali-kali, tercampur dengan kartu orang lain) — DIROMBAK jadi **1 baris per ORANG** (nama muncul SEKALI saja) — klik nama itu untuk membuka SEMUA riwayat kartunya (aktif maupun nonaktif) di 1 tempat.

## Context — Alasan (Diskusi 2026-08-13)

Sebagai tindak lanjut insiden T165 (kartu lama tertukar menyebabkan tap ditolak) — user eksplisit minta perombakan tampilan: **"sekarang orang yang memiliki kartu aktif dan non aktif masih tampil semua, saya ingin yang tampil di halaman kartu hanya 1 nama saya. ketika di klik baru masuk menampilkan semua riwayat kartunya baik untuk guru atau siswa. untuk menu registrasi biarkan hanya siswa atau guru yang belum memiliki kartu."**

**Riset kode mengonfirmasi struktur SAAT INI**:
- `apps/web/src/app/(admin)/kartu/kartu-view.tsx` — 3 tab (Siswa/Guru/Karyawan), MASING-MASING murni **flat list 1-baris-per-Card** (`cardsSiswa.map((card) => ...)`, key `card.id` bukan `student.id`). Kalau 1 siswa punya 3 kartu histori (2 nonaktif + 1 aktif), dia muncul **3 KALI** di tabel yang sama.
- `GET /cards` (`cards.service.ts` `findAll()`) — query FLAT ke tabel `Card`, TIDAK ADA grouping per-owner sama sekali.
- **TIDAK ADA endpoint baru yang perlu dibuat untuk riwayat per-orang** — `GET /students/:id` dan `GET /teachers/:id` (SUDAH ADA, dipakai halaman detail siswa/guru) **SUDAH mengembalikan field `cards[]` LENGKAP** (semua histori, `orderBy: issuedAt desc`, TANPA filter status) — REUSE endpoint ini, JANGAN buat endpoint baru.
- **Preseden UI riwayat kartu SUDAH ADA** — halaman detail siswa (`apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx`, section "Riwayat Kartu" baris ~352-397) SUDAH menampilkan tabel riwayat kartu 1 siswa (UID, Status, Diterbitkan, Dicabut) — TAPI READ-ONLY (tidak ada tombol aksi). Task ini MEMBUTUHKAN versi YANG PUNYA AKSI (Aktifkan Kembali dari T165, Cabut, Ganti Kartu) — REUSE POLA TAMPILAN section itu sebagai referensi struktur, TAPI perlu diperluas dengan tombol aksi.
- **Halaman Registrasi Kartu (`tap-assign`) SUDAH BENAR sesuai keinginan user** — filter "Belum Punya Kartu" SUDAH memakai kondisi `cards: { none: { status: active } }` (orang TANPA kartu aktif, baik yang belum pernah punya kartu SAMA SEKALI, maupun yang PERNAH punya tapi sekarang semua nonaktif) — **TIDAK PERLU diubah**, TAPI perlu DIPASTIKAN TIDAK BENTROK secara UX dengan tombol Aktifkan Kembali baru di halaman Kartu (lihat Edge Cases).

## Spec Detail

### 1. Backend — TIDAK ADA endpoint baru WAJIB, TAPI PERTIMBANGKAN 1 endpoint ringkasan untuk performa

- **Opsi A (REKOMENDASI, sederhana)** — Halaman Kartu BARU mengambil daftar ORANG (bukan Card) dari endpoint yang SUDAH ADA: `GET /students` (dengan filter kelas/jurusan yang sudah ada) untuk tab Siswa, `GET /teachers` untuk tab Guru/Karyawan — MASING-MASING baris tampilkan RINGKASAN status kartu orang itu (misal "1 kartu aktif" / "Tidak ada kartu aktif" / "2 kartu nonaktif") — TAPI `GET /students`/`GET /teachers` (list) SAAT INI KEMUNGKINAN BESAR TIDAK include field `cards[]` di response LIST (beda dari `findOne` yang include lengkap) — VERIFIKASI ini saat implementasi, KALAU list endpoint TIDAK include ringkasan kartu, PERLU DIPERLUAS (tambah `_count: { cards: { where: { status: active } } }` atau serupa) supaya baris tabel bisa menampilkan ringkasan TANPA fetch detail penuh untuk semua orang sekaligus (N+1 query problem kalau naif).
- **Opsi B (alternatif)** — buat 1 endpoint BARU khusus halaman ini, misal `GET /cards/summary-by-owner` yang GROUP BY owner dan return ringkasan (nama, jumlah kartu aktif, jumlah kartu nonaktif) — LEBIH EKSPLISIT tapi endpoint baru yang perlu dirawat.
- **PUTUSKAN saat implementasi** mana yang lebih murah/rapi — REKOMENDASI Opsi A KALAU perluasan `GET /students`/`GET /teachers` list terasa ringan (cukup tambah `_count` di Prisma select), Opsi B KALAU terasa lebih bersih secara pemisahan tanggung jawab modul (Card module vs Student/Teacher module, ingat ADR-003 batas modul — HATI-HATI: `Card` bukan milik modul Core, jadi query LANGSUNG ke tabel Student/Teacher dari `CardsService` PERLU melalui service layer Core yang sesuai, BUKAN query Prisma langsung lintas modul — VERIFIKASI pendekatan mana yang tidak melanggar ADR-003).

### 2. Frontend — halaman Kartu jadi daftar ORANG, bukan daftar KARTU

- `apps/web/src/app/(admin)/kartu/kartu-view.tsx` — ROMBAK TOTAL struktur tabel di 3 tab (Siswa/Guru/Karyawan): SETIAP BARIS = 1 ORANG (bukan 1 kartu). Kolom BARU (per orang): Nama, NISN/NIY, Kelas/Jurusan (siswa)/Status Kepegawaian (guru-karyawan), **Ringkasan Status Kartu** (misal badge "1 Aktif" / "Tidak Ada Kartu Aktif" / "2 Nonaktif"), Aksi ("Lihat Riwayat").
- **Klik nama ATAU tombol "Lihat Riwayat"** → buka DIALOG/SHEET (KONSISTEN pola dialog yang sudah ada di halaman ini, misal `ReplaceCardForm` yang sudah pakai Dialog) menampilkan **SEMUA kartu histori orang itu** — fetch LAZY saat dialog dibuka (panggil `GET /students/:id` atau `GET /teachers/:id`, ambil field `cards[]`) — JANGAN fetch semua riwayat SEMUA orang di awal (mahal, tidak perlu).
- **Dalam dialog riwayat** — tabel kecil: UID, Status (badge), Diterbitkan, Dicabut, **Aksi PER KARTU** (Aktifkan Kembali untuk yang nonaktif dari T165, Cabut/Ganti Kartu untuk yang aktif — REUSE `CardActions` yang SUDAH ADA, TINGGAL DIPASANG DI KONTEKS BARU ini alih-alih di baris tabel utama).
- Search+filter (kelas/jurusan untuk siswa, status kepegawaian untuk guru-karyawan) — TETAP ADA seperti sekarang, TAPI beroperasi terhadap DAFTAR ORANG (bukan daftar kartu) — SESUAIKAN query params yang dikirim ke `GET /students`/`GET /teachers` (endpoint yang SUDAH punya filter serupa, KEMUNGKINAN BESAR bisa reuse langsung tanpa perubahan).
- **Aturan tabel permanen proyek** — WAJIB tetap dipatuhi di tabel BARU ini: search box, kolom No, SEMUA kolom sortable (KONSISTEN memory `feedback_tabel_wajib_search_sort_kolom_no`).

### 3. Registrasi Kartu (`tap-assign`) — TIDAK DIUBAH, TAPI PASTIKAN KONSISTENSI UX

- **JANGAN ubah logic filter** `unassignedPersons()` — SUDAH BENAR sesuai keinginan user (orang tanpa kartu aktif, termasuk yang pernah punya tapi sekarang nonaktif semua).
- **PERTIMBANGKAN** (opsional, UX enhancement): di halaman `tap-assign`, untuk orang yang MUNCUL di situ TAPI SEBENARNYA punya histori kartu nonaktif (bukan benar-benar belum pernah punya kartu) — TAMPILKAN PETUNJUK KECIL ("Pernah punya kartu, lihat riwayat di halaman Kartu") supaya admin SADAR ada opsi "Aktifkan Kembali" sebagai alternatif dari "daftar kartu baru" — TIDAK WAJIB untuk v1, PUTUSKAN saat implementasi apakah worth ditambahkan.

## Edge Cases
- Orang yang BELUM PERNAH punya kartu SAMA SEKALI (0 baris `Card` di database) — di halaman Kartu BARU, baris orang itu TETAP MUNCUL dengan ringkasan "Tidak Ada Kartu" (BUKAN "Tidak Ada Kartu Aktif" — beda pesan untuk membedakan "belum pernah" vs "pernah tapi sekarang nonaktif", PUTUSKAN redaksi yang jelas saat implementasi) — klik "Lihat Riwayat" untuk orang ini akan menampilkan riwayat KOSONG, TIDAK BOLEH error/crash.
- Guru/Karyawan dengan BANYAK kartu aktif sekaligus (T119, multi-kartu) — ringkasan status HARUS mencerminkan jumlah yang benar (misal "3 Aktif"), BUKAN disederhanakan seolah cuma boleh 1 seperti siswa.
- Performa — kalau jumlah siswa/guru CUKUP BANYAK (ratusan), pastikan pendekatan poin 1 (ringkasan count) TIDAK menyebabkan N+1 query (1 query terpisah per baris) — WAJIB pakai agregasi Prisma yang efisien (`_count` dengan `where` di level `include`, atau `groupBy`), BUKAN loop query manual.

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kartu/kartu-view.tsx` (rombak total struktur tabel+dialog riwayat), `apps/api/src/cards/cards.controller.ts`/`cards.service.ts` (KALAU Opsi B endpoint baru dipilih) ATAU `apps/api/src/core/students/students.service.ts`+`apps/api/src/core/teachers/teachers.service.ts` (KALAU Opsi A, perluas list endpoint dengan ringkasan count).
- **Jangan sentuh:** `apps/web/src/app/(admin)/kartu/tap-assign/tap-assign-view.tsx` dan `unassignedPersons()` (logic SUDAH BENAR, tidak diubah), `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` section Riwayat Kartu (halaman TERPISAH, biarkan tetap ada apa adanya sebagai read-only view, TIDAK perlu dihapus meski sekarang ada juga di halaman Kartu — REDUNDANSI KECIL yang bisa diterima, atau EVALUASI apakah salah satu sebaiknya cukup link ke yang lain, TIDAK WAJIB diputuskan di task ini).

## Acceptance Criteria
- [x] Halaman Kartu (semua 3 tab) menampilkan **1 baris per orang**, TIDAK LAGI 1 baris per kartu — orang dengan banyak kartu histori HANYA muncul SEKALI.
- [x] Setiap baris punya ringkasan status kartu (jumlah aktif/nonaktif) yang AKURAT — verified live dengan skenario 2 aktif + 1 nonaktif.
- [x] Klik baris/tombol "Lihat Riwayat" → dialog menampilkan SEMUA kartu histori orang itu (aktif+nonaktif), dengan aksi (Aktifkan Kembali/Cabut/Ganti Kartu) LANGSUNG bisa dilakukan dari situ.
- [x] Search+filter (kelas/jurusan/status kepegawaian) TETAP berfungsi terhadap daftar ORANG.
- [x] Halaman Registrasi Kartu (`tap-assign`) TIDAK BERUBAH perilakunya — verified live, `unassigned-persons` tetap 9 hasil, tidak disentuh.
- [x] Aturan tabel permanen (search+No+sortable semua kolom) DIPATUHI di tabel baru.
- [x] TIDAK ADA N+1 query untuk ringkasan status kartu — VERIFIKASI KONKRET via Prisma query-event log: query `cards` menggunakan `WHERE student_id IN (...)` BATCHED (1 query untuk SEMUA siswa di halaman itu), BUKAN 1 query per siswa. Total 6 query TETAP (findMany+count+kelas+kampus+jurusan+cards) terlepas dari jumlah siswa.
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] **JANGAN** ubah logic `unassignedPersons()`/halaman `tap-assign` — TIDAK disentuh sama sekali, dikonfirmasi via curl (masih 9 hasil, sama seperti sebelum task ini).
- [x] **REUSE** `GET /students/:id`/`GET /teachers/:id` untuk riwayat per-orang — TIDAK ADA endpoint riwayat baru, dialog `RiwayatKartuDialog` fetch lazy ke endpoint yang SUDAH ADA ini.
- [x] **PATUHI ADR-003** — Opsi A dipilih (bukan Opsi B), jadi tidak relevan (tidak ada cross-module service call baru) — StudentsService/TeachersService (keduanya Core) diperluas dengan data milik mereka sendiri (`cards` relation ke `Card`), bukan `CardsService` yang query lintas modul.
- [x] Verifikasi PERFORMA (tidak N+1) — DILAKUKAN via skrip Prisma query-event log sementara, dikonfirmasi query `cards` batched (`IN (...)`), BUKAN diasumsikan aman.
- [x] Dikerjakan SETELAH T165 — dialog riwayat LANGSUNG punya tombol "Aktifkan Kembali" terpasang (reuse `CardActions` T165 apa adanya).

## Status Eksekusi (2026-08-13)

**Selesai.** Keputusan arsitektur (Opsi A vs B) dikonfirmasi eksplisit ke user SEBELUM implementasi — user pilih **Opsi A** (perluas `GET /students`/`GET /teachers`, bukan endpoint baru di CardsModule).

**Backend — Core (`StudentsService`, `TeachersService`)**:
- `StudentsService.findAll()` — tambah parameter opsional `includeCardSummary` (DTO baru `ListStudentsDto.includeCardSummary`) — kalau `true`, `cards: {select: {status}}` disertakan DALAM query `findMany` yang SAMA (nested include, bukan query terpisah). Tanpa flag ini (semua caller lama: plot-siswa, kenaikan-massal, dst), behavior 100% identik seperti sebelumnya.
- `TeachersService.findAll()` — SEBELUMNYA sangat minim (`this.prisma.teacher.findMany({orderBy: {nama: "asc"}})`, TANPA filter/pagination/sort sama sekali) — DIPERLUAS penuh mengikuti pola `StudentsService` PERSIS: `ListTeachersDto` baru (statusKepegawaian, status, nama search, sortBy/sortDir dari 4 field, page/pageSize opsional, includeCardSummary). Pagination TETAP opsional — 6 caller existing (`GET /teachers` tanpa query param) dikonfirmasi TIDAK regresi (masih return `Teacher[]` polos, verified live).
- `TeachersController` — `@Query() filter: ListTeachersDto` ditambahkan ke `findAll()`.

**Frontend (`apps/web/src/app/(admin)/kartu/`)**:
- `page.tsx` — dirombak total, fetch `GET /students`+`GET /teachers` (dengan `includeCardSummary=true`) alih-alih `GET /cards`. Filter "status kartu" (aktif/nonaktif per BARIS) LAMA DIHAPUS — tidak relevan lagi untuk daftar per-orang, sesuai spec yang hanya minta filter kelas/jurusan (siswa) + statusKepegawaian (guru/karyawan, sudah otomatis via query terpisah guru/karyawan).
- `kartu-view.tsx` — ROMBAK TOTAL: 3 tab sekarang render 1 baris per Student/Teacher (bukan Card), kolom baru: No, Nama (klik untuk buka riwayat), NISN/NIY, Kelas/Jurusan (siswa saja), **Status Kartu** (badge `CardSummaryBadge`, warna hijau kalau ada kartu aktif/merah kalau ada tapi semua nonaktif/abu kalau tidak pernah punya — 3 STATE beda sesuai edge case spec), Aksi ("Lihat Riwayat"). `RiwayatKartuDialog` baru — fetch LAZY (`GET /students/:id` atau `GET /teachers/:id`) saat dialog dibuka, render tabel kartu penuh dengan `CardActions` (T165, REUSE apa adanya) per baris — Aktifkan Kembali/Ganti/Cabut LANGSUNG dari dialog ini, refresh riwayat + halaman utama setelah aksi berhasil.
- `core-types.ts` — `TeacherPage` interface baru (pola sama `StudentPage`).

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh):
1. `GET /students?includeCardSummary=true` — `cards: []` untuk siswa tanpa kartu, `cards: [{status: "active"}]` untuk siswa dengan 1 kartu aktif (data T165) — akurat.
2. `GET /teachers?includeCardSummary=true&statusKepegawaian=guru` — filter benar (4 guru, bukan campur karyawan), ringkasan akurat.
3. Skenario multi-kartu (guru dengan 2 aktif + 1 nonaktif, disiapkan manual) — list summary tampilkan PERSIS `[active, active, inactive]` (3 entri, akan jadi badge "2 Aktif, 1 Nonaktif" di UI), `GET /teachers/:id` (dipakai dialog riwayat) tampilkan LENGKAP ketiga kartu dengan UID masing-masing — sesuai edge case "guru multi-kartu, ringkasan HARUS akurat".
4. `GET /teachers` TANPA query param — tetap array polos 4 guru, TIDAK ADA field `cards` (regresi nol untuk 6 caller lama).
5. `GET /students` TANPA query param — tetap array polos 48 siswa, TIDAK ADA field `cards` (regresi nol).
6. `GET /cards/unassigned-persons` (backing `tap-assign`) — TIDAK disentuh, hasil tetap 9 orang, sama seperti sebelum task ini.
7. `GET /kartu` (halaman) tanpa login — 307 redirect ke login (BUKAN 500), konfirmasi tidak ada crash server-side.
8. **N+1 verifikasi KONKRET** — skrip sementara dengan Prisma query-event logging: `findAll({includeCardSummary: true, page: 1})` untuk 48 siswa menghasilkan TEPAT 6 query SQL (findMany utama, count, kelas, kampus, jurusan, cards) — query `cards` dikonfirmasi `WHERE student_id IN (48 nilai)` (SATU query batched, BUKAN 48 query terpisah). Jumlah query TIDAK bertambah seiring jumlah siswa — BUKAN N+1.
9. Semua data uji (test admin, activity_log terkait, 3 kartu test guru) dibersihkan setelah verifikasi.
10. `tsc --noEmit` bersih `apps/api` + `apps/web`. Jest `apps/api` 273/273 pass, tidak ada regresi.

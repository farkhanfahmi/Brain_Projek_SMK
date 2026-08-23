# T133 — API+Web: Filter Jurusan+Kelas di Halaman Registrasi Kartu (Belum Punya Kartu)

## Depends on
Tidak ada dependency teknis. Perluasan endpoint+UI existing, tidak ada model/skema baru.

## Objective
Section "Belum Punya Kartu" di halaman Registrasi Kartu (`/kartu/tap-assign`) punya filter Jurusan dan Kelas — saat ini HANYA ada search bebas nama/NISN/NIP, tidak ada cara mempersempit daftar siswa/guru yang belum punya kartu berdasarkan jurusan atau kelas tertentu.

## Context
- **App:** `apps/api` (tambah query param + filter ke endpoint) + `apps/web` (tambah UI filter)
- **Riset 2026-08-07 (Explore agent, baca kode langsung)** — dikonfirmasi ada 2 tempat berbeda yang menampilkan status "Belum Punya Kartu"/"Belum Memiliki Kartu", dan task ini SPESIFIK untuk salah satunya (dikonfirmasi user):
  - **Halaman Registrasi Kartu** (`apps/web/src/app/(admin)/kartu/tap-assign/tap-assign-view.tsx:106`, judul section literal "Belum Punya Kartu") — **INI YANG DIMAKSUD task ini**. Dipakai piket/admin untuk mendaftarkan kartu RFID baru ke siswa/guru yang belum punya. SAAT INI cuma ada 1 filter: search bebas nama/NISN/NIP (`search` state, baris ±17, ±32-36) — TIDAK ADA filter Jurusan/Kelas sama sekali.
  - **BUKAN** kolom status "Belum Memiliki Kartu" di halaman Rekap (`rekap-view.tsx`, hasil T132) — itu SUDAH punya filter Jurusan+Kelas (berlaku ke seluruh tabel rekap termasuk baris status ini), tidak perlu kerja tambahan, di luar scope task ini.
  - **Backend endpoint SAAT INI TIDAK MENDUKUNG filter apa pun** — `GET /cards/unassigned-persons` (`apps/api/src/cards/cards.controller.ts:24-26`) TIDAK PUNYA `@Query()` parameter sama sekali. `CardsService.unassignedPersons()` (`apps/api/src/cards/cards.service.ts:112-130`) — method TANPA parameter, `findMany` flat tanpa filter kelas/jurusan apa pun. **Filter ini harus dibangun dari nol di backend**, bukan sekadar expose parameter yang sudah ada tapi belum dipakai FE (beda dari kasus Rekap yang backend-nya sudah siap).
  - **Pola filter Jurusan+Kelas SUDAH ADA di halaman lain** untuk dicontek — `kartu-view.tsx` (baris ±111-254, cascading Jurusan→Kelas) dan `rekap-view.tsx` (baris ±113-114 state, ±405-419 UI) — REUSE pola yang sama (cascading: pilih Jurusan mempersempit opsi Kelas), JANGAN bikin pola filter baru yang berbeda.

## Spec Detail

### Backend
- `apps/api/src/cards/cards.controller.ts` — endpoint `GET /cards/unassigned-persons` (baris ±24-26): tambah `@Query()` DTO baru, terima `kelasId?: number`, `jurusanId?: number` (opsional, konsisten pola filter opsional di endpoint lain proyek ini).
- `apps/api/src/cards/cards.service.ts` — `unassignedPersons()` (baris ±112-130): terima parameter filter baru, tambahkan ke `where` clause query siswa. **PENTING**: method ini kemungkinan mengembalikan GABUNGAN siswa DAN guru tanpa kartu (cek nama fungsi "Persons" — generic, bukan cuma "Students") — filter Jurusan/Kelas HANYA relevan untuk SISWA (guru tidak punya jurusan/kelas), pastikan filter ini TIDAK diterapkan ke query guru, atau guru otomatis dikecualikan kalau filter Jurusan/Kelas aktif (perlu diputuskan: apakah saat filter Jurusan/Kelas aktif, guru tetap tampil semua, atau disembunyikan karena filter itu tidak relevan untuk mereka — REKOMENDASI: guru TETAP tampil apa adanya, filter Jurusan/Kelas hanya mempengaruhi baris siswa, karena guru memang tidak punya atribut itu untuk difilter).
- Cek dulu struktur DTO/enum yang dikembalikan `unassignedPersons()` — pastikan response tetap membedakan siswa vs guru dengan jelas (kemungkinan sudah ada, cek saja) supaya FE bisa tetap render dengan benar.

### Frontend
- `apps/web/src/app/(admin)/kartu/tap-assign/tap-assign-view.tsx` — tambah 2 dropdown filter: Jurusan lalu Kelas (urutan Search → Jurusan → Kelas, cascading, KONSISTEN dengan aturan permanen proyek soal urutan filter bertingkat — lihat pola persis di `kartu-view.tsx`/`rekap-view.tsx`, REUSE styling/struktur yang sama, bukan bikin baru).
- Filter Jurusan/Kelas HANYA relevan untuk daftar SISWA di section ini — kalau section ini juga menampilkan guru dalam 1 daftar campur, pastikan UI tidak membingungkan (misal filter Kelas aktif tapi tetap ada baris guru tanpa kelas — beri indikasi jelas atau pisahkan tampilan siswa/guru kalau perlu, putuskan saat implementasi berdasarkan struktur data aktual).
- State filter terhubung ke query param yang dikirim ke `GET /cards/unassigned-persons`, konsisten pola fetch yang sudah ada di halaman ini.

## Edge Cases
- Filter Jurusan dipilih tapi Kelas belum dipilih → tampilkan semua siswa di jurusan itu (semua kelas), pola sama seperti halaman lain yang sudah punya cascading filter ini.
- Kombinasi filter menghasilkan 0 hasil → tampilkan empty state yang jelas (bukan tabel kosong tanpa penjelasan), konsisten pola empty state yang sudah dipakai di halaman lain.

## Files
- **Modifikasi:** `apps/api/src/cards/cards.controller.ts`, `apps/api/src/cards/cards.service.ts` (kemungkinan DTO baru di `apps/api/src/cards/dto/`), `apps/web/src/app/(admin)/kartu/tap-assign/tap-assign-view.tsx`.
- **Jangan sentuh:** halaman Rekap (`rekap-view.tsx`, sudah punya filter ini, di luar scope), tab Murid/Guru/Karyawan di `kartu-view.tsx` (halaman berbeda, tidak terkait).

## Acceptance Criteria
- [x] Section "Belum Punya Kartu" di Registrasi Kartu punya filter Jurusan dan Kelas, cascading (Jurusan mempersempit opsi Kelas). Diverifikasi live via Playwright — pilih Jurusan DKV, opsi Kelas otomatis mempersempit ke kelas DKV saja.
- [x] Search nama/NISN/NIP yang sudah ada TETAP berfungsi bersamaan dengan filter baru (kombinasi search+filter, bukan saling menggantikan). Diverifikasi live — ketik "Aarta" dengan filter aktif, hasil tetap benar (1 dari 9).
- [x] Filter HANYA mempengaruhi baris siswa — guru tidak hilang/salah terfilter. Diverifikasi live+curl: filter `kelasId`/`jurusanId` apa pun, jumlah guru selalu tetap 4 (konsisten), hanya jumlah siswa yang berubah.
- [x] Build + type-check `apps/api` dan `apps/web` hijau. `tsc --noEmit` bersih kedua app, jest 187/187 tetap lulus (tidak ada test cards existing, tidak ada regresi modul lain).

## Status Eksekusi — SELESAI (2026-08-07)
**Backend**: DTO baru `UnassignedPersonsQueryDto` (`kelasId?`, `jurusanId?`, keduanya opsional int) — BUKAN reuse `ListCardsDto` karena scope beda (endpoint ini tidak butuh status/pagination/sort). `CardsController.unassignedPersons()` terima `@Query() filter`. `CardsService.unassignedPersons(filter?)` — filter HANYA ditambahkan ke `where` query `student.findMany` (`kelasId: filter?.kelasId`, `kelas: { jurusanId: filter?.jurusanId }`), query `teacher.findMany` TIDAK disentuh sama sekali (guru selalu tampil apa adanya, sesuai rekomendasi spec).

**Frontend** (`tap-assign-view.tsx`): props baru `kelasList`/`jurusanList` (di-load di `page.tsx` via `Promise.all` dengan fetch existing, pola sama `kartu/page.tsx`). State `jurusanId`/`kelasId` lokal (`useState`, BUKAN URL searchParams — beda dari `kartu-view.tsx` karena halaman ini sudah client-only murni tanpa reload). `useEffect` fetch ulang `/cards/unassigned-persons` dengan query params tiap filter berubah (server-side filter), search TETAP client-side di atas hasil fetch (tidak round-trip server tiap huruf diketik). `handleJurusanChange()` — REUSE logic reset Kelas dari `kartu-view.tsx` (reset HANYA kalau Kelas yang dipilih tidak lagi cocok Jurusan baru, BUKAN selalu reset — kalau Jurusan direset ke "Semua", Kelas yang sudah valid tetap dipertahankan, cuma opsi dropdown melebar lagi). Urutan filter: Search → Jurusan → Kelas (konsisten aturan permanen proyek).

**Verifikasi live**: curl langsung ke API (filter kosong = 9 orang, `kelasId=43` = 3 siswa+4 guru, `jurusanId=4` = 2 siswa+4 guru — guru konsisten 4 di semua kasus) + UI browser penuh via Playwright (pilih Jurusan → Kelas ter-cascade → reset Jurusan → Kelas tetap valid sampai direset manual → search+filter kombinasi bekerja).

## Validasi Claudian
- [x] Struktur data `unassignedPersons()` dicek langsung — array CAMPUR siswa+guru dengan discriminator `type: "student" | "teacher"` (bukan 2 array terpisah), sesuai spec asli. Filter diterapkan HANYA ke query siswa.
- [x] Pola cascading Jurusan→Kelas di-reuse PERSIS dari `kartu-view.tsx` (`kelasOptions` filter + `handleJurusanChange` reset-kondisional) — TIDAK reinvent state management filter yang berbeda gaya, hanya adaptasi ke `useState` lokal (bukan URL searchParams) karena halaman ini sudah client-only.

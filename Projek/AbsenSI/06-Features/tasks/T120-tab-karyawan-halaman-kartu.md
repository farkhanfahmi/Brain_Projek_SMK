# T120 — API+Web: Tab "Karyawan" Baru di Halaman Kartu (Murid | Guru | Karyawan)

## Depends on
Tidak ada dependency teknis keras. Kalau T121 (ganti label "Siswa"→"Murid") dikerjakan berdekatan, urutan bebas — tapi kalau T120 dikerjakan LEBIH DULU, langsung pakai label "Murid" untuk tab pertama (bukan "Siswa") supaya tidak perlu diubah 2 kali. Kalau T121 dikerjakan duluan, tab baru ini otomatis konsisten mengikuti pola label yang sudah diperbarui.

## Objective
Halaman Kartu (`(admin)/kartu/`) punya 3 tab terpisah: **Murid | Guru | Karyawan** — saat ini cuma 2 tab (Siswa/Guru), semua kartu milik `Teacher` (baik `statusKepegawaian: guru` maupun `karyawan`) tercampur jadi satu di tab "Guru". Tab baru memisahkan karyawan/staf non-guru ke tampilannya sendiri.

## Context
- **App:** `apps/api` (tambah filter `statusKepegawaian` ke endpoint kartu) + `apps/web` (tab ke-3 + fetch data ke-3)
- **Riset 2026-08-06 (Explore agent, baca kode langsung)**:
  - `apps/web/src/app/(admin)/kartu/kartu-view.tsx` — pakai `Tabs`/`TabsList`/`TabsTrigger`/`TabsContent` dari `@absensi/ui` (Radix-based). Saat ini persis 2 tab: `<Tabs defaultValue="siswa">` (baris ±198), `<TabsTrigger value="siswa">Kartu Siswa</TabsTrigger>` (±201), `<TabsTrigger value="guru">Kartu Guru</TabsTrigger>` (±202). Tiap tab punya STATE SENDIRI-SENDIRI yang paralel (search/sort/pagination/filter terpisah per tab, lihat interface `ActiveFilter` ±42-57) — **pola ini yang harus ditiru untuk tab ke-3**, bukan bikin pola baru.
  - `apps/web/src/app/(admin)/kartu/page.tsx` — TIDAK pakai `tab=` query param untuk pilih tab aktif; sebaliknya **fetch KEDUA tab sekaligus** (`cardsSiswa` dan `cardsGuru`, paralel via `Promise.all`) lalu keduanya dilempar ke `KartuView`, dan Radix `Tabs` cuma toggle visibility client-side. **Tab ke-3 ikut pola sama**: fetch `cardsKaryawan` paralel juga di `page.tsx`.
  - `apps/api/src/cards/cards.service.ts` `findAll()` (baris ±30) — filter SAAT INI cuma `ownerType: "student" | "teacher"` (binary). **TIDAK ADA filter `statusKepegawaian`** — kartu guru dan karyawan tercampur dalam `ownerType: "teacher"` tanpa dibedakan. `StatusKepegawaian` enum (`guru`/`karyawan`) sudah ada di model `Teacher` (`schema.prisma:224-227`), dipakai di modul lain (attendance, realtime), tapi belum pernah dipakai di `cards.service.ts`/`list-cards.dto.ts`.

## Spec Detail

### Backend
- `apps/api/src/cards/dto/list-cards.dto.ts` — tambah field opsional `statusKepegawaian?: "guru" | "karyawan"` (ikuti pola field opsional yang sudah ada seperti `kelasId`/`jurusanId`).
- `apps/api/src/cards/cards.service.ts` `findAll()` — kalau `filter.statusKepegawaian` terisi, tambahkan kondisi `where` join ke `teacher.statusKepegawaian` (SELAIN filter `ownerType: "teacher"` yang sudah ada, bukan menggantikan — filter tambahan yang mempersempit lagi).
- Tidak perlu migration — `StatusKepegawaian` enum sudah ada di schema.

### Frontend
- `apps/web/src/app/(admin)/kartu/page.tsx` — tambah fetch ke-3 (`cardsKaryawan`) paralel dengan yang sudah ada, query `ownerType=teacher&statusKepegawaian=karyawan`. **Tab Guru yang sudah ada perlu disesuaikan juga**: query untuk `cardsGuru` sekarang harus TAMBAH `statusKepegawaian=guru` (supaya karyawan tidak lagi muncul di tab Guru — dulu tercampur, sekarang harus dipisah tegas).
- `apps/web/src/app/(admin)/kartu/kartu-view.tsx`:
  - Tambah `TabsTrigger value="karyawan">Kartu Karyawan</TabsTrigger>` (posisi ketiga, setelah Guru).
  - Tambah `TabsContent value="karyawan">` — struktur SAMA PERSIS seperti `TabsContent value="guru"` (tabel dengan search+sort+kolom No dari T106, `SortableHeader` yang sudah diekstrak reusable — pakai ulang, jangan reimplementasi).
  - Tambah state paralel untuk tab karyawan (search/sort/pagination) mengikuti pola `ActiveFilter` yang sudah ada untuk siswa/guru — TIGA blok state paralel sekarang, bukan dua.
  - Label tab pertama: pakai **"Kartu Murid"** (bukan "Kartu Siswa") kalau T121 sudah/akan dikerjakan bersamaan — cek urutan pengerjaan T120 vs T121 dulu (lihat Depends on).

## Edge Cases
- Guru dengan `statusKepegawaian: null`/tidak terisi (kalau ada data lama yang belum lengkap) → tentukan default masuk ke tab mana (kemungkinan aman default ke "Guru" supaya tidak hilang dari kedua tab, TAPI klarifikasi ke user kalau ternyata ada banyak data begini saat implementasi — bisa jadi butuh perbaikan data terpisah).

## Files
- **Modifikasi:** `apps/api/src/cards/dto/list-cards.dto.ts`, `apps/api/src/cards/cards.service.ts`, `apps/web/src/app/(admin)/kartu/page.tsx`, `apps/web/src/app/(admin)/kartu/kartu-view.tsx`.
- **Jangan sentuh:** `SortableHeader` (`apps/web/src/components/sortable-header.tsx`, reuse apa adanya), tab Siswa/Murid existing (struktur/logicnya tidak berubah, cuma mungkin label kalau T121 turut dikerjakan).

## Acceptance Criteria
- [x] Halaman Kartu punya 3 tab: Murid (atau Siswa kalau T121 belum jalan) | Guru | Karyawan.
- [x] Tab Guru HANYA menampilkan kartu milik `Teacher` dengan `statusKepegawaian: guru` (karyawan tidak lagi tercampur).
- [x] Tab Karyawan menampilkan kartu milik `Teacher` dengan `statusKepegawaian: karyawan`, dengan search+sort+kolom No yang sama seperti 2 tab lain.
- [x] Registrasi kartu baru untuk karyawan tetap berfungsi normal (reuse alur registrasi existing, T118/T119 tidak terpengaruh scope-nya oleh perubahan tab ini — tidak ada perubahan di `create()`/`ensureOwnerExistsAndHasNoActiveCard()`, hanya `findAll()`).
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] Cek urutan T120 vs T121 sebelum eksekusi — T121 BELUM dikerjakan (masih "Belum dikerjakan" di STATUS.md saat T120 dieksekusi 2026-08-08), jadi tab pertama tetap label lama "Kartu Siswa"/"Siswa" — T121 nanti perlu update label tab ini juga saat dikerjakan.
- [x] Pastikan filter `statusKepegawaian` baru TIDAK mengubah perilaku endpoint kartu untuk pemanggil lain — grep konfirmasi satu-satunya caller `GET /cards` adalah `apps/web/src/app/(admin)/kartu/page.tsx`, filter opsional (default undefined = tidak filter), aman.

## Status Eksekusi (2026-08-08)
- Backend: `list-cards.dto.ts` tambah `statusKepegawaian?: StatusKepegawaian` (opsional, `@IsEnum`). `cards.service.ts` `findAll()` tambah `where.teacher: filter.statusKepegawaian ? { statusKepegawaian: filter.statusKepegawaian } : undefined` — filter TAMBAHAN di atas `ownerType:"teacher"` yang sudah ada, bukan pengganti.
- Frontend: `page.tsx` sekarang fetch 3 query paralel — query guru ditambah `statusKepegawaian=guru` (default query, bukan dari searchParams — user tidak bisa ubah ini, murni pemisahan tab), query karyawan baru `statusKepegawaian=karyawan`. `kartu-view.tsx` tab ke-3 "Kartu Karyawan" struktur identik tab Guru (search+SortableHeader UID/Nama/Status/Diterbitkan+kolom No+Pagination), state URL paralel (`karyawanNama`, `karyawanSortBy`, `karyawanSortDir`, `karyawanPage`, `karyawanPageSize`, `statusKaryawan`) — total 3 blok state paralel sekarang sesuai spec.
- Edge case null `statusKepegawaian` di spec TIDAK RELEVAN — cek schema (`schema.prisma:212`) kolom `statusKepegawaian` NOT NULL dengan `@default(guru)`, jadi tidak ada baris Teacher yang bisa punya nilai null/kosong.
- Verifikasi: `tsc --noEmit` bersih di `apps/api` & `apps/web`. `jest` 203/203 pass (tidak ada suite khusus cards). Live end-to-end TIDAK bisa diuji penuh — dev DB saat ini nol baris `cards` dengan `teacher_id` terisi (belum ada kartu guru/karyawan terdaftar sama sekali) dan nol Teacher dengan `statusKepegawaian: karyawan`, jadi split 3 tab tidak actionable diuji lewat UI/curl saat ini. Sebagai gantinya, query construction di-cross-check manual lewat raw SQL setara terhadap dev DB (`WHERE t.status_kepegawaian = 'guru'` dkk) dan pola `where.teacher: {...}` identik pola `where.student: {...}` yang sudah ada & battle-tested di file yang sama — risiko rendah.
- Belum di-commit — menunggu instruksi user soal scope/timing commit (pola sesi ini: user sensitif soal jam sibuk ekstra/absensi aktif).

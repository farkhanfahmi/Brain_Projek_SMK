# T145 — Schema+API+Web: Jam Masuk Sekolah 3-Lapis (Default Global → Per Tingkat → Override Per Kelas)

## Depends on
Tidak ada dependency teknis ke T144. Independen, TAPI **T146 (Alfa Live di Papan Piket) depends on task ini** — T146 tidak boleh dikerjakan sebelum T145 selesai.

## Objective
Ganti mekanisme "jam masuk sekolah" (`Schedule{type: jam_sekolah}`) dari 1 baris global sembarangan (bug) menjadi 3 lapis resolusi eksplisit: **Default Global** (fallback dasar) → **Default Per Tingkat** (X/XI/XII, override default global) → **Override Per Kelas individual** (opsional, override default tingkatnya) — sesuai kebutuhan nyata sekolah yang punya jam masuk berbeda antar tingkat/kelas.

## Context — Bug Dikonfirmasi + Kebutuhan Bisnis Nyata (Riset+Diskusi 2026-08-08)

**Bug ditemukan** saat investigasi T144 (bug alfa=0): `AttendanceService.determineStatus()` (`apps/api/src/attendance/attendance.service.ts:634-636`) menentukan status "Terlambat" siswa dengan:
```ts
const schedule = await this.prisma.schedule.findFirst({
  where: { type: "jam_sekolah", hari: dayOfWeek },
});
```
Query ini **TIDAK memfilter `kelasId`** — kalau ada LEBIH DARI 1 baris `jam_sekolah` untuk hari yang sama (yang MEMANG dibutuhkan karena tingkat/kelas berbeda punya jam masuk berbeda, dikonfirmasi user), `findFirst()` mengambil SALAH SATU SECARA SEMBARANGAN (urutan default DB, bukan keputusan bisnis) dan dipakai untuk SEMUA siswa apa pun kelasnya.

**Bug KEDUA di lokasi lain, pola sama**: `PermitsService.tandaiIzinTidakKembali()` (`apps/api/src/permits/permits.service.ts:417-419`) — estimasi jam pulang untuk kasus "izin tidak kembali" JUGA query `jam_sekolah` tanpa filter kelas, PADAHAL `record.student.kelas` sudah ter-load di scope yang sama (baris 403) — fix-nya trivial begitu struktur data baru siap.

**BUKAN bug** (di luar scope, keputusan desain terpisah): `TeachingSessionsService.izinSeharianSudahLewat()` (`apps/api/src/teaching-sessions/teaching-sessions.service.ts:390-416`) — ini untuk GURU (izin seharian), bukan siswa. Guru tidak terikat 1 kelas (bisa ngajar lintas kelas/jurusan), jadi konsep "kelasId per baris" TIDAK otomatis applicable — method ini TETAP pakai jam_sekolah global untuk keperluan guru, **TIDAK diubah oleh task ini** (kalau nanti perlu disesuaikan, itu task terpisah dengan keputusan desain sendiri).

**Skema sudah siap** — `Schedule.kelasId` (`schema.prisma:274`) sudah ada sebagai field nullable, jadi TIDAK PERLU migration skema untuk field ini. Yang belum ada: field untuk assign per TINGKAT (X/XI/XII) — task ini PERLU menambah itu.

**Keputusan final struktur 3-lapis** (dikonfirmasi user 2026-08-08, verbatim: "ada 1 default. jadi perbaiki sekalian menu jadwal - jam masuk sekolah, yang dapat menentukan default jam masuk dan jam masuk setiap tingkat per harinya" dan "bagaimana jika keduanya... di menu jam masuk sekolah akan input per tingkat saja, tapi setiap tingkat ketika di klik akan masuk kehalaman yang menampilkan semua kelas yang ada di tingkat tersebut. secara default semua kelas akan sesuai jam tingkatan tapi bisa di edit menjadi jam yang berbeda di halaman tersebut"):

1. **Lapis 1 — Default Global**: 1 set jam masuk (per hari, Senin-Sabtu) berlaku untuk SEMUA kelas yang tidak punya override di lapis 2/3.
2. **Lapis 2 — Default Per Tingkat**: admin bisa set jam masuk BERBEDA untuk tingkat X/XI/XII (per hari). Kalau diisi untuk tingkat tertentu, MENANG atas Lapis 1 untuk SEMUA kelas di tingkat itu (kecuali di-override lagi di Lapis 3).
3. **Lapis 3 — Override Per Kelas**: dari halaman detail 1 tingkat (menampilkan semua kelas di tingkat itu), admin bisa override jam masuk untuk KELAS INDIVIDUAL tertentu (misal karena alasan khusus 1 kelas itu beda dari tingkatnya). Kalau diisi, MENANG atas Lapis 2 DAN Lapis 1 untuk kelas itu SAJA.

**Resolusi urutan prioritas saat lookup** (dipakai backend): Kelas spesifik (Lapis 3) → kalau tidak ada, Tingkat kelas itu (Lapis 2) → kalau tidak ada, Default Global (Lapis 1) → kalau Default Global juga tidak ada untuk hari itu, PERLAKUKAN SAMA seperti sekarang (`determineStatus()` return `hadir`, tidak pernah alfa/terlambat untuk hari tanpa jadwal — TIDAK diubah, di luar scope).

## Spec Detail

### 1. Schema (Prisma) — restrukturisasi `Schedule{type: jam_sekolah}`

**Opsi A (direkomendasikan)** — tambah kolom `tingkat` (nullable) ke model `Schedule` yang sudah ada, DIPAKAI BERSAMA `kelasId` (yang sudah ada) untuk menentukan lapis mana sebuah baris berlaku:
```prisma
// Tambah ke model Schedule yang sudah ada:
tingkat Tingkat? @map("tingkat") // enum X/XI/XII yang SUDAH ADA di schema (dipakai Kelas.tingkat), nullable
```
Aturan pengisian PER BARIS (WAJIB divalidasi di service, bukan cuma dokumentasi):
- **Baris Lapis 1 (Default Global)**: `kelasId: null`, `tingkat: null`.
- **Baris Lapis 2 (Default Per Tingkat)**: `kelasId: null`, `tingkat: "X"` (atau XI/XII) — 1 baris per kombinasi (tingkat, hari).
- **Baris Lapis 3 (Override Per Kelas)**: `kelasId: <id>`, `tingkat: null` (kelasId sudah cukup spesifik, tingkat tidak perlu diisi ganda — TAPI service BOLEH auto-isi `tingkat` dari `kelas.tingkat` saat create untuk kemudahan query, putuskan saat implementasi mana yang lebih bersih).
- **VALIDASI WAJIB di service**: TOLAK kombinasi yang tidak masuk akal (misal `kelasId` DAN `tingkat` keduanya diisi dengan tingkat yang BEDA dari `kelas.tingkat` aktual — inkonsisten data; atau lebih dari 1 baris untuk kombinasi (hari, kelasId) yang sama; atau lebih dari 1 baris untuk kombinasi (hari, tingkat, kelasId:null) yang sama — masing-masing kombinasi harus UNIK).
- Pertimbangkan `@@unique` composite kalau Prisma bisa mengekspresikan uniqueness kondisional (kemungkinan tidak bisa langsung karena nullable composite unique berperilaku khusus di MySQL — cek dan putuskan saat implementasi apakah unique constraint DB memungkinkan atau validasi cukup di level service saja, seperti pola `AttendanceLockConfig` dkk yang "enforce di service, bukan DB constraint").

**Opsi B (alternatif, evaluasi kalau Opsi A terasa janggal saat implementasi)** — pisah jadi 2 konsep berbeda secara eksplisit alih-alih overload 1 model: model baru `SchoolEntryTimeDefault` (per hari, 1 baris global) + model baru `SchoolEntryTimeByTingkat` (per hari+tingkat) + `Schedule{type:jam_sekolah, kelasId}` yang SUDAH ADA dipakai murni untuk Lapis 3. **PUTUSKAN saat implementasi** mana yang lebih bersih (Opsi A lebih sedikit model baru tapi 1 model jadi overload; Opsi B lebih eksplisit tapi 3 model terpisah untuk 1 konsep) — SUDAH CUKUP INFORMASI di sini untuk memutuskan tanpa perlu tanya balik ke user, ini murni keputusan teknis implementasi bukan keputusan bisnis.

- Migration baru (baik Opsi A maupun B).

### 2. Backend — helper resolusi terpusat (WAJIB, satu sumber kebenaran)

Buat 1 method BARU, contoh `ScheduleService.resolveJamMasuk(kelasId: number, hari: number): Promise<{ jamMulai: string; sumberLapis: 1 | 2 | 3 } | null>` — logic:
1. Ambil `kelas.tingkat` dari `kelasId` yang diberikan.
2. Cek Lapis 3 (baris `kelasId` match persis) — kalau ada, return itu.
3. Kalau tidak, cek Lapis 2 (baris `tingkat` match, `kelasId: null`) — kalau ada, return itu.
4. Kalau tidak, cek Lapis 1 (`kelasId: null`, `tingkat: null`) — kalau ada, return itu.
5. Kalau tidak ada ketiganya untuk hari itu — return `null` (perilaku SAMA seperti sekarang saat tidak ada jadwal sama sekali, `determineStatus()` fallback ke `hadir`).

**WAJIB dipakai KONSISTEN oleh SEMUA pemakai** (JANGAN duplikasi logic 3-lapis di berbagai tempat):
- `AttendanceService.determineStatus()` (baris 634-636) — GANTI query `findFirst` langsung dengan panggil `resolveJamMasuk(card.studentId → kelasId, dayOfWeek)`.
- `PermitsService.tandaiIzinTidakKembali()` (baris 417-419) — GANTI dengan panggil `resolveJamMasuk(record.student.kelasId, dayOfWeek)` (kelasId sudah ter-load di scope, cek baris 403).

### 3. Frontend — Redesign Halaman `apps/web/src/app/(admin)/jadwal/jadwal-view.tsx`

Struktur BARU (menggantikan tabel 1-baris-per-hari yang sekarang):
- **Bagian atas — Default Global**: tabel 6 hari (Senin-Sabtu) SEPERTI SEKARANG, tapi LABEL jelas "Jam Masuk Default (berlaku untuk semua kelas kecuali di-override)".
- **Bagian bawah — 3 kartu/link Tingkat** (X, XI, XII) — masing-masing menampilkan ringkasan jam masuk tingkat itu (kalau sudah di-set) atau "Mengikuti Default" (kalau belum). Klik salah satu tingkat → navigasi ke halaman/panel BARU.
- **Halaman/panel Detail Tingkat** (route baru, misal `apps/web/src/app/(admin)/jadwal/tingkat/[tingkat]/` atau dialog/sheet kalau lebih sesuai pola existing — putuskan saat implementasi mana yang lebih konsisten UX admin lain):
  - Form set jam masuk 6-hari UNTUK TINGKAT ini (Lapis 2).
  - **Tabel semua kelas di tingkat ini** — tiap baris kelas menampilkan jam masuk yang BERLAKU untuk kelas itu SAAT INI (hasil resolusi: kalau belum di-override, tampilkan jam TINGKAT dengan indikator "mengikuti tingkat"; kalau sudah di-override, tampilkan jam KELAS dengan indikator "override khusus" + tombol untuk reset kembali ke default tingkat).
  - Aksi edit per baris kelas → form kecil (dialog/inline) set jam masuk 6-hari KHUSUS kelas itu (Lapis 3), atau tombol "Hapus Override" untuk kembali ikut default tingkat.
- Sesuai aturan tabel permanen proyek (memory `feedback_tabel_wajib_search_sort_kolom_no`) — tabel kelas di halaman detail tingkat WAJIB search+kolom No+sortable KALAU jumlah kelas per tingkat berpotensi banyak (>10) — cek jumlah kelas realistis per tingkat di sekolah ini saat implementasi, kalau memang sedikit (<10, tidak akan pernah dipaginasi) boleh dikecualikan konsisten pola halaman master-data-pendek lain.

### 4. Endpoint API baru/diperluas
- `GET /schedules/jam-masuk/default` — 6 baris (hari) Lapis 1.
- `PATCH /schedules/jam-masuk/default` — update Lapis 1 (existing endpoint kemungkinan sudah ada dalam bentuk berbeda, CEK dulu `schedules.controller.ts` sebelum bikin baru — kemungkinan besar bisa REUSE endpoint existing dengan penyesuaian kecil, bukan endpoint baru total).
- `GET /schedules/jam-masuk/tingkat/:tingkat` — 6 baris (hari) Lapis 2 untuk tingkat itu, PLUS daftar semua kelas di tingkat itu dengan status resolusi masing-masing (ikut tingkat / override).
- `PATCH /schedules/jam-masuk/tingkat/:tingkat` — update Lapis 2.
- `PATCH /schedules/jam-masuk/kelas/:kelasId` — set/update override Lapis 3 untuk 1 kelas.
- `DELETE /schedules/jam-masuk/kelas/:kelasId` — hapus override (kembali ikut default tingkat).
- SEMUA endpoint PATCH/DELETE — `@Roles(UserRole.super_admin)` (konsisten pola pengaturan lain), `@LogActivity` wajib.

## Edge Cases
- Kelas dengan `tingkat` yang BERUBAH (misal karena `kenaikanMassal` — walau belum pernah dipakai, tetap harus tidak crash kalau nanti dipakai) — override Lapis 3 kelas itu (kalau ada) TETAP menempel ke `kelasId` yang sama (kelas fisik tidak berubah saat kenaikan kelas, siswa yang pindah — ingat dari riset sebelumnya `kenaikanMassal` mengubah `Student.kelasId`, BUKAN identitas `Kelas` itu sendiri, jadi ini TIDAK relevan sebagai edge case sebenarnya, override tetap valid).
- Tingkat yang belum PERNAH di-set adminnya sama sekali (baru install sistem) — SEMUA kelas di tingkat itu otomatis ikut Default Global (Lapis 1), tidak error.
- Kelas BARU dibuat setelah tingkatnya sudah punya Lapis 2 — otomatis ikut Lapis 2 tingkat itu tanpa perlu setting tambahan (resolusi dinamis by tingkat, bukan disalin manual per kelas).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (tambah field/model sesuai Opsi A/B yang dipilih), `apps/api/src/core/schedules/schedules.service.ts` (`resolveJamMasuk()` baru + validasi), `apps/api/src/core/schedules/schedules.controller.ts` (endpoint baru/diperluas), `apps/api/src/attendance/attendance.service.ts` (`determineStatus()` pakai `resolveJamMasuk`), `apps/api/src/permits/permits.service.ts` (`tandaiIzinTidakKembali()` pakai `resolveJamMasuk`), `apps/web/src/app/(admin)/jadwal/jadwal-view.tsx` (redesign total), halaman/route baru detail tingkat.
- **Jangan sentuh:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts` (`izinSeharianSudahLewat`, guru — di luar scope, keputusan desain terpisah), `ScheduleConfig.toleransiTerlambatMenit` (dipakai jalur guru, tidak terkait task ini).

## Acceptance Criteria
- [x] Admin bisa set Default Global (6 hari) — perilaku sama seperti "Jadwal" existing sekarang, cuma label diperjelas.
- [x] Admin bisa set jam masuk PER TINGKAT (X/XI/XII, masing-masing 6 hari) yang override Default Global.
- [x] Dari halaman detail tingkat, admin bisa lihat semua kelas di tingkat itu + override jam masuk PER KELAS individual yang menang atas tingkat.
- [x] Resolusi 3-lapis (Kelas → Tingkat → Default) dipakai KONSISTEN oleh `determineStatus()` (status Terlambat siswa) DAN `tandaiIzinTidakKembali()` (estimasi jam pulang izin) — satu helper `SchedulesService.resolveJamMasuk()`, tidak ada duplikasi logic.
- [x] Kelas yang belum di-override otomatis ikut jam tingkatnya; tingkat yang belum di-set otomatis ikut Default Global — regresi nol dikonfirmasi live (Skenario 1: 0 jadwal sama sekali → null, sama seperti sebelum T145).
- [x] `izinSeharianSudahLewat()` (guru) TIDAK diubah — dikonfirmasi `git diff` file itu, diff yang ada murni pre-existing T139 (`AcademicPeriodService` injection), bukan dari T145.
- [x] Build + type-check `apps/api` dan `apps/web` hijau. Test suite existing lulus 100% (207→219, 12 test baru).

## Validasi Claudian
- [x] **JANGAN** mengubah `izinSeharianSudahLewat()` — dikonfirmasi tidak disentuh.
- [x] **Opsi A dipilih** (tambah field `tingkat` nullable ke model `Schedule` yang sudah ada) — lihat alasan di Status Eksekusi.
- [x] Validasi service MENOLAK data inkonsisten — `upsertJamMasukHariList()` cek duplikat hari dalam 1 request (400), dan pola delete-then-create (bukan upsert-by-id) menjamin TIDAK ADA cara membuat 2 baris untuk kombinasi (hari, kelasId/tingkat) yang sama lewat endpoint ini.
- [x] `resolveJamMasuk()` selesai dan diverifikasi benar — 6 skenario live end-to-end PASS (lihat Status Eksekusi). **T146 SEKARANG BOLEH DIKERJAKAN.**

## Status Eksekusi (2026-08-09)
- **Keputusan skema — Opsi A dipilih**: tambah `tingkat Tingkat?` ke model `Schedule` yang sudah ada (bukan Opsi B, 3 model terpisah). Alasan: `kelasId` (Lapis 3) sudah ada di model yang sama, jadi menambah `tingkat` (Lapis 2) di model yang sama menjaga SEMUA baris jam_sekolah (3 lapis) queryable lewat 1 tabel dengan filter sederhana (`kelasId`+`tingkat` null/non-null menentukan lapis) — tidak perlu JOIN/UNION lintas 3 tabel untuk resolusi, dan `resolveJamMasuk()` jadi 3 query paralel simetris (`findFirst` per lapis) alih-alih logic percabangan per-model yang beda bentuk. Trade-off diterima: 1 model sedikit "overload" makna kolom `tingkat`/`kelasId` tergantung kombinasi, tapi didokumentasikan jelas di komentar schema + validasi terpusat di 1 helper (`upsertJamMasukHariList`) mencegah kombinasi liar.
- **Migration**: `20260808232837_t145_schedule_tingkat_3_lapis_jam_masuk`, tambah kolom nullable, diterapkan bersih ke dev DB (0 downtime risk, kolom baru nullable).
- **Backend**: `resolveJamMasuk(kelasId, hari)` baru di `SchedulesService` — 3 query paralel (`Promise.all`) untuk Lapis 3/2/1, pilih hit pertama yang non-null sesuai prioritas. 6 endpoint baru di bawah `/schedules/jam-masuk/*` (GET+PATCH default, GET+PATCH tingkat/:tingkat, PATCH+DELETE kelas/:kelasId) — **route `jam-masuk/*` didaftarkan SEBELUM `@Patch(":id")` generik** di controller (NestJS/Express match route berurutan, kalau didaftarkan setelah `:id` maka "jam-masuk" bisa ketangkep sebagai param id).
- **Activity log**: endpoint jam-masuk TIDAK pakai decorator `@LogActivity` (pola sama T143/T144 — endpoint di-key oleh kelasId/tingkat, bukan id baris Schedule tunggal, snapshot-fetch otomatis decorator akan salah ambil baris). Manual `activityLogService.record()` di tiap service method.
- **Validasi konsistensi**: `upsertJamMasukHariList()` — helper private yang dipakai SEMUA 3 endpoint PATCH (default/tingkat/kelas), pola "delete-then-create dalam 1 transaction" (bukan upsert-by-id) — otomatis menjamin uniqueness 1-baris-per-hari per lapis tanpa perlu `@@unique` composite di schema (yang memang bermasalah untuk nullable composite di MySQL, sesuai dugaan spec). Duplikat hari dalam 1 request PATCH ditolak eksplisit (400) sebelum transaction dimulai.
- **Caller 1 — `AttendanceService.determineStatus()`**: constructor inject `SchedulesService` (`CoreModule` sudah diimpor `AttendanceModule`, tidak perlu ubah module wiring). Ganti query `findFirst` langsung dengan `resolveJamMasuk(card.student.kelasId, dayOfWeek)`. Edge case BARU yang ditangani eksplisit: siswa `kelasId: null` (belum di-plot, T072) → fallback `hadir` langsung (tidak ada tingkat untuk diresolusi), behavior sama seperti "tidak ada jadwal" sebelumnya.
- **Caller 2 — `PermitsService.tandaiIzinTidakKembali()`**: constructor inject `SchedulesService`, **`PermitsModule` ditambah import `CoreModule`** (sebelumnya tidak ada — dicek dulu tidak ada circular dependency risk lewat transitive imports `PiketSchedulesModule`/`AcademicPeriodModule`). `record.student.kelasId` dijamin non-null di titik pemanggilan (guard `ForbiddenException` di atasnya sudah menyaring kelas null).
- **Frontend**: `jadwal-view.tsx` dirombak total — bagian atas Default Global (form 6-hari sekaligus, bukan 1-hari-per-submit seperti sebelumnya), bagian bawah 3 kartu Tingkat (klik → `/jadwal/tingkat/[tingkat]`). Halaman baru `jadwal/tingkat/[tingkat]/` — form Lapis 2 + tabel kelas dengan badge status ("Mengikuti tingkat" / "Override khusus") + aksi edit (dialog form Lapis 3) dan reset (DELETE override). `JamMasukForm`/`JamMasukTable` diekstrak reusable, dipakai 3 tempat (Default Global, Tingkat, per-Kelas override) — TIDAK ada duplikasi form 6-hari. Tabel kelas per tingkat TIDAK pakai search/sort/kolom No (dicek dulu: dev DB max 3 kelas per tingkat, jauh di bawah threshold 10 di memory `feedback_tabel_wajib_search_sort_kolom_no`).
- **Verifikasi**: `tsc --noEmit` bersih `apps/api`+`apps/web`. `jest` 219/219 pass (12 test baru: 5 `resolveJamMasuk` skenario lapis di `schedules.service.spec.ts` yang sudah ada, 7 CRUD jam-masuk baru). **Live end-to-end** via script `NestFactory.createApplicationContext` terhadap dev DB (kelas real: X DKV 1/X TKJ 1/X TKR 3 tingkat X, XI DKV 1/XI TKR 1 tingkat XI, XII TKJ 1 tingkat XII) — 6 skenario BERURUTAN dikonfirmasi PASS: (1) tanpa jadwal sama sekali → null; (2) set Default Global → Lapis 1 resolve benar; (3) set Lapis 2 tingkat X → menang atas default UNTUK kelas tingkat X, kelas tingkat LAIN (XI) tetap Lapis 1 (isolasi antar-tingkat benar); (4) set Lapis 3 override 1 kelas → menang atas tingkat+default UNTUK kelas itu SAJA, kelas lain tingkat sama (X TKJ 1) tetap ikut Lapis 2 (isolasi antar-kelas benar); (5) `getJamMasukTingkat` response `kelasList` status resolusi akurat; (6) hapus override → kelas kembali ikut Lapis 2. Semua data test dibuat+dihapus bersih, dev DB `schedules` type=jam_sekolah kembali 0 baris.
- **T146 dependency**: `resolveJamMasuk()` selesai diverifikasi benar sesuai syarat Validasi Claudian di spec ini — T146 (Alfa Live di Papan Piket) sekarang boleh dikerjakan.

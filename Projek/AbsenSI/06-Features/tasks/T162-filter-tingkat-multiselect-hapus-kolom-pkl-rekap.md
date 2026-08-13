# T162 — Web+API: Filter Tingkat Multi-Select + Hapus Tampilan Kolom PKL di Rekap Kehadiran

## Depends on
Tidak ada dependency teknis. Independen dari T161.

## Objective
1. Halaman Rekap Kehadiran admin — filter **Tingkat** bisa memilih **LEBIH DARI 1 tingkat sekaligus** (misal X DAN XI bersamaan), bukan cuma 1 tingkat per kali lihat.
2. Kolom/badge/status **"PKL"** DIHAPUS dari SEMUA tampilan rekap (tabel, badge, grafik, PDF, Excel) — TAPI logic yang mengecualikan siswa PKL dari perhitungan Alfa **TETAP HARUS JALAN PERSIS SEPERTI SEKARANG**, TIDAK BOLEH berubah sama sekali.

## Context — Alasan (dari Diskusi 2026-08-12)

User eksplisit: **"absensi PKL tidak akan di ambil dari aplikasi ini, sudah ada aplikasi khusus pkl. status PKL disini hanya agar tidak terhitung alfa saja."** — jadi kolom PKL yang SAAT INI ditampilkan di rekap (menunjukkan siswa mana yang sedang PKL) TIDAK PERLU DILIHAT ADMIN LAGI (aplikasi PKL terpisah sudah menangani pelaporan PKL), TAPI mekanisme di baliknya (siswa PKL tidak dihukum Alfa karena memang tidak wajib hadir di sekolah) **HARUS TETAP BERFUNGSI**.

**Riset kode mengonfirmasi**: logic exclude-dari-Alfa dan penghitungan kolom tampilan `pkl` di `attendance-report.service.ts` (`reportInternal()`, baris ~225-233) adalah **1 BLOK KODE YANG SAMA** yang PERLU DIPISAHKAN dengan hati-hati:
```ts
for (const dateKey of wajibDates) {
  if (!firstCardDate || dateKey < firstCardDate) { belumMemilikiKartu += 1; continue; }
  if (bucket.presentDates.has(dateKey) || excusedDates.has(dateKey)) continue;
  if (pklDates.has(dateKey)) pkl += 1;    // <- baris exclude-dari-alfa SEKALIGUS hitung tampilan
  else alfa += 1;
}
```
**KEPUTUSAN**: variabel `pkl` TETAP di-increment secara INTERNAL (logic exclude tidak berubah SAMA SEKALI — `if (pklDates.has(dateKey)) pkl += 1; else alfa += 1;` TIDAK DIUBAH), TAPI field `pkl` **TIDAK LAGI di-expose** ke response API (`ReportRow` interface dan objek return) — perbedaannya HANYA di titik expose, bukan di titik hitung.

## Spec Detail

### 1. Filter Tingkat — Multi-Select

**Riset mengonfirmasi TIDAK ADA komponen multi-select di codebase ini sama sekali** (dicek: tidak ada `MultiSelect`/checkbox-group di `packages/ui/` maupun halaman lain manapun) — ini KOMPONEN BARU yang perlu dibuat.

- **REKOMENDASI implementasi**: karena Tingkat cuma 3 opsi TETAP (X/XI/XII, tidak akan pernah bertambah — ini enum tetap, bukan data dinamis), JANGAN buat komponen dropdown-multi-select KOMPLEKS — CUKUP **3 CHECKBOX SEDERHANA** ("X", "XI", "XII") berjajar/berdampingan menggantikan dropdown Select yang sekarang. Ini LEBIH SEDERHANA diimplementasikan DAN lebih jelas UX-nya untuk cuma 3 opsi tetap, dibanding membangun komponen multi-select generik yang overkill untuk kasus ini.
- Frontend `apps/web/src/app/(admin)/rekap/rekap-view.tsx` — GANTI state `tingkat` dari `useState(ALL)` (single string) jadi `useState<Tingkat[]>([])` (array, kosong = semua tingkat/tidak difilter — KONSISTEN semantik dengan `ALL` yang sekarang, cuma representasinya array kosong bukan string khusus).
- `buildParams()` — kirim SEMUA tingkat terpilih ke backend. Format query string: gunakan pola **repeated key** (`?tingkat=X&tingkat=XI`), KONSISTEN dengan cara NestJS/`class-validator` menerima array dari query string secara native — VERIFIKASI di backend (poin 2) bahwa format ini yang diharapkan.
- `kelasTerfilter` (filter turunan Kelas berdasar Tingkat, client-side) — ubah dari `tingkat === ALL || k.tingkat === tingkat` (single equality) jadi `tingkatTerpilih.length === 0 || tingkatTerpilih.includes(k.tingkat)` (array membership, kosong = semua).
- `buildFilterLabel()` (label kop PDF/Excel) — sesuaikan untuk menampilkan SEMUA tingkat terpilih (misal "Tingkat X, XI" kalau 2 dipilih, bukan cuma 1).

### 2. Backend — DTO dan query jadi array

- `apps/api/src/attendance/dto/report-query.dto.ts` — ganti field `tingkat` dari `@IsIn(["X","XI","XII"]) tingkat?: Tingkat` (single) jadi **array**:
```ts
@IsOptional()
@IsArray()
@IsIn(["X", "XI", "XII"], { each: true })
@Transform(({ value }) => (Array.isArray(value) ? value : [value]))  // toleransi kalau cuma 1 nilai terkirim (query string tunggal jadi string, bukan array)
tingkat?: Tingkat[];
```
- `ReportExportQueryDto` (extends `ReportQueryDto`) — otomatis ikut berubah, TIDAK PERLU perubahan terpisah (VERIFIKASI ini benar saat implementasi).
- `attendance-report.service.ts` — di KEDUA lokasi yang query `kelas.tingkat` (`reportInternal()` baris ~113-117 DAN `reportSingleDay()` baris ~292-297) — ganti dari equality (`tingkat: query.tingkat`) jadi **IN**:
```ts
kelas:
  query.jurusanId || (query.tingkat && query.tingkat.length > 0)
    ? { jurusanId: query.jurusanId, tingkat: query.tingkat && query.tingkat.length > 0 ? { in: query.tingkat } : undefined }
    : undefined,
```
(SESUAIKAN struktur PERSIS dengan kode existing di kedua lokasi, JANGAN asal tempel — baca dulu kode aslinya untuk pastikan perubahan minimal dan konsisten).

### 3. Hapus tampilan PKL — SEMUA 5+ lokasi yang teridentifikasi riset

**Backend** (`attendance-report.service.ts`):
- `ReportRow` interface — HAPUS field `pkl` dari interface (baris ~45), TAPI variabel `pkl` LOKAL di dalam loop TETAP ADA (untuk exclude-dari-alfa) — JANGAN expose ke object return (baris ~247, hapus `pkl` dari object yang dikembalikan sebagai `ReportRow`).
- `reportSingleDay()` (mode 1-hari) — status `"pkl"` (baris ~353, prioritas tertinggi di antara status lain) — **KEPUTUSAN WAJIB DIPUTUSKAN**: kalau status PKL TIDAK BOLEH tampil sama sekali, siswa yang sebenarnya PKL harus JATUH ke status LAIN saat ditampilkan — REKOMENDASI: siswa PKL pada mode single-day sebaiknya TIDAK MUNCUL SAMA SEKALI di tabel rekap (bukan muncul dengan status yang salah/menyesatkan seperti "Belum Absen") — KONSULTASI/PUTUSKAN saat implementasi: apakah siswa PKL DIKECUALIKAN TOTAL dari daftar/tabel rekap mode single-day (tidak muncul barisnya sama sekali, KONSISTEN dengan filosofi "PKL tidak diambil dari aplikasi ini"), ATAU tetap muncul tapi dengan status netral tanpa label eksplisit "PKL" (misal kosongkan status/keterangan) — REKOMENDASI KUAT: **kecualikan total dari tabel** (siswa PKL SAMA SEKALI tidak relevan ditampilkan di rekap kehadiran harian, sesuai maksud user), BUKAN cuma sembunyikan labelnya.
- Union type `SingleDayStatus` — KALAU status `"pkl"` dihapus total dari daftar siswa yang ditampilkan (rekomendasi di atas), TETAP PERTIMBANGKAN apakah union type ini masih perlu include `"pkl"` sebagai NILAI INTERNAL (dipakai untuk FILTER siswa yang dikecualikan SEBELUM sampai ke response), atau bisa dihapus total dari type publik. PUTUSKAN saat implementasi, PASTIKAN type-safety tetap terjaga.

**Frontend** (`rekap-view.tsx`):
- `SINGLE_DAY_STATUS_ORDER`/`LABEL`/`CHART_COLOR`/`BADGE_CLASS` (baris ~66-95) — HAPUS entry `"pkl"` dari SEMUA konstanta ini.
- Badge status tabel single-day (baris ~728-733) — otomatis tidak akan render PKL lagi setelah backend TIDAK MENGIRIM baris siswa PKL (rekomendasi di atas).
- Grafik Chart.js single-day (baris ~229-256) — HAPUS `pkl` dari perhitungan `totals` dan dari label/warna sumbu grafik.
- Header kolom "PKL" tabel mode range (baris ~801-807) — HAPUS kolom ini SEPENUHNYA dari tabel.
- Cell kolom PKL tabel mode range (baris ~836) — HAPUS.

**Export** (`attendance-report-export.service.ts`):
- Excel mode range — HAPUS kolom `{ header: "PKL", key: "pkl", width: 8 }` (baris ~196).
- Excel mode single-day — kolom Status TIDAK LAGI menampilkan label "PKL" (otomatis konsisten kalau siswa PKL sudah dikecualikan dari data yang diterima, poin di atas).
- PDF mode range (`buildHtml`) — HAPUS `<th>PKL</th>` (baris ~310) dan `<td>${row.pkl}</td>` (baris ~233).
- PDF mode single-day (`buildSingleDayHtml`) — pastikan status "PKL" TIDAK muncul di tabel (baris ~369) MAUPUN grafik (baris ~376-378) — konsisten dengan keputusan pengecualian siswa PKL dari data yang diproses.
- Konstanta `SINGLE_DAY_STATUS_LABEL`/`ORDER`/`COLOR` di FILE INI (duplikat terpisah dari frontend, baris ~17-43) — HAPUS entry `"pkl"` JUGA DI SINI (perubahan harus dilakukan **2 KALI**, backend export service DAN frontend rekap-view.tsx, KARENA ini bukan shared code — konfirmasi riset).

## Edge Cases
- Semua siswa di 1 kelas sedang PKL (kondisi realistis untuk kelas XII di akhir semester) — kalau siswa PKL dikecualikan total dari tabel single-day, kelas itu BISA tampil TANPA BARIS SAMA SEKALI untuk tanggal itu — pastikan UI menampilkan state KOSONG YANG WAJAR ("Tidak ada siswa untuk ditampilkan" atau serupa), BUKAN error/crash.
- `riwayatCatatan()` (halaman LAIN, riwayat per-siswa, BUKAN rekap) — punya entry `jenis: "pkl"` JUGA (di luar scope task ini, dikonfirmasi riset) — **JANGAN SENTUH**, task ini SCOPE-nya HANYA halaman Rekap Kehadiran (`report()`/`reportSingleDay()`/export terkait), bukan riwayat individual siswa.

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (filter multi-select, hapus 5 lokasi tampilan PKL), `apps/api/src/attendance/dto/report-query.dto.ts` (`tingkat` jadi array), `apps/api/src/attendance/attendance-report.service.ts` (query IN, hapus expose field `pkl`, exclude siswa PKL dari `reportSingleDay`), `apps/api/src/attendance/attendance-report-export.service.ts` (hapus kolom/label PKL di PDF+Excel).
- **Jangan sentuh:** `riwayatCatatan()` (di luar scope, halaman berbeda), logic INTERNAL exclude-dari-alfa (`if (pklDates.has(dateKey)) pkl += 1; else alfa += 1;` — baris ini TIDAK BERUBAH, cuma titik expose-nya yang dihapus).

## Acceptance Criteria
- [x] Filter Tingkat di halaman Rekap bisa pilih X, XI, XII secara BEBAS gabungan (0, 1, 2, atau 3 sekaligus), hasil rekap menampilkan gabungan tingkat yang dipilih.
- [x] Kolom "PKL" TIDAK ADA LAGI di tabel rekap mode range (tampilan, PDF, Excel).
- [x] Badge/status "PKL" TIDAK ADA LAGI di tabel rekap mode single-day (tampilan, PDF, Excel) — siswa PKL dikecualikan total dari daftar yang ditampilkan.
- [x] Grafik (Chart.js di layar+PDF) TIDAK LAGI punya kategori "PKL".
- [x] **VERIFIKASI KRITIKAL**: kolom Alfa TETAP DIHITUNG BENAR — siswa yang statusnya PKL TETAP TIDAK DIHITUNG ALFA (verified live, lihat Status Eksekusi — alfa=4 tanpa PKL record, alfa=3 dengan PKL record menutup 1 hari, PERSIS selisih 1 sesuai hari PKL).
- [x] Build + type-check `apps/api` dan `apps/web` hijau. Test suite existing lulus 100% (4 test yang eksplisit cek field `pkl` DIPERBARUI menjadi menguji efeknya lewat `alfa`, bukan dihapus — tetap menguji logic exclude-dari-alfa yang mendasarinya).

## Validasi Claudian
- [x] **JANGAN** mengubah logic exclude-dari-alfa (`pklDates.has(dateKey)` check) — TIDAK diubah, baris `if (pklDates.has(dateKey)) pkl += 1; else alfa += 1;` identik seperti sebelumnya, HANYA titik expose ke `ReportRow` yang dihapus.
- [x] **VERIFIKASI EKSPLISIT** dengan test/data nyata — dilakukan via curl langsung ke dev API (bukan cuma unit test): insert `StudentPkl` nyata untuk 1 siswa, bandingkan `alfa` sebelum/sesudah insert, hasilnya PERSIS sesuai ekspektasi matematis.
- [x] **JANGAN** sentuh `riwayatCatatan()` — TIDAK disentuh, method itu (dan entry `jenis: "pkl"` di dalamnya) tetap seperti semula.
- [x] Perubahan konstanta label PKL dilakukan DI DUA TEMPAT terpisah (`rekap-view.tsx` DAN `attendance-report-export.service.ts`) — keduanya diperbarui.

## Status Eksekusi (2026-08-13)

**Selesai.** Backend + frontend + test + verifikasi live, semua hijau.

**Filter Tingkat multi-select**:
- `report-query.dto.ts` — `tingkat` jadi `Tingkat[]` dengan `@Transform` toleransi 1 nilai (query string tunggal → array).
- `attendance-report.service.ts` — KEDUA lokasi (`reportInternal()`, `reportSingleDay()`) query `kelas.tingkat` diubah dari equality jadi `{ in: query.tingkat }`.
- `rekap-view.tsx` — dropdown Select diganti 3 tombol pill toggle (`aria-pressed`, konsisten desain sistem `rounded-full`) — TIDAK ada komponen multi-select generik dibuat, sesuai rekomendasi spec (3 opsi tetap, checkbox sederhana cukup). `buildParams()` kirim repeated query key `?tingkat=X&tingkat=XI`.
- **Verified live**: `tingkat=X` → hanya kelas X; `tingkat=X&tingkat=XI` → union kedua tingkat (dikonfirmasi via curl langsung terhadap dev API, bukan cuma baca kode).

**Hapus tampilan PKL** (titik expose SAJA, logic exclude-dari-alfa TIDAK disentuh):
- `ReportRow` interface — field `pkl` dihapus. Variabel lokal `pkl` di dalam loop `reportInternal()` TETAP dihitung persis sama (`if (pklDates.has(dateKey)) pkl += 1; else alfa += 1;`), hanya tidak masuk object return.
- `reportSingleDay()` — siswa PKL DIKECUALIKAN TOTAL dari array yang di-return (`.filter((s) => !pklStudentIds.has(s.id))` sebelum `.map()`), `no` dihitung ULANG setelah filter supaya penomoran tetap berurutan tanpa celah. Status `"pkl"` dan seluruh union `SingleDayStatus` terkait dihapus (sekarang 6 kategori, bukan 7) — TypeScript otomatis memaksa semua `Record<SingleDayStatus, ...>` di export service DAN frontend menghapus key `pkl` juga (compiler-enforced consistency, bukan manual tracking 2 tempat terpisah).
- Export service (`attendance-report-export.service.ts`) — kolom Excel `PKL` dihapus, `<th>PKL</th>`+`<td>${row.pkl}</td>` PDF range dihapus, konstanta `SINGLE_DAY_STATUS_ORDER/LABEL/COLOR` (duplikat terpisah dari frontend) diperbarui juga.
- `rekap-view.tsx` — konstanta duplikat yang sama diperbarui, kolom "PKL" tabel range + `SortableHeader` dihapus.
- `core-types.ts` — `AttendanceReportRow.pkl` dan `SingleDayStatus` union `"pkl"` dihapus, konsisten dengan backend.

**Test** — 4 test existing (`attendance-report.service.spec.ts`) yang assert `row.pkl` DIPERBARUI (bukan dihapus) untuk menguji efeknya via `alfa` (nilai yang benar-benar penting secara bisnis), + 1 test "PKL menang dari status lain" diganti jadi "PKL dikecualikan total dari daftar" (`toBeUndefined()`). Full suite 265/265 pass.

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh):
1. Insert `StudentPkl` nyata untuk 1 siswa (tanggal 2026-08-13) → `GET /attendance/report-flexible?from=2026-08-13&to=2026-08-13` (single-day) → siswa itu TIDAK muncul di 39 baris hasil (dikonfirmasi by studentId, bukan asumsi).
2. Mode range (`2026-08-10..2026-08-13`) DENGAN PKL record → `alfa: 3`, field `pkl` TIDAK ADA di response JSON sama sekali.
3. Record PKL DIHAPUS sementara → re-query rentang SAMA → `alfa: 4` — selisih PERSIS 1 (hari PKL yang di-exclude), membuktikan logic exclude-dari-alfa 100% identik dengan sebelum perubahan.
4. `tingkat=X` vs `tingkat=X&tingkat=XI` (repeated query key) → hasil `kelas` yang terlibat berbeda sesuai ekspektasi (union yang benar).
5. Export Excel (`?tingkat=X&tingkat=XI`) → 200 OK, file valid, `unzip` konten XML dikonfirmasi TIDAK mengandung string "PKL" di `sheet1.xml` maupun `sharedStrings.xml`.
6. Export PDF single-day → 200 OK, file PDF valid 2 halaman.
7. Semua data uji (user test admin, activity_log terkait, `StudentPkl` test record) dibersihkan setelah verifikasi.
8. `tsc --noEmit` bersih `apps/api` + `apps/web`. Jest `apps/api` 265/265 pass.

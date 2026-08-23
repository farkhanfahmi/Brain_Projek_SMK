# T222 — Web+API: Replikasi Filter Per-Kolom + Pilih Kolom Export ke Rekap Guru/Karyawan

## Depends on
**REKOMENDASI KUAT dikerjakan SETELAH T218 dan T221 selesai** (rekap murid) — task ini murni REPLIKASI pola yang sama ke halaman rekap guru, supaya konvensi komponen (`column-filter-header.tsx` dari T218, checkbox export dari T221) sudah stabil dan tidak perlu didesain 2x secara paralel dengan variasi kecil yang tidak konsisten. Kalau TERPAKSA dikerjakan paralel dengan T218/T221, WAJIB koordinasi ketat pola komponen yang dipakai (jangan sampai rekap murid dan rekap guru punya 2 implementasi filter-kolom yang beda gaya).

## Konteks — Permintaan User (2026-08-18)

User minta fitur yang SAMA PERSIS dengan T218 (filter per-kolom ala Excel) dan T221 (checkbox pilih-kolom export + warna tombol PDF merah/Excel hijau) diterapkan JUGA ke halaman **Rekap Kehadiran Guru & Karyawan** (`apps/web/src/app/(admin)/rekap-guru/rekap-guru-view.tsx`).

## Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-18)

- **Hanya 1 halaman**, guru+karyawan digabung (bukan 2 halaman terpisah) — `apps/web/src/app/(admin)/rekap-guru/rekap-guru-view.tsx` (591 baris). "Karyawan" bukan entitas terpisah, hanya nilai `Teacher.statusKepegawaian: "guru" | "karyawan"`.
- Backend terpisah total dari rekap murid: modul `apps/api/src/teacher-attendance-report/` (`teacher-attendance-report.service.ts`, `teacher-attendance-report-export.service.ts`) — **paralel, sengaja diduplikasi** dari modul `attendance` (komentar eksplisit di kode: "DUPLIKASI SENGAJA bukan reuse import karena modul ini paralel/terpisah dari rekap siswa").
- Fetch **tanpa pagination**, sama seperti rekap murid — filter+pilih-kolom bisa murni client-side.
- `SortableHeader` **sudah terpasang** di semua kolom kedua mode.
- Search box **sudah ada** tapi HANYA global (nama), **belum ada** `ColumnFilterHeader`/filter per-kolom sama sekali (T218 belum diterapkan di sini).
- Tombol Download PDF/Excel **sudah ada**, KEDUANYA masih `variant="outline"` polos (T221 juga belum diterapkan — bukan cuma di rekap guru, di rekap murid pun T221 belum dikerjakan saat task ini ditulis, jadi task INI harus tunggu T221 selesai duluan supaya ada polanya untuk direplikasi).
- Backend export (`teacher-attendance-report-export.service.ts`) — kolom **hardcode** di `sheet.columns` (Excel, 2 tempat: single-day baris 247-255, range baris 280-293) dan template HTML `<th>` (PDF, 2 tempat: range baris 437-450, single-day baris 552-559) — TIDAK ADA mekanisme pilih-kolom, sama seperti kondisi rekap murid sebelum T221.

## Kolom Tabel Rekap Guru (untuk referensi filter+export)

**Mode single-day (Per Hari)** — 7 kolom: No, Nama, NIY, Kepegawaian (`statusKepegawaian`, enum guru/karyawan), Status (enum 7 nilai: hadir/terlambat/belum_absen/sakit/izin_pribadi/tugas_dinas/pelatihan), Waktu, Keterangan.

**Mode range (Per Rentang)** — 12 kolom: No, Nama, NIY, Kepegawaian, Hadir, Terlambat, Sakit, Izin Pribadi, Tugas Dinas, Pelatihan, Alfa, Jumlah Hari Wajib.

## Spec Detail

### 1. Filter per-kolom (replikasi T218)

- REUSE KOMPONEN yang sama persis dibuat T218 (`apps/web/src/components/column-filter-header.tsx` atau perluasan `sortable-header.tsx`, sesuai keputusan implementasi T218) — JANGAN buat komponen filter-kolom kedua yang terpisah, pasang komponen yang sudah ada ke header tabel `rekap-guru-view.tsx`.
- **Kolom kategori** → dropdown checklist multi-select: **Kepegawaian** (guru/karyawan — nilai tetap cuma 2, checklist sangat ringan), **Status** (mode single-day, 7 nilai enum).
- **Kolom angka** → filter range min-max: Hadir, Terlambat, Sakit, Izin Pribadi, Tugas Dinas, Pelatihan, Alfa, Jumlah Hari Wajib (mode range).
- **Kolom teks bebas** → text-contains: Nama, NIY, Keterangan.
- State filter (`rangeColumnFilters`/`singleDayColumnFilters`, dsb) — pola SAMA PERSIS T218, TAPI instance state terpisah dari rekap murid (halaman berbeda, tidak share state).
- Filter kolom RESET saat filter bar atas (Kepegawaian pill Semua/Guru/Karyawan, tanggal, dsb — cek filter bar existing di baris 391-416) berubah — KONSISTEN aturan T218.

### 2. Pilih kolom export + warna tombol (replikasi T221)

- Checkbox "sertakan di export" di header tiap kolom (kecuali "No") — REUSE komponen gabungan sort+filter+export-toggle dari T221 (SATU komponen header, bukan 3 elemen terpisah).
- localStorage key TERPISAH dari rekap murid, misal `absensi:rekap-guru:export-columns:range` dan `absensi:rekap-guru:export-columns:single-day`.
- Tombol Download PDF → `variant="destructive"` (merah, SUDAH tersedia sejak T221 nambah). Tombol Download Excel → `variant="success"` (hijau, varian yang DIBUAT T221 — reuse langsung, JANGAN buat varian baru lagi).
- Backend `teacher-attendance-report-export.service.ts` — SAMA PERLAKUAN seperti `attendance-report-export.service.ts` di T221: `sheet.columns`/template HTML `<th>` diubah data-driven dari parameter `kolom` baru di DTO export (`apps/api/src/teacher-attendance-report/dto/` — cek nama file DTO export yang setara `ReportExportQueryDto`, replikasi field `kolom` yang sama).
- Whitelist nama kolom valid — SESUAIKAN daftar kolom guru (Kepegawaian, Status, Hadir, Terlambat, Sakit, Izin Pribadi, Tugas Dinas, Pelatihan, Alfa, Jumlah Hari Wajib — BUKAN daftar kolom murid, jangan copy-paste whitelist murid apa adanya).

### 3. Edge case khusus rekap guru (tidak ada padanan di rekap murid)

- **Filter Kepegawaian = Karyawan + mode Per Hari** — sudah ada pesan khusus existing "karyawan tidak punya jadwal mengajar" (`karyawanDikecualikan`, baris 327, 447-451) — PASTIKAN filter kolom BARU (T218-style) tidak bentrok/menutupi pesan edge case ini yang sudah ada, kolom "Status" untuk karyawan di mode Per Hari mungkin punya semantik berbeda dari guru — VERIFIKASI saat implementasi apakah filter dropdown Status perlu penyesuaian nilai enum yang relevan untuk karyawan vs guru.

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/rekap-guru/rekap-guru-view.tsx` (pasang komponen header T218+T221, localStorage terpisah, warna tombol), `apps/api/src/teacher-attendance-report/dto/` (field `kolom`, cek nama file DTO export yang tepat), `apps/api/src/teacher-attendance-report/teacher-attendance-report-export.service.ts` (kolom data-driven).
- **Jangan sentuh:** modul `attendance`/`attendance-report-export.service.ts` (rekap murid) — SENGAJA terpisah, jangan digabung/direfaktor jadi shared code kalau bukan itu yang diminta (konsisten pola duplikasi sengaja yang sudah ada di proyek ini).

## Acceptance Criteria
- [x] Filter per-kolom tampil di header tabel rekap guru, pola SAMA PERSIS T218 (komponen di-reuse, bukan reimplementasi).
- [x] Checkbox pilih-kolom export tampil di header, localStorage terpisah dari rekap murid, pola SAMA PERSIS T221.
- [x] Tombol Download PDF merah, Excel hijau — reuse varian `success`/`destructive` yang sudah dibuat T221 (tidak bikin varian baru lagi).
- [x] Backend export terima parameter `kolom` dengan whitelist SESUAI kolom rekap guru (bukan kolom murid).
- [x] Edge case "karyawan tidak punya jadwal mengajar" tetap berfungsi normal, tidak bentrok dengan filter kolom baru.
- [x] Modul `attendance`/rekap murid TIDAK tersentuh sama sekali oleh task ini.
- [x] Build + type-check hijau, jest baru mengikuti pola test T218/T221 tapi untuk `teacher-attendance-report-export.service.ts`.

## Validasi Claudian
- [x] Konfirmasi task ini dikerjakan SETELAH T218+T221 selesai (atau kalau paralel, konfirmasi komponen shared benar-benar sama, bukan 2 implementasi berbeda gaya untuk fitur yang sama). (T218+T221 keduanya sudah Selesai sebelum T222 dimulai, dicek via STATUS.md)
- [x] Konfirmasi whitelist kolom backend SESUAI kolom rekap guru (Kepegawaian/Status/dst), bukan hasil copy-paste whitelist kolom murid yang salah konteks. (`TEACHER_EXPORTABLE_COLUMNS` di `teacher-report-export-query.dto.ts` — Kepegawaian/Status/Hadir/Terlambat/Sakit/Izin Pribadi/Tugas Dinas/Pelatihan/Alfa/Jumlah Hari Wajib, DTO test eksplisit menolak kolom murid seperti "jurusan")

# T224d — Web: Wali Kelas — Integrasi Frontend (Daftar Siswa, Detail Siswa, RekapView Scoped)

## Depends on
**WAJIB SETELAH T224a, T224b, T224c SEMUA selesai** (3 endpoint backend yang diintegrasikan task ini). Bagian 4 dari 4, murni frontend.

## Konteks

3 endpoint backend baru sudah tersedia (asumsi T224a/b/c selesai):
- `GET /journal/kelas-wali-siswa` (T224a) — daftar siswa kelas wali.
- `GET /journal/kelas-wali-siswa/:id` + `GET /journal/kelas-wali-siswa/:id/riwayat-catatan` (T224b) — biodata + riwayat.
- `GET /journal/kelas-wali-rekap` (T224c) — rekap kehadiran scoped, terima `studentIds?`.

Halaman `(guru)/guru/wali-kelas/` **SUDAH ADA** dengan 3 tab (Ringkasan Kehadiran, Rekap Per Mapel, Catatan Guru Mapel) — task ini MENAMBAH 2 tab baru, TIDAK menghapus/mengubah 3 tab existing.

## Spec Detail

### 1. Tab baru "Daftar Siswa"

- Fetch `GET /journal/kelas-wali-siswa` (T224a) saat tab dibuka/mount.
- Tabel: No, Nama, NISN, Status (badge — hijau "Aktif" / abu-abu "Nonaktif" untuk siswa nonaktif) — KONSISTEN aturan tabel wajib proyek (search box, sortable per kolom via `SortableHeader`).
- Klik baris/nama siswa → buka detail (Sheet, KONSISTEN pola form panjang proyek — REKOMENDASI dari task asli) berisi:
  - Biodata lengkap (fetch `GET /journal/kelas-wali-siswa/:id`, T224b) — foto, NISN, tempat/tanggal lahir, jenis kelamin, agama, alamat, nama orang tua, no HP.
  - Riwayat Catatan (fetch `GET /journal/kelas-wali-siswa/:id/riwayat-catatan`, T224b) — REUSE komponen render tabel yang SAMA dengan admin (`RiwayatCatatanSection`/tabel di `siswa-detail-view.tsx` admin) — EKSTRAK jadi komponen shared `apps/web/src/components/riwayat-catatan-table.tsx` kalau belum ada, dipakai baik oleh halaman admin maupun wali kelas (JANGAN duplikasi kode render tabel 4-kolom yang sama).
  - **TIDAK ADA section riwayat kartu** — response T224b sudah tidak mengandung field itu, FE tidak perlu logic exclude tambahan.

### 2. Tab baru "Rekap Detail" (atau nama lain, terpisah dari "Ringkasan Kehadiran" existing)

- REUSE komponen `RekapView` (`apps/web/src/app/(admin)/rekap/rekap-view.tsx`, 1183 baris) — TAMBAH prop baru opsional:
  ```ts
  interface RekapViewProps {
    // ...props existing...
    scopeMode?: "admin" | "wali-kelas";
    fixedKelasId?: number;
  }
  ```
  - Mode `wali-kelas`: SEMBUNYIKAN filter dropdown Jurusan/Tingkat/Kelas (kelas sudah fixed dari `kelasIdWali`), TAMBAH checklist pilih siswa (state baru, list didapat dari `GET /journal/kelas-wali-siswa` yang SAMA seperti tab Daftar Siswa — REUSE fetch, jangan panggil 2x kalau bisa share state antar tab) di atas tabel.
  - Endpoint yang dipanggil ditentukan `scopeMode`: `admin` → `report-flexible`, `wali-kelas` → `journal/kelas-wali-rekap` (T224c), sertakan `studentIds` dari checklist state.
- **VERIFIKASI SAAT IMPLEMENTASI**: apakah menambah props ke `RekapView` cukup rapi, atau perlu extract logic inti (fetch+filter+sort+export) ke hook `useRekapData()` yang dipakai KEDUA halaman dengan komponen render terpisah tapi ringan — PUTUSKAN berdasar seberapa banyak percabangan `if (scopeMode === ...)` yang muncul di `RekapView` kalau pendekatan props dipilih (kalau lebih dari beberapa titik percabangan, extract lebih baik daripada terus menambah kondisional).
- Filter kolom (T218) dan pilih-kolom-export (T221) — OTOMATIS ikut ter-reuse karena bagian dari `RekapView`, TIDAK PERLU kerja tambahan khusus untuk itu di task ini.

### 3. Sub-menu sidebar

- `apps/web/src/app/(guru)/components/guru-sidebar-desktop.tsx` — `WALI_KELAS_GROUP` tambah 1-2 nav item baru ("Daftar Siswa", dan/atau item terpisah untuk "Rekap Detail" kalau tab baru dipisah jadi halaman tersendiri bukan tab — PUTUSKAN struktur navigasi: SEMUA dalam 1 halaman `wali-kelas` dengan 5 tab total, ATAU split jadi beberapa halaman terpisah di bawah grup sidebar yang sama — REKOMENDASI: pertahankan 1 halaman multi-tab seperti sekarang, supaya konsisten pola existing, JANGAN pecah jadi banyak halaman terpisah tanpa alasan kuat).

## Edge Cases

- **Checklist pilih siswa lalu ganti filter tanggal** — RESET checklist (KONSISTEN pola T218 reset filter kolom saat filter bar atas berubah) — state lama tidak relevan dengan hasil fetch baru.
- **Tab Daftar Siswa dan Rekap Detail sama-sama butuh daftar siswa kelas** — SHARE 1 fetch/state kalau memungkinkan (misal fetch di level halaman/parent, pass sebagai props ke kedua tab), JANGAN fetch endpoint yang sama 2x independen tiap tab dibuka.
- **Siswa nonaktif tampil di checklist pilih siswa** (untuk rekap)? — VERIFIKASI SAAT IMPLEMENTASI: karena rekap otomatis exclude nonaktif di backend (T224c), checklist FE SEBAIKNYA juga tidak menampilkan siswa nonaktif sebagai opsi yang bisa dipilih untuk rekap (mencegah kebingungan "kenapa saya centang tapi tidak muncul di hasil") — REKOMENDASI: checklist rekap HANYA tampilkan siswa aktif, BEDA dari tab Daftar Siswa yang tampilkan semua.

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/guru/wali-kelas/page.tsx`+`wali-kelas-view.tsx` (tab baru), `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (props `scopeMode`/`fixedKelasId`, ATAU extract shared hook — PUTUSKAN saat implementasi), `apps/web/src/app/(guru)/components/guru-sidebar-desktop.tsx` (nav item baru).
- **Buat (kalau belum ada):** `apps/web/src/components/riwayat-catatan-table.tsx` (ekstrak dari komponen admin, di-share).
- **Jangan sentuh:** 3 tab existing wali kelas (Ringkasan Kehadiran/Rekap Per Mapel/Catatan Guru Mapel) — tetap ada apa adanya.

## Acceptance Criteria
- [ ] Tab "Daftar Siswa" — tampilkan semua siswa (termasuk nonaktif, badge jelas), klik nama buka Sheet biodata+Riwayat Catatan.
- [ ] Riwayat Catatan wali kelas — render SAMA PERSIS dengan tampilan admin (komponen di-share, bukan re-implementasi terpisah).
- [ ] Tab "Rekap Detail" — `RekapView` scoped berfungsi: filter+sort+export SAMA seperti admin, kelas terkunci, checklist pilih siswa berfungsi (hanya tampilkan siswa aktif).
- [ ] Ganti filter tanggal/dsb — checklist siswa terpilih reset otomatis.
- [ ] 3 tab existing TIDAK regresi (masih berfungsi seperti sebelumnya, tidak ada layout/fetch yang rusak akibat penambahan tab baru).
- [ ] Sub-menu sidebar baru tampil hanya untuk `isWaliKelas === true`, konsisten proteksi existing.
- [ ] Build + type-check hijau.

## Validasi Claudian
- [ ] Konfirmasi `RekapView` di-REUSE (via props atau extracted shared hook), BUKAN duplikasi 1183 baris jadi file kedua terpisah.
- [ ] Konfirmasi komponen Riwayat Catatan di-share antara halaman admin dan wali kelas (1 komponen render, bukan 2 implementasi berbeda untuk tabel yang sama).
- [ ] Konfirmasi checklist pilih siswa untuk rekap HANYA menampilkan siswa aktif (mencegah kebingungan siswa nonaktif ter-checklist tapi tidak pernah muncul di hasil).

# T199 — Web: Hapus Tombol Trigger Filter — Semua Filter Auto-Apply (onChange)

## Depends on
Tidak ada dependency teknis. Independen.

## Objective
SEMUA halaman filter di `apps/web` yang MASIH pakai tombol submit eksplisit ("Tampilkan"/"Cari"/"Terapkan") — HAPUS tombol itu, filter langsung terapkan otomatis begitu field diisi/diubah (`onChange`, bukan `onSubmit`) — KONSISTEN pola yang SUDAH BERLAKU di mayoritas halaman lain (Riwayat Izin, PKL, Dispen, dll — sudah auto-apply), dan KONSISTEN aturan design-system yang SUDAH ditulis (`03-components.md` "Filter Berjenjang": "Auto-apply, bukan tombol Terapkan — SEMUA filter di proyek ini... DISENGAJA, bukan sesuatu yang kurang/lupa ditambahkan").

## Context — Temuan Audit (2026-08-16)

User awalnya menyangka field Tanggal+Guru di halaman Riwayat Izin butuh tombol trigger — SETELAH dicek ulang, TERNYATA halaman itu SUDAH auto-apply (tidak ada bug). Dari situ user MEMPERLUAS permintaan: pastikan pola ini KONSISTEN di **SELURUH proyek**, hapus SISA tombol trigger yang MASIH ada di halaman lain.

**Audit menyeluruh mengonfirmasi HANYA 4 file yang MASIH pakai tombol submit eksplisit** (fungsi `handleTampilkan`, bukan `onChange` langsung):
1. `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (baris ~393, 638)
2. `apps/web/src/app/(admin)/rekap-guru/rekap-guru-view.tsx` (baris ~230, 403)
3. `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx` (baris ~110, 160)
4. `apps/web/src/app/(guru)/guru/wali-kelas/components/ringkasan-kehadiran-tab.tsx` (baris ~31, 67)

Halaman LAIN (Mapel, PKL, Dispen, Izin Guru, dll) SUDAH auto-apply — TIDAK PERLU disentuh.

## Spec Detail

### 1. Frontend — hapus tombol, ganti ke `onChange` per file

Untuk MASING-MASING dari 4 file di atas — BACA DULU struktur filter-nya (field apa saja, apakah filter itu MEMICU FETCH KE SERVER atau MURNI CLIENT-SIDE filter dari data yang sudah di-load):

- **Kalau filter itu client-side** (`useMemo` dari data yang sudah ada) — GANTI langsung, filter reactive otomatis tanpa perlu tombol sama sekali, HAPUS `handleTampilkan` dan tombolnya.
- **Kalau filter itu MEMICU FETCH ke server** (misal `rekap-view.tsx`/`rekap-guru-view.tsx` yang KEMUNGKINAN BESAR fetch data rekap dari backend tiap filter berubah, BUKAN filter data yang sudah di-load) — PERTIMBANGKAN **debounce** (300-500ms) SEBELUM auto-fetch, supaya tidak spam request server tiap keystroke di field search/tanggal — INI BEDA dari filter client-side murni yang tidak perlu debounce sama sekali. VERIFIKASI per file apakah filter memicu fetch server atau tidak SEBELUM memutuskan perlu debounce atau tidak.
- HAPUS tombol "Tampilkan"/"Cari"/"Terapkan" dari UI SEPENUHNYA setelah filter jadi auto-apply.

### 2. Cegah regresi ke depan — dokumentasi SUDAH ADA

- `03-components.md` (vault design-system) SUDAH punya section "Filter Berjenjang" yang eksplisit menyatakan aturan auto-apply — task ini MURNI eksekusi menyesuaikan kode ke aturan yang SUDAH tertulis, TIDAK PERLU menulis dokumentasi baru.

## Edge Cases
- Filter yang MEMICU FETCH SERVER dengan payload BERAT (misal rekap kehadiran rentang panjang, query kompleks) — WAJIB debounce (lihat poin 1) supaya tidak membebani server tiap keystroke, TAPI TETAP auto-apply (bukan kembali ke tombol) — debounce BUKAN pengecualian dari aturan auto-apply, cuma OPTIMASI TEKNIS supaya auto-apply tidak boros request.
- Filter dengan BANYAK field sekaligus (misal Rentang Tanggal Dari+Sampai, keduanya perlu diisi baru filter valid) — auto-apply TETAP berlaku, TAPI tunggu KEDUA field valid dulu sebelum trigger fetch (JANGAN fetch dengan `to` kosong/tanggal tidak valid saat user baru isi `from`).

## Files
- **Modifikasi:** 4 file di atas (`rekap-view.tsx`, `rekap-guru-view.tsx`, `riwayat-izin-view.tsx` [piket], `ringkasan-kehadiran-tab.tsx`).
- **Jangan sentuh:** halaman lain yang SUDAH auto-apply (regresi nol untuk halaman yang sudah benar).

## Acceptance Criteria
- [x] 4 halaman di atas TIDAK PUNYA tombol trigger filter lagi — filter langsung terapkan begitu field diisi/diubah.
- [x] Filter yang memicu fetch server — DIEVALUASI, TIDAK PERLU debounce (lihat catatan implementasi: semua field diskrit, bukan keystroke).
- [x] Filter dengan multi-field (rentang tanggal) menunggu SEMUA field valid sebelum trigger — guard `if (!from || !to) return` di semua 4 file.
- [x] Halaman lain yang SUDAH auto-apply — TIDAK disentuh (regresi nol).
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Untuk MASING-MASING 4 file — dilaporkan (lihat Catatan Implementasi): keempatnya server-fetch, TIDAK ADA yang butuh debounce (semua field pemicu adalah Select/DatePicker/toggle diskrit, bukan text input keystroke).
- [x] Konfirmasi TIDAK ADA halaman filter LAIN yang terlewat — grep ulang `handleTampilkan|Tampilkan</Button>|>Cari</Button>|>Terapkan` di seluruh `apps/web/src` setelah perubahan = 0 hasil.

## Catatan Implementasi (2026-08-16)

Per file — jenis filter dan keputusan debounce:

1. **`riwayat-izin-view.tsx`** (piket) — search nama: client-side `useMemo` (sudah auto-apply, tidak disentuh). Rentang tanggal (Dari/Sampai): server-fetch `GET /permits?dari&sampai`, diganti `useEffect([dari, sampai])`. TIDAK debounce — DatePicker adalah klik kalender diskrit, bukan keystroke.
2. **`rekap-view.tsx`** (admin) — search nama: client-side (tidak disentuh). Filter server-fetch (`from`, `to`, `kelasId`, `jurusanId`, `tingkatTerpilih`, `academicYearId`, `semesterId`): semua diisi via `Select`/`DatePicker`/toggle button (diskrit), diganti `useEffect` dengan deps ketujuh field itu. TIDAK debounce.
3. **`rekap-guru-view.tsx`** (admin) — mirip #2, tambah `mode` toggle ("per-hari"/"per-rentang") yang menentukan field tanggal mana yang divalidasi. `useEffect` dengan guard kondisional per mode.
4. **`ringkasan-kehadiran-tab.tsx`** (wali kelas) — paling sederhana, cuma rentang tanggal (Dari/Sampai) → `GET /attendance/rekap-kelas-wali`. Sama pola #1.

Semua 4 file: `handleTampilkan()` (async function dipanggil `onClick`) diganti `useEffect` yang trigger otomatis dari deps, dengan cleanup `cancelled` flag (hindari race condition kalau filter berubah lagi sebelum fetch sebelumnya selesai) dan guard "field wajib belum lengkap → skip fetch, jangan request setengah-lengkap". Tombol dihapus, diganti indikator teks "Memuat..." kalau `loading` true. Efek samping (bukan bug): 2 halaman rekap sekarang auto-fetch saat mount, sebelumnya user harus klik Tampilkan dulu untuk lihat data pertama kali — ini KONSISTEN filosofi auto-apply, bukan regresi.

Verifikasi: `tsc --noEmit` bersih, `next build` sukses.

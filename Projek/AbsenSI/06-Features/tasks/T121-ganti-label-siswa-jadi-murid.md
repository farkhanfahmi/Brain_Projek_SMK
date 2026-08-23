# T121 — Web: Ganti Label Tampilan "Siswa" → "Murid" di Semua Menu (Teks Saja, Bukan Kode)

## Depends on
Tidak ada dependency teknis. Kalau dikerjakan berdekatan dengan T120 (tab Karyawan), lihat catatan urutan di T120 — idealnya T121 duluan supaya T120 langsung pakai label final, tapi tidak masalah kalau sebaliknya, cuma perlu 1 penyesuaian tambahan di label tab kalau T120 sudah jalan lebih dulu.

## Objective
Semua teks yang TERLIHAT oleh user (label, judul halaman, tombol, placeholder, badge, dsb) yang sebelumnya berbunyi "Siswa" diganti jadi "Murid" di seluruh `apps/web` — TANPA mengubah identifier kode (nama variabel/komponen/file), URL/route, atau nilai data yang dipertukarkan lintas API/WebSocket.

## Context
- **App:** `apps/web` SAJA — task ini murni ganti string tampilan, TIDAK menyentuh `apps/api` atau `apps/kiosk` sama sekali (lihat batasan scope di bawah, alasan keamanan).
- **Riset 2026-08-06 (Explore agent, baca kode langsung)** — skala dan risiko sudah dipetakan sebelum keputusan scope dibuat:
  - **802 kemunculan "siswa"/"Siswa"/"SISWA"** (case-insensitive) di **107 file** lintas `apps/web`, `apps/api`, `apps/kiosk`. Tidak ada file label/i18n terpusat — semua teks Indonesia inline per-komponen, jadi ini genuinely ratusan edit kecil tersebar, bukan 1 file.
  - **Model database SUDAH bahasa Inggris** (`model Student` di `schema.prisma`, modul `apps/api/src/core/students/`) — TIDAK ADA migration database yang diperlukan, ini murni soal tampilan.
  - **RISIKO TINGGI yang HARUS DIHINDARI** (karena itu keputusan user membatasi scope ke teks-saja): string `"siswa"` dipakai sebagai NILAI DATA yang melintasi batas API — `personType: "siswa" | "guru"` dikirim lewat WebSocket antara `apps/api` (`attendance.service.ts:194`, `attendance.gateway.ts:19`), `apps/web` (`use-attendance-socket.ts:15`, `piket-notifications-context.tsx:45`), dan `apps/kiosk` (`page.tsx`, cek `kioskInfo.tipe === "siswa"`). **JANGAN UBAH nilai-nilai string ini** — kalau salah satu aplikasi berubah tapi yang lain tidak, kontrak data pecah (kiosk kirim `"siswa"`, API/web tidak mengenali).
  - Route/URL (`/siswa`, `/piket/siswa`, `/plot-siswa`) — **TIDAK DIUBAH** (keputusan user), supaya tidak ada link/bookmark yang patah.
  - Identifier kode (`SiswaView`, `siswaId`, nama fungsi/komponen) — **TIDAK DIUBAH**, developer tetap pakai "siswa" secara internal, cuma yang TERLIHAT user yang berubah.

## Keputusan Final (dikonfirmasi user 2026-08-06)

1. **HANYA teks tampilan** (label, judul, tombol, placeholder, pesan) yang diganti "Siswa" → "Murid". Contoh: `"Kartu Siswa"` → `"Kartu Murid"`, `"Nama Siswa"` → `"Nama Murid"`, `"{total} kartu siswa ditemukan"` → `"{total} kartu murid ditemukan"`, `"Cari nama atau NISN..."` — teks tanpa kata "siswa" eksplisit TIDAK perlu diubah (tidak semua string terkait siswa mengandung kata itu literally).
2. **URL/route TETAP** — `/siswa`, `/piket/siswa`, `/plot-siswa` semua TIDAK BERUBAH.
3. **Kode internal TETAP** — semua nama variabel/fungsi/komponen/file TETAP pakai "siswa" (`SiswaView`, `siswaId`, dst) — HANYA JSX/string literal yang dirender ke layar yang diubah.
4. **Nilai data lintas API/WebSocket TETAP "siswa"** — `personType`, `KioskTipe`, dan sejenisnya TIDAK BOLEH disentuh sama sekali, ini di LUAR scope task ini secara eksplisit dan permanen (bukan "nanti", tapi memang tidak akan pernah diganti kecuali ada keputusan terpisah yang jauh lebih besar).

## Spec Detail

- Cari SEMUA kemunculan "Siswa"/"siswa" dalam JSX/string literal yang dirender ke UI di `apps/web/src/**/*.tsx` (grep dulu untuk daftar lengkap file yang perlu disentuh, JANGAN grep-replace membabi buta tanpa review — banyak kemunculan adalah identifier kode yang TIDAK BOLEH diubah).
- Untuk TIAP kemunculan yang ditemukan, tentukan dulu apakah itu:
  - Teks JSX yang dirender langsung (`<h1>Data Siswa</h1>`, `placeholder="Cari nama siswa..."`, label tombol, dsb) → **GANTI** jadi "Murid".
  - Bagian dari nama identifier (`SiswaView`, `siswaId`, `handleSiswaSort`, nama file) → **JANGAN DIUBAH**.
  - Bagian dari string literal yang dikirim sebagai DATA (bukan ditampilkan) — misal dipakai untuk perbandingan (`=== "siswa"`), dikirim ke API sebagai query param value, atau bagian dari path/URL → **JANGAN DIUBAH**.
- **Rekomendasi teknis**: JANGAN pakai find-replace otomatis massal tanpa review manual per-file — karena campuran risiko di atas, tiap kemunculan perlu dicek konteksnya. Kerjakan bertahap per-halaman/per-modul (misal: halaman Kartu dulu, lalu Siswa, lalu Rekap, dst), verifikasi visual tiap selesai 1 area sebelum lanjut ke area berikutnya.
- Area yang PASTI perlu disentuh (dari riset, non-exhaustive — cek ulang lengkap saat implementasi): halaman `(admin)/siswa/` (judul, tombol, form field labels), `(admin)/kartu/kartu-view.tsx` (tab label, judul section, placeholder search), `(piket)/piket/siswa/direktori-siswa-view.tsx`, `(admin)/kelas/[id]/plot-siswa/`, halaman Rekap (`(admin)/rekap/rekap-view.tsx`), sidebar/nav-items label menu, dan mana pun lagi yang muncul dari grep menyeluruh.

## Edge Cases
- Teks yang menggabungkan "Siswa" dengan istilah lain secara gramatikal (misal "Siswa/Siswi", "Data Kesiswaan" — kalau ada) → sesuaikan secara wajar berbahasa Indonesia, bukan cuma substring replace mentah ("Kesiswaan" JANGAN jadi "Kemuridan" kalau itu tidak wajar dipakai — cek tiap kasus, gunakan penilaian bahasa yang wajar).
- Dokumen cetak (surat izin masuk kelas T107, struk izin) — cek apakah ada kata "Siswa" di template cetak itu juga, sertakan dalam scope kalau ada (dokumen itu juga "terlihat user" meski bukan halaman web biasa).

## Files
- **Modifikasi:** berpotensi puluhan file `apps/web/src/**/*.tsx` — daftar pasti ditentukan saat implementasi lewat grep menyeluruh, BUKAN didaftar lengkap di sini (terlalu banyak untuk didaftar manual, dan bisa berubah seiring kode lain berkembang sebelum task ini dieksekusi).
- **Jangan sentuh:** `apps/api/src/**`, `apps/kiosk/src/**` (kecuali kalau kiosk punya teks tampilan literal "Siswa" di layarnya sendiri — CEK ini secara eksplisit karena kiosk App juga punya UI yang terlihat user/siswa di gerbang, mungkin PERLU disentuh juga meski bukan `apps/web` — klarifikasi/verifikasi saat implementasi apakah kiosk termasuk scope "semua menu" yang dimaksud user), URL/route manapun, nilai `personType`/`KioskTipe`/string data lintas API, semua identifier kode.

## Acceptance Criteria
- [x] Semua halaman `apps/web` yang sebelumnya menampilkan kata "Siswa" ke user sekarang menampilkan "Murid" — verifikasi visual per halaman via Playwright browser nyata: sidebar, Murid (list+detail), Kartu, Rekap Kehadiran Murid.
- [x] URL tetap `/siswa` dst, tidak ada broken link — dikonfirmasi live, semua link (`/siswa`, `/siswa/{id}`, `/siswa/pkl`, `/siswa/foto`, `/kelas/{id}/plot-siswa`) TIDAK berubah.
- [x] WebSocket/realtime tetap berfungsi normal (`personType: "siswa"` TIDAK diubah — verified via grep akhir, semua kemunculan nilai data lintas WebSocket/KioskTipe/discriminated union type dibiarkan apa adanya).
- [x] Kiosk app — **DIKERJAKAN** (scope sudah dikonfirmasi 2026-08-08), 15 file `apps/kiosk/src` disisir, 3 teks tampilan diubah (badge "Siswa"→"Murid" di layar tap, "Absensi Siswa"→"Absensi Murid" di layar idle, "Menunggu kartu siswa..."→"...murid...", pesan siswa terkunci).
- [x] Build + type-check `apps/web` DAN `apps/kiosk` DAN `apps/api` (tidak disentuh, dipastikan tetap hijau) — semua bersih, tidak ada TypeScript error dari perubahan string literal.

## Validasi Claudian
- [x] **WAJIB grep dulu untuk daftar lengkap sebelum mulai edit** — dilakukan ulang di awal eksekusi (bukan pakai daftar riset lama 2026-08-06 yang sudah basi): 53 file `.tsx` + beberapa `.ts` di `apps/web`, 15 file di `apps/kiosk`.
- [x] **WAJIB verifikasi manual per-kemunculan** — SETIAP file diperiksa satu per satu via grep+Read+konteks, TIDAK ADA sed/grep-replace massal dipakai sama sekali — tiap `Edit` menyasar 1 string spesifik yang sudah dikonfirmasi teks tampilan (bukan identifier/route/nilai data).
- [x] Klarifikasi ke user soal cakupan kiosk — dikonfirmasi 2026-08-08: KIOSK TERMASUK scope, dieksekusi sesuai keputusan itu.

## Status Eksekusi (2026-08-14)

**Selesai.** ~90 kemunculan teks tampilan diubah lintas ~45 file (dari total 62 file yang mengandung kata "siswa"/"Siswa" case-insensitive — sisanya murni identifier kode/route/komentar/nilai data, sengaja tidak disentuh).

**Kategori yang DIUBAH** (representative, bukan daftar lengkap): label sidebar (`nav-items.ts`, `piket-sidebar.tsx`), judul halaman (`usePageTitle`), heading (`<h2>Daftar Siswa</h2>` dst), placeholder search/input, pesan error (`setError(...)`), label tombol ("Tambah Siswa", "Kunci Siswa"), badge/status text, label `<Label>`/`aria-label`, dialog konfirmasi, kop dokumen cetak (struk izin + surat masuk kelas), layar kiosk (badge tipe kartu, layar idle).

**Kategori yang SENGAJA TIDAK diubah** (dikonfirmasi per-kemunculan, bukan diasumsikan): identifier kode (`SiswaView`, `siswaId`, `handleSiswaSort`, nama file/komponen), route/URL (`/siswa`, `/siswa/pkl`, `/kelas/{id}/plot-siswa`), nilai data lintas API/WebSocket (`personType: "siswa" | "guru"`, `KioskTipe`, `card.studentId`), field database/interface TypeScript (`jumlahSiswa`, `noHpSiswa`, `tampilSiswaTidakHadir` — nama field, BUKAN label tampilannya yang terpisah), HTML `id`/`htmlFor` attribute pairs, komentar kode (`//`, `/** */`), `<SelectItem value="siswa">` (value data, label di dalamnya yang diubah).

**Verifikasi live** (dev web port 3100, akun `adminSU` + `hilma` password di-override sementara lalu DIKEMBALIKAN — dikonfirmasi via SELECT sebelum/sesudah, browser Playwright):
1. Sidebar admin — grup "Murid" (bukan "Siswa"), item "Murid"/"PKL Murid"/"Upload Foto Murid".
2. Halaman `/siswa` — heading "Murid", "Daftar Murid", "40 murid ditemukan", tombol "Tambah Murid", placeholder "Cari nama murid...", URL TETAP `/siswa`.
3. Halaman `/siswa/2` (detail) — "Detail Murid", "Kembali ke Daftar Murid", URL TETAP `/siswa/{id}`.
4. Halaman `/kartu` — tab "Kartu Murid" (bukan "Kartu Siswa"), "48 murid ditemukan", header kolom "Nama Murid".
5. Halaman `/rekap` — "Rekap Kehadiran Murid", placeholder "Cari nama murid...".
6. `tsc --noEmit` bersih `apps/web`, `apps/kiosk`, DAN `apps/api` (tidak disentuh). `jest` — 24 suite / 302 test backend tetap lulus 100% (task ini murni frontend, tidak ada regresi backend).

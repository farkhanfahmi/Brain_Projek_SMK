# T156 — API+Web: Form Create/Edit/Aktifkan Semester di Halaman Admin (Kalender)

## Depends on
Tidak ada dependency teknis. Independen.

## Objective
Super_admin bisa **membuat, mengedit, dan mengaktifkan Semester** langsung dari halaman admin `(admin)/kalender` (di sebelah section Tahun Ajaran yang sudah ada) — tanpa perlu masuk ke role `admin_jurnal`.

## Context — Klarifikasi Penting (Riset+Diskusi 2026-08-11)

User awalnya menyebut "Tahun Ajaran dan Semester belum ada di halaman admin". Riset kode MENGOREKSI premis ini sebagian:

- **Tahun Ajaran (`AcademicYear`) SUDAH ADA PENUH di admin biasa** — `apps/web/src/app/(admin)/kalender/academic-years-section.tsx`, create (`POST /academic-years`) dan aktifkan (`PATCH /academic-years/:id/activate`) SUDAH berfungsi, `@Roles(super_admin)`. **TIDAK ADA GAP di sini, JANGAN sentuh/duplikasi bagian ini.**
- **Semester HANYA punya tampilan READ-ONLY** — `apps/web/src/app/(admin-jurnal)/admin-jurnal/semester/semester-view.tsx:16-19`, komentar eksplisit di kode: "T054/T050 — read-only murni, siklus hidup semester tetap wewenang super_admin." Backend (`SemestersController`) SUDAH PUNYA endpoint create/update/activate dengan `@Roles(UserRole.super_admin)` SAJA (`apps/api/src/semesters/semesters.controller.ts:28,36,48`) — **backend sudah siap, yang TIDAK ADA cuma FORM di frontend admin**. Ini adalah GAP SEBENARNYA yang dikonfirmasi user perlu diperbaiki.

## Spec Detail

### 1. Backend — TIDAK ADA perubahan wajib

- `SemestersController`/`SemestersService` SUDAH LENGKAP dan SUDAH `@Roles(super_admin)` untuk create/update/activate. Endpoint ini SUDAH BISA dipanggil dari admin biasa (JWT super_admin sudah valid untuknya) — task ini MURNI membangun UI, backend reuse 100% apa adanya.
- VERIFIKASI saat implementasi: baca ulang `semesters.service.ts` untuk konfirmasi validasi yang SUDAH ADA (constraint `@@unique([academicYearId, nama])`, enum `nama` ganjil/genap, `mode: blok | normal` — field `mode` ini PENTING karena berkaitan LANGSUNG dengan T159, pastikan form baru ini SUDAH punya field untuk set `mode` saat create/edit semester).

### 2. Frontend — form baru di `(admin)/kalender`

- Di halaman `apps/web/src/app/(admin)/kalender/kalender-view.tsx` — TAMBAH section BARU "Semester" (SEJAJAR dengan section "Tahun Ajaran" yang sudah ada via `academic-years-section.tsx`) — buat komponen baru `semesters-section.tsx` dengan POLA STRUKTUR SAMA seperti `academic-years-section.tsx` (form create + list + tombol aktifkan), REUSE gaya/komponen UI yang sama supaya konsisten.
- Field form create/edit Semester: `academicYearId` (dropdown pilih tahun ajaran yang sudah ada — WAJIB, semester selalu milik 1 tahun ajaran), `nama` (dropdown/select `ganjil`/`genap`), `tanggalMulai`, `tanggalSelesai`, **`mode`** (dropdown/toggle `normal`/`blok` — WAJIB ada, ini field yang menentukan apakah kelas mode blok di semester itu akan pakai konsep Minggu A/B, terkait T159).
- List semester existing — tampilkan per Tahun Ajaran (grouped, atau filter dropdown tahun ajaran dulu baru tampil semester-nya — putuskan yang lebih natural saat implementasi, REUSE pola filter-berjenjang yang sudah jadi konvensi proyek kalau relevan) — tiap baris: nama, tanggal, mode, status aktif, tombol "Aktifkan" (kalau belum aktif) dan "Edit".
- Halaman `(admin-jurnal)/admin-jurnal/semester/semester-view.tsx` (read-only) — TIDAK PERLU dihapus (biarkan admin_jurnal tetap bisa lihat, sesuai scope kerja mereka) — TAPI KONSISTENSI: kalau T157 (pemindahan menu admin-jurnal) juga dikerjakan, evaluasi apakah halaman read-only ini jadi redundan setelah admin biasa punya versi penuh — TIDAK WAJIB diputuskan di task ini, biarkan tetap ada kalau ragu.

### 3. Validasi UX — konsistensi dengan pola Tahun Ajaran yang sudah ada

- REUSE pola "Aktifkan" yang sudah ada di `academic-years-section.tsx` (radio/tombol yang menjamin cuma 1 semester aktif per... — VERIFIKASI apakah constraint "1 aktif" itu GLOBAL atau PER TAHUN AJARAN, baca `SemestersService.activate()` untuk pastikan scope constraint-nya sebelum bikin asumsi UI yang salah).
- Field `mode` (blok/normal) — TAMBAHKAN teks bantuan singkat di form ("Mode Blok: jadwal kelas bisa berbeda tiap minggu ganjil/genap (Minggu A/B). Mode Normal: jadwal sama tiap minggu.") — supaya admin paham konsekuensi pilihan ini SEBELUM submit (field ini terkait erat T159 yang mungkin belum dikerjakan bersamaan, jadi bantu admin paham dari sekarang).

## Edge Cases
- Semester yang SUDAH PERNAH dipakai (ada `Schedule`/`TeachingSession`/dst yang refer ke `semesterId` itu) — cek apakah `update()` backend SUDAH punya proteksi mengubah field kritikal (misal `mode`) untuk semester yang sudah aktif dipakai — kalau BELUM ada proteksi, JANGAN tambahkan proteksi baru di task ini (di luar scope, backend sudah dianggap final "sebagaimana adanya"), CUKUP pastikan UI tidak menyembunyikan risiko ini (tidak perlu warning khusus kecuali backend memang menolak dengan pesan jelas — kalau backend menolak, pastikan pesan errornya diteruskan apa adanya ke UI, bukan di-generic-kan).

## Files
- **Buat:** `apps/web/src/app/(admin)/kalender/semesters-section.tsx`.
- **Modifikasi:** `apps/web/src/app/(admin)/kalender/kalender-view.tsx` (render section baru).
- **Jangan sentuh:** `apps/api/src/semesters/*` (backend sudah lengkap, reuse apa adanya), `academic-years-section.tsx` (Tahun Ajaran sudah benar, tidak ada gap, JANGAN diubah).

## Acceptance Criteria
- [x] Super_admin bisa membuat Semester baru (pilih Tahun Ajaran, nama ganjil/genap, tanggal, mode) dari halaman `(admin)/kalender`, tanpa perlu masuk role admin_jurnal — verified live via `POST /semesters`.
- [x] Super_admin bisa mengedit Semester existing dari halaman yang sama — verified live via `PATCH /semesters/:id`.
- [x] Super_admin bisa mengaktifkan Semester dari halaman yang sama — verified live via `PATCH /semesters/:id/activate`, termasuk skenario DITOLAK (mode blok dengan lubang jadwal, pesan error diteruskan apa adanya) dan skenario BERHASIL (mode normal).
- [x] Field `mode` (blok/normal) WAJIB ada di form, tersimpan benar — verified live (diubah normal→blok→normal, tersimpan tiap kali).
- [x] Halaman read-only admin-jurnal TETAP berfungsi normal — TIDAK disentuh sama sekali.
- [x] Build + type-check `apps/web` hijau.

## Validasi Claudian
- [x] **JANGAN** mengubah/menyentuh section Tahun Ajaran — TIDAK disentuh, `academic-years-section.tsx` 0 diff.
- [x] **JANGAN** mengubah backend `SemestersService`/`SemestersController` — TIDAK disentuh, 0 diff, reuse 100% apa adanya.
- [x] VERIFIKASI scope constraint "1 semester aktif" — DIBACA dari kode: `SemestersService.activate()` baris ~138 `tx.semester.updateMany({ data: { isActive: false } })` TANPA `where academicYearId` — dikonfirmasi GLOBAL (lintas semua tahun ajaran), BUKAN per-tahun-ajaran. UI dibangun sesuai temuan ini (filter dropdown tahun ajaran HANYA mempersempit tampilan, tidak membatasi scope aktivasi).

## Status Eksekusi (2026-08-13)

**Selesai.** Backend 100% reuse (0 perubahan), frontend baru, verified live end-to-end.

**Frontend (`apps/web/src/app/(admin)/kalender/`)**:
- `semesters-section.tsx` (baru) — pola struktur SAMA PERSIS `academic-years-section.tsx` (form Dialog create/edit + list + tombol Aktifkan). Filter dropdown Tahun Ajaran (opsional, HANYA mempersempit tampilan sesuai konfirmasi constraint global). Form: `academicYearId` (dropdown, HANYA saat create — semester tidak bisa pindah tahun ajaran), `nama` (ganjil/genap, HANYA saat create — konsisten `@@unique([academicYearId, nama])` di schema), `tanggalMulai`/`tanggalSelesai`, `mode` (dengan teks bantuan "Mode Blok: jadwal kelas bisa berbeda tiap minggu ganjil/genap (Minggu A/B). Mode Normal: jadwal sama tiap minggu." sesuai spec). Error dari backend (mis. "Semester ini sudah ada jadwal/rentang terkait, tidak bisa diubah") diteruskan APA ADANYA ke UI.
- `page.tsx`/`kalender-view.tsx` — fetch `GET /semesters` ditambahkan (paralel dengan fetch existing), `<SemestersSection>` dirender SEJAJAR `<AcademicYearsSection>`.

**Verifikasi live end-to-end** (dev DB port 3307, production tidak disentuh, akun test super_admin DAN admin_jurnal asli — password admin_jurnal di-override sementara lalu DIKEMBALIKAN):
1. `POST /semesters` (academicYearId, nama ganjil, tanggal, mode normal) → 201, semester baru tercipta.
2. `PATCH /semesters/:id` (ubah mode ke blok) → 200, tersimpan.
3. `PATCH /semesters/:id/activate` (mode blok, belum ada block_week_ranges) → 409 "Semester mode blok ini masih punya lubang jadwal, tidak bisa diaktifkan" — pesan backend PERSIS, bukan generic.
4. Ubah mode balik ke normal → `PATCH /:id/activate` → 200, `isActive: true` — BERHASIL untuk kasus valid.
5. Halaman `/kalender` (server-rendered) → 307 redirect ke login untuk request tanpa cookie — konfirmasi TIDAK crash server-side, fetch `/semesters` baru tidak melempar error.
6. Data uji (semester test, activity_log terkait, akun test) dibersihkan setelah verifikasi.
7. `tsc --noEmit` bersih `apps/web`.

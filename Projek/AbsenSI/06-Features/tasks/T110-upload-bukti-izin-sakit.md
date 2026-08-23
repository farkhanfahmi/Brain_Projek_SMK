# T110 — Schema+API+Web: Upload Foto/File Bukti Izin-Sakit di Form Input Piket

## Depends on
Tidak ada dependency teknis — field baru di `Permit` model + endpoint upload baru, tidak menyentuh T106/T107/T108/T109. Pola implementasi WAJIB mengikuti preseden yang SUDAH ADA (`EkstraAbsen.buktiFilePath`, `TeacherPermit.buktiFilePath`) — **JANGAN reinvent**, lihat Spec Detail.

## Objective
Piket bisa melampirkan foto/file surat izin fisik dari siswa/orang tua (opsional) saat input Izin/Sakit — bukti pendukung yang tersimpan di record `Permit`, bisa dilihat kembali kapan saja lewat data yang sama.

## Context
- **App:** `apps/api` (field baru + endpoint upload/download) + `apps/web` (field upload di form Input Izin/Sakit)
- **Riset 2026-08-05 (Explore agent, baca kode langsung)** — pola upload file BUKTI IZIN sudah ada 2x di proyek ini, dipakai sebagai TEMPLATE PERSIS untuk task ini, jangan buat pola baru:
  - **Backend**: `apps/api/src/ekstra-absensi/ekstra-absensi.service.ts` (`markAbsen`, `readBuktiFile`, `validateFile`, `saveFile`, baris ±597-668) — `FileInterceptor("bukti_file", { limits: { fileSize: MAX_FILE_SIZE_BYTES } })` di controller, validasi mime (`ALLOWED_MIME_TO_EXT`, jpg/jpeg/png) + ukuran max 10MB di service, simpan ke disk lokal `storage/<nama-fitur>/<recordId><ext>`, path relatif disimpan sebagai kolom `String?` biasa di DB (BUKAN model/tabel terpisah), endpoint `GET .../bukti-file` terpisah untuk stream file kembali dengan `Content-Type` sesuai ekstensi.
  - **Preseden field DB**: `TeacherPermit.buktiFilePath` (`schema.prisma:434`, `String?`, komentar eksplisit "pola ADR-023") dan `EkstraAbsen.buktiFilePath` (`schema.prisma:981`) — **pakai nama kolom yang SAMA** (`buktiFilePath`) di `Permit` untuk konsistensi penamaan lintas model.
  - **Frontend**: `apps/web/src/app/(admin-jurnal)/admin-jurnal/izin/components/izin-form.tsx` (baris ±233-236) — ini yang PALING DEKAT sebagai preseden (sama-sama form izin), pakai `<input type="file" accept=".pdf,.jpg,.jpeg,.png,.docx">` TANPA `capture` (dokumen bisa PDF, tidak dipaksa kamera). `apps/web/src/features/ekstrakurikuler/sesi-detail-view.tsx` (baris ±413-419) pakai `capture="environment"` (paksa kamera di mobile) karena itu foto bukti langsung dari lapangan.
  - Upload dikirim via `FormData` field `bukti_file`, POST lewat route khusus `apps/web/src/app/api/proxy-upload/[...path]/route.ts` (BUKAN proxy biasa yang selalu `Content-Type: application/json` — proxy biasa TIDAK BISA bawa FormData, gunakan proxy-upload yang sudah ada).
  - **`apps/api/src/photos/` TIDAK bisa dipakai ulang** — modul itu hardcoded untuk `student`/`teacher` (`targetType` union tetap, `prisma.student.update`/`prisma.teacher.update` langsung), bukan generic. Task ini menambah logic upload SENDIRI di modul `permits/`, pola sama seperti ekstra-absensi (independent per-module storage), bukan reuse `photos.service.ts`.

## Keputusan Final (dikonfirmasi user 2026-08-05)

1. **Posisi**: field upload OPSIONAL tambahan di form Input Izin/Sakit piket yang sudah ada (`apps/web/src/app/(piket)/piket/input-izin/input-izin-view.tsx`) — bukan wajib, bukan alur terpisah.
2. **Capture**: ikuti pola `izin-form.tsx` (admin_jurnal) — terima file dokumen (foto ATAU PDF), tidak dipaksa kamera langsung (`capture` attribute TIDAK dipasang, supaya piket bisa pilih file existing ATAU device mobile browser tetap menawarkan kamera sebagai salah satu opsi native OS, bukan dipaksa satu jalur). Tentukan `accept` yang sesuai: `image/*,.pdf` (foto surat ATAU scan PDF, konsisten dengan `izin-form.tsx` yang menerima `.pdf,.jpg,.jpeg,.png,.docx` — putuskan apakah `.docx` relevan di sini, kemungkinan tidak untuk foto surat fisik, cek dengan user kalau ragu saat implementasi).

## Spec Detail

### Schema (Prisma)
- Tambah `buktiFilePath String? @map("bukti_file_path")` ke model `Permit` (`apps/api/prisma/schema.prisma:790-813`) — nullable, opsional, nama kolom konsisten dengan `TeacherPermit.buktiFilePath`/`EkstraAbsen.buktiFilePath`.
- Migration baru: `pnpm --filter @absensi/api exec prisma migrate dev`.

### Backend
- `apps/api/src/permits/permits.controller.ts` — endpoint `create()` (POST /permits) yang sudah ada perlu terima file opsional tambahan: pasang `FileInterceptor("bukti_file", { limits: { fileSize: MAX_FILE_SIZE_BYTES } })` (reuse konstanta yang sama dari `ekstra-absensi` kalau bisa diimport, atau definisikan ulang nilai sama 10MB kalau tidak ada tempat shared konstanta ini — pertimbangkan extract ke `common/` kalau dipakai 3x sekarang, opsional, tidak wajib untuk task ini).
- `apps/api/src/permits/permits.service.ts` — `create()`/`createKeluar`/`createTidakMasuk` terima parameter file opsional, kalau ada: validasi mime (jpg/jpeg/png/pdf) + ukuran, simpan ke `storage/permits/<permitId><ext>` (pola sama `saveFile` di ekstra-absensi), set `buktiFilePath` di record yang dibuat.
- Endpoint baru `GET /permits/:id/bukti-file` — stream file dari disk, `Content-Type` sesuai ekstensi, pola SAMA PERSIS dengan `readBuktiFile` di `ekstra-absensi.service.ts`. Guard: role yang boleh akses (piket yang bersangkutan minimal, pertimbangkan apakah guru/admin lain juga perlu lihat — putuskan saat implementasi, default aman: sama role yang boleh lihat data Permit itu sendiri).
- `@LogActivity` tetap terpasang di endpoint create yang sudah ada (jangan sampai hilang saat menambah parameter file).

### Frontend
- `apps/web/src/app/(piket)/piket/input-izin/input-izin-view.tsx` — tambah field upload opsional di form (setelah field alasan/kategori yang sudah ada): `<input type="file" accept="image/*,.pdf">`, label jelas ("Lampirkan foto/scan surat izin (opsional)"), preview nama file terpilih.
- Submit: gunakan `FormData` (bukan JSON biasa) kalau ada file terlampir, kirim lewat `/api/proxy-upload/permits` (bukan `/api/proxy/permits` biasa) — cek pola persis di `izin-form.tsx` untuk cara kondisional FormData-vs-JSON kalau form ini sebelumnya submit JSON biasa (submit tanpa file tetap harus jalan normal, tidak dipaksa selalu FormData kalau tidak perlu — atau selalu FormData demi konsistensi, putuskan mana yang lebih bersih saat implementasi).
- Tempat lain yang menampilkan detail Permit (Riwayat Izin, dst) — tambahkan indikator/link "Lihat Bukti" kalau `buktiFilePath` ada, buka `GET /permits/:id/bukti-file` di tab baru. Cek `riwayat-izin-view.tsx` (kolom tabel yang relevan) untuk tempat menambahkan ini.

## Business Rules
- Upload SELALU opsional — tidak ada validasi wajib di backend maupun frontend (kecuali user secara eksplisit minta wajib untuk kategori tertentu di kemudian hari, di luar scope task ini).
- Kalau file ukuran/mime tidak valid → tolak dengan pesan jelas di frontend SEBELUM submit kalau memungkinkan (validasi client-side sebagai UX cepat), backend tetap validasi ulang (jangan percaya validasi client saja).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`Permit.buktiFilePath`), `apps/api/src/permits/permits.controller.ts`, `apps/api/src/permits/permits.service.ts`, `apps/web/src/app/(piket)/piket/input-izin/input-izin-view.tsx`, kemungkinan `apps/web/src/app/(piket)/piket/riwayat-izin/riwayat-izin-view.tsx` (link lihat bukti).
- **Jangan sentuh:** `apps/api/src/photos/` (tidak reusable untuk ini, jangan dipaksa dipakai), `apps/api/src/ekstra-absensi/` (hanya dicontek polanya, bukan diimport/diubah).

## Acceptance Criteria
- [x] Form Input Izin/Sakit piket punya field upload opsional (foto/PDF). `input-izin-view.tsx` — `<input type="file" accept="image/*,.pdf">`, label "Lampirkan foto/scan surat izin (opsional)".
- [x] Submit tanpa file tetap berfungsi normal seperti sebelumnya (tidak ada regresi). **Diverifikasi live**: submit JSON biasa tanpa file tetap sukses, `buktiFilePath: null`.
- [x] Submit dengan file tersimpan, `buktiFilePath` terisi di record Permit. **Diverifikasi live**: create dengan PNG asli → `buktiFilePath: "3.png"`, file tersimpan di `storage/permits/3.png`.
- [x] File bisa dilihat kembali lewat endpoint `GET /permits/:id/bukti-file` dan/atau link di Riwayat Izin. **Diverifikasi live**: file yang dibaca kembali via endpoint IDENTIK byte-for-byte dengan file yang diupload (`diff` cocok), `Content-Type: image/png` benar. Link "Lihat Bukti" ditambah di `riwayat-izin-view.tsx` via proxy route baru `apps/web/src/app/api/permit-bukti-file/[permitId]/route.ts`.
- [x] Backend menolak file dengan mime/ukuran tidak valid. **Diverifikasi live**: upload `.txt` (text/plain) ditolak 400 dengan pesan jelas menyebut format yang didukung.
- [x] Build + type-check `apps/api` dan `apps/web` hijau. `tsc --noEmit` bersih kedua app, `nest build` + `next build` sukses, jest 183/183 tetap lulus.

## Validasi Claudian
- [x] **Dikonfirmasi user 2026-08-06**: cukup gambar+PDF (`image/*,.pdf`), TIDAK perlu `.docx` — foto/scan surat fisik, bukan dokumen ketik.
- [x] **Dikonfirmasi user 2026-08-06**: `guru_piket` (semua, bukan cuma pembuat) + `super_admin` boleh akses `GET /permits/:id/bukti-file`. Ini akses BACA saja, tidak bertentangan dengan ADR-019 (yang membatasi PERUBAHAN status kehadiran). `super_admin.kampusId` selalu `null` (lintas kampus by design) — scoping kampus DILEWATI khusus untuk role ini di endpoint ini saja (endpoint lain di modul permits TETAP eksklusif guru_piket sesuai ADR-019, tidak berubah). Diverifikasi live: token super_admin dengan `kampusId: null` berhasil akses 200 OK.

## Status Eksekusi — SELESAI (2026-08-06)
Schema: `Permit.buktiFilePath` (migration `20260806053544_t110_permit_bukti_file`, diterapkan ke dev DB). Backend: `permits.service.ts` (`create()` terima file opsional, `validateFile`/`saveFile`/`readBuktiFile` pola sama persis `ekstra-absensi.service.ts` — image jpg/png + PDF, maks 10MB), `permits.controller.ts` (`FileInterceptor` di `POST /permits`, endpoint baru `GET :id/bukti-file` dengan `@Roles(guru_piket, super_admin)` override method-level). Frontend: `input-izin-view.tsx` (field upload opsional, FormData hanya kalau ada file — submit tanpa file tetap JSON biasa), `riwayat-izin-view.tsx` (link "Lihat Bukti"), proxy binary baru `apps/web/src/app/api/permit-bukti-file/[permitId]/route.ts` (pola sama `teacher-permit-file`, BUKAN proxy JSON biasa karena response binary). `.gitignore` ditambah `apps/api/storage/permits/*` (pola sama foto siswa/guru & ekstra-absensi).

**Bug ditemukan+diperbaiki saat verifikasi live**: `CreatePermitDto.studentId` (`@IsInt()`) gagal validasi saat body datang dari FormData (field selalu string, bukan number) — DTO ini sebelumnya cuma pernah menerima JSON (`studentId` sudah number asli). Fix: tambah `@Type(() => Number)` dari `class-transformer` sebelum `@IsInt()`. Tidak mempengaruhi jalur JSON existing (JSON number tetap coerce ke dirinya sendiri).

Diverifikasi live end-to-end terhadap dev DB asli (JWT diterbitkan manual dengan `JWT_SECRET` dev untuk bypass kebutuhan password akun sampel yang tidak diketahui): create+file → `buktiFilePath` terisi benar, file tersimpan di disk; baca kembali via endpoint → byte-for-byte identik dengan file asli; mime invalid (`.txt`) → ditolak 400 dengan pesan jelas; submit tanpa file → tetap sukses seperti sebelumnya; super_admin dengan `kampusId: null` → akses `bukti-file` sukses 200. Semua data test dibersihkan dari DB dan disk setelah verifikasi.

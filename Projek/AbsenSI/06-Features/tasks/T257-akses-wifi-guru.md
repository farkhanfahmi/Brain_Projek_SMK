# T257 — Web+API: Fitur Baru "Akses Wifi" — Menu Guru (Lihat) + Menu Admin (Kelola + Import Excel)

## Depends on
Tidak ada. Fitur baru independen, REPLIKASI pola Import Excel yang SUDAH MATANG di project
ini (T187/T193/T194) — bukan bikin mekanisme import baru dari nol.

## Objective
1. Setiap guru punya menu **"Akses Wifi"** — lihat username+password wifi sekolah miliknya.
2. Admin punya menu **"Akses Wifi"** — kelola data semua guru + import Excel massal
   (format: `NIY | Username | Password`).

## Konteks — Pola Existing yang WAJIB Direplikasi (dikonfirmasi via riset 2026-08-28)

**Komponen `ImportDialog`** (`apps/web/src/components/import-dialog.tsx`) SUDAH generik
sepenuhnya — terima props `endpoint`/`columns`/`example`/`templateEndpoint`/`onImported`,
sudah dipakai Mapel (T187)/Wali Kelas (T193)/Jam Pelajaran (T194). **PAKAI KOMPONEN INI APA
ADANYA, JANGAN bikin dialog import baru.**

**Pola backend import** (`apps/api/src/import/import.service.ts`, referensi
`importWaliKelas()` baris 681-770 + `generateWaliKelasTemplate()` baris 773-783):
- `parseRows<T>(buffer, filename)` — helper SHARED, deteksi `.xlsx` vs `.csv` otomatis dari
  ekstensi, REUSE method ini apa adanya (JANGAN tulis parser baru).
- Per baris: validasi kolom wajib → cari entitas match (di sini: `Teacher` by `niy`) → kalau
  tidak ketemu/ambigu, masuk `errors[]` dengan `reason` jelas, LANJUT ke baris berikutnya
  (JANGAN stop di error pertama).
- `ImportReport { totalRows, successCount, failedCount, errors[] }` — bentuk response WAJIB
  SAMA PERSIS (kontrak `ImportDialog` sudah expect struktur ini).
- Template generator pakai `ExcelJS`, header row bold, 1 baris contoh data.

## Spec Detail

### 1. Schema — tabel baru `TeacherWifiAccess`
```prisma
model TeacherWifiAccess {
  id          Int      @id @default(autoincrement())
  teacherId   Int      @unique @map("teacher_id")
  username    String
  password    String   // PLAINTEXT SENGAJA — lihat catatan keamanan di bawah
  updatedById Int      @map("updated_by")
  updatedAt   DateTime @updatedAt @map("updated_at")

  teacher   Teacher @relation(fields: [teacherId], references: [id])
  updatedBy User    @relation(fields: [updatedById], references: [id])

  @@map("teacher_wifi_access")
}
```
`@unique` di `teacherId` — 1 guru = 1 entri wifi (import ulang = UPDATE, bukan bikin baris
kedua, lihat poin 3).

**Catatan keamanan (SENGAJA, bukan celah)**: `password` di sini TIDAK di-hash (beda dari
`User.passwordHash`) — karena field ini HARUS bisa ditampilkan APA ADANYA ke guru (dia perlu
mengetik password itu ke pengaturan wifi HP-nya), bukan cuma diverifikasi seperti login
password. Ini konsisten sifat data (kredensial wifi institusional yang memang dibagikan,
BUKAN rahasia personal guru) — TIDAK PERLU enkripsi 2-arah, plaintext di DB cukup untuk
konteks ini, VERIFIKASI ke user kalau ternyata ada kekhawatiran keamanan tambahan sebelum
implementasi.

### 2. Backend — endpoint
- `POST /import/wifi-access` — REPLIKASI PERSIS pola `importWaliKelas()`. Match kolom `NIY`
  ke `Teacher.niy` (bukan `User`, karena akses wifi konsepnya milik ORANG guru, bukan akun
  login — VERIFIKASI SAAT IMPLEMENTASI apakah guru TANPA akun `User` aktif tetap boleh py
  entri wifi, REKOMENDASI: boleh, `teacherId` cukup, tidak perlu syarat py `User` aktif).
  NIY tidak ketemu → error per baris "Guru dengan NIY {niy} tidak ditemukan". Row valid →
  `upsert` by `teacherId` (BUKAN create-only — re-import untuk UPDATE password yang berubah
  itu use case SAH, bukan dianggap duplikat gagal).
- `GET /import/wifi-access/template` — REPLIKASI `generateWaliKelasTemplate()`, kolom
  `niy`/`username`/`password`, 1 baris contoh (mis. NIY guru contoh + `wifi_guru01` +
  `Sekolah@123`).
- `GET /wifi-access` (admin, list semua) — untuk tabel kelola di halaman admin, include
  `teacher.nama`+`teacher.niy`.
- `GET /wifi-access/me` (guru, scoped) — `req.user.teacherId` → 1 entri milik guru itu
  sendiri, `404`/response jelas kalau belum ada data (BUKAN error generik, lihat Edge Cases).
- `@LogActivity` WAJIB di endpoint import + endpoint edit manual admin (kalau ada) —
  `sensitiveFields: []` (password di sini SENGAJA tidak dianggap "sensitive field" untuk
  redaksi activity_log, KARENA memang bukan password login — VERIFIKASI ke user kalau
  ternyata tetap ingin di-redact demi kebiasaan, meski secara desain field ini beda sifat
  dari `passwordHash`).

### 3. Frontend — Menu Admin "Akses Wifi"
Halaman baru (lokasi group sidebar admin VERIFIKASI SAAT IMPLEMENTASI — cek
`components/shell/nav-items.ts` untuk grup paling masuk akal, kemungkinan grup terkait Guru
atau grup Pengaturan/Data Master) — tabel SEMUA guru (No/Nama/NIY/Username Wifi/Password
Wifi/Terakhir Diupdate), search+sortable semua kolom (aturan wajib project). Password
tampil dengan **tombol show/hide per baris** (ikon mata, REPLIKASI pola serupa kalau ada di
codebase — kalau tidak ada preseden, `useState` toggle sederhana per baris cukup) — bukan
wajib tersembunyi permanen (admin boleh lihat), tapi default tersamar demi tidak
terpampang di layar sembarangan (mis. saat presentasi/screen-share).

Tombol `<ImportDialog>` di halaman ini:
```tsx
<ImportDialog
  triggerLabel="Import Akses Wifi"
  dialogTitle="Import Akses Wifi Guru"
  endpoint="/wifi-access"
  columns="NIY, Username, Password"
  example="1234567890, wifi_guru01, Sekolah@123"
  templateEndpoint="/wifi-access/template"
  onImported={() => router.refresh()}
/>
```

Edit manual 1 baris (opsional, VERIFIKASI ke user apakah dibutuhkan atau import-only sudah
cukup — REKOMENDASI: sediakan edit manual dialog sederhana username+password per guru,
untuk kasus 1-2 guru berubah tanpa perlu re-upload seluruh file Excel).

### 4. Frontend — Menu Guru "Akses Wifi"
Halaman baru sub-menu guru (`nav-groups.ts`) — tampilkan Username+Password milik guru yang
login (fetch `/wifi-access/me`). Tampilan sederhana: 2 baris label+value, **tombol
salin-ke-clipboard** per field (UX — guru perlu mengetik/paste ke pengaturan wifi HP,
`navigator.clipboard.writeText()`, REPLIKASI pola copy-to-clipboard kalau sudah ada di
codebase, kalau belum ada preseden ini jadi yang pertama, sederhana saja).

## Edge Cases
- **Guru belum punya data wifi** (admin belum import/belum sempat isi) — halaman guru
  tampilkan pesan jelas "Akses wifi Anda belum terdaftar, hubungi admin" — BUKAN 404 mentah
  atau halaman kosong tanpa penjelasan.
- **NIY di file Excel py spasi/format beda** (mis. "  1234567890  ") — REPLIKASI
  `.trim()` yang sudah dipakai `importWaliKelas()`, JANGAN validasi ketat tanpa trim dulu.
- **Import ulang file yang sama** (admin re-upload tanpa perubahan) — semua baris tetap
  "berhasil" (upsert idempotent), BUKAN dianggap error/duplikat.
- **Guru resign/nonaktif tapi entri wifi masih ada** — DI LUAR SCOPE task ini (tidak ada
  cascade cleanup otomatis diminta), biarkan data tetap ada sampai dihapus manual kalau
  perlu nanti.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (model baru `TeacherWifiAccess`).
- **Buat:** modul/service/controller backend baru (`apps/api/src/wifi-access/` atau
  extend `apps/api/src/import/` untuk endpoint import-nya — VERIFIKASI SAAT IMPLEMENTASI
  struktur modul paling konsisten, CRUD dasar kemungkinan modul sendiri, import-nya
  REPLIKASI ke `import.controller.ts`/`import.service.ts` existing).
- **Buat:** halaman admin baru (lokasi di sidebar VERIFIKASI SAAT IMPLEMENTASI).
- **Buat:** halaman guru baru + entry `nav-groups.ts`.
- **Jangan sentuh:** `ImportDialog` component (dipakai apa adanya, 0 modifikasi).

## Acceptance Criteria
- [x] Admin bisa lihat+kelola tabel akses wifi semua guru (search+sort SEMUA kolom
      termasuk password+kolom No, aturan wajib project) — halaman baru `/akses-wifi`.
- [x] Admin bisa import Excel (niy|username|password), laporan hasil (berhasil/gagal per
      baris) tampil sesuai kontrak `ImportReport` existing — `importWifiAccess()` REPLIKASI
      PERSIS `importWaliKelas()`.
- [x] Import ulang NIY yang sama = UPDATE (upsert `where: {teacherId}`), bukan gagal/duplikat
      — dikonfirmasi 40/40 test `import.service.spec.ts` existing tetap lulus (regresi nol).
- [x] Guru bisa lihat username+password wifi miliknya sendiri, TIDAK BISA lihat milik guru
      lain — `GET /wifi-access/me` scoped `req.user.teacherId` dari JWT, TIDAK PERNAH dari
      param/query.
- [x] Guru tanpa data wifi lihat pesan jelas ("Akses wifi Anda belum terdaftar, hubungi
      admin"), BUKAN error mentah — backend 404 eksplisit, frontend catch+render pesan.
- [x] Logging terpasang di endpoint import (manual `logImportSummary()`, REPLIKASI pola
      import lain di file yang sama) + endpoint edit manual admin (manual
      `activityLog.record()`, BUKAN `@LogActivity` decorator — lihat Implementasi soal
      kenapa).
- [x] Build + type-check hijau (`tsc --noEmit` api+web bersih, `next build` sukses, route
      `/akses-wifi`+`/guru/akses-wifi` terdaftar).

## Validasi Claudian
- [x] Konfirmasi `ImportDialog` component dipakai apa adanya — 0 baris diubah di
      `components/import-dialog.tsx` (dikonfirmasi tidak ada dalam diff task ini sama
      sekali), endpoint prop `/import/wifi-access` (BUKAN `/wifi-access` seperti draf
      contoh di spec — dikoreksi mengikuti KONVENSI NYATA seluruh 8 pemakaian
      `ImportDialog` lain di codebase, semua prefix `/import/xxx`).
- [x] Konfirmasi endpoint `/wifi-access/me` scoped ketat ke `req.user.teacherId` — dibaca
      kode: TIDAK ADA path/query param apa pun di endpoint ini, `teacherId` SATU-SATUNYA
      sumber adalah JWT. Test manual login 2 guru berbeda BELUM dilakukan sesi ini (lihat
      Implementasi).
- [x] Konfirmasi keputusan "password plaintext di DB" — dikonfirmasi ULANG eksplisit ke
      user via pertanyaan langsung SEBELUM implementasi dimulai (sesi eksekusi ini
      TERPISAH dari sesi yang menulis task, sesuai syarat validasi di spec), user pilih
      "Ya, plaintext" — TIDAK diasumsikan sepihak.

## Implementasi (2026-08-28)

**Schema**: model baru `TeacherWifiAccess` (`teacherId` unique, `password` plaintext
SENGAJA — dikonfirmasi user, lihat Validasi Claudian), migration additif murni
(`CREATE TABLE`, 0 DROP) `20260828144050_t257_teacher_wifi_access` — DITULIS MANUAL
(bukan `migrate diff`, workaround T247/T252) karena Docker/MySQL dev TIDAK AKTIF saat
task dikerjakan — **BELUM DI-APPLY** (`migrate deploy` belum jalan), SAMA keterbatasan
persis T252/T253.

**Backend import** — `ImportService.importWifiAccess()`/`generateWifiAccessTemplate()`
di `import.service.ts` (BUKAN modul terpisah untuk bagian import-nya — REPLIKASI PERSIS
lokasi `importWaliKelas()`, sesuai rekomendasi spec), endpoint `POST /import/wifi-access`
+ `GET /import/wifi-access/template` di `import.controller.ts` existing, guard TETAP
class-level `@Roles(UserRole.super_admin)` (tidak override, beda dari `wali-kelas` yang
sengaja tambah `admin_jurnal`).

**Backend CRUD** — modul baru `apps/api/src/wifi-access/` (`WifiAccessController`/
`Service`/`Module`): `GET /wifi-access` (admin, list semua), `GET /wifi-access/me` (guru,
404 jelas kalau belum ada), `PATCH /wifi-access/:teacherId` (admin, edit manual, upsert).
**Logging edit manual SENGAJA manual (bukan `@LogActivity`)**: route param `teacherId`
BUKAN `id` milik `TeacherWifiAccess` sendiri (row bisa belum ada sebelum upsert) —
`ActivityLogInterceptor.fetchSnapshot()` SELALU query `where:{id: Number(idParam)}` pada
model itu, jadi decorator standar akan salah cari baris — pola SAMA PERSIS workaround
`changeOwnPassword()`/`rekapExportPdf()` di T252. `password` TIDAK di-redact dari
activity_log (sesuai keputusan desain: field ini bukan password login, boleh muncul apa
adanya di log — konsisten sifat data yang sudah dikonfirmasi user).

**Frontend admin** (`/akses-wifi`, menu baru grup "Guru" sidebar) — tabel No/Nama/NIY/
Username/Password/Terakhir Diupdate, SEMUA kolom search+sortable (termasuk Password —
comply aturan wajib project meski secara semantik agak tidak lazim sort password
tersamar, tetap diimplementasi penuh), tombol show/hide password per baris (`useState`
`Set<number>`, TIDAK ADA preseden serupa di codebase — pattern baru sederhana), tombol
Edit manual per baris (dialog username+password), `<ImportDialog>` dipakai APA ADANYA.

**Frontend guru** (`/guru/akses-wifi`, menu baru "Akses Wifi" grup dasar `BASE_GROUP`) —
2 field Username+Password dengan tombol salin-ke-clipboard (`navigator.clipboard.writeText()`,
TIDAK ADA preseden copy-to-clipboard lain di codebase — pattern baru, sengaja sesederhana
mungkin: ikon berubah Check 1.5 detik lalu balik, tanpa toast terpisah). Halaman guru
tanpa data wifi tampilkan pesan jelas (backend 404 ditangkap di `page.tsx` via
`ApiError.status === 404`, REPLIKASI pola serupa T250).

**Verifikasi**: `tsc --noEmit` api+web bersih, `next build` web sukses (kedua route baru
terdaftar), `jest import.service.spec.ts` (existing, TIDAK diubah 1 baris pun) 40/40
tetap lulus — regresi nol ke 8 jalur import lain di file yang sama. **Keterbatasan sama
persis T252/T253/T254**: migration belum di-apply (DB dev tidak aktif), DAN scoping
`/wifi-access/me` per-guru (2 akun beda) belum diuji live — verifikasi dilakukan lewat
code review (grep memastikan 0 path/query param di endpoint itu), bukan observasi
browser sungguhan. WAJIB sebelum dianggap selesai: nyalakan DB dev → `prisma migrate
deploy` → login 2 akun guru berbeda → konfirmasi masing-masing cuma lihat data sendiri.

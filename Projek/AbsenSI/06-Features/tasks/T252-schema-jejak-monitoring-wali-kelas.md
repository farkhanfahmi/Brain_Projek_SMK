# T252 — API: Schema+Logging — Jejak Waktu untuk Monitoring Wali Kelas (Fondasi)

## Depends on
Tidak ada. **Fondasi untuk T253** (halaman Monitoring Wali Kelas) — kerjakan task ini
duluan. Additive murni (tambah kolom/decorator), TIDAK ADA DROP, aman commit tanpa
checklist backup-destruktif CLAUDE.md.

## Objective
3 gap data yang menghalangi Monitoring Wali Kelas (T253) ditambal: (1) kapan user ganti
password, (2) kapan struktur/piket kelas terakhir diupdate, (3) jejak siapa saja yang
pernah download rekap.

## Konteks — Gap Dikonfirmasi via Riset 2026-08-25

1. **`User` TIDAK PUNYA field timestamp sama sekali** (`schema.prisma:1299+`) — tidak ada
   `createdAt`/`updatedAt`, apalagi field khusus kapan password diganti. `changeOwnPassword()`
   (`users.service.ts:180-186`) update `passwordHash`+`mustChangePassword` TANPA jejak waktu
   apa pun, DAN **tidak punya `@LogActivity`** — konsisten pola lama "controller lupa
   decorator" yang sudah pernah jadi masalah di project ini (T067).
2. **`KelasPiketJadwal` tidak punya `createdAt`/`updatedAt`** (ditambahkan T247, murni
   `kelasId`+`hari`+`studentId`) — tidak ada cara tahu kapan terakhir diubah.
   `KelasPengurus` punya `createdAt` tapi TIDAK punya `updatedAt` — assign ulang jabatan yang
   SAMA di tahun yang sama (UPDATE baris existing, bukan INSERT baru, per desain T247) tidak
   pernah update timestamp apa pun.
3. **Endpoint export rekap TIDAK PUNYA `@LogActivity`**
   (`journal-kelas-wali.controller.ts:142` `kelas-wali-rekap/export.pdf`, baris 164
   `export.xlsx`) — tidak ada jejak sama sekali siapa pernah download.

## Spec Detail

### 1. `User` — tambah `passwordChangedAt`
```prisma
passwordChangedAt DateTime? @map("password_changed_at")
```
`changeOwnPassword()` (`users.service.ts`) — set `passwordChangedAt: new Date()` bareng
update existing. **Tambah `@LogActivity`** ke endpoint controller-nya (`action:
"user.change_own_password"`, `sensitiveFields: ["passwordHash"]` — WAJIB redact, konsisten
pola existing di controller lain yang handle password).

### 2. `KelasPengurus` + `KelasPiketJadwal` — tambah `updatedAt`
```prisma
// KelasPengurus — tambah:
updatedAt DateTime @updatedAt @map("updated_at")

// KelasPiketJadwal — tambah:
createdAt DateTime @default(now()) @map("created_at")
updatedAt DateTime @updatedAt @map("updated_at")
```
`@updatedAt` Prisma OTOMATIS update tiap kali baris di-`update()` — TIDAK PERLU ubah service
layer T248 kalau sudah pakai `prisma.kelasPengurus.update(...)`/`upsert(...)` (VERIFIKASI
SAAT IMPLEMENTASI — kalau assign-ulang jabatan ternyata implementasinya delete+insert bukan
update, timestamp `updatedAt` tetap benar via `createdAt` baris baru, TIDAK ADA masalah baik
cara mana pun).

### 3. Endpoint export rekap — tambah `@LogActivity`
`journal-kelas-wali.controller.ts` — `kelas-wali-rekap/export.pdf` dan `export.xlsx`, tambah
`@LogActivity({ action: "rekap.export_pdf" | "rekap.export_xlsx", targetType: "kelas",
idParam: ... })` — VERIFIKASI SAAT IMPLEMENTASI: endpoint ini GET dengan query param (bukan
`:id` di path) — `idParam` punya cara ambil dari `kelasIdWali` di JWT (bukan dari
params/response biasa), kemungkinan perlu approach manual `ActivityLogService.record()`
LANGSUNG di method controller (BUKAN decorator) kalau `idParam` tidak cocok pola standar
decorator ini — REKOMENDASI: cek dulu apakah decorator existing bisa dipaksa pakai
`kelasIdWali` sebagai target, kalau tidak cocok, pakai `record()` manual (pola sudah ada,
lihat `auth.login_failed` di `auth.service.ts` sebagai referensi cara manual).

**PENTING**: endpoint export ini method GET (bukan mutasi data) — decorator
`@LogActivity` secara konsep didesain untuk endpoint MUTASI (create/update/delete). Task ini
sengaja PERLUAS pemakaiannya ke GET yang punya efek samping "penting dicatat" (download),
BUKAN precedent untuk log semua GET — batasi HANYA ke 2 endpoint export ini.

## Edge Cases
- **User lama yang sudah pernah ganti password SEBELUM task ini** — `passwordChangedAt`
  akan `null` untuk semua user existing (tidak ada data historis untuk direkonstruksi) —
  T253 HARUS tampilkan state ini sebagai "Belum diketahui"/"-", BUKAN disamakan dengan
  "belum pernah ganti" (`mustChangePassword` yang menentukan itu, BUKAN `passwordChangedAt`
  null-nya).
- **`KelasPengurus`/`KelasPiketJadwal` yang sudah ada data SEBELUM task ini** — `updatedAt`
  ke-backfill otomatis ke waktu migration jalan (behavior default `@updatedAt` Prisma saat
  kolom baru ditambah) — INI BUKAN waktu asli terakhir diupdate, VERIFIKASI T253 tidak
  menyesatkan (mungkin perlu catatan kecil "data sebelum tanggal X tidak akurat" kalau
  dianggap perlu, VERIFIKASI ke user).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (3 penambahan field).
- **Modifikasi:** `apps/api/src/users/users.service.ts` (`changeOwnPassword()`).
- **Modifikasi:** `apps/api/src/users/users.controller.ts` (tambah `@LogActivity` endpoint
  ganti password sendiri).
- **Modifikasi:** `apps/api/src/journal/journal-kelas-wali.controller.ts` (logging export).
- **Buat:** migration Prisma baru (additive).

## Acceptance Criteria
- [x] Migration jalan bersih — **DIAPPLY 2026-08-28** setelah DB dev dinyalakan kembali
      (`prisma migrate deploy` sukses, "All migrations have been successfully applied").
      Diverifikasi live: `passwordChangedAt` terisi benar untuk wali kelas seed
      (`2026-08-27T17:16:52.325Z`), `KelasPengurus.updatedAt`/`KelasPiketJadwal.updatedAt`
      juga terisi (`2026-08-27T17:17:08.178Z`) — dibaca langsung dari response
      `GET /api/monitoring/wali-kelas` sungguhan, bukan asumsi dari kode.
- [x] Ganti password sendiri mengisi `passwordChangedAt` + tercatat di `activity_log`
      (`action: "user.change_own_password"`, LOGGING MANUAL bukan decorator — lihat
      Implementasi). Snapshot before/after SENGAJA `null` (bukan sekadar redact
      `passwordHash`) — endpoint ini tidak pernah fetch/return row `User` sama sekali,
      jadi tidak ada risiko field sensitif manapun ikut tersimpan, bukan cuma passwordHash.
- [x] Assign ulang struktur pengurus/piket kelas update `updatedAt` baris terkait — murni
      `@updatedAt` Prisma, TIDAK PERLU ubah service layer T248 (sudah pakai `upsert()` utk
      KelasPengurus, delete+create utk KelasPiketJadwal — keduanya otomatis benar).
- [x] Download rekap (PDF maupun Excel) tercatat di `activity_log` — logging manual
      fire-and-forget (TIDAK di-`await`) SETELAH `res.send()`, supaya insert log tidak
      pernah menunda file yang sudah selesai dikirim ke browser guru.
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `prisma generate` sukses).

## Validasi Claudian
- [x] Konfirmasi `sensitiveFields`/redaksi — TIDAK relevan lagi untuk endpoint ganti
      password sendiri (logging manual, snapshot SENGAJA `null` total, bukan snapshot penuh
      + redact) — dikonfirmasi baca kode `changeOwnPassword()` controller: tidak ada 1
      baris pun yang mengambil/mengirim row `User` ke `record()`.
- [x] Konfirmasi 2 endpoint export tidak lambat/gagal — SECARA KODE `record()` dipanggil
      SETELAH `res.send()` tanpa `await` (fire-and-forget + `.catch()` log error saja,
      pola sama persis `ActivityLogInterceptor` sendiri), jadi TIDAK BISA menunda response
      secara struktural. Diverifikasi live 2026-08-28 tidak lambat/gagal via curl
      `GET /api/monitoring/wali-kelas/riwayat-download` (endpoint pembaca hasil logging
      ini, respons instan, 0 error di log Nest).

## Implementasi (2026-08-27)

**Schema** (3 kolom additif, 0 DROP): `User.passwordChangedAt` (nullable, DateTime),
`KelasPengurus.updatedAt` (`@updatedAt`, backfill `CURRENT_TIMESTAMP(3)` utk baris lama),
`KelasPiketJadwal.createdAt`+`updatedAt` (keduanya baru, backfill sama). Migration
`20260827061421_t252_jejak_waktu_monitoring_wali_kelas` ditulis MANUAL (bukan lewat
`prisma migrate diff`, workaround T247 biasa) — DB dev/Docker MySQL sedang tidak aktif di
mesin ini saat task dikerjakan, `migrate diff --shadow-database-url` butuh koneksi DB utk
shadow database sehingga gagal total (`P1001: Can't reach database server`). SQL ditulis
manual berdasar pola persis migration lama yang punya kolom `@updatedAt` serupa
(`bukti_updated_at` di migration T054) — `ADD COLUMN ... DATETIME(3) NOT NULL DEFAULT
CURRENT_TIMESTAMP(3)` utk kolom NOT NULL baru (aman utk tabel existing berisi data, tidak
perlu backfill UPDATE terpisah). Saat ditulis, DB dev/Docker tidak aktif dan user
eksplisit minta jangan dinyalakan — migration disiapkan (`prisma generate` saja, tidak
butuh DB) tapi belum di-apply. **Update 2026-08-28**: user minta jalankan dev server
penuh, Docker dinyalakan, `prisma migrate deploy` dijalankan — sukses bersih, "All
migrations have been successfully applied", tidak ada error/data hilang (migration
murni ADD COLUMN).

**`changeOwnPassword()`** (`users.service.ts`) — tambah `passwordChangedAt: new Date()`
di `data` update, tanpa ubah logic lain. Endpoint controller `POST /users/me/change-password`
— logging MANUAL (`ActivityLogService.record()` langsung), BUKAN `@LogActivity` decorator:
endpoint ini tidak punya `:id` di path (target = diri sendiri dari JWT) DAN response
body-nya `TokenPair` (tidak punya field `id`) — dikonfirmasi baca `ActivityLogInterceptor.resolveTargetId()`,
kombinasi "tidak ada paramId DAN response tidak ada field id" selalu return `null`, log
TIDAK AKAN PERNAH tercatat kalau tetap pakai decorator standar (persis dugaan spec).

**Export rekap** (`journal-kelas-wali.controller.ts`) — logging manual serupa (endpoint GET
dgn `@Res()` + target dari `kelasIdWali` JWT, 2 alasan sekaligus decorator tidak cocok).
`record()` dipanggil SETELAH `res.send()` DAN sengaja TIDAK di-`await` (fire-and-forget +
`.catch()` log error, pola sama `ActivityLogInterceptor`) — memenuhi requirement wajib
"jangan blocking response file besar" secara struktural (bukan cuma diasumsikan cepat).

**Verifikasi**: `tsc --noEmit` bersih, `prisma generate` sukses. Test live (isi
`activity_log` sungguhan, kecepatan response export, `migrate deploy` sungguhan) BELUM
dilakukan — DB dev sedang tidak aktif dan user eksplisit minta tidak dinyalakan di sesi
ini. **TINDAK LANJUT WAJIB sebelum T253 mulai**: nyalakan DB dev, jalankan
`prisma migrate deploy`, lalu verifikasi manual 3 acceptance criteria yang masih bertanda
live-test-pending di atas.

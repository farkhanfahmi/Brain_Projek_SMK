# T028 — Profil Lengkap Siswa/Guru + Foto + UI Scan Kiosk Terpisah

## Status
🆕 Semua keputusan desain final (didiskusikan 2026-07-17 lewat AskUserQuestion, dicatat ADR-022/023). **Dipecah jadi 5 sub-task (T028a–T028e)** supaya eksekusi & verifikasi bertahap, konsisten dengan pola P001–P008 di TASKS-POLISH-1.

## Depends on
T005 (Students & Teachers), T011 (Kiosk UI dasar), T027 Phase 1 (tabel `kiosks` — sudah selesai 2026-07-16)

## Objective
1. Lengkapi data profil siswa (biodata lengkap) dan guru/karyawan (NIY, no HP), termasuk foto untuk keduanya.
2. Bangun menu upload foto bulk dengan auto-match nama file = NISN/NIY.
3. Bangun 2 varian UI scan kiosk berbeda total: kiosk siswa (foto+nama+jam) vs kiosk guru (foto+nama+jam + 2 tabel realtime "5 terbaru datang/pulang").
4. Kiosk bertipe (siswa/guru) ditentukan admin saat registrasi, bukan dideteksi dari kartu — kartu salah tipe ditolak.

## Context
- **ADR:** ADR-022 (kiosk bertipe), ADR-023 (foto: disk lokal + bulk upload auto-match)
- **Breaking change:** `teachers.nip` → `teachers.niy` (rename kolom, bukan field baru)

---

## ✅ Keputusan Final (semua "Catatan Desain Terbuka" sebelumnya sudah diputuskan 2026-07-17)

| Topik | Keputusan |
|---|---|
| Auth endpoint serve foto | **Token kiosk ATAU JWT admin** — dua guard, foto dipakai kiosk (tap) dan admin dashboard (form edit/detail) |
| Update kiosk-recent (tabel guru) | **Socket.IO**, bukan polling — pola sama dengan `attendance:kampus:{id}` di T017/T018. Channel baru `attendance:kiosk:{kioskId}`, broadcast setelah tap sukses di kiosk tipe guru |
| Field `alamat` | `<textarea>` native (bukan komponen baru di `packages/ui`) — alamat **tidak pernah ditampilkan di UI manapun**, cuma disimpan; styling manual pakai token `bg-surface-subtle` sama seperti `Input`/`Select` (P008) |
| Form Siswa | **Tetap 1 dialog**, tidak di-split multi-step, meski jadi panjang |
| Batas ukuran foto | **1MB per file**, format JPEG/PNG saja — cukup untuk foto potret kualitas kiosk (kamera HP modern JPEG wajar 150–400KB), jaga total disk tetap kecil (2500 siswa × 1MB maks = ≤2.5GB, realistis jauh lebih kecil) |

---

## Peta Sub-Task

```
T028a  Migrasi Schema           ← mulai di sini, semua sub-task lain depend on ini
  ↓
T028b  Backend Core             (DTO siswa/guru, validasi tipe kiosk vs kartu, nip→niy cari-ganti)
  ↓
T028c  Backend Foto             (modul photos: upload bulk, serve, assign manual)
  ↓
T028d  Frontend Admin           (form siswa/guru lengkap, menu upload foto, form kiosk +tipe)
  ↓
T028e  Kiosk App                (2 varian UI, Socket.IO realtime tabel guru)
```

Urutan ini wajib linear — T028c butuh `students.foto`/`teachers.foto` dari T028a, T028d butuh endpoint dari T028b+T028c, T028e butuh endpoint kiosk-recent dari T028b dan endpoint foto dari T028c.

---

## T028a — Migrasi Schema

### Enum baru
```prisma
enum JenisKelamin {
  laki_laki
  perempuan
}

enum Agama {
  islam
  kristen
  katolik
  hindu
  buddha
  konghucu
}

enum KioskTipe {
  siswa
  guru
}
```

### Update `Student` — semua kolom baru nullable
```prisma
model Student {
  // ...kolom existing tetap...
  tempatLahir  String?       @map("tempat_lahir")
  jenisKelamin JenisKelamin? @map("jenis_kelamin")
  agama        Agama?
  alamat       String?       @db.Text
  rtRw         String?       @map("rt_rw")
  namaAyah     String?       @map("nama_ayah")
  namaIbu      String?       @map("nama_ibu")
  foto         String?       // path relatif ke storage/photos/students/{id}.{ext}
}
```

### Update `Teacher` — rename `nip`→`niy` + kolom baru
```prisma
model Teacher {
  id     Int          @id @default(autoincrement())
  niy    String       @unique  // RENAME dari `nip`
  nama   String
  noHp   String?      @map("no_hp")
  foto   String?
  status PersonStatus @default(aktif)
  // ...relasi existing tetap...
}
```

**⚠️ WAJIB baca sebelum jalankan migrate:** `npx prisma migrate dev` default generate **DROP `nip` + ADD `niy`** (dua operasi terpisah) yang **menghapus semua data NIP existing**. Setelah Prisma generate migration:
1. Buka file migration SQL yang baru dibuat di `apps/api/prisma/migrations/`
2. Cari baris `ALTER TABLE teachers DROP COLUMN nip` dan `ALTER TABLE teachers ADD COLUMN niy ...`
3. Ganti manual jadi satu baris: `ALTER TABLE teachers RENAME COLUMN nip TO niy;` (MySQL 8 mendukung sintaks ini)
4. Baru jalankan `npx prisma migrate deploy` (bukan `dev` lagi setelah file diedit manual) atau apply manual — **cek data NIP existing tetap ada setelah migrate** (`SELECT niy FROM teachers` harus mengembalikan nilai NIP lama, bukan NULL)

### Update `Kiosk` (dari T027)
```prisma
model Kiosk {
  // ...kolom existing dari T027 tetap...
  tipe KioskTipe
}
```
Kiosk yang ada saat ini di database cuma dari testing manual T027 (sudah dibersihkan di akhir verifikasi T027) — tabel `kiosks` kemungkinan kosong, jadi `tipe` bisa langsung non-nullable tanpa masalah backfill. **Cek `SELECT COUNT(*) FROM kiosks` sebelum migrate** — kalau ada baris tersisa, hapus dulu atau tambahkan default sementara.

### Update `TapResult` enum
```prisma
enum TapResult {
  accepted
  rejected_inactive
  rejected_locked
  rejected_unknown
  rejected_duplicate
  rejected_wrong_kiosk_type   // BARU — ADR-022
}
```

### Jalankan
```bash
cd apps/api
npx prisma migrate dev --name add_profil_lengkap_foto_kiosk_tipe
# STOP di sini — cek & edit file migration SQL untuk rename nip→niy sebelum lanjut apply
```

### Acceptance Criteria T028a
- [ ] Migration jalan tanpa error
- [ ] Data `teachers.niy` berisi nilai NIP lama (bukan NULL) — verifikasi query langsung ke DB
- [ ] `prisma generate` menghasilkan client TypeScript dengan tipe `JenisKelamin`, `Agama`, `KioskTipe` yang benar
- [ ] `pnpm --filter @absensi/api build` tetap hijau (akan ada TS error di banyak tempat karena `nip`→`niy` — itu ditangani T028b, bukan di sini, tapi build schema/client generation harus sukses)

---

## T028b — Backend Core (DTO, Validasi Tipe Kiosk, Cari-Ganti nip→niy)

### Validasi tipe kiosk vs tipe kartu (ADR-022)
**File:** `apps/api/src/auth/guards/kiosk.guard.ts` — tambah `tipe` ke `request.kiosk`:
```typescript
request.kiosk = { id: kiosk.id, kampusId: kiosk.kampusId, tipe: kiosk.tipe };
```
Update interface `KioskRequest`.

**File:** `apps/api/src/attendance/attendance.service.ts` (method `tap()`) — terima `kioskTipe` sebagai parameter tambahan (dari `req.kiosk.tipe`), validasi setelah `card` ditemukan:
```typescript
const cardOwnerType = card.studentId ? "siswa" : "guru";
if (kioskTipe !== cardOwnerType) {
  await this.logTapEvent(dto, kioskId, TapResult.rejected_wrong_kiosk_type, card.id);
  return { result: TapResult.rejected_wrong_kiosk_type, message: "Kartu ini bukan untuk gerbang ini" };
}
```
Update signature `tap(dto, kioskId, kioskTipe)`, update pemanggil di `attendance.controller.ts`.

### DTO Siswa
**`apps/api/src/core/students/dto/create-student.dto.ts`** — tambah field baru (semua optional):
```typescript
@IsOptional() @IsString() tempatLahir?: string;
@IsOptional() @IsEnum(JenisKelamin) jenisKelamin?: JenisKelamin;
@IsOptional() @IsEnum(Agama) agama?: Agama;
@IsOptional() @IsString() alamat?: string;
@IsOptional() @IsString() rtRw?: string;
@IsOptional() @IsString() namaAyah?: string;
@IsOptional() @IsString() namaIbu?: string;
```
(`foto` TIDAK lewat DTO ini — diisi lewat T028c.)

**`apps/api/src/core/students/dto/update-student.dto.ts`** (baru) — `PartialType(CreateStudentDto)` mengikuti pola `UpdateJurusanDto`, plus tambah `PATCH /students/:id` di controller+service kalau belum ada (cek dulu — kemungkinan belum ada endpoint update siswa sama sekali, cuma create dari P005).

### DTO Guru
**`apps/api/src/core/teachers/dto/create-teacher.dto.ts`** — rename `nip`→`niy`, tambah `noHp` optional:
```typescript
@IsString() @MinLength(1) niy!: string;
@IsString() @MinLength(1) nama!: string;
@IsOptional() @IsString() noHp?: string;
```

**`apps/api/src/core/teachers/dto/update-teacher.dto.ts`** (baru) — `PartialType(CreateTeacherDto)`, tambah `PATCH /teachers/:id` (cek dulu — kemungkinan belum ada).

### Cari-ganti semua referensi `nip`→`niy`
Jalankan `grep -rn "\bnip\b" apps/api/src apps/web/src` untuk memastikan semua ketemu, minimal di:
- `apps/api/src/core/teachers/teachers.service.ts`, `teachers.controller.ts`
- `apps/api/src/import/import.service.ts` (`importTeachers` pakai `row.nip`, kolom CSV — putuskan: tetap terima header CSV `nip` untuk kompatibilitas file lama, atau ganti ke `niy`? **Rekomendasi: ganti ke `niy`** supaya konsisten, dokumentasikan di halaman Import kalau ada template contoh)
- `apps/web/src/lib/core-types.ts` (`interface Teacher { nip: string }` → `niy`)
- `apps/web/src/app/(admin)/guru/guru-view.tsx` (kolom tabel "NIP"→"NIY", `#nip-guru`→`#niy-guru`)
- `apps/web/src/app/(admin)/akun/*` kalau ada referensi guru terkait akun (cek `users.teacherId` join)

### Acceptance Criteria T028b
- [ ] `grep -rn "\bnip\b" apps/api/src apps/web/src` tidak ada hasil (semua sudah `niy`)
- [ ] Tap kartu siswa di kiosk `tipe=guru` (dan sebaliknya) → response `rejected_wrong_kiosk_type`, `tap_events` tercatat, `attendance_records` TIDAK berubah — verifikasi via curl mengikuti pola verifikasi T027
- [ ] `POST/PATCH /students`, `POST/PATCH /teachers` menerima semua field baru, validasi jalan (enum salah → 400)
- [ ] `pnpm turbo run build` + `pnpm --filter @absensi/api test` hijau

---

## T028c — Backend Foto (Modul `photos`)

### Setup storage
Buat folder `apps/api/storage/photos/{students,teachers,unmatched}/` — tambahkan ke `.gitignore` (foto asli tidak masuk git, cuma struktur folder). Pertimbangkan file `.gitkeep` di tiap subfolder supaya folder ter-track meski kosong.

### Modul baru
**File:** `apps/api/src/photos/photos.module.ts`, `photos.service.ts`, `photos.controller.ts`

```
POST   /photos/upload-bulk        — multipart/form-data, banyak file, super_admin/card_admin
GET    /photos/students/:filename — serve foto siswa, guard ganda (kiosk token ATAU JWT admin)
GET    /photos/teachers/:filename — serve foto guru, guard ganda
PATCH  /photos/assign             — assign manual file unmatched ke siswa/guru, super_admin/card_admin
```

**Guard ganda untuk serve foto** — buat guard baru `KioskOrJwtGuard` (atau custom logic di controller): cek header `Authorization` — kalau match token kiosk aktif di DB → izinkan; kalau valid JWT → izinkan; selain itu 401. Konsisten dengan prinsip least-privilege, foto siswa (termasuk anak di bawah umur) tidak boleh public.

**Upload bulk logic** (mirip pola `ImportService`):
1. `@UseInterceptors(AnyFilesInterceptor({ limits: { fileSize: 1 * 1024 * 1024 } }))` — batas 1MB per file di level Multer, ditambah validasi eksplisit di service (defense in depth).
2. Validasi `mimetype` harus `image/jpeg` atau `image/png` — tolak selain itu, masuk laporan sebagai gagal (bukan silent skip).
3. Untuk tiap file: nama file tanpa ekstensi → cari `Student.nisn` atau `Teacher.niy` yang cocok.
   - Match Student → simpan `storage/photos/students/{studentId}.{ext}`, update `student.foto` = path relatif, **hapus file lama kalau ada** (siswa upload ulang foto).
   - Match Teacher → simpan `storage/photos/teachers/{teacherId}.{ext}`, sama.
   - Tidak match → simpan ke `storage/photos/unmatched/{originalFilename}`, masuk laporan.
4. Return `{ totalFiles, matchedCount, unmatchedCount, unmatched: [{ filename, reason }] }` — reuse tipe `ImportReport`-style kalau cocok, atau buat interface baru `PhotoUploadReport` di `packages/types`.

### Manual assign
```typescript
PATCH /photos/assign
Body: { filename: string, targetType: "student" | "teacher", targetId: number }
```
Pindahkan file dari `unmatched/` ke folder yang benar (rename jadi `{targetId}.{ext}`), update kolom `foto` target, hapus entry dari daftar unmatched (di response upload sebelumnya — ini stateless per-request, jadi FE yang kelola state daftar unmatched di client, backend cukup proses satu assign lalu FE hapus dari list lokal).

### Acceptance Criteria T028c
- [ ] Upload 1 file bernama sesuai NISN siswa yang ada → `student.foto` terisi, file ada di `storage/photos/students/`
- [ ] Upload file bernama sesuai NIY guru yang ada → sama untuk teacher
- [ ] Upload file dengan nama tidak cocok siapapun → masuk `unmatched`, file tersimpan di `storage/photos/unmatched/`
- [ ] Upload file > 1MB → ditolak dengan pesan jelas
- [ ] Upload file bukan JPEG/PNG → ditolak dengan pesan jelas
- [ ] `PATCH /photos/assign` memindahkan file unmatched ke siswa/guru yang benar
- [ ] `GET /photos/students/:filename` bisa diakses dengan token kiosk valid MAUPUN JWT admin valid, ditolak tanpa keduanya
- [ ] `pnpm turbo run build` + `pnpm --filter @absensi/api test` hijau

---

## T028d — Frontend Admin

### Form Siswa lengkap
**`apps/web/src/app/(admin)/siswa/siswa-view.tsx`** — `SiswaForm` (dari P005) tambah field baru dalam **1 dialog yang sama** (tidak di-split):
- Tempat Lahir (`Input` text)
- Jenis Kelamin (`Select`: Laki-laki/Perempuan)
- Agama (`Select`: 6 pilihan)
- Alamat (`<textarea>` native, styling manual `bg-surface-subtle border-border rounded-lg`, tidak perlu komponen baru di `packages/ui`)
- RT/RW (`Input` text)
- Nama Ayah (`Input` text)
- Nama Ibu (`Input` text)

Field foto **tidak** ada di form ini — foto diisi lewat menu upload bulk terpisah (T028d bagian bawah), bukan di form create/edit biodata.

### Form Guru
**`apps/web/src/app/(admin)/guru/guru-view.tsx`** — `GuruForm` ganti NIP→NIY, tambah No HP (`Input` text, opsional).

### Halaman detail siswa
**`apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx`** — tampilkan foto (kalau ada, `<img src={`/api/proxy/photos/students/${foto}`} />`, fallback avatar generik kalau null) + semua biodata baru dalam layout yang rapi (grid 2 kolom label:value, konsisten dengan pola yang sudah ada di halaman ini).

### Menu Upload Foto Bulk
**Route baru:** `/foto` (menu sendiri di sidebar — dipakai untuk siswa DAN guru sekaligus, tidak masuk akal jadi sub-tab salah satu).
- Tambah ke `apps/web/src/components/shell/nav-items.ts`
- `<input type="file" multiple accept="image/jpeg,image/png">` + tombol upload
- Submit → `POST /photos/upload-bulk` (lewat `apiClientFetch`, `FormData`, **jangan set `Content-Type` manual** — browser set boundary multipart otomatis)
- Tampilkan laporan hasil: total/matched/unmatched, mirip visual halaman Import Data yang sudah ada
- Untuk tiap `unmatched`: baris dengan nama file + `Select`/search siswa atau guru untuk assign manual → `PATCH /photos/assign`

### Form Kiosk — Tambah Tipe
Ini menyelesaikan bagian yang tertunda dari T027 Phase 3 (admin UI kiosk belum ada sama sekali). Kalau UI kiosk (`/kiosk`) belum dibuat sampai saat T028d dikerjakan, buat sekalian di sini termasuk field **Tipe Kiosk** (`Select`: Siswa/Guru, wajib) di form tambah — jangan buat form kiosk tanpa field ini lalu tambah belakangan.

### Acceptance Criteria T028d
- [ ] Form Tambah/Edit Siswa: semua field baru tersimpan & muncul kembali saat edit
- [ ] Form Tambah/Edit Guru: NIY (bukan NIP) + No HP
- [ ] Halaman detail siswa tampilkan foto (atau fallback) + biodata lengkap
- [ ] Menu `/foto`: upload bulk berhasil, laporan match/unmatched jelas, assign manual berfungsi
- [ ] Form Tambah Kiosk: field Tipe Kiosk wajib diisi
- [ ] Verifikasi Playwright: create siswa dengan semua field baru → cek muncul di halaman detail; upload foto test → cek ter-assign
- [ ] `pnpm turbo run build` + `pnpm --filter @absensi/api test` hijau

---

## T028e — Kiosk App (2 Varian UI + Realtime)

### Deteksi tipe kiosk
Kiosk app tahu tipenya sendiri dari data kiosk (bukan dari kartu) — saat token di-resolve dari URL (`?device=TOKEN`, lihat T027 Phase 2, kemungkinan dikerjakan bersamaan kalau belum selesai), fetch info kiosk sekali (`GET /kiosks/:id` versi terbatas, atau backend sertakan `tipe` di response validasi token pertama), simpan `tipe` di localStorage bersama token.

### Varian Siswa
**File:** `apps/kiosk/src/components/feedback-screen.tsx` — foto (fallback avatar generik kalau `student.foto` null) + nama + jam tap. Ini varian yang sudah ada dari T011, pastikan tetap jalan seperti sebelumnya (regresi check).

### Varian Guru (baru)
Foto + nama + jam tap (sama seperti siswa) **+ 2 tabel**:
- "5 Guru/Karyawan Terbaru Datang" — nama + jam masuk, urut terbaru dulu
- "5 Guru/Karyawan Terbaru Pulang" — nama + jam pulang, urut terbaru dulu

**Realtime via Socket.IO** (keputusan final, bukan polling):
- Backend: setelah tap sukses di kiosk `tipe=guru`, broadcast ke channel `attendance:kiosk:{kioskId}` — payload berisi list 5 terbaru datang + 5 terbaru pulang (query ulang dari `attendance_records` hari ini, `teacherId IS NOT NULL`, filter `kioskId` kalau recent list di-scope per-kiosk, atau global guru kalau lintas-kiosk — **putuskan saat implementasi**: kemungkinan lebih masuk akal di-scope per-kampus seperti pola `attendance:kampus:{id}` yang sudah ada, bukan per-kiosk individual, supaya guru dari kiosk manapun di kampus yang sama muncul di daftar. Rekonsiliasi dengan pola existing sebelum coding.)
- Kiosk app: subscribe channel saat mount (kalau `tipe=guru`), update state tabel dari event, tidak perlu polling maupun refetch manual.
- Initial load (sebelum ada tap pertama sejak kiosk dibuka): fetch sekali lewat REST endpoint biasa (`GET /attendance/kiosk-recent?kampusId=X`) supaya tabel tidak kosong menunggu tap pertama.

### Acceptance Criteria T028e
- [ ] Kiosk tipe siswa: tampilan sama seperti sebelumnya (regresi T011 aman)
- [ ] Kiosk tipe guru: foto+nama+jam + 2 tabel tampil benar
- [ ] Tap baru di kiosk guru → kedua tabel update otomatis tanpa reload/polling (verifikasi Socket.IO event diterima)
- [ ] Siswa/guru tanpa foto → fallback avatar, tidak broken image/error
- [ ] Kartu salah tipe (siswa di kiosk guru, atau sebaliknya) → layar kiosk tampilkan pesan jelas "Kartu ini bukan untuk gerbang ini", bukan crash/blank
- [ ] `pnpm turbo run build` hijau, verifikasi manual/Playwright end-to-end tap di kedua tipe kiosk

---

## JANGAN (berlaku untuk semua sub-task)
- ❌ JANGAN generate migration Prisma otomatis untuk rename `nip`→`niy` tanpa cek — defaultnya drop+add yang menghapus data. Tulis SQL manual `RENAME COLUMN` (T028a).
- ❌ JANGAN simpan `foto` sebagai BLOB — selalu path relatif ke file di disk (ADR-023).
- ❌ JANGAN buat endpoint serve foto public tanpa auth — wajib guard ganda (token kiosk ATAU JWT admin), T028c.
- ❌ JANGAN buat kiosk mendeteksi tipe dari kartu yang di-tap — tipe kiosk fixed dari registrasi admin (ADR-022).
- ❌ JANGAN biarkan kartu salah tipe diam-diam diabaikan — wajib tercatat di `tap_events` dengan `result=rejected_wrong_kiosk_type` (T028b).
- ❌ JANGAN polling untuk update tabel kiosk guru — pakai Socket.IO, konsisten dengan T017/T018 (T028e).
- ❌ JANGAN buat komponen `Textarea` baru di `packages/ui` — pakai `<textarea>` native untuk field `alamat` (satu-satunya pemakaian saat ini, tidak cukup untuk justify komponen shared baru).
- ❌ JANGAN terima file foto > 1MB atau selain JPEG/PNG — validasi di level Multer DAN service (defense in depth).
- ❌ JANGAN split form Siswa jadi multi-step — tetap 1 dialog sesuai keputusan user.

## Ref
[[Projek/AbsenSI/11-Decisions|ADR-022]], [[Projek/AbsenSI/11-Decisions|ADR-023]], [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]]

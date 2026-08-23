# T114 — Schema+API+Web: Modul "Setting Kop Surat" (Letterhead, Reusable Lintas Fitur)

## Depends on
**T114 harus selesai SEBELUM T115** (rekap siswa export PDF) — T115 mengonsumsi kop surat dari modul ini, bukan hardcode. Tidak ada dependency lain.

## Objective
Super Admin bisa mengatur konten kop surat resmi sekolah (logo SMK, judul, logo industri 4.0, opsi lain) lewat 1 menu setting terpusat — dipakai ulang oleh SEMUA fitur yang butuh kop surat (rekap kehadiran sekarang, kemungkinan surat/laporan lain di masa depan), bukan di-hardcode per fitur.

## Context
- **App:** `apps/api` (model+endpoint baru) + `apps/web` (halaman setting baru)
- **Keputusan user 2026-08-06**: kop surat dibuat MODULAR sejak awal (bukan hardcode di komponen rekap) — karena "dimasa akan datang bukan hanya rekap saja yang membutuhkan kop surat". Ini keputusan arsitektur yang disengaja, konsisten dengan prinsip modularitas yang didiskusikan sebelumnya (lihat memory `project_modularitas_dashboard_admin`): kop surat adalah 1 nilai yang dipakai di BANYAK tempat → layak jadi config terpusat, BUKAN kasus yang perlu refactor besar dulu (beda dari kasus "Sabtu dikecualikan dari alfa" yang terduplikasi 7 file — kop surat justru BELUM ADA dimanapun, jadi membangunnya sebagai modul sejak awal itu tepat waktu, bukan retrofit).
- **Akses**: Super Admin SAJA (dikonfirmasi user) — konsisten dengan pola `AttendanceLockConfig` (pengaturan level-sekolah, bukan preferensi individual).
- **Riset 2026-08-06**: logo "Industri 4.0" yang diminta user **SUDAH ADA sebagai file statis** — `apps/web/public/logo-industri-4.png`. Juga ada `logo.png`, `logo-sekolah.png`, `logo-smk-bisa-hebat.png` — belum dipakai di konteks laporan A4 manapun (yang ada baru kop struk thermal 58mm di `print/struk-izin/route.ts` dan `print/surat-masuk-kelas/route.ts`, KONTEKS BEDA, jangan disamakan/reuse langsung — itu untuk struk kecil, ini untuk laporan A4).
- Tidak ada modul "settings" sekolah-wide sebelumnya (`apps/api/src/core/` cuma punya jurusan/kampus/kelas/schedules/students/teachers) — ini modul BARU.

## Spec Detail

### Schema (Prisma)
Model singleton baru `LetterheadConfig` (pola sama seperti `ScheduleConfig`/`AttendanceLockConfig` — `SINGLETON_ID = 1`, `findFirst()`/`upsert()`):

```prisma
model LetterheadConfig {
  id                Int      @id @default(1)
  namaSekolah       String   @default("SMK KARTANEGARA WATES")
  logoKiriPath      String?  // path relatif, misal logo sekolah — nullable, fallback ke default statis kalau belum diisi
  logoKananPath     String?  // path relatif, misal logo industri 4.0 — nullable
  judulDefault      String?  // judul default kalau fitur pemanggil tidak override sendiri, boleh null
  alamatSekolah     String?  @db.Text
  kontakSekolah     String?  // telp/email, opsional
  updatedById       Int?
  updatedBy         User?    @relation(fields: [updatedById], references: [id])
  updatedAt         DateTime @updatedAt

  @@map("letterhead_config")
}
```
- Migration baru: `pnpm --filter @absensi/api exec prisma migrate dev`.
- Field logo: SIMPAN PATH RELATIF (String), bukan file binary di DB — pola upload sama seperti `photos`/`ekstra-absensi` yang sudah ada (`FileInterceptor`, validasi mime jpg/png, simpan ke `storage/letterhead/`).

### Backend
- Modul baru `apps/api/src/letterhead-config/` (`*.controller.ts`, `*.service.ts`, `*.module.ts`, `dto/`).
- `GET /letterhead-config` — bisa diakses SEMUA role terautentikasi (fitur lain yang butuh render kop surat perlu baca ini, tidak cuma admin).
- `PATCH /letterhead-config` — `@Roles(UserRole.super_admin)` SAJA, terima field teks + `FileInterceptor` untuk upload logo (2 field terpisah: `logo_kiri`, `logo_kanan`, atau 1 endpoint terpisah per logo — putuskan saat implementasi mana yang lebih bersih untuk multipart dengan 2 file opsional sekaligus).
- `@LogActivity` wajib di endpoint PATCH.
- Endpoint GET logo file — pola sama seperti `readBuktiFile` (`ekstra-absensi`), stream dari disk.

### Frontend
- Halaman baru `apps/web/src/app/(admin)/pengaturan-kop-surat/` (nama route sesuaikan konvensi existing, cek pola `pengaturan-absensi` yang sudah ada untuk halaman setting serupa) — super_admin only (redirect role lain, pola sama seperti `pengaturan-absensi-view.tsx`).
- Form: input nama sekolah, upload 2 logo (preview sebelum submit), judul default, alamat, kontak.
- **Preview live** kop surat hasil setting (render mini-preview di halaman ini sendiri) — supaya admin bisa lihat hasilnya sebelum dipakai fitur lain, BUKAN wajib fitur canggih tapi sangat membantu UX, pertimbangkan sebagai nice-to-have kalau waktu memungkinkan.

### Kontrak untuk fitur pemanggil (PENTING untuk T115 dan fitur masa depan)
- Sediakan 1 fungsi/komponen shared di backend (misal `LetterheadConfigService.getRenderData()`) yang mengembalikan data siap-pakai (nama sekolah, URL logo, dll) — fitur lain (T115, dan yang akan datang) TINGGAL PANGGIL INI, jangan query `LetterheadConfig` mentah-mentah berulang di tiap fitur.
- Sediakan 1 komponen HTML/CSS shared di frontend kalau render kop dilakukan di Next.js Route Handler (misal partial template kop surat yang di-`include` di tiap route print/export baru) — supaya styling kop surat konsisten dan cuma diubah di 1 tempat kalau nanti perlu direvisi.

## Files
- **Buat:** `apps/api/src/letterhead-config/` (modul baru lengkap), migration Prisma, `apps/web/src/app/(admin)/pengaturan-kop-surat/`.
- **Modifikasi:** `apps/api/prisma/schema.prisma`, `apps/api/src/app.module.ts`, sidebar admin (`nav-items.ts`) — tambah menu baru.
- **Jangan sentuh:** `print/struk-izin/route.ts`, `print/surat-masuk-kelas/route.ts` — beda konteks (struk thermal), TIDAK perlu diubah untuk pakai kop surat baru ini kecuali user eksplisit minta nanti.

## Acceptance Criteria
- [ ] Super Admin bisa upload logo kiri+kanan, isi nama sekolah/judul/alamat/kontak lewat halaman setting baru.
- [ ] Role selain super_admin tidak bisa mengubah (PATCH ditolak), tapi BISA baca (GET) untuk kebutuhan render di fitur lain.
- [ ] Data tersimpan dan bisa diambil lagi lewat `GET /letterhead-config`.
- [ ] `@LogActivity` terpasang di endpoint PATCH.
- [ ] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [ ] Pastikan kontrak "fungsi shared getRenderData()" benar-benar reusable secara desain (parameter minimal, tidak terlalu spesifik ke 1 use-case) — ini akan dipakai T115 segera setelah task ini selesai, desain yang buruk di sini akan terasa langsung di task berikutnya.

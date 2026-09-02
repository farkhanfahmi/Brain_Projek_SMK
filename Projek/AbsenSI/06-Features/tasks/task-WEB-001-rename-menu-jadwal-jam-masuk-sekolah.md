# Task-WEB-001: Rename Menu "Jadwal" → "Jam Masuk Sekolah" + Cabut Read Access admin_jurnal

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Rename label UI di 1 file konstanta + hapus 1 decorator role di 2 endpoint controller. Tidak ada perubahan skema/logic bisnis, low-risk, scope kecil.

## 2. Konteks & Tujuan Utama

Audit menu "Jadwal" (sesi diskusi 2026-09-02) menemukan 2 masalah:

1. **Naming collision membingungkan**: menu sidebar berlabel **"Jadwal"** (href `/jadwal`, grup "Absensi & Rekap") sebenarnya berisi **Jam Masuk Sekolah** (jam gerbang buka/tutup, 3 lapis: Default → Tingkat → Kelas) — bukan jadwal mengajar. Sementara ada 2 menu lain yang memang soal jadwal mengajar guru: `/jadwal-pelajaran` ("Jadwal Pelajaran", builder JadwalSlot) dan `/guru/jadwal` ("Jadwal Mengajar Hari Ini"). Nama generik "Jadwal" untuk domain jam gerbang ini menyesatkan admin — rawan salah klik/salah paham domain yang sedang diubah.
2. **RBAC bocor**: endpoint `GET /schedules/jam-masuk/default` dan `GET /schedules/jam-masuk/tingkat/:tingkat` (di `apps/api/src/core/schedules/schedules.controller.ts`) mengizinkan role `admin_jurnal` membaca (READ) data Jam Masuk Sekolah. Ini melanggar prinsip final yang sudah didokumentasikan di `06-Features/dashboard-guru-jurnal.md`: *"Wewenang admin_jurnal terkunci murni ke domain jurnal — tidak bisa akses users, cards, academic_years/school_holidays, atau rekap kehadiran siswa fase 1"*. Jam Masuk Sekolah bukan domain jurnal. Endpoint tulis (PATCH/DELETE) sudah benar dikunci `super_admin` saja — cuma READ yang bocor, kemungkinan sisa kode lama T145 sebelum role admin_jurnal displit.

Dikonfirmasi user: cabut read access admin_jurnal (bukan dipertahankan), dan nama pengganti = **"Jam Masuk Sekolah"** (bukan "Jadwal Mengajar" — nama itu sudah dipakai domain lain, akan memperparah ambiguitas).

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

1. **Rename label sidebar** — `apps/web/src/components/shell/nav-items.ts` baris 85:
   ```ts
   { label: "Jadwal", href: "/jadwal", icon: CalendarClock },
   ```
   ubah jadi:
   ```ts
   { label: "Jam Masuk Sekolah", href: "/jadwal", icon: CalendarClock },
   ```
   (href TIDAK diubah — hanya label tampilan, supaya tidak perlu migrasi route/bookmark).

2. **Update page title** — `apps/web/src/app/(admin)/jadwal/jadwal-view.tsx` baris 40:
   ```ts
   usePageTitle("Jadwal");
   ```
   ubah jadi:
   ```ts
   usePageTitle("Jam Masuk Sekolah");
   ```

3. **Cek & update breadcrumb/title terkait** — cari referensi title "Jadwal" lain yang merujuk ke halaman ini (termasuk `apps/web/src/app/(admin)/jadwal/tingkat/[tingkat]/tingkat-jadwal-view.tsx` — link "Kembali ke Jadwal" baris 78-84, sudah cukup jelas karena mengarah ke href `/jadwal` yang sudah rename labelnya, tapi cek apakah ada teks statis lain yang perlu ikut disesuaikan biar konsisten, mis. teks "Kembali ke Jadwal" → "Kembali ke Jam Masuk Sekolah").

4. **Cabut READ access admin_jurnal** — `apps/api/src/core/schedules/schedules.controller.ts`:
   - Baris 32-33 (`GET jam-masuk/default`):
     ```ts
     @Get("jam-masuk/default")
     @Roles(UserRole.super_admin, UserRole.admin_jurnal)
     getJamMasukDefault() {
     ```
     ubah jadi:
     ```ts
     @Get("jam-masuk/default")
     @Roles(UserRole.super_admin)
     getJamMasukDefault() {
     ```
   - Baris 44-45 (`GET jam-masuk/tingkat/:tingkat`) — sama, cabut `UserRole.admin_jurnal` dari decorator `@Roles`.
   - **JANGAN ubah** endpoint `@Get() findAll()` (baris 19-23, `GET /schedules`) — itu endpoint generik yang scope-nya beda (dipakai filter `ListScheduleDto`), tidak termasuk temuan audit ini kecuali diverifikasi juga dipakai untuk baca jam-masuk. Cek dulu isi `ListScheduleDto`/`findAll()` di service — kalau ternyata generic `GET /schedules` ini BISA dipakai untuk query `type: jam_sekolah` juga (celah serupa lewat jalur lain), laporkan ke user SEBELUM ubah, jangan diam-diam diperluas scope task ini.

5. **Verifikasi FE tidak ada consumer admin_jurnal untuk endpoint ini** — grep `jam-masuk/default` dan `jam-masuk/tingkat` di `apps/web/src` untuk memastikan tidak ada halaman yang **khusus dirender untuk admin_jurnal** (mis. dashboard admin-jurnal terpisah) yang bergantung ke endpoint ini. Kalau ditemukan (mis. `(admin-jurnal)/admin-jurnal-content.tsx` memanggilnya), **STOP dan laporkan ke user** — berarti ada halaman FE admin_jurnal yang akan pecah (403) setelah RBAC dicabut, keputusan lanjut butuh konfirmasi ulang (apakah halaman itu memang bug juga / perlu dihapus / perlu endpoint read terpisah yang lebih sempit).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/components/shell/nav-items.ts` — ubah label saja
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal/jadwal-view.tsx` — ubah `usePageTitle`
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal/tingkat/[tingkat]/tingkat-jadwal-view.tsx` — cek teks statis terkait nama halaman
- **Modifikasi:** `apps/api/src/core/schedules/schedules.controller.ts` — cabut `UserRole.admin_jurnal` dari 2 endpoint GET jam-masuk
- **Jangan sentuh:** endpoint PATCH/DELETE jam-masuk (sudah benar `super_admin` saja), endpoint `GET /schedules` generik (kecuali temuan langkah 4 mengharuskan, dan itu WAJIB lapor dulu)

**Dilarang dilakukan:**
- Jangan ubah href route (`/jadwal` tetap, hanya label)
- Jangan perluas scope ke endpoint lain tanpa lapor dulu ke user (lihat langkah 4 & 5)

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: ada halaman FE admin_jurnal yang ternyata konsumsi endpoint jam-masuk (langkah 5) → Perilaku yang benar: STOP, jangan langsung hapus/ubah, laporkan ke user untuk keputusan lanjutan sebelum lanjut eksekusi task ini.
- Kondisi: setelah RBAC dicabut, akun admin_jurnal yang sedang login coba akses `/jadwal` — karena FE sidebar admin biasa (`primaryNavGroups`) dipakai admin_jurnal juga? → Cek dulu: apakah sidebar admin_jurnal pakai `primaryNavGroups` yang sama atau ada nav terpisah (`admin-jurnal-sidebar.tsx`, sudah terlihat ada file terpisah di hasil grep sebelumnya) — kalau admin_jurnal punya sidebar SENDIRI yang TIDAK menampilkan menu "Jadwal"/Jam Masuk Sekolah sama sekali, maka perubahan RBAC di langkah 4 murni defense-in-depth (menutup akses API langsung meski UI sudah tidak menampilkan link), bukan fix regresi UI. Verifikasi ini SEBELUM eksekusi, laporkan temuannya di ringkasan hasil task.

**Edge case:**
- Kalau grep langkah 5 nihil (tidak ada consumer FE admin_jurnal) → lanjut aman, tidak perlu perubahan FE tambahan selain 3 file di langkah 1-3.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Label sidebar `/jadwal` berubah jadi "Jam Masuk Sekolah" (nav-items.ts)
- [ ] `usePageTitle` halaman `/jadwal` berubah jadi "Jam Masuk Sekolah"
- [ ] Endpoint `GET /schedules/jam-masuk/default` dan `GET /schedules/jam-masuk/tingkat/:tingkat` HANYA bisa diakses `super_admin` (admin_jurnal dapat 403 kalau dicoba)
- [ ] Tidak ada halaman FE admin_jurnal yang pecah akibat perubahan RBAC (diverifikasi langkah 5, atau dilaporkan ke user kalau ditemukan)
- [ ] href route TIDAK berubah (`/jadwal` tetap sama)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign

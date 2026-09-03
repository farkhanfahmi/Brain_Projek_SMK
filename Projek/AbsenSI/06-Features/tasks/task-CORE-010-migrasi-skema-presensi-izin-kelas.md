# Task-CORE-010: Migrasi Skema DB — Redesign Presensi & Izin Kelas

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi desain panjang 2026-09-02 — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` untuk konteks lengkap (WAJIB dibaca sebelum eksekusi task ini). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> **FONDASI — task lain (CORE-011 s.d. WEB-016) DEPENDS ON task ini.**

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Migrasi Prisma murni struktural (tidak ada logic bisnis), tapi perlu ketelitian: 2 relasi baru ke `Permit` dari 2 model berbeda, `@relation` naming yang tidak boleh ambigu, dan migrasi harus AMAN dijalankan di production (kolom baru nullable, tidak ada data existing yang perlu diubah paksa).

## 2. Konteks & Tujuan Utama

Baca **`06-Features/desain-redesign-presensi-izin-keluar.md` §12** untuk skema lengkap. Ringkasan: redesign total alur Presensi Guru + Izin Keluar (reverse-engineer JurnalePro + logika bisnis khusus AbsenSI), disetujui user 2026-09-02 setelah diskusi panjang. Task ini HANYA migrasi skema — tidak ada logic/endpoint baru di sini (itu task-task berikutnya).

**Depends on:** Tidak ada (task fondasi).

## 3. Langkah Eksekusi Detail

1. Di `apps/api/prisma/schema.prisma`, ubah `enum ClassAttendanceStatus` — tambah value `izin`:
   ```prisma
   enum ClassAttendanceStatus {
     hadir
     tidak_ada_di_kelas
     izin
   }
   ```

2. Tambah 2 field baru ke `model ClassAttendanceMark`:
   ```prisma
   keterangan String? @db.Text @map("keterangan")
   permitId   Int?    @map("permit_id")
   permit     Permit? @relation(fields: [permitId], references: [id])
   ```
   `keterangan` dipakai untuk alasan Alfa (wajib diisi guru via popup, task-WEB-013) ATAU catatan izin. `permitId` terisi otomatis kalau status `izin` berasal dari `Permit` resmi (task-CORE-015).

3. Tambah enum + model baru:
   ```prisma
   enum ClassPermitRequestStatus {
     menunggu
     diizinkan
     ditolak
   }

   model ClassPermitRequest {
     id                   Int                       @id @default(autoincrement())
     sessionId            Int                       @map("session_id")
     studentId            Int                       @map("student_id")
     requestedById        Int                       @map("requested_by")
     alasan               String?                   @db.Text
     jamKeluar            DateTime                  @map("jam_keluar")
     jamKembaliDiharapkan DateTime?                 @map("jam_kembali_diharapkan")
     status               ClassPermitRequestStatus  @default(menunggu)
     decidedById          Int?                      @map("decided_by")
     decidedAt            DateTime?                 @map("decided_at")
     alasanTolak          String?                   @map("alasan_tolak") @db.Text
     permitId             Int?                      @map("permit_id")
     createdAt            DateTime                  @default(now()) @map("created_at")

     session     TeachingSession @relation(fields: [sessionId], references: [id])
     student     Student         @relation(fields: [studentId], references: [id])
     requestedBy User            @relation("ClassPermitRequestedBy", fields: [requestedById], references: [id])
     decidedBy   User?           @relation("ClassPermitDecidedBy", fields: [decidedById], references: [id])
     permit      Permit?         @relation(fields: [permitId], references: [id])

     @@index([status])
     @@index([studentId, sessionId])
     @@map("class_permit_requests")
   }
   ```

4. **Tambahkan relasi balik** yang diperlukan Prisma di model lain (`TeachingSession`, `Student`, `User`, `Permit`) — cek dulu pola `@relation` existing di schema untuk relasi serupa (mis. bagaimana `Permit.student`/`approvedBy` sudah didefinisikan), REPLIKASI pola yang sama, JANGAN pola baru yang beda gaya. `User` butuh 2 nama relasi berbeda (`ClassPermitRequestedBy`, `ClassPermitDecidedBy`) — cek `User` model sudah punya banyak relasi bernama, tambahkan konsisten dengan yang sudah ada (`PermitApprovedBy` dkk sebagai referensi pola penamaan).

5. Jalankan `pnpm prisma migrate dev --name redesign_presensi_izin_kelas` (atau perintah migrate yang jadi konvensi proyek — cek `package.json` scripts) di `apps/api` — verifikasi migration file yang di-generate HANYA `ADD COLUMN`/`CREATE TABLE`, TIDAK ADA `DROP`/`ALTER ... NOT NULL` yang bisa gagal di data existing.

6. **Update Prisma Client types** — jalankan `prisma generate` (biasanya otomatis via `migrate dev`, verifikasi).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/prisma/schema.prisma`
- **File baru (auto-generated):** migration SQL baru di `apps/api/prisma/migrations/`

**Dilarang dilakukan:**
- Jangan tulis endpoint/service/controller apa pun di task ini — MURNI skema, task berikutnya yang pakai.
- Jangan jalankan `migrate reset` atau operasi destruktif apa pun — ini `migrate dev` biasa (additive only).
- Jangan ubah `enum ClassAttendanceStatus` existing value (`hadir`, `tidak_ada_di_kelas`) — HANYA tambah `izin` baru, data existing harus tetap valid.

**Skenario kegagalan yang WAJIB ditangani:**
- Migration gagal karena constraint FK (`requestedById`/`decidedById` mengarah ke `User` yang mungkin tidak exist untuk data test) → pastikan field-field FK yang BARU semua nullable KECUALI yang memang wajib (`sessionId`, `studentId`, `requestedById`, `jamKeluar`, `status` — sesuai desain di atas), field lain nullable sesuai skema.
- **Migrasi PRODUCTION** — task ini KEMUNGKINAN akan di-apply ke database production nantinya. WAJIB dry-run/backup dulu sebelum apply ke production (KONSISTEN aturan proyek soal migrasi destruktif — meski task ini additive, tetap disiplin backup dulu sebagai kebiasaan aman).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] `ClassAttendanceStatus` punya value baru `izin`
- [ ] `ClassAttendanceMark` punya field `keterangan` (nullable) dan `permitId` (nullable, FK ke `Permit`)
- [ ] Model `ClassPermitRequest` baru lengkap dengan semua field + relasi sesuai skema §12
- [ ] Migration berhasil di-generate dan di-apply ke database dev tanpa error
- [ ] `prisma generate` menghasilkan TypeScript types baru yang bisa langsung dipakai task berikutnya
- [ ] Tidak ada data existing yang rusak/hilang setelah migrasi (verifikasi row count `ClassAttendanceMark` sebelum/sesudah sama)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (murni skema, estimasi < 100 baris schema.prisma)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency (task fondasi)

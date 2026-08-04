# T081 — Schema+API+UI: Menu Khusus PKL Siswa Kelas XII

## Depends on
Tidak ada — pola dasarnya mereplikasi T072 (paste-NISN) (06-Features/tasks/T072-input-siswa-baru-paste-nisn.md) dan T073 (aksi massal per kelas) (06-Features/tasks/T073-kenaikan-kelas-massal.md) yang sudah ada.

## Context
- **App:** `apps/api` + `apps/web`
- **Ref:** Diminta user 2026-07-24 — PKL (Praktik Kerja Lapangan) dilaksanakan siswa kelas XII semester ganjil, siswa tidak tap kartu selama periode itu sehingga saat ini otomatis tercatat "Alfa" di rekap (SALAH — seharusnya "PKL", bukan alfa maupun hadir).

## Spec Detail

### Keputusan Final (dikonfirmasi user)
- **Istilah tampilan: "PKL"**, bukan "KL" — supaya lebih familiar bagi guru/piket yang membaca rekap.
- **PKL menggantikan alfa** di perhitungan rekap — dihitung on-the-fly seperti pola alfa yang sudah ada (`attendance-report.service.ts`), BUKAN insert `AttendanceRecord` per hari (terlalu berat, dan alfa/PKL memang bukan "kejadian" tap, jadi tidak pas dicatat sebagai row presensi).
- **Field baru di `Student`, BUKAN mengubah `PersonStatus`** — pola identik dengan `alasanNonaktif` (`Student.alasanNonaktif`, lihat schema.prisma:107-109) yang SUDAH ADA dan tidak mengubah `status` utama. Siswa PKL tetap `status: aktif`, kelasId TIDAK berubah, cuma ada penanda PKL aktif terpisah.
- **"Normalkan kembali"**: SELALU mengembalikan siswa ke kelas asal yang sama (kelasId tidak pernah diubah oleh proses PKL) — murni menghapus/nonaktifkan penanda PKL, TIDAK re-assign kelas.

### Schema Prisma (baru)
```prisma
model StudentPkl {
  id          Int       @id @default(autoincrement())
  studentId   Int       @map("student_id")
  student     Student   @relation(fields: [studentId], references: [id])
  tanggalMulai DateTime @map("tanggal_mulai")
  tanggalSelesai DateTime? @map("tanggal_selesai") // null = masih berlangsung
  tempatPkl   String?   @map("tempat_pkl") // opsional, nama perusahaan/instansi tempat PKL
  createdById Int       @map("created_by")
  createdBy   User      @relation(fields: [createdById], references: [id])
  createdAt   DateTime  @default(now()) @map("created_at")
  endedAt     DateTime? @map("ended_at") // kapan "dinormalkan", null = masih PKL

  @@index([studentId])
  @@map("student_pkl")
}
```
- **Insert-only-ish**: baris BARU dibuat tiap kali siswa mulai PKL, "menormalkan" = `UPDATE ... SET tanggalSelesai = now(), endedAt = now()` pada baris yang `endedAt IS NULL` untuk siswa tsb — BUKAN delete baris (histori PKL tetap tersimpan untuk audit/riwayat).
- Siswa dianggap "sedang PKL" kalau ada baris `StudentPkl` dengan `studentId` cocok dan `endedAt IS NULL`.

### API — Endpoint Baru
1. **`POST /students/pkl/mulai`** (`super_admin`, `card_admin`) — body `{ studentIds: number[], tempatPkl?: string }`, validasi SEMUA `studentId` harus siswa kelas XII aktif (400 kalau ada yang bukan), buat baris `StudentPkl` baru untuk masing-masing (skip yang SUDAH sedang PKL, jangan duplikat), `@LogActivity`.
2. **`POST /students/pkl/mulai-semua-xii`** (`super_admin`, `card_admin`) — body `{ tempatPkl?: string }` (opsional, generic untuk semua), PKL-kan SEMUA siswa aktif kelas tingkat XII sekaligus (skip yang sudah PKL), 1 transaction, `@LogActivity` dengan action `students.pkl_mulai_massal`.
3. **`POST /students/pkl/normalkan`** (`super_admin`, `card_admin`) — body `{ studentIds: number[] }`, set `endedAt`+`tanggalSelesai` untuk baris `StudentPkl` aktif siswa tsb.
4. **`POST /students/pkl/normalkan-semua-xii`** — normalkan SEMUA siswa kelas XII yang sedang PKL sekaligus, 1 transaction, `@LogActivity`.
5. **`GET /students/pkl`** — list siswa yang sedang PKL saat ini (join `StudentPkl` where `endedAt IS NULL`), dengan info kelas+jurusan+tanggalMulai+tempatPkl.

### Integrasi ke Rekap Kehadiran (WAJIB)
- `attendance-report.service.ts` — di titik yang sekarang menghitung `alfa` (baris ~102-105 dan ~195-199 versi saat ini), tambah pengecekan: kalau siswa punya `StudentPkl` aktif yang overlap tanggal wajib tsb (`tanggalMulai <= dateKey <= (tanggalSelesai ?? today)`), masukkan ke hitungan **`pkl`** (field baru di response), BUKAN `alfa`.
- Tambah field `pkl: number` di response rekap (`RekapRow` atau tipe sejenis) dan di `RiwayatCatatanEntry` (`jenis: "pkl"`).

### UI — Menu Baru `/siswa/pkl`
- Menu baru di sidebar admin, dekat menu Siswa/Kelas (bukan submenu tersembunyi)
- **Bagian 1 — Input manual (paste NISN)**: REUSE pola UI dari T072 (06-Features/tasks/T072-input-siswa-baru-paste-nisn.md) (textarea paste banyak NISN sekaligus, validasi live pakai `validate-nisn-batch` endpoint yang sudah ada — extend validasinya untuk kasus PKL: cek juga siswa harus kelas XII, kalau bukan tampilkan error per-baris "Bukan kelas XII")
- **Bagian 2 — Tombol massal**:
  - Tombol "PKL-kan Semua Kelas XII" (dengan dialog konfirmasi ringkasan jumlah siswa yang akan diproses, field opsional Tempat PKL kalau seragam)
  - Tombol "Normalkan Semua Kelas XII" (dialog konfirmasi, kembalikan SEMUA siswa XII yang sedang PKL ke status normal)
- **Bagian 3 — Daftar siswa PKL aktif**: tabel (Nama, NISN, Kelas, Jurusan, Tanggal Mulai, Tempat PKL) dengan tombol "Normalkan" per baris (individual, tidak harus tunggu tombol massal)
- Tabel siswa (`/siswa`) dan halaman detail siswa (`/siswa/[id]`) — tampilkan badge "PKL" (StatusBadge variant baru atau reuse "processing") di kolom Status kalau siswa sedang PKL, tetap tampilkan status aktif/nonaktif di sampingnya (2 badge terpisah, jangan menimpa)

## JANGAN
- ❌ JANGAN ubah `PersonStatus` enum atau `Student.status` — PKL BUKAN status keluar/nonaktif, siswa PKL tetap aktif penuh
- ❌ JANGAN insert `AttendanceRecord` per hari untuk siswa PKL — dihitung on-the-fly sama seperti alfa
- ❌ JANGAN ubah `Student.kelasId` saat mulai/normalkan PKL — kelas asal tidak pernah berubah karena PKL
- ❌ JANGAN izinkan PKL untuk siswa non-kelas-XII — validasi tegas di backend (400), bukan cuma UI
- ❌ JANGAN lupa `@LogActivity` di 4 endpoint POST baru (aturan permanen — cek memory `feedback_wajib_log_activity`)

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` — model `StudentPkl` baru + migration
- **Buat:** `apps/api/src/core/students/dto/pkl.dto.ts` (beberapa DTO: mulai, mulai-semua, normalkan, normalkan-semua)
- **Modifikasi:** `apps/api/src/core/students/students.controller.ts`, `students.service.ts` — 5 endpoint baru
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` — integrasi PKL ke perhitungan rekap+riwayat
- **Modifikasi:** `apps/api/src/core/students/dto/validate-nisn-batch.dto.ts` atau service terkait — extend validasi kelas-XII-only untuk konteks PKL (kemungkinan perlu parameter context, cek dulu apakah bisa reuse langsung atau butuh varian)
- **Buat:** `apps/web/src/app/(admin)/siswa/pkl/page.tsx`, `pkl-view.tsx`
- **Modifikasi:** sidebar admin — tambah menu "PKL Siswa"
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/siswa-view.tsx`, `siswa/[id]/siswa-detail-view.tsx` — badge PKL
- **Modifikasi:** `apps/web/src/lib/core-types.ts` — type `StudentPkl`, extend `RekapRow`/`RiwayatCatatanEntry`

## Acceptance Criteria
- [ ] `POST /students/pkl/mulai-semua-xii` mem-PKL-kan semua siswa XII aktif, siswa non-XII tidak terpengaruh
- [ ] Siswa yang sedang PKL: rekap kehadiran hari-hari PKL menampilkan "PKL", BUKAN "Alfa"
- [ ] `POST /students/pkl/normalkan-semua-xii` mengembalikan status normal, kelasId siswa TIDAK berubah
- [ ] Input manual paste-NISN menolak NISN yang bukan kelas XII dengan pesan jelas
- [ ] Badge "PKL" muncul di tabel Siswa dan halaman detail siswa yang sedang PKL
- [ ] `@LogActivity` tercatat di keempat endpoint mutasi PKL

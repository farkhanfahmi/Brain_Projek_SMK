---
tags: [absensi, feature, ekstrakurikuler]
status: in-progress
updated: 2026-07-30
---

# Feature — Ekstrakurikuler (Pendaftaran + Absensi oleh Pembina)

← Index (00-INDEX AbsenSI.md)

> Modul di dalam AbsenSI (bukan aplikasi terpisah) — reuse `students`, `users`, RBAC guard, dan infrastruktur JWT yang sudah ada.

---

## 📋 Status

| Item | Detail |
|---|---|
| Pendaftaran ekstra (publik, self-service siswa) | ✅ **Selesai & live** — lihat bagian "Yang Sudah Ada" |
| Monitoring pendaftaran (admin) | ✅ **Selesai & live** |
| Absensi ekstra oleh pembina (dashboard baru) | 🟡 **Task ditulis 2026-07-30, belum dikerjakan** — lihat T096 (06-Features/tasks/T096-absensi-ekstrakurikuler-pembina.md) |

**PENTING:** Draft versi 2026-07-27 dokumen ini SUDAH USANG di beberapa bagian — beberapa keputusan sudah berubah lewat implementasi nyata dan diskusi lanjutan 2026-07-30. Perbedaan utama vs draft lama:
- Pendaftaran ekstra ternyata **self-service oleh siswa sendiri** (halaman publik `/daftar-ekstra`, tanpa login), BUKAN input admin dari formulir kertas seperti draft lama asumsikan.
- Siswa **HANYA boleh daftar 1 ekstra** (constraint `unique(studentId)` di `EkstraPendaftaran`, submit ulang = ganti ekstra lewat UPDATE, bukan tambah baris baru) — pertanyaan terbuka draft lama soal ini sudah terjawab oleh implementasi.
- `EkstraPendaftaran` **TIDAK** terikat `academicYearId` — tidak ada konsep "daftar ulang tiap tahun", cukup 1 baris per siswa yang di-update kapan saja.
- Tidak ada tabel `pembina_ekstra` terpisah maupun `ekstra_pembina` many-to-many — hasil diskusi 2026-07-30 menyederhanakan jadi `Ekstrakurikuler.pembinaId` (1:1 ke `User`, BUKAN ke `Teacher`) karena user eksplisit konfirmasi **1 ekstra = tepat 1 pembina** (bukan banyak-ke-banyak).
- Tidak ada field `kuota`, `hari`, `jam_mulai`, `jam_selesai` di `Ekstrakurikuler` — tidak pernah diminta user, jangan tambahkan tanpa instruksi eksplisit.
- Field `deskripsi` di `Ekstrakurikuler` juga tidak pernah diminta — skema aktual cuma `id`, `nama` (unique), `urutan`.

---

## 👤 Aktor & Akses (final, dikonfirmasi 2026-07-30)

| Role | Login? | Akses |
|---|---|---|
| **Guru yang jadi pembina** | Pakai akun `guru` existing (TIDAK dapat akun baru) | Menu "Ekstrakurikuler" MUNCUL TAMBAHAN di sidebar guru — pola identik Wali Kelas (`kelasIdWali`, lihat `apps/web/src/app/(guru)/guru-sidebar.tsx`): hanya tampil kalau akun ini adalah pembina suatu ekstra. |
| **Pembina eksternal** (bukan guru sekolah) | Role BARU `pembina_ekstra`, akun terpisah dibuatkan admin | Dashboard sendiri, scope sempit — hanya ekstra yang dia bina. Tidak bisa akses modul lain (jurnal, piket, dst). |
| **Super Admin** | Existing | CRUD `Ekstrakurikuler` (sudah ada), assign `pembinaId` (BARU), monitoring pendaftaran (sudah ada). **Tidak bisa** ubah data absensi ekstra — domain eksklusif pembina, pola sama ADR-019. |
| **Siswa** | Tidak login (pendaftaran publik existing tetap sama) | Tidak ada interaksi dengan absensi — pembina yang mencatat, bukan siswa self-report. |

---

## 🗄️ Yang Sudah Ada (jangan ubah tanpa alasan kuat)

```prisma
model Ekstrakurikuler {
  id     Int    @id @default(autoincrement())
  nama   String @unique
  urutan Int    @default(0)
  pendaftaran EkstraPendaftaran[]
}

model EkstraPendaftaran {
  id                Int      @id @default(autoincrement())
  studentId         Int      @unique @map("student_id") // 1 siswa = 1 baris, submit ulang = UPDATE
  ekstrakurikulerId Int      @map("ekstrakurikuler_id")
  submittedAt       DateTime @default(now())
  updatedAt         DateTime @updatedAt
}
```
- Endpoint publik: `apps/api/src/ekstra-publik/ekstra-publik.controller.ts` (submit, tanpa guard)
- Endpoint admin monitoring: `ekstra-monitoring.controller.ts` (JWT + Roles super_admin/card_admin) — termasuk `DELETE /ekstra-monitoring/siswa/:studentId/pendaftaran` (batalkan, siswa isi ulang sendiri)
- Endpoint admin CRUD ekstra: `ekstra-kurikuler.controller.ts`
- Halaman publik: `apps/web/src/app/daftar-ekstra/`
- Halaman admin: `apps/web/src/app/(admin)/ekstra-kurikuler/`, `apps/web/src/app/(admin)/ekstra-monitoring/`

**Tidak ada** yang perlu diubah di bagian ini untuk fitur absensi — murni penambahan, lihat T096 (06-Features/tasks/T096-absensi-ekstrakurikuler-pembina.md) untuk detail lengkap skema baru dan implementasi.

---

## 🔗 Lihat Juga
- T096 — Absensi Ekstrakurikuler oleh Pembina (06-Features/tasks/T096-absensi-ekstrakurikuler-pembina.md) — task detail lengkap, siap dieksekusi
- 11-Decisions (11-Decisions.md) — ADR-010 (dual-FK nullable), ADR-019 (domain eksklusif pencatat lapangan)
- `apps/api/prisma/schema.prisma` — model `TeachingSession`/`ClassAttendanceMark`/`JournalEntry`, pola arsitektur yang ditiru untuk `EkstraSesi`/`EkstraAbsen`

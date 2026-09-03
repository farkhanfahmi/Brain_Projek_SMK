# Task-CORE-020: `ClassPermitRequest` — Tambah `@LogActivity` + Unique Constraint Cegah Duplikasi Pengajuan

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> Migration di task ini MURNI ADDITIF (`CREATE UNIQUE INDEX`) — TIDAK ada DROP/TRUNCATE, tapi tetap WAJIB baca `apps/api/prisma/migrations/<nama>/migration.sql` sebelum commit sesuai aturan CLAUDE.md §Commit-Auto-Deploy untuk konfirmasi ini benar-benar additif.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low-medium
**Alasan pemilihan:** 2 perbaikan kecil digabung (audit log + unique constraint) di file yang sama, tapi unique constraint butuh migration Prisma + penanganan error P2002 yang perlu ketelitian sedikit lebih dari fix biasa.

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) menemukan 2 kejanggalan di `ClassPermitRequestsService.create()` (`apps/api/src/class-permit-requests/class-permit-requests.service.ts` baris ~31-76, dipanggil dari `journal.controller.ts` endpoint `POST /teaching-sessions/:sessionId/permit-request`):

**A. Tidak ada audit log sama sekali** — endpoint ini adalah mutasi data (guru kelas membuat `ClassPermitRequest` baru), tapi `journal.controller.ts` tidak meng-import `LogActivity`, dan service-nya tidak meng-inject `ActivityLogService`/tidak ada `activityLog.record()` di manapun dalam method `create()`. Ini melanggar aturan wajib proyek #1 (`@LogActivity` wajib di endpoint mutasi). Bandingkan dengan `izinkan()`/`tolak()` di controller yang SAMA yang sudah punya `@LogActivity`.

**B. Cek duplikasi rawan race condition** — `create()` baris ~56-63 melakukan "check-then-create" murni (`findFirst` baca, baru `create()`) TANPA unique constraint di database. User sudah memutuskan (keputusan eksplisit 2026-09-03): **kalau unique constraint memang cara terbaik, gunakan itu** — bukan sekadar transaction guard di level aplikasi.

**Depends on:** Tidak ada. **CATATAN**: task-CORE-018 (fix race `izinkan()`) independen dari task ini — keduanya bisa dikerjakan dalam urutan apapun, tapi TIDAK saling tumpang tindih area kodenya (task-CORE-018 fokus `izinkan()`, task ini fokus `create()`).

## 3. Langkah Eksekusi Detail

### A. Audit Log

1. Cek pola `@LogActivity` yang sudah dipakai di controller yang SAMA (`journal.controller.ts`, endpoint `izinkan()`/`tolak()` kalau ada di situ, ATAU cek `class-permit-requests.controller.ts` untuk endpoint `izinkan`/`tolak` — VERIFIKASI SAAT IMPLEMENTASI endpoint `create()` persisnya di controller mana, sesuai temuan audit endpoint ini di-mount di `journal.controller.ts` bukan `class-permit-requests.controller.ts`).
2. Tambahkan `@LogActivity` decorator ke endpoint controller `POST /teaching-sessions/:sessionId/permit-request` (REPLIKASI pola persis yang dipakai endpoint mutasi lain di controller yang sama — action name, targetType, dst sesuai konvensi `activity_log` existing, mis. `class_permit_request.create` atau pola penamaan `<domain>.<action>` yang sudah dipakai di seluruh proyek).
3. **VERIFIKASI SAAT IMPLEMENTASI**: kalau ternyata endpoint di `journal.controller.ts` ini TIDAK cocok dipasangi decorator `@LogActivity` (karena decorator itu didesain untuk 1 target route param tunggal dan situasi di sini beda — cek aturan CLAUDE.md §Aturan Wajib Endpoint Mutasi Baru poin 1: "operasi BULK/internal-service-call pakai `ActivityLogService.record()` manual, decorator hanya cocok 1 target route param") — kalau memang tidak cocok, pakai `ActivityLogService.record()` manual di dalam `ClassPermitRequestsService.create()` (inject `ActivityLogService` ke constructor), REPLIKASI pola manual logging yang sudah dipakai `JournalService.updateJournal()`/`updateAttendance()` di file yang sama (`journal.service.ts`).

### B. Unique Constraint

4. **Schema Prisma** (`apps/api/prisma/schema.prisma`, model `ClassPermitRequest`) — tambahkan unique constraint. **MASALAH TEKNIS PENTING**: MySQL tidak mendukung *partial unique index* native (unique HANYA untuk baris `status: menunggu`) — VERIFIKASI SAAT IMPLEMENTASI pendekatan yang valid untuk MySQL 8, kemungkinan pendekatan:
   - **Opsi disarankan**: tambah kolom generated/computed TIDAK diperlukan kalau desainnya diubah — cukup buat `@@unique([sessionId, studentId])` TANPA syarat status, TAPI ini akan mencegah siswa yang SUDAH diproses (`diizinkan`/`ditolak`) untuk mengajukan ulang di sesi yang sama — PERLU DIPASTIKAN apakah ini memang perilaku yang diinginkan (kemungkinan besar TIDAK — siswa yang izinnya ditolak WAJAR mengajukan ulang di sesi yang sama kalau alasannya berbeda).
   - **Opsi alternatif (lebih tepat)**: MySQL 8 mendukung *functional/generated column* untuk mensimulasikan partial unique index — buat kolom generated `pendingKey` yang berisi `CONCAT(sessionId, '-', studentId)` HANYA kalau `status = 'menunggu'` (`NULL` untuk status lain, MySQL mengizinkan banyak `NULL` di kolom unique), lalu `@@unique` di kolom generated itu. INI PENDEKATAN YANG DIREKOMENDASIKAN — replikasi teknik "nullable generated column sebagai partial unique index" yang umum dipakai untuk MySQL. VERIFIKASI SAAT IMPLEMENTASI cara Prisma schema mendefinisikan generated column MySQL (`@db.VirtualColumn`/raw SQL di migration kalau Prisma schema-nya tidak native support, mungkin perlu edit `migration.sql` manual setelah `prisma migrate dev` generate skeleton-nya).
   - **PUTUSKAN saat implementasi** pendekatan mana yang dipakai berdasar VERIFIKASI teknis di atas — dokumentasikan keputusannya di bagian Implementasi task ini.

5. **Migration** — jalankan `prisma migrate dev` untuk generate migration baru, BACA isi `migration.sql` yang dihasilkan, PASTIKAN hanya `ADD COLUMN`/`CREATE UNIQUE INDEX` (additif murni, TIDAK ADA DROP apa pun) — sesuai aturan CLAUDE.md, migration additif AMAN commit langsung tanpa backup manual wajib, TAPI tetap baca isinya untuk konfirmasi.

6. **Service layer** — `create()` di `class-permit-requests.service.ts`:
   - TETAP pertahankan cek `findFirst()` di awal (untuk pesan error yang FRIENDLY sebelum coba insert — UX lebih baik daripada langsung andalkan P2002 generik).
   - TAMBAHKAN try/catch di sekitar `prisma.classPermitRequest.create()` — tangkap `PrismaClientKnownRequestError` dengan `code === 'P2002'` (constraint violation, race yang lolos dari cek `findFirst` duluan) → lempar `ConflictException` dengan pesan SAMA seperti cek awal ("... sudah punya pengajuan izin yang masih menunggu untuk sesi ini") — REPLIKASI pola penanganan P2002 yang sudah ada di `attendance.service.ts` (disebutkan di audit sebagai referensi existing).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/prisma/schema.prisma` — model `ClassPermitRequest`
- **Buat:** migration baru di `apps/api/prisma/migrations/`
- **Modifikasi:** `apps/api/src/class-permit-requests/class-permit-requests.service.ts` — `create()`, tangkap P2002
- **Modifikasi:** `apps/api/src/journal/journal.controller.ts` (atau `class-permit-requests.controller.ts`, VERIFIKASI lokasi endpoint) — tambah `@LogActivity`/manual log
- **⚠️ Kalau ternyata perlu ubah `packages/types` (shared)**: WAJIB stop dan minta konfirmasi user dulu.

**Dilarang dilakukan:**
- Jangan pakai `@@unique([sessionId, studentId, status])` polos TANPA mempertimbangkan implikasinya — itu TIDAK mencegah race untuk status `menunggu` yang sama (karena `status` ikut jadi bagian key, dua baris `menunggu` dengan `sessionId`+`studentId` sama TETAP dianggap unik kalau kombinasinya literal identik — JUSTRU INI YANG DIMAKSUD DICEGAH, jadi constraint ini SUDAH BENAR mencegah 2 baris `(sessionId, studentId, status='menunggu')` sama — VERIFIKASI LOGIKA INI SAAT IMPLEMENTASI, opsi generated column di langkah 4 sebenarnya opsional kalau constraint 3-kolom sederhana ini sudah cukup: 2 request `menunggu` untuk siswa+sesi sama akan collision, TAPI siswa yang sudah `diizinkan`/`ditolak` bisa mengajukan ulang karena `status` beda → SEBENARNYA INI OPSI PALING SEDERHANA DAN SUDAH CUKUP, PRIORITASKAN opsi `@@unique([sessionId, studentId, status])` dulu sebelum generated column yang lebih rumit, KECUALI terbukti ada masalah nyata saat implementasi).
- Jangan hapus cek `findFirst()` di awal — tetap dipertahankan untuk UX pesan error yang cepat/jelas sebelum sempat kena race.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: 2 request `create()` untuk siswa+sesi yang sama nyaris bersamaan → Perilaku benar: HANYA 1 yang berhasil, yang kedua dapat `ConflictException` jelas (via catch P2002), BUKAN 500 generik.
- Kondisi: siswa yang izinnya SUDAH `ditolak` mengajukan izin BARU untuk sesi yang sama (skenario wajar — guru re-submit dengan alasan beda) → Perilaku benar: BERHASIL, tidak collision (karena `status` beda antara baris lama `ditolak` dan baris baru `menunggu`, asumsi pakai constraint 3-kolom).
- Kondisi: migration dijalankan di DB dev yang SUDAH punya data — VERIFIKASI dulu tidak ada baris existing yang melanggar constraint baru (2 baris `menunggu` untuk siswa+sesi sama) SEBELUM apply migration, kalau ada data begitu di dev, bersihkan dulu (dev only, BUKAN production — cek env sebelum bertindak).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Endpoint `create()` (pengajuan izin guru kelas) tercatat di `activity_log` setiap kali dipanggil.
- [ ] Unique constraint baru di `ClassPermitRequest` mencegah 2 baris `menunggu` untuk `(sessionId, studentId)` yang sama di level database.
- [ ] Race condition 2 request bersamaan → hanya 1 sukses, yang lain dapat pesan error jelas (bukan 500).
- [ ] Siswa yang izinnya `ditolak` tetap bisa mengajukan ulang untuk sesi yang sama.
- [ ] Migration murni additif (diverifikasi baca `migration.sql`), tidak ada DROP.
- [ ] Test unit baru untuk skenario race (mock P2002) + skenario re-submit setelah ditolak.
- [ ] Full test suite lulus tanpa regresi, typecheck bersih.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 150 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada — TAPI VERIFIKASI migration additif SEBELUM commit sesuai aturan CLAUDE.md

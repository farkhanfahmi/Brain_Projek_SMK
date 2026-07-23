# T067 — Lengkapi `@LogActivity` di Semua Controller Mutasi yang Belum Tercatat

## Depends on
Tidak ada — infrastruktur `ActivityLogInterceptor`/`@LogActivity` sudah ada dan berfungsi (`apps/api/src/activity-log/`), task ini murni menambahkan decorator yang belum terpasang.

## Objective
Audit (2026-07-22) menemukan **14 dari 22 controller** dengan endpoint mutasi (POST/PATCH/PUT/DELETE) **tidak** memakai `@LogActivity` — termasuk beberapa aksi berdampak luas (aktivasi semester, CRUD guru, izin guru dengan upload bukti, perubahan jadwal blok). Task ini menambahkan decorator yang hilang secara konsisten ke semua controller yang sudah punya endpoint mutasi.

## Context
- **App:** `apps/api`
- **Mekanisme existing:** `apps/api/src/activity-log/decorators/log-activity.decorator.ts` (decorator `@LogActivity({ action, targetType, prismaModel, idParam?, sensitiveFields? })`) + `activity-log.interceptor.ts` (global interceptor, HANYA bertindak kalau method ditandai decorator DAN request punya `req.user`)
- **Contoh pola yang SUDAH BENAR** (jadikan referensi): `apps/api/src/cards/cards.controller.ts`, `apps/api/src/core/students/students.controller.ts`, `apps/api/src/permits/permits.controller.ts`

## Spec Detail

### Daftar Controller yang BELUM Punya `@LogActivity` (WAJIB ditambahkan)

| Controller | Endpoint | `action` yang disarankan | `targetType` | `prismaModel` |
|---|---|---|---|---|
| `core/kampus/kampus.controller.ts` | POST, PATCH, DELETE | `kampus.create`/`kampus.update`/`kampus.delete` | `kampus` | `kampus` |
| `core/kelas/kelas.controller.ts` | POST, PATCH, DELETE | `kelas.create`/`kelas.update`/`kelas.delete` | `kelas` | `kelas` |
| `core/jurusan/jurusan.controller.ts` | POST, PATCH, DELETE | `jurusan.create`/`jurusan.update`/`jurusan.delete` | `jurusan` | `jurusan` |
| `core/teachers/teachers.controller.ts` | POST, PATCH | `teacher.create`/`teacher.update` | `teacher` | `teacher` |
| `core/schedules/schedules.controller.ts` | POST, POST(copy-from-semester), PATCH | `schedule.create`/`schedule.copy_from_semester`/`schedule.update` | `schedule` | `schedule` |
| `mapel/mapel.controller.ts` | POST, PATCH | `mapel.create`/`mapel.update` | `mapel` | `mapel` |
| `schedule-config/schedule-config.controller.ts` | PATCH | `schedule_config.update` | `schedule_config` | `scheduleConfig` |
| `semesters/semesters.controller.ts` | POST, PATCH, PATCH(activate) | `semester.create`/`semester.update`/**`semester.activate`** | `semester` | `semester` |
| `block-week-ranges/block-week-ranges.controller.ts` | POST, DELETE | `block_week_range.create`/`block_week_range.delete` | `block_week_range` | `blockWeekRange` |
| `teaching-sessions/teaching-sessions.controller.ts` | POST(generate-today), POST(start), POST(run-auto-close) | `teaching_session.generate`/`teaching_session.start`/`teaching_session.auto_close` | `teaching_session` | `teachingSession` |
| `journal/journal.controller.ts` | PATCH(journal), PATCH(attendance) | `journal_entry.update`/`class_attendance_mark.update` | `journal_entry`/`class_attendance_mark` | `journalEntry`/`classAttendanceMark` |
| `teacher-permits/teacher-permits.controller.ts` | POST, PATCH(bukti), POST(submit-tugas) | `teacher_permit.create`/`teacher_permit.update_bukti`/`teacher_permit.submit_tugas` | `teacher_permit` | `teacherPermit` |
| `calendar/academic-years/academic-years.controller.ts` | POST, PATCH(activate) | `academic_year.create`/**`academic_year.activate`** | `academic_year` | `academicYear` |
| `photos/photos.controller.ts` | POST(upload-bulk), PATCH(assign), POST/DELETE(students/teachers) | `photo.upload`/`photo.assign`/`photo.delete` | `student`/`teacher` (sesuai target) | `student`/`teacher` |
| `import/import.controller.ts` | POST(students/teachers/cards) | `import.students`/`import.teachers`/`import.cards` | `import_batch` (atau target individual kalau memungkinkan) | *(lihat catatan di bawah)* |

### Catatan Khusus per Kasus

**`import.controller.ts`** — ini bulk import, bukan mutasi 1 entity. `@LogActivity` yang ada sekarang dirancang untuk 1 target ID (`idParam`). Untuk bulk import, pertimbangkan: log 1 entry ringkasan (`action: import.students`, `targetType: import_batch`, `snapshotAfter: { jumlah_baris, filename }`) BUKAN log per-baris (bisa ribuan entry activity_log untuk 1 import) — kalau `@LogActivity` decorator existing tidak cukup fleksibel untuk pola ini, panggil `ActivityLogService.record()` langsung di service (bukan decorator) dengan snapshot ringkasan.

**`photos.controller.ts`** — target bisa `student` ATAU `teacher` tergantung endpoint (`students/:id` vs `teachers/:id`) — perlu 2 decorator config berbeda untuk 2 route berbeda di controller yang sama, bukan 1 config untuk semua method.

**`teaching-sessions.controller.ts`** — endpoint `generate-today` dan `run-auto-close` dipanggil SISTEM/CRON (bukan user manual dari UI), bukan hanya trigger manual admin. Cek: apakah `req.user` selalu ada untuk endpoint ini (interceptor butuh `req.user` untuk mencatat) — kalau endpoint ini juga dipanggil job internal tanpa JWT user, log tidak akan tercatat untuk trigger otomatis (HANYA tercatat kalau dipicu manual oleh admin_jurnal/super_admin lewat UI). Ini bukan bug yang perlu diperbaiki di task ini, tapi catat sebagai batasan yang disadari — trigger otomatis oleh job cron TIDAK BISA dicatat sebagai "aktivitas user" karena memang bukan aksi user.

**`schedule_config`/`teaching_session`/`journal_entry`/`class_attendance_mark`/`teacher_permit`/`block_week_range` sebagai nama Prisma model** — cek casing PERSIS sesuai definisi model di `schema.prisma` (`prismaModel` di decorator harus cocok nama property `PrismaClient`, biasanya camelCase dari nama model, misal model `ScheduleConfig` → property `prisma.scheduleConfig`).

## JANGAN
- ❌ JANGAN tambahkan `@LogActivity` ke endpoint GET/read-only — hanya endpoint yang MENGUBAH data (POST/PATCH/PUT/DELETE)
- ❌ JANGAN buat 1 `action` generik untuk semua endpoint (misal `"update"` polos) — ikuti format `{resource}.{verb}` yang sudah jadi konvensi (`card.create`, `school_holiday.delete`, dst)
- ❌ JANGAN lupa `sensitiveFields` untuk endpoint yang datanya sensitif — cek dulu apakah target modelya punya field seperti password/token yang perlu di-redact (kemungkinan besar tidak relevan untuk daftar controller di atas, tapi tetap cek satu-satu, jangan asumsi semua aman)
- ❌ JANGAN ubah/hapus `@LogActivity` yang SUDAH BENAR di 8 controller existing (`cards`, `students`, `attendance`, `kiosks`, `permits`, `users`, `school-holidays`, `piket-schedules`) — task ini HANYA menambah yang hilang

## Files
- **Modifikasi:** 14 controller file yang disebutkan di tabel atas — tambah `@LogActivity({...})` di atas tiap method mutasi

## Acceptance Criteria
- [ ] Semua 14 controller di atas punya `@LogActivity` di setiap endpoint POST/PATCH/PUT/DELETE-nya
- [ ] Uji minimal 3 kasus: buat `Kampus` baru → cek `activity_log` (via MySQL MCP) ada 1 baris baru dengan `action: kampus.create`; aktifkan `Semester` → cek `action: semester.activate` tercatat; update `TeacherPermit` bukti → cek `action: teacher_permit.update_bukti` tercatat dengan `snapshotBefore`/`snapshotAfter` yang benar
- [ ] Tidak ada `action` yang bentrok/duplikat nama dengan yang sudah ada di 8 controller existing
- [ ] `import.controller.ts` mencatat ringkasan (bukan per-baris) untuk operasi bulk

## Handoff
Setelah task ini, jadikan **wajib** menambahkan `@LogActivity` sebagai bagian dari checklist "selesai" tiap kali membuat controller/endpoint mutasi BARU ke depan — lihat memory permanen yang mencatat aturan ini (`feedback_wajib_log_activity` di sistem memory Claude).

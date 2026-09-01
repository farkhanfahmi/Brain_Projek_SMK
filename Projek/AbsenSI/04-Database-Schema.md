---
tags: [absensi, database, schema]
updated: 2026-08-31
---

# 04 — Database Schema (Lengkap, Auto-Extract dari Prisma)

← Index (00-INDEX AbsenSI.md)

> **[2026-08-31] Ditulis ulang total** — versi sebelumnya HANYA mencakup skema s.d. Fase 1b (peringatan eksplisit di dalamnya sendiri). Dokumen ini di-generate dari ekstraksi langsung `apps/api/prisma/schema.prisma` (59 model, 749 baris field) — **59/59 model dikonfirmasi tercakup**, tidak ada yang terlewat.
>
> **Cara baca**: kolom **Field** = nama di kode Prisma/TypeScript (camelCase). Kolom **Kolom DB** = nama fisik di tabel MySQL (snake_case, cuma ditulis kalau beda dari field via `@map`). Relasi (foreign key ke model lain) TIDAK didaftar sebagai kolom data — cukup disebut ringkas di bagian "Relasi" tiap model, karena itu bukan kolom fisik tambahan (relasi 1-ke-banyak tidak menambah kolom di tabel ini).
>
> **Untuk skema paling akurat mutlak, selalu cross-check ke `apps/api/prisma/schema.prisma` langsung** — dokumen ini snapshot per 2026-08-31, akan basi lagi seiring migration baru.

---

## Ringkasan

**Total: 59 model / tabel**, dikelompokkan 8 domain berikut:

- **Master Data & Struktur Sekolah** (12 tabel): Kampus, Jurusan, Kelas, Student, StudentPkl, KelasPengurus, KelasPiketJadwal, Teacher, TeacherWifiAccess, Card, Mapel, MapelJurusan
- **Konfigurasi Sistem (Singleton)** (10 tabel): ScheduleConfig, AttendanceLockConfig, KampusTapConfig, ServerHealthAlert, GoogleDriveBackupConfig, KaryawanJamKerjaConfig, SystemLiveConfig, ForcePasswordChangeConfig, TvPiketDisplayConfig, EkstraRegistrationConfig
- **Jadwal & Jurnal Mengajar Guru** (9 tabel): Schedule, TeachingSession, JournalEntry, ClassAttendanceMark, GradeAssessment, GradeAssessmentSession, GradeEntry, TeacherPermit, TeacherPermitSession
- **Absensi Inti (Tap RFID)** (5 tabel): AttendanceSession, AttendanceRecord, Kiosk, TvSession, TapEvent
- **Audit & Kalender Akademik** (4 tabel): ActivityLog, AcademicYear, Semester, SchoolHoliday
- **Jadwal Pelajaran (Fondasi Baru T203+)** (8 tabel): AlokasiWaktu, AlokasiWaktuSlot, OpsiJadwal, OpsiJadwalTingkat, OpsiJadwalMingguGenerate, MapelGuru, JadwalSlot, JadwalSlotGuru
- **Akun & Perizinan** (5 tabel): User, Permit, LateEntrySlip, PiketSchedule, PiketJournalEntry
- **Ekstrakurikuler** (6 tabel): Ekstrakurikuler, EkstraKelompok, EkstraKelompokAnggota, EkstraPendaftaran, EkstraSesi, EkstraAbsen

---

## Master Data & Struktur Sekolah

### `Kampus` → tabel `kampus`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |
| `lokasiLat` | Decimal? | lokasi_lat |
| `lokasiLng` | Decimal? | lokasi_lng |
| `radiusGeofenceMeter` | Int? | radius_geofence_meter |

**Relasi:** `kelas`→Kelas, `users`→User, `kiosks`→Kiosk, `tvSessions`→TvSession, `tvPiketDisplayConfig`→TvPiketDisplayConfig

### `Jurusan` → tabel `jurusan`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |

**Relasi:** `kelas`→Kelas, `mapelRelations`→MapelJurusan

### `Kelas` → tabel `kelas`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |
| `kampusId` | Int | kampus_id |
| `jurusanId` | Int | jurusan_id |
| `tingkat` | Tingkat | — |
| `ruangan` | String? | — |
| `lantai` | Int? | — |

**Relasi:** `kampus`→Kampus, `jurusan`→Jurusan, `students`→Student, `schedules`→Schedule, `teachingSessions`→TeachingSession, `waliKelas`→User, `gradeAssessments`→GradeAssessment, `jadwalSlots`→JadwalSlot, `pengurus`→KelasPengurus, `piketJadwal`→KelasPiketJadwal

### `Student` → tabel `students`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nisn` | String | — |
| `nama` | String | — |
| `kelasId` | Int? | kelas_id |
| `kelasTerakhirNama` | String? | kelas_terakhir_nama |
| `status` | PersonStatus | — |
| `tanggalLahir` | DateTime? | tanggal_lahir |
| `alasanNonaktif` | AlasanNonaktif? | alasan_nonaktif |
| `tahunLulus` | Int? | tahun_lulus |
| `lockedAt` | DateTime? | locked_at |
| `lockedReason` | String? | locked_reason |
| `lockedById` | Int? | locked_by |
| `unlockedAt` | DateTime? | unlocked_at |
| `unlockedById` | Int? | unlocked_by |
| `unlockNote` | String? | unlock_note |
| `lateStrikeResetAt` | DateTime? | late_strike_reset_at |
| `tempatLahir` | String? | tempat_lahir |
| `jenisKelamin` | JenisKelamin? | jenis_kelamin |
| `agama` | Agama? | — |
| `alamat` | String? | — |
| `rtRw` | String? | rt_rw |
| `namaAyah` | String? | nama_ayah |
| `namaIbu` | String? | nama_ibu |
| `foto` | String? | — |
| `noHpSiswa` | String? | no_hp_siswa |
| `noHpAyah` | String? | no_hp_ayah |
| `noHpIbu` | String? | no_hp_ibu |
| `tinggalKelasPada` | DateTime? | tinggal_kelas_pada |

**Relasi:** `kelas`→Kelas, `lockedBy`→User, `unlockedBy`→User, `cards`→Card, `attendanceRecords`→AttendanceRecord, `permits`→Permit, `classAttendanceMarks`→ClassAttendanceMark, `pklRecords`→StudentPkl, `ekstraPendaftaran`→EkstraPendaftaran, `ekstraAbsen`→EkstraAbsen, `ekstraKelompok`→EkstraKelompokAnggota, `lateEntrySlips`→LateEntrySlip, `gradeEntries`→GradeEntry, `kelasPengurus`→KelasPengurus, `kelasPiketJadwal`→KelasPiketJadwal, `userAkun`→User

### `StudentPkl` → tabel `student_pkl`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `studentId` | Int | student_id |
| `tanggalMulai` | DateTime | tanggal_mulai |
| `tanggalSelesai` | DateTime? | tanggal_selesai |
| `tempatPkl` | String? | tempat_pkl |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |
| `endedAt` | DateTime? | ended_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `student`→Student, `createdBy`→User, `academicYear`→AcademicYear, `semester`→Semester

### `KelasPengurus` → tabel `kelas_pengurus`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `kelasId` | Int | kelas_id |
| `studentId` | Int | student_id |
| `jabatan` | JabatanPengurus | — |
| `academicYearId` | Int | academic_year_id |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `kelas`→Kelas, `student`→Student, `academicYear`→AcademicYear, `createdBy`→User

### `KelasPiketJadwal` → tabel `kelas_piket_jadwal`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `kelasId` | Int | kelas_id |
| `hari` | Int | — |
| `studentId` | Int | student_id |
| `createdAt` | DateTime | created_at |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `kelas`→Kelas, `student`→Student

### `Teacher` → tabel `teachers`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `niy` | String | — |
| `nama` | String | — |
| `noHp` | String? | no_hp |
| `foto` | String? | — |
| `status` | PersonStatus | — |
| `gelarDepan` | String? | gelar_depan |
| `gelarBelakang` | String? | gelar_belakang |
| `tempatLahir` | String? | tempat_lahir |
| `tanggalLahir` | DateTime? | tanggal_lahir |
| `jenisKelamin` | JenisKelamin? | jenis_kelamin |
| `agama` | Agama? | — |
| `alamat` | String? | — |
| `statusPernikahan` | StatusPernikahan? | status_pernikahan |
| `statusKepegawaian` | StatusKepegawaian | status_kepegawaian |

**Relasi:** `users`→User, `cards`→Card, `schedules`→Schedule, `attendanceRecords`→AttendanceRecord, `teachingSessions`→TeachingSession, `teacherPermits`→TeacherPermit, `gradeAssessments`→GradeAssessment, `mapelPengampu`→MapelGuru, `jadwalSlotGuru`→JadwalSlotGuru, `wifiAccess`→TeacherWifiAccess

### `TeacherWifiAccess` → tabel `teacher_wifi_access`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `teacherId` | Int | teacher_id |
| `username` | String | — |
| `password` | String | — |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `teacher`→Teacher, `updatedBy`→User

### `Card` → tabel `cards`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `uid` | String | — |
| `studentId` | Int? | student_id |
| `teacherId` | Int? | teacher_id |
| `status` | CardStatus | — |
| `issuedAt` | DateTime | issued_at |
| `revokedAt` | DateTime? | revoked_at |

**Relasi:** `student`→Student, `teacher`→Teacher, `tapEvents`→TapEvent

### `Mapel` → tabel `mapel`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |
| `kode` | String? | — |
| `createdAt` | DateTime | created_at |

**Relasi:** `jurusanRelations`→MapelJurusan, `schedules`→Schedule, `teachingSessions`→TeachingSession, `gradeAssessments`→GradeAssessment, `guruPengampu`→MapelGuru, `jadwalSlots`→JadwalSlot

### `MapelJurusan` → tabel `mapel_jurusan`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `mapelId` | Int | mapel_id |
| `jurusanId` | Int | jurusan_id |

**Relasi:** `mapel`→Mapel, `jurusan`→Jurusan

---

## Konfigurasi Sistem (Singleton)

### `ScheduleConfig` → tabel `schedule_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `toleransiSiswaMenit` | Int | toleransi_siswa_menit |
| `toleransiGuruMenit` | Int | toleransi_guru_menit |
| `toleransiKaryawanMenit` | Int | toleransi_karyawan_menit |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `updatedBy`→User

### `AttendanceLockConfig` → tabel `attendance_lock_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `lateLockAutoEnabled` | Boolean | late_lock_auto_enabled |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `updatedBy`→User

### `KampusTapConfig` → tabel `kampus_tap_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `kampusMatchEnabled` | Boolean | kampus_match_enabled |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `updatedBy`→User

### `ServerHealthAlert` → tabel `server_health_alerts`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `kategori` | String | — |
| `pesan` | String | — |
| `createdAt` | DateTime | created_at |

### `GoogleDriveBackupConfig` → tabel `google_drive_backup_config`

| Field             | Tipe     | Kolom DB          |
| ----------------- | -------- | ----------------- |
| `id`              | Int      | —                 |
| `refreshTokenEnc` | String   | refresh_token_enc |
| `driveFolderId`   | String   | drive_folder_id   |
| `driveFolderNama` | String?  | drive_folder_nama |
| `connectedEmail`  | String?  | connected_email   |
| `updatedById`     | Int      | updated_by        |
| `updatedAt`       | DateTime | updated_at        |

**Relasi:** `updatedBy`→User

### `KaryawanJamKerjaConfig` → tabel `karyawan_jam_kerja_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `jamMulai` | String | jam_mulai |
| `jamSelesai` | String | jam_selesai |
| `toleransiAktif` | Boolean | toleransi_aktif |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `updatedBy`→User

### `SystemLiveConfig` → tabel `system_live_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `liveSince` | DateTime | live_since |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `updatedBy`→User

### `ForcePasswordChangeConfig` → tabel `force_password_change_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `forceAdmin` | Boolean | force_admin |
| `forcePiket` | Boolean | force_piket |
| `forceGuru` | Boolean | force_guru |
| `forcePembinaEkstra` | Boolean | force_pembina_ekstra |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `updatedBy`→User

### `TvPiketDisplayConfig` → tabel `tv_piket_display_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `kampusId` | Int | kampus_id |
| `tampilBarPersentase` | Boolean | tampil_bar_persentase |
| `tampilSiswaTidakHadir` | Boolean | tampil_siswa_tidak_hadir |
| `tampilGuruBelumMulai` | Boolean | tampil_guru_belum_mulai |
| `tampilGuruIzin` | Boolean | tampil_guru_izin |
| `tampilSiswaIzinBelumKembali` | Boolean | tampil_siswa_izin_belum_kembali |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `kampus`→Kampus, `updatedBy`→User

### `EkstraRegistrationConfig` → tabel `ekstra_registration_config`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `lockPindahEkstra` | Boolean | lock_pindah_ekstra |
| `isOpen` | Boolean | is_open |
| `bukaMulai` | DateTime? | buka_mulai |
| `bukaSampai` | DateTime? | buka_sampai |
| `updatedById` | Int | updated_by |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `updatedBy`→User

---

## Jadwal & Jurnal Mengajar Guru

### `Schedule` → tabel `schedules`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `type` | ScheduleType | — |
| `teacherId` | Int? | teacher_id |
| `kelasId` | Int? | kelas_id |
| `mapelId` | Int? | mapel_id |
| `hari` | Int | — |
| `jamMulai` | String? | jam_mulai |
| `jamSelesai` | String? | jam_selesai |
| `thresholdTerlambatMenit` | Int | threshold_terlambat_menit |
| `tanggalBerlakuMulai` | DateTime? | tanggal_berlaku_mulai |
| `tanggalBerlakuSelesai` | DateTime? | tanggal_berlaku_selesai |
| `semesterId` | Int? | semester_id |
| `tingkat` | Tingkat? | — |

**Relasi:** `teacher`→Teacher, `kelas`→Kelas, `mapel`→Mapel, `semester`→Semester

### `TeachingSession` → tabel `teaching_sessions`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `jadwalSlotId` | Int | jadwal_slot_id |
| `teacherId` | Int | teacher_id |
| `kelasId` | Int | kelas_id |
| `mapelId` | Int | mapel_id |
| `tanggal` | DateTime | — |
| `startedAt` | DateTime? | started_at |
| `closedAt` | DateTime? | closed_at |
| `status` | SesiJurnalStatus | — |
| `lokasiLat` | Decimal? | lokasi_lat |
| `lokasiLng` | Decimal? | lokasi_lng |
| `terlambatMenit` | Int? | terlambat_menit |
| `createdAt` | DateTime | created_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `jadwalSlot`→JadwalSlot, `teacher`→Teacher, `kelas`→Kelas, `mapel`→Mapel, `journalEntry`→JournalEntry, `attendanceMarks`→ClassAttendanceMark, `teacherPermitSessions`→TeacherPermitSession, `academicYear`→AcademicYear, `semester`→Semester, `gradeAssessmentSessions`→GradeAssessmentSession

### `JournalEntry` → tabel `journal_entries`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `sessionId` | Int | session_id |
| `elemen` | String? | — |
| `capaianPembelajaran` | String? | capaian_pembelajaran |
| `materi` | String? | — |
| `tujuanPembelajaran` | String? | tujuan_pembelajaran |
| `tugasPenilaian` | String? | tugas_penilaian |
| `catatan` | String? | — |
| `createdAt` | DateTime | created_at |
| `updatedAt` | DateTime | updated_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `session`→TeachingSession, `academicYear`→AcademicYear, `semester`→Semester

### `ClassAttendanceMark` → tabel `class_attendance_marks`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `sessionId` | Int | session_id |
| `studentId` | Int | student_id |
| `status` | ClassAttendanceStatus | — |
| `markedById` | Int | marked_by |
| `markedAt` | DateTime | marked_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `session`→TeachingSession, `student`→Student, `markedBy`→User, `academicYear`→AcademicYear, `semester`→Semester

### `GradeAssessment` → tabel `grade_assessments`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `teacherId` | Int | teacher_id |
| `kelasId` | Int | kelas_id |
| `mapelId` | Int | mapel_id |
| `judul` | String | — |
| `createdAt` | DateTime | created_at |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `teacher`→Teacher, `kelas`→Kelas, `mapel`→Mapel, `sessions`→GradeAssessmentSession, `entries`→GradeEntry

### `GradeAssessmentSession` → tabel `grade_assessment_sessions`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `assessmentId` | Int | assessment_id |
| `sessionId` | Int | session_id |

**Relasi:** `assessment`→GradeAssessment, `session`→TeachingSession

### `GradeEntry` → tabel `grade_entries`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `assessmentId` | Int | assessment_id |
| `studentId` | Int | student_id |
| `nilai` | Int? | — |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `assessment`→GradeAssessment, `student`→Student

### `TeacherPermit` → tabel `teacher_permits`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `teacherId` | Int | teacher_id |
| `tanggal` | DateTime | — |
| `tanggalSelesai` | DateTime? | tanggal_selesai |
| `kategori` | TeacherPermitKategori | — |
| `status` | TeacherPermitStatus | — |
| `buktiFilePath` | String | bukti_file_path |
| `buktiUpdatedAt` | DateTime | bukti_updated_at |
| `approvedById` | Int | approved_by |
| `approvedAt` | DateTime | approved_at |
| `tugasFilePath` | String? | tugas_file_path |
| `tugasKeterangan` | String? | tugas_keterangan |
| `submittedAt` | DateTime? | submitted_at |
| `followUpNeeded` | Boolean | follow_up_needed |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `teacher`→Teacher, `sessions`→TeacherPermitSession, `approvedBy`→User, `academicYear`→AcademicYear, `semester`→Semester

### `TeacherPermitSession` → tabel `teacher_permit_sessions`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `permitId` | Int | permit_id |
| `sessionId` | Int | session_id |

**Relasi:** `permit`→TeacherPermit, `session`→TeachingSession

---

## Absensi Inti (Tap RFID)

### `AttendanceSession` → tabel `attendance_sessions`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `locationType` | LocationType | location_type |
| `kelasId` | Int? | kelas_id |

**Relasi:** `attendanceRecords`→AttendanceRecord

### `AttendanceRecord` → tabel `attendance_records`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `studentId` | Int? | student_id |
| `teacherId` | Int? | teacher_id |
| `sessionId` | Int? | session_id |
| `tanggal` | DateTime | — |
| `waktuMasuk` | DateTime | waktu_masuk |
| `waktuPulang` | DateTime? | waktu_pulang |
| `status` | AttendanceStatus | — |
| `masukVia` | MasukVia? | masuk_via |
| `pulangVia` | PulangVia? | pulang_via |
| `clientUuid` | String? | client_uuid |
| `kioskId` | Int? | kiosk_id |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `student`→Student, `teacher`→Teacher, `session`→AttendanceSession, `kiosk`→Kiosk, `tapEvents`→TapEvent, `lateEntrySlips`→LateEntrySlip, `academicYear`→AcademicYear, `semester`→Semester

### `Kiosk` → tabel `kiosks`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |
| `kampusId` | Int | kampus_id |
| `deviceToken` | String | device_token |
| `allowedIp` | String | allowed_ip |
| `tipe` | KioskTipe | — |
| `isActive` | Boolean | is_active |
| `lastSeenAt` | DateTime? | last_seen_at |
| `lastSeenIp` | String? | last_seen_ip |
| `lastFailedIp` | String? | last_failed_ip |
| `lastFailedAt` | DateTime? | last_failed_at |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |

**Relasi:** `kampus`→Kampus, `createdByUser`→User, `tapEvents`→TapEvent, `attendanceRecords`→AttendanceRecord

### `TvSession` → tabel `tv_sessions`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `kampusId` | Int | kampus_id |
| `token` | String | — |
| `isActive` | Boolean | is_active |
| `revokedAt` | DateTime? | revoked_at |
| `revokedById` | Int? | revoked_by |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |

**Relasi:** `kampus`→Kampus, `revokedBy`→User, `createdBy`→User

### `TapEvent` → tabel `tap_events`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `uid` | String | — |
| `cardId` | Int? | card_id |
| `kioskId` | Int? | kiosk_id |
| `scannedAt` | DateTime | scanned_at |
| `result` | TapResult | — |
| `attendanceRecordId` | Int? | attendance_record_id |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `card`→Card, `attendanceRecord`→AttendanceRecord, `kiosk`→Kiosk, `academicYear`→AcademicYear, `semester`→Semester

---

## Audit & Kalender Akademik

### `ActivityLog` → tabel `activity_log`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `actorId` | Int | actor_id |
| `action` | String | — |
| `targetType` | String | target_type |
| `targetId` | Int | target_id |
| `snapshotBefore` | Json? | snapshot_before |
| `snapshotAfter` | Json? | snapshot_after |
| `ipAddress` | String? | ip_address |
| `createdAt` | DateTime | created_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `actor`→User, `academicYear`→AcademicYear, `semester`→Semester

### `AcademicYear` → tabel `academic_years`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |
| `tanggalMulai` | DateTime | tanggal_mulai |
| `tanggalSelesai` | DateTime | tanggal_selesai |
| `isActive` | Boolean | is_active |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |

**Relasi:** `createdBy`→User, `schoolHolidays`→SchoolHoliday, `semesters`→Semester, `studentPkl`→StudentPkl, `teachingSessions`→TeachingSession, `journalEntries`→JournalEntry, `classAttendanceMarks`→ClassAttendanceMark, `teacherPermits`→TeacherPermit, `attendanceRecords`→AttendanceRecord, `tapEvents`→TapEvent, `activityLogs`→ActivityLog, `permits`→Permit, `lateEntrySlips`→LateEntrySlip, `piketJournalEntries`→PiketJournalEntry, `ekstraSesi`→EkstraSesi, `ekstraAbsen`→EkstraAbsen, `kelasPengurus`→KelasPengurus

### `Semester` → tabel `semesters`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `academicYearId` | Int | academic_year_id |
| `nama` | SemesterNama | — |
| `tanggalMulai` | DateTime | tanggal_mulai |
| `tanggalSelesai` | DateTime | tanggal_selesai |
| `isActive` | Boolean | is_active |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |

**Relasi:** `academicYear`→AcademicYear, `createdBy`→User, `schedules`→Schedule, `opsiJadwal`→OpsiJadwal, `studentPkl`→StudentPkl, `teachingSessions`→TeachingSession, `journalEntries`→JournalEntry, `classAttendanceMarks`→ClassAttendanceMark, `teacherPermits`→TeacherPermit, `attendanceRecords`→AttendanceRecord, `tapEvents`→TapEvent, `activityLogs`→ActivityLog, `permits`→Permit, `lateEntrySlips`→LateEntrySlip, `piketJournalEntries`→PiketJournalEntry, `ekstraSesi`→EkstraSesi, `ekstraAbsen`→EkstraAbsen

### `SchoolHoliday` → tabel `school_holidays`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `academicYearId` | Int? | academic_year_id |
| `tanggalMulai` | DateTime | tanggal_mulai |
| `tanggalSelesai` | DateTime | tanggal_selesai |
| `jenis` | HolidayJenis | — |
| `keterangan` | String? | — |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |
| `updatedById` | Int? | updated_by |
| `updatedAt` | DateTime? | updated_at |

**Relasi:** `academicYear`→AcademicYear, `createdBy`→User, `updatedBy`→User

---

## Jadwal Pelajaran (Fondasi Baru T203+)

### `AlokasiWaktu` → tabel `alokasi_waktu`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | — |
| `updatedAt` | DateTime | — |

**Relasi:** `slots`→AlokasiWaktuSlot, `opsiJadwal`→OpsiJadwal, `createdBy`→User

### `AlokasiWaktuSlot` → tabel `alokasi_waktu_slots`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `alokasiWaktuId` | Int | alokasi_waktu_id |
| `hari` | Int | — |
| `jamKe` | Int? | jam_ke |
| `jamMulai` | String | jam_mulai |
| `jamSelesai` | String | jam_selesai |
| `keterangan` | String? | — |
| `urutan` | Int | — |

**Relasi:** `alokasiWaktu`→AlokasiWaktu

### `OpsiJadwal` → tabel `opsi_jadwal`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `semesterId` | Int | semester_id |
| `alokasiWaktuId` | Int | alokasi_waktu_id |
| `nama` | String | — |
| `mode` | ModeJadwal | — |
| `isActive` | Boolean | is_active |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | — |
| `updatedAt` | DateTime | — |

**Relasi:** `semester`→Semester, `alokasiWaktu`→AlokasiWaktu, `tingkatScopes`→OpsiJadwalTingkat, `mingguGenerate`→OpsiJadwalMingguGenerate, `jadwalSlots`→JadwalSlot, `createdBy`→User

### `OpsiJadwalTingkat` → tabel `opsi_jadwal_tingkat`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `opsiJadwalId` | Int | opsi_jadwal_id |
| `tingkat` | Tingkat | — |

**Relasi:** `opsiJadwal`→OpsiJadwal

### `OpsiJadwalMingguGenerate` → tabel `opsi_jadwal_minggu_generate`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `opsiJadwalId` | Int | opsi_jadwal_id |
| `tanggal` | DateTime | — |
| `minggu` | MingguAB | — |

**Relasi:** `opsiJadwal`→OpsiJadwal

### `MapelGuru` → tabel `mapel_guru`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `mapelId` | Int | mapel_id |
| `teacherId` | Int | teacher_id |

**Relasi:** `mapel`→Mapel, `teacher`→Teacher

### `JadwalSlot` → tabel `jadwal_slots`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `opsiJadwalId` | Int | opsi_jadwal_id |
| `kelasId` | Int | kelas_id |
| `mapelId` | Int | mapel_id |
| `hari` | Int | — |
| `jamKe` | Int | jam_ke |
| `minggu` | MingguAB? | — |
| `jamKeAkhirRentang` | Int? | jam_ke_akhir_rentang |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | — |
| `updatedAt` | DateTime | — |

**Relasi:** `opsiJadwal`→OpsiJadwal, `kelas`→Kelas, `mapel`→Mapel, `guru`→JadwalSlotGuru, `createdBy`→User, `teachingSessions`→TeachingSession

### `JadwalSlotGuru` → tabel `jadwal_slot_guru`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `jadwalSlotId` | Int | jadwal_slot_id |
| `teacherId` | Int | teacher_id |

**Relasi:** `jadwalSlot`→JadwalSlot, `teacher`→Teacher

---

## Akun & Perizinan

### `User` → tabel `users`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `username` | String | — |
| `passwordHash` | String | password_hash |
| `role` | UserRole | — |
| `teacherId` | Int? | teacher_id |
| `studentId` | Int? | student_id |
| `kampusId` | Int? | kampus_id |
| `kelasIdWali` | Int? | kelas_id_wali |
| `status` | UserStatus | — |
| `mustChangePassword` | Boolean | must_change_password |
| `passwordChangedAt` | DateTime? | password_changed_at |
| `lockedAt` | DateTime? | locked_at |
| `lockedReason` | String? | locked_reason |
| `lockedById` | Int? | locked_by |
| `unlockedAt` | DateTime? | unlocked_at |
| `unlockedById` | Int? | unlocked_by |
| `unlockNote` | String? | unlock_note |

**Relasi:** `teacher`→Teacher, `student`→Student, `kampus`→Kampus, `kelasWali`→Kelas, `lockedBy`→User, `unlockedBy`→User, `activityLogs`→ActivityLog, `academicYearsCreated`→AcademicYear, `holidaysCreated`→SchoolHoliday, `holidaysUpdated`→SchoolHoliday, `studentsLocked`→Student, `studentsUnlocked`→Student, `permitsApproved`→Permit, `permitsKembaliConfirmed`→Permit, `kiosksCreated`→Kiosk, `piketSchedules`→PiketSchedule, `piketSchedulesCreated`→PiketSchedule, `scheduleConfigsUpdated`→ScheduleConfig, `attendanceLockConfigsUpdated`→AttendanceLockConfig, `kampusTapConfigsUpdated`→KampusTapConfig, `googleDriveBackupConfigsUpdated`→GoogleDriveBackupConfig, `karyawanJamKerjaConfigsUpdated`→KaryawanJamKerjaConfig, `systemLiveConfigsUpdated`→SystemLiveConfig, `forcePasswordChangeConfigsUpdated`→ForcePasswordChangeConfig, `ekstraRegistrationConfigsUpdated`→EkstraRegistrationConfig, `classAttendanceMarksMade`→ClassAttendanceMark, `teacherPermitsApproved`→TeacherPermit, `semestersCreated`→Semester, `tvSessionsCreated`→TvSession, `tvSessionsRevoked`→TvSession, `studentPklsCreated`→StudentPkl, `ekstrakurikulerDibina`→Ekstrakurikuler, `ekstraSesiCreated`→EkstraSesi, `ekstraAbsenMarked`→EkstraAbsen, `lateEntrySlipsPrinted`→LateEntrySlip, `piketJournalEntries`→PiketJournalEntry, `tvPiketDisplayConfigsUpdated`→TvPiketDisplayConfig, `alokasiWaktuCreated`→AlokasiWaktu, `opsiJadwalCreated`→OpsiJadwal, `jadwalSlotCreated`→JadwalSlot, `usersLocked`→User, `usersUnlocked`→User, `kelasPengurusCreated`→KelasPengurus, `teacherWifiAccessUpdated`→TeacherWifiAccess

### `Permit` → tabel `permits`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `studentId` | Int | student_id |
| `jenis` | PermitJenis | — |
| `alasanKategori` | PermitAlasanKategori | alasan_kategori |
| `alasanDetail` | String? | alasan_detail |
| `tanggal` | DateTime | — |
| `tanggalSelesai` | DateTime? | tanggal_selesai |
| `jamKeluar` | DateTime? | jam_keluar |
| `jamKembaliDiharapkan` | DateTime? | jam_kembali_diharapkan |
| `statusKembali` | StatusKembali | status_kembali |
| `kembaliDikonfirmasiAt` | DateTime? | kembali_dikonfirmasi_at |
| `kembaliDikonfirmasiById` | Int? | kembali_dikonfirmasi_by |
| `approvedById` | Int | approved_by |
| `kodeVerifikasi` | String? | kode_verifikasi |
| `suratPrintedAt` | DateTime? | surat_printed_at |
| `buktiFilePath` | String? | bukti_file_path |
| `createdAt` | DateTime | created_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `student`→Student, `approvedBy`→User, `kembaliDikonfirmasiBy`→User, `academicYear`→AcademicYear, `semester`→Semester

### `LateEntrySlip` → tabel `late_entry_slips`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `studentId` | Int | student_id |
| `attendanceRecordId` | Int? | attendance_record_id |
| `tanggal` | DateTime | — |
| `jamTap` | DateTime | jam_tap |
| `alasan` | String? | — |
| `catatan` | String? | — |
| `kodeVerifikasi` | String | kode_verifikasi |
| `printedById` | Int | printed_by |
| `createdAt` | DateTime | created_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `student`→Student, `attendanceRecord`→AttendanceRecord, `printedBy`→User, `academicYear`→AcademicYear, `semester`→Semester

### `PiketSchedule` → tabel `piket_schedules`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `hari` | Int | — |
| `userId` | Int | user_id |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |

**Relasi:** `user`→User, `createdBy`→User

### `PiketJournalEntry` → tabel `piket_journal_entries`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `userId` | Int | user_id |
| `tanggal` | DateTime | — |
| `catatan` | String | — |
| `filledAt` | DateTime | filled_at |
| `createdAt` | DateTime | created_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `user`→User, `academicYear`→AcademicYear, `semester`→Semester

---

## Ekstrakurikuler

### `Ekstrakurikuler` → tabel `ekstrakurikuler`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `nama` | String | — |
| `urutan` | Int | — |
| `pembinaId` | Int? | pembina_id |
| `hari` | Int? | hari |
| `jamMulai` | String? | jam_mulai |
| `jamSelesai` | String? | jam_selesai |

**Relasi:** `pembina`→User, `pendaftaran`→EkstraPendaftaran, `sesi`→EkstraSesi, `kelompok`→EkstraKelompok

### `EkstraKelompok` → tabel `ekstra_kelompok`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `ekstrakurikulerId` | Int | ekstrakurikuler_id |
| `nama` | String | — |
| `jamMulai` | String | jam_mulai |
| `jamSelesai` | String | jam_selesai |

**Relasi:** `ekstrakurikuler`→Ekstrakurikuler, `anggota`→EkstraKelompokAnggota, `sesi`→EkstraSesi

### `EkstraKelompokAnggota` → tabel `ekstra_kelompok_anggota`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `kelompokId` | Int | kelompok_id |
| `studentId` | Int | student_id |

**Relasi:** `kelompok`→EkstraKelompok, `student`→Student

### `EkstraPendaftaran` → tabel `ekstra_pendaftaran`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `studentId` | Int | student_id |
| `ekstrakurikulerId` | Int | ekstrakurikuler_id |
| `submittedAt` | DateTime | submitted_at |
| `updatedAt` | DateTime | updated_at |

**Relasi:** `student`→Student, `ekstrakurikuler`→Ekstrakurikuler

### `EkstraSesi` → tabel `ekstra_sesi`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `ekstrakurikulerId` | Int | ekstrakurikuler_id |
| `kelompokId` | Int? | kelompok_id |
| `tanggal` | DateTime | — |
| `jamMulai` | String? | jam_mulai |
| `jamSelesai` | String? | jam_selesai |
| `catatan` | String? | — |
| `createdById` | Int | created_by |
| `createdAt` | DateTime | created_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `ekstrakurikuler`→Ekstrakurikuler, `kelompok`→EkstraKelompok, `createdBy`→User, `absen`→EkstraAbsen, `academicYear`→AcademicYear, `semester`→Semester

### `EkstraAbsen` → tabel `ekstra_absen`

| Field | Tipe | Kolom DB |
|---|---|---|
| `id` | Int | — |
| `sesiId` | Int | sesi_id |
| `studentId` | Int | student_id |
| `status` | EkstraAbsenStatus? | — |
| `buktiFilePath` | String? | bukti_file_path |
| `markedById` | Int? | marked_by |
| `markedAt` | DateTime? | marked_at |
| `academicYearId` | Int? | academic_year_id |
| `semesterId` | Int? | semester_id |

**Relasi:** `sesi`→EkstraSesi, `student`→Student, `markedBy`→User, `academicYear`→AcademicYear, `semester`→Semester

---

## Enum yang Dipakai

Daftar semua 30 enum ada langsung di `schema.prisma` (cari `enum `) — tidak diduplikasi di sini karena enum baru sering ditambah seiring fitur baru dan gampang basi. Enum penting yang sering direferensikan dokumen lain: `UserRole`, `PersonStatus`, `AttendanceStatus`, `PulangVia`, `MasukVia`, `JabatanPengurus`.

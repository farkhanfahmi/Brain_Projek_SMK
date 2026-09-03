# Task-CORE-011: Backend — Endpoint Bulk Checklist Presensi + Validasi "Belum Terabsen"

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` §3 untuk konteks lengkap. Dieksekusi oleh Claude Code.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Perubahan arsitektur interaksi (instant-PATCH-per-klik → bulk-checklist-lalu-submit) menyentuh validasi silang data tap gerbang vs checklist guru — perlu ketelitian query dan pesan error presisi per-siswa.

## 2. Konteks & Tujuan Utama

Baca `06-Features/desain-redesign-presensi-izin-keluar.md` §2-3. Saat ini `JournalService.updateAttendance()` (`apps/api/src/journal/journal.service.ts`) menerima PATCH per-klik langsung tersimpan. Desain baru: guru checklist SEMUA siswa dulu (state lokal FE), baru submit sekali — backend WAJIB validasi kelengkapan SEBELUM menyimpan.

**Depends on:** task-CORE-010 (skema `keterangan` di `ClassAttendanceMark` harus sudah ada).

## 3. Langkah Eksekusi Detail

1. Update `apps/api/src/journal/dto/update-attendance.dto.ts` — `marks[]` tiap entry tambah field opsional `keterangan?: string` (wajib diisi FE untuk status `tidak_ada_di_kelas`, tapi validasi WAJIB-nya di backend juga — jangan percaya FE saja).

2. Di `JournalService.updateAttendance()` — SEBELUM melakukan write apa pun, tambahkan **validasi kelengkapan**:
   - Ambil daftar `studentId` yang **tap gerbang** hari itu (`AttendanceRecord`, REUSE query yang SUDAH ADA di method ini — `attendanceRecords` sudah di-fetch di `getDetail()`, cek pola serupa).
   - Bandingkan dengan `dto.marks[]` yang dikirim FE — kalau ada `studentId` yang tap gerbang TAPI TIDAK ADA di `dto.marks[]` (guru belum checklist sama sekali untuk siswa itu), **throw `BadRequestException`** dengan pesan PERSIS format: `"<nama siswa> belum terabsen"` (ambil nama dari `Student`, JANGAN cuma `studentId` mentah di pesan error).
   - Kalau ADA beberapa siswa belum terabsen sekaligus, gabungkan jadi 1 pesan jelas (mis. daftar nama dipisah koma) — JANGAN cuma tampilkan siswa pertama yang ketemu, guru perlu tahu SEMUA yang kurang sekaligus supaya tidak submit berkali-kali.

3. **Validasi keterangan wajib untuk status `tidak_ada_di_kelas`** — kalau `mark.status === "tidak_ada_di_kelas"` DAN `mark.keterangan` kosong/undefined → `BadRequestException` (jangan percaya FE sudah wajibkan popup, backend WAJIB re-validasi).

4. **Simpan `keterangan`** ke `ClassAttendanceMark` saat status `tidak_ada_di_kelas` — field `permitId` TIDAK disentuh di task ini (itu task-CORE-015, auto-sync dari Permit).

5. **PERTAHANKAN logic existing** yang sudah benar: status `hadir` tetap DIHAPUS barisnya (bukan disimpan eksplisit, KONSISTEN filosofi T038 "jangan simpan baris untuk status default") — TAPI untuk status `izin` (BARU), tetap disimpan eksplisit sama seperti `tidak_ada_di_kelas` (BUKAN default implisit).

6. **Tambah endpoint/param untuk "Hadir Semua"** — bisa berupa flag khusus di DTO (`applyAllHadir: boolean`) yang kalau true, backend hapus SEMUA baris `ClassAttendanceMark` existing untuk sesi itu (kembali ke default hadir semua) TANPA perlu FE kirim `marks[]` lengkap satu-satu. Verifikasi validasi "belum terabsen" (langkah 2) TIDAK berlaku kalau flag ini true (karena tujuannya justru skip checklist manual).

7. `activityLog.record()` existing tetap dipertahankan, TAMBAHKAN `keterangan` ke `snapshotAfter` untuk audit trail siapa isi alasan apa.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/journal/journal.service.ts` — `updateAttendance()`
- **Modifikasi:** `apps/api/src/journal/dto/update-attendance.dto.ts`

**Dilarang dilakukan:**
- Jangan ubah endpoint route (`PATCH :sessionId/attendance` tetap sama) — MURNI ubah validasi+payload di dalamnya, kompatibel dengan tetap 1 request per submit (bukan per-klik).
- Jangan hapus validasi `assertSessionStarted()` existing — tetap berlaku, sesi harus started dulu.

**Skenario kegagalan yang WAJIB ditangani:**
- Guru submit dengan 1 siswa belum terabsen → 400 dengan pesan nama siswa jelas, TIDAK ADA baris yang tersimpan sama sekali (all-or-nothing, JANGAN partial save yang bikin state tanggung).
- Guru pilih "Hadir Semua" tapi kelas kosong (0 siswa) → tidak error, cukup no-op.
- `studentId` di `dto.marks[]` yang BUKAN siswa kelas sesi ini → tetap ditolak (validasi existing SUDAH benar, JANGAN dihapus).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Submit presensi dengan siswa tap-gerbang-tapi-belum-checklist → 400, pesan `"<nama> belum terabsen"` (gabung semua nama kalau lebih dari 1)
- [ ] Submit status `tidak_ada_di_kelas` tanpa `keterangan` → 400
- [ ] `keterangan` tersimpan ke DB untuk status `tidak_ada_di_kelas`
- [ ] Status `izin` disimpan eksplisit (tidak dihapus seperti `hadir`)
- [ ] Endpoint "Hadir Semua" bekerja tanpa perlu validasi kelengkapan
- [ ] Activity log mencatat `keterangan` di snapshot

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 150 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: task-CORE-010 WAJIB selesai dulu (field `keterangan` harus ada di skema)

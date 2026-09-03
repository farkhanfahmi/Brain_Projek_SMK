# Task-CORE-013 / WEB-014: Edit Presensi Setelah Sesi Closed (Window 1 Minggu) + Cascade Auto-Unlock

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` §5. Dieksekusi oleh Claude Code.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Fitur BARU (belum ada UI/UX proper sebelumnya), menyentuh backend (buka jalur edit terkontrol + cascade unlock) DAN frontend (UI edit mode + entry point). Cascade unlock adalah bagian paling rawan bug kalau tidak presisi.

## 2. Konteks & Tujuan Utama

Baca `06-Features/desain-redesign-presensi-izin-keluar.md` §5. **Temuan investigasi**: backend `updateAttendance()` TIDAK block berdasarkan status closed (cuma cek `startedAt`), tapi frontend `/guru/presensi` hardcode `readOnly=true` dengan komentar "sementara". Task ini membangun fitur edit PROPER — bukan sekadar buka kunci yang ada.

**Keputusan final:**
- Window edit: **1 minggu** dari tanggal sesi.
- Cascade unlock: **AUTO-UNLOCK LANGSUNG** saat guru ubah Alfa→Hadir/Izin (bukan flag manual piket).

**Depends on:** task-CORE-010, task-CORE-011, task-CORE-012 (lock harus sudah bisa terjadi sebelum ada yang perlu di-cascade-unlock).

## 3. Langkah Eksekusi Detail

### Backend

1. Di `JournalService.updateAttendance()` (`apps/api/src/journal/journal.service.ts`) — TAMBAHKAN validasi window waktu SETELAH `assertSessionStarted()`: kalau `session.closedAt !== null` (sesi sudah closed, berarti ini mode EDIT bukan input pertama kali), cek `session.tanggal` masih dalam **1 minggu** dari `now` — kalau lewat, `ForbiddenException` dengan pesan jelas (mis. `"Presensi hanya bisa diedit dalam 1 minggu setelah sesi berlangsung"`).

2. **Cascade auto-unlock** — untuk tiap `mark` di payload yang statusnya BERUBAH dari `tidak_ada_di_kelas` (Alfa) menjadi `hadir`/`izin` DIBANDING data lama (`before`, sudah di-fetch existing di method ini) — cek apakah `Student` itu **sedang terkunci** DENGAN `lockedReason` yang match konteks sesi ini (REUSE keterangan/referensi yang disimpan task-CORE-012). Kalau match:
   - `Student.update({ lockedAt: null, unlockedAt: now, unlockedById: <system actor>, unlockNote: "Auto-unlock — presensi dikoreksi guru dari Alfa" })` — pakai `activityLog.getSystemActorId()` (SUDAH ADA, dipakai `lockGuruTanpaTapPulang`) sebagai `unlockedById`, REPLIKASI pola unlock yang sudah ada di proyek (cek endpoint unlock manual existing kalau ada, untuk konsistensi field mana yang diisi).
   - Catat ke `activityLog` — action `"student.unlock_auto_koreksi_presensi"`.

3. **Endpoint terpisah untuk cek "boleh edit atau tidak"** (opsional, untuk FE tampilkan/sembunyikan tombol Edit) — bisa ditambahkan sebagai field di response `GET /teaching-sessions/:id/detail` (mis. `bisa_diedit: boolean`, dihitung dari `closedAt` + window 1 minggu) — REUSE endpoint detail yang sudah ada, tidak perlu endpoint baru terpisah.

### Frontend

4. Di `apps/web/src/app/(guru)/guru/presensi/components/presensi-detail.tsx` — GANTI `readOnly` hardcoded `true` jadi kondisional berdasarkan `bisa_diedit` dari response (langkah 3) — tambah tombol "Edit Presensi" yang toggle mode (mirip pola state `expanded`/edit mode lain di proyek).

5. **Konfirmasi sebelum submit edit** — REUSE `ConfirmationDialog` (kalau sudah jadi komponen shared) atau `Dialog` biasa — tampilkan ringkasan perubahan sebelum benar-benar submit (mis. "3 siswa akan diubah statusnya"), KONSISTEN aturan proyek "operasi yang mengubah data penting wajib konfirmasi".

6. Kalau window 1 minggu terlewat — tombol Edit TIDAK ditampilkan sama sekali (bukan ditampilkan lalu error saat diklik) — cek `bisa_diedit` dari backend, jangan hitung ulang di FE (single source of truth di server).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/journal/journal.service.ts` — `updateAttendance()` (validasi window + cascade unlock)
- **Modifikasi:** `apps/api/src/journal/journal.controller.ts` — kemungkinan tambah `bisa_diedit` ke response detail
- **Modifikasi:** `apps/web/src/app/(guru)/guru/presensi/components/presensi-detail.tsx` — UI edit mode

**Dilarang dilakukan:**
- Jangan buka edit untuk sesi yang BELUM pernah closed (`closedAt === null`) — itu jalur normal existing via `/guru/sesi/[id]`, BUKAN scope task ini.
- Jangan auto-unlock siswa yang terkunci untuk ALASAN LAIN (bukan dari kontradiksi Alfa+tap-gerbang task-CORE-012) — cascade unlock HANYA untuk kasus yang match konteks sesi ini, JANGAN unlock serampangan siswa yang kebetulan sedang terkunci sebab lain.

**Skenario kegagalan yang WAJIB ditangani:**
- Guru edit presensi H+8 (lewat 1 minggu) → backend tolak, FE bahkan tidak tampilkan tombol Edit.
- Guru ubah Alfa→Hadir tapi siswa itu TERNYATA sudah di-unlock manual sebelumnya oleh piket (`lockedAt` sudah null) → cascade unlock jadi no-op (tidak ada yang perlu di-unlock lagi), TIDAK error.
- Guru ubah status BUKAN dari Alfa (mis. Hadir→Izin) → tidak ada cascade unlock sama sekali (hanya trigger dari Alfa→lainnya).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Presensi sesi closed bisa diedit dalam window 1 minggu dari tanggal sesi
- [ ] Edit setelah window lewat → ditolak backend, tombol tidak muncul di FE
- [ ] Ubah Alfa→Hadir/Izin men-trigger auto-unlock kartu siswa (kalau lock match konteks sesi ini)
- [ ] Semua perubahan tercatat ke activity log (termasuk unlock)
- [ ] Konfirmasi dialog muncul sebelum submit edit

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 200 baris perubahan gabungan BE+FE)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: task-CORE-010, task-CORE-011, task-CORE-012 WAJIB selesai dulu

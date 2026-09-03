# Task-CORE-015: Auto-Sync Permit → ClassAttendanceMark (Status Izin Otomatis)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` §7, §12. Dieksekusi oleh Claude Code.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Logic lintas-modul (Permit ↔ TeachingSession ↔ ClassAttendanceMark) yang harus dipanggil dari 2 titik berbeda (task-CORE-014 alur guru-kelas, DAN alur piket-langsung existing `IzinKeluarForm`/`PermitsService`) — perlu desain 1 service yang reusable dari kedua titik, bukan duplikasi logic.

## 2. Konteks & Tujuan Utama

Baca `06-Features/desain-redesign-presensi-izin-keluar.md` §7 (logika revisi final) dan §12 (alur "Permit unified"). **Keputusan final user**: izin di LUAR jam pelajaran aktif (istirahat dkk) — keputusan SEPENUHNYA di piket, guru kelas TIDAK perlu approve apa pun, tinggal MENERIMA status izin yang piket buat. Ini butuh mekanisme: begitu `Permit` (row) dibuat LEWAT JALUR MANAPUN, sistem otomatis cari sesi terkait dan set status presensi jadi Izin.

**Depends on:** task-CORE-010 (skema `permitId` di `ClassAttendanceMark`).

## 3. Langkah Eksekusi Detail

1. Buat method baru `syncPermitToAttendance(permit: Permit)` di service yang tepat (kemungkinan `PermitsService`, `apps/api/src/permits/permits.service.ts` — atau service baru terpisah kalau `PermitsService` sudah terlalu besar, TENTUKAN saat implementasi) — dipanggil di **2 titik**:
   - Setelah `PermitsService.create()` sukses (alur piket-langsung existing, `IzinKeluarForm` — untuk kasus izin di LUAR jam pelajaran aktif).
   - Setelah `ClassPermitRequestService.izinkan()` sukses (task-CORE-014, alur guru-kelas — untuk kasus SAAT jam pelajaran aktif).

2. Logic method:
   - Ambil `permit.tanggal` (+ `tanggalSelesai` kalau multi-hari) dan `permit.jamKeluar`/`jamKembaliDiharapkan`.
   - Cari SEMUA `TeachingSession` milik `permit.studentId` (via `Kelas` siswa itu, JOIN `TeachingSession.kelasId`) pada tanggal yang SAMA, yang **jendela waktu sesinya (jamMulai-jamSelesai, REUSE `resolveJamSesi()` existing di `TeachingSessionsService` — VERIFIKASI apakah perlu extract jadi shared util atau service ini punya akses ke method itu) OVERLAP dengan rentang `jamKeluar`-`jamKembaliDiharapkan` permit.
   - Untuk SETIAP sesi yang overlap — `ClassAttendanceMark.upsert({ status: "izin", permitId: permit.id, keterangan: <ringkasan dari Permit.alasanDetail> })`.

3. **Kasus khusus §7 (logika revisi final)**: kalau `jamKembaliDiharapkan` melewati BATAS jam pelajaran berikutnya (setelah istirahat) — method ini TETAP set status `izin` untuk SEMUA sesi yang overlap, TIDAK ADA pengecualian "guru kelas harus approve dulu" (logika lama yang SUDAH DIBATALKAN user) — cukup terapkan overlap waktu apa adanya.

4. **Idempotency** — kalau dipanggil 2x untuk `Permit` yang sama (mis. race condition), `upsert` mencegah duplikasi baris, TAPI pastikan tidak menimpa `keterangan` custom yang mungkin sudah diisi manual sebelumnya jika ada — VERIFIKASI urutan operasi (permit sync SEHARUSNYA jadi sumber utama untuk kasus ini, override manual guru sebelumnya wajar ditimpa KARENA ini kasus di luar jam pelajaran di mana guru memang tidak checklist).

5. **Tidak perlu** menyentuh kasus `Permit.jenis === "tidak_masuk"` (izin/sakit SEHARIAN) — task ini fokus `jenis: "keluar"` (SEBAGIAN hari) yang overlap dengan sesi tertentu. Kalau `tidak_masuk` juga perlu sync serupa, itu di luar scope task ini (kemungkinan sudah tertangani beda mekanisme, VERIFIKASI saat implementasi apakah ada gap di sana juga, laporkan terpisah kalau ditemukan).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/permits/permits.service.ts` — panggil `syncPermitToAttendance()` setelah create
- **Modifikasi:** service `ClassPermitRequestService` (task-CORE-014) — panggil method yang sama setelah `izinkan()`
- **File baru (kemungkinan):** util/service `syncPermitToAttendance()` kalau dipisah jadi shared

**Dilarang dilakukan:**
- Jangan duplikasi logic overlap-waktu — REUSE `resolveJamSesi()` yang sudah ada di `TeachingSessionsService`, jangan tulis ulang resolusi jam dari awal.
- Jangan sync untuk `Permit.jenis === "tidak_masuk"` di task ini — di luar scope eksplisit.

**Skenario kegagalan yang WAJIB ditangani:**
- Siswa tidak punya sesi mengajar sama sekali pada tanggal itu (hari libur/tidak ada jadwal) → tidak ada yang di-sync, tidak error.
- `TeachingSession` untuk sesi yang overlap SUDAH `closed` (sesi sudah selesai sebelum izin diproses piket, kasus telat) → TETAP sync statusnya (data historis dikoreksi retroaktif) — TAPI ini MENYENTUH kembali aturan window edit (task-CORE-013, 1 minggu) — VERIFIKASI interaksi kedua task ini tidak saling blokir (sync otomatis ini BUKAN "edit manual guru", jadi TIDAK perlu tunduk ke validasi window 1 minggu — auto-sync sistem punya previllege beda dari edit manual guru).
- Guru SUDAH checklist manual siswa itu Hadir SEBELUM izin piket diproses → sync ini AKAN MENIMPA jadi Izin (permit adalah otoritas lebih tinggi, konsisten §2.4 "guru tidak perlu checklist manual untuk kasus izin resmi").

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] `Permit` dibuat dari alur piket-langsung (istirahat dkk) → sesi yang overlap otomatis ter-set status Izin
- [ ] `Permit` dibuat dari alur guru-kelas-disetujui-piket (task-CORE-014) → sesi terkait otomatis ter-set status Izin
- [ ] `permitId` di `ClassAttendanceMark` terisi, bisa di-trace balik ke `Permit` asal
- [ ] Tidak ada duplikasi logic resolusi jam — REUSE `resolveJamSesi()`
- [ ] Idempotent terhadap panggilan berulang

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 120 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: task-CORE-010 WAJIB selesai. **Disarankan dieksekusi BARENGAN/SETELAH task-CORE-014** karena saling memanggil satu sama lain.

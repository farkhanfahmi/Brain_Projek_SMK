# Task-CORE-009: Fix Kalender Presensi Tidak Tampil untuk Sesi Belum Auto-Close

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kode Presensi 2026-09-02. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Perubahan kondisi query 1 baris (`status: "closed"` → kondisi lebih longgar), tapi perlu ketelitian memastikan tidak menampilkan sesi yang benar-benar belum ada datanya sama sekali (guru belum submit presensi apa pun).

## 2. Konteks & Tujuan Utama

Audit kode Presensi Guru 2026-09-02: `JournalService.getKalenderBulan()` (`apps/api/src/journal/journal.service.ts`) HANYA menandai tanggal di kalender kalau sesi berstatus `closed` DAN `closedAt` terisi. Kalau guru sudah submit presensi tapi sesi BELUM di-close (jam belum lewat, atau cron `autoCloseDueSessions` belum sempat jalan — job 5-menitan, ada jeda), tanggal itu **tidak muncul di kalender sama sekali** — meski datanya sudah tersimpan di `ClassAttendanceMark`. Guru bisa bingung "saya sudah isi presensi tapi kok kalendernya kosong".

**Root cause:** `closed`+`closedAt` dipakai sebagai proxy "sudah ada presensi", padahal proxy yang lebih akurat adalah **sesi sudah `startedAt` terisi** (sesi sudah dimulai, guru sudah punya kesempatan isi presensi) — TIDAK PERLU menunggu sesi ditutup untuk presensi dianggap "ada".

## 3. Langkah Eksekusi Detail

1. Di `apps/api/src/journal/journal.service.ts`, method `getKalenderBulan()` (baris ~398-432) — ubah kondisi `where` dari:
   ```ts
   status: "closed", closedAt: { not: null }
   ```
   menjadi:
   ```ts
   startedAt: { not: null }
   ```
   (sesi sudah dimulai = guru sudah punya kesempatan mengisi presensi, terlepas sudah/belum auto-close).

2. **Pertimbangkan pembedaan visual** di kalender antara "sesi sudah dimulai TAPI belum ditutup" (masih bisa berubah) vs "sesi sudah ditutup" (final) — kalau user MAU ada beda visual (mis. dot warna beda), diskusikan dulu saat eksekusi; kalau tidak, treat sama (kedua kondisi cukup ditandai dot yang sama seperti sekarang).

3. **Verifikasi tidak ada regresi** — kondisi baru (`startedAt: { not: null }`) harus TETAP menangkap semua kasus lama (`closed`+`closedAt` adalah subset dari `startedAt terisi`, karena sesi tidak bisa closed tanpa pernah started — VERIFIKASI asumsi ini benar dengan cek constraint/flow `autoCloseDueSessions()`, jangan asumsi tanpa cek).

4. Method `myClasses()`/`getKelasDiajar()` — TIDAK PERLU diubah (sudah scope by `teacherId` distinct, tidak terkait bug ini).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/journal/journal.service.ts` — `getKalenderBulan()`

**Dilarang dilakukan:**
- Jangan ubah definisi `jumlahHadir`/`jumlahTidakAdaDiKelas` — perhitungan itu sudah benar, cuma KONDISI SELEKSI tanggalnya yang salah.
- Jangan sentuh `autoCloseDueSessions()` — di luar scope, itu jadwal cron terpisah yang tidak perlu diubah.

**Skenario kegagalan yang WAJIB ditangani:**
- Sesi `startedAt` terisi TAPI guru belum sempat isi presensi sama sekali (baru mulai, semua siswa masih default "hadir" implisit, tidak ada baris `ClassAttendanceMark`) → tetap tampil di kalender (karena "hadir" adalah default valid, bukan berarti belum diisi — presensi TETAP ada meski semua implisit hadir, KONSISTEN filosofi T038 "jangan simpan baris untuk status default").

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Sesi yang sudah `startedAt` terisi (meski belum auto-close) tampil di kalender presensi
- [ ] Tidak ada regresi — sesi yang sebelumnya tampil (closed+closedAt) tetap tampil
- [ ] Perhitungan `jumlahHadir`/`jumlahTidakAdaDiKelas` tidak berubah logic-nya

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 15 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency

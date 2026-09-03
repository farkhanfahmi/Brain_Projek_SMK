# Task-CORE-008 / WEB-011: Tampilkan Jam KBM Terjadwal + Jam Mulai Aktual (dari Scan QR)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi dengan user (referensi visual "Jam KBM" vs "Jam KBM dilaksanakan" JurnalePro). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Data yang dibutuhkan (`startedAt`) SUDAH ADA di database DAN sudah di-query di service (`session.startedAt`), TAPI belum jelas apakah sudah di-expose ke response API sebagai field terpisah untuk keperluan tampilan ini. Effort rendah karena murni field baru + render tambahan, bukan logic baru.

## 2. Konteks & Tujuan Utama

Referensi user: JurnalePro menampilkan 2 baris waktu per sesi — baris atas "Jadwal KBM" (jam terjadwal, mis. `08:30 - 09:30 WIB`, warna netral/abu) dan baris bawah "Jam KBM dilaksanakan" (jam AKTUAL guru mulai, mis. `08:36 - 09:30 WIB`, warna berbeda/menonjol menunjukkan keterlambatan visual).

**Keputusan user:** di card sesi AbsenSI, tampilkan jam terjadwal (SUDAH ADA: `sesi.jam_mulai`–`sesi.jam_selesai`) DAN di bawahnya jam AKTUAL guru mulai mengajar — **dihitung dari saat berhasil scan QR siswa** (yaitu `TeachingSession.startedAt`, field yang di-set saat `startSession()` sukses, dicek di `teaching-sessions.service.ts`).

**Temuan teknis (dicek Hermes sebelum menulis task ini):**
- `TeachingSession.startedAt` (kolom `DateTime?`) SUDAH ADA di schema DAN SUDAH di-return di `SesiHariIni` interface backend (`startedAt: session.startedAt` di `teaching-sessions.service.ts`) — TAPI belum dicek apakah field ini SUDAH sampai ke frontend `SesiHariIniRow` (`core-types.ts`) dalam format yang bisa langsung dipakai (perlu verifikasi nama field snake_case + format tanggal/waktu saat eksekusi).
- Kalau field sudah ada di FE types tapi belum dirender — task ini MURNI frontend (render saja). Kalau field BELUM ada di FE types — perlu tambah mapping di controller/DTO response juga.

## 3. Langkah Eksekusi Detail

1. **Verifikasi dulu** apakah `startedAt`/`started_at` SUDAH sampai ke response JSON `GET /teaching-sessions/my-today` yang dikonsumsi frontend (cek `teaching-sessions.controller.ts` method yang handle endpoint ini — apakah dia `return` langsung objek dari service atau ada DTO/mapping terpisah yang mungkin men-strip field ini). Kalau field BELUM di-passthrough ke response HTTP, tambahkan.
2. Update `apps/web/src/lib/core-types.ts` — pastikan `SesiHariIniRow` punya field `started_at: string | null` (ISO datetime string atau null kalau sesi belum dimulai) — cek field lain di interface yang sama (`closed_at` SUDAH ADA di baris 561, kemungkinan besar `started_at` juga sudah ada, TINGGAL VERIFIKASI bukan asumsi ada/tidak).
3. Di `apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx`, di bawah baris jam terjadwal (baris 147-149 existing: `{sesi.jam_mulai} – {sesi.jam_selesai}`), tambahkan baris BARU kondisional:
   ```
   {sesi.started_at && (
     <p className="text-caption text-primary-hover">
       Dimulai: {formatJamDariISOString(sesi.started_at)} WIB
     </p>
   )}
   ```
   Format jam dari ISO string ke `HH:MM` — cek apakah sudah ada util format jam serupa di proyek (`apps/web/src/lib/`) sebelum menulis fungsi baru dari nol, REUSE kalau ada.
4. **Styling** — warna teks BERBEDA dari jam terjadwal (mis. jam terjadwal `text-ink-secondary` netral, jam aktual `text-primary-hover` atau `text-success-text` untuk menandakan "ini realisasi, bukan rencana") — SELARAS dengan task-WEB-009 (warna beda per status), jangan bentrok pilihan warna, koordinasikan token yang dipakai supaya konsisten skema warna keseluruhan card.
5. **Tidak perlu bandingkan/hitung keterlambatan secara eksplisit** (mis. "Terlambat 6 menit") di task ini — SEKADAR tampilkan 2 angka jam berdampingan seperti referensi JurnalePro, biarkan guru/admin yang menyimpulkan visual. Kalau user MAU indikator keterlambatan eksplisit nanti, itu task terpisah (perlu keputusan definisi "terlambat" — bandingkan ke jam mulai jadwal atau ke jendela toleransi tertentu, BELUM didiskusikan).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi (kemungkinan, VERIFIKASI dulu perlu atau tidak):** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` — pastikan `startedAt` passthrough ke response
- **Modifikasi (kemungkinan, VERIFIKASI dulu):** `apps/web/src/lib/core-types.ts` — field `started_at`
- **Modifikasi:** `apps/web/src/app/(guru)/guru/jadwal/components/sesi-card.tsx` — render baris jam aktual

**Dilarang dilakukan:**
- Jangan hitung/tampilkan durasi keterlambatan (di luar scope eksplisit task ini — user cuma minta TAMPILKAN jam mulai aktual, bukan analisis keterlambatan).
- Jangan ubah logic `startSession()` backend — `startedAt` SUDAH di-set dengan benar saat scan QR sukses (dikonfirmasi `T043`/kode existing), task ini MURNI menampilkan data yang sudah ada, bukan mengubah kapan/bagaimana data itu terisi.

**Skenario kegagalan yang WAJIB ditangani:**
- Sesi belum dimulai (`started_at === null`) → baris jam aktual TIDAK DITAMPILKAN sama sekali (bukan "-" atau string kosong membingungkan) — kondisional render murni.
- Sesi sudah `closed_at` terisi (sesi sudah selesai) → jam aktual TETAP tampil (histori "jam berapa dia mulai", relevan meski sesi sudah selesai — BEDA dari task real-time lain, ini historical record).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Jam mulai AKTUAL (dari `startedAt`, hasil scan QR sukses) tampil di bawah jam terjadwal
- [ ] Field kosong/null saat sesi belum dimulai → baris tidak dirender, bukan placeholder kosong
- [ ] Data historis tetap tampil untuk sesi yang sudah selesai
- [ ] Warna teks berbeda dari jam terjadwal, dikoordinasikan dengan skema warna task-WEB-009

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 40 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency, TAPI disarankan dieksekusi SETELAH task-WEB-009 (warna status) supaya pilihan warna jam aktual tidak bentrok/perlu direvisi ulang

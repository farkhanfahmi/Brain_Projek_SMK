# Task-CORE-003: Refactor N+1 Query di cekKetersediaanGuru (Batch Query)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Refactor logic query (bukan tambah fitur baru), butuh ketelitian supaya hasil akhir (deteksi bentrok) TETAP SAMA PERSIS dengan versi lama — regresi logic bentrok dampaknya besar (guru bisa ke-assign ganda kalau refactor salah). Bukan task trivial meski "cuma performa".

**⚠️ CATATAN PENTING:** Task ini **HARUS dieksekusi SETELAH task-CORE-002 selesai** (fix silent-fail bentrok), karena keduanya menyentuh method yang sama (`findBentrokConflict`/`cekKetersediaanGuru`) — mengerjakan bersamaan/terbalik urutan berisiko konflik logic atau salah satu fix ke-overwrite yang lain.

## 2. Konteks & Tujuan Utama

Audit menu Jadwal Pelajaran (sesi diskusi 2026-09-02) menemukan pola N+1 query di `JadwalSlotService.cekKetersediaanGuru()` (`apps/api/src/jadwal-slot/jadwal-slot.service.ts`, baris ~303-340).

**Masalah:** endpoint ini dipanggil **real-time** oleh FE setiap kali admin membuka dropdown guru di builder Jadwal Pelajaran (badge "Mengajar di Kelas X" real-time, T212). Untuk tiap kandidat guru (`candidates`, hasil query `mapelGuru.findMany` — bisa banyak untuk mapel umum seperti PJOK/Agama yang diampu banyak guru), method ini memanggil `findBentrokConflict()` di dalam loop — yang di dalamnya SENDIRI melakukan query lagi (`jadwalSlotGuru.findMany` + `resolveJamForOpsiJadwal` per kandidat slot). Total query bisa jadi `O(guru × slot_existing_guru)`, bukan konstan.

User mengonfirmasi **memang terasa lag** setiap klik menu ini pertama kali (tapi tidak lag lagi setelah cache/kembali ke menu yang sama) — sesuai gejala N+1 query yang mahal di request pertama.

**Tujuan:** batch-kan semua query jadi jumlah tetap (tidak bergantung jumlah kandidat guru), sambil **mempertahankan hasil akhir yang identik** dengan implementasi lama.

**Depends on:** task-CORE-002 (fix silent-fail bentrok) — HARUS selesai duluan, karena refactor ini menyentuh method yang sama persis.

## 3. Langkah Eksekusi Detail

1. **Baca ulang `cekKetersediaanGuru()` versi TERBARU** (setelah task-CORE-002 diterapkan) sebelum mulai — pastikan refactor ini dibangun di atas kontrak response yang sudah termasuk field "tidak bisa dipastikan" dari task sebelumnya, bukan versi lama.

2. **Batch query kandidat slot existing** — alih-alih `findBentrokConflict()` dipanggil per-kandidat-guru (yang di dalamnya query `jadwalSlotGuru.findMany` per teacherId satu-satu), lakukan SEKALI di awal `cekKetersediaanGuru()`:
   ```ts
   const allCandidateSlots = await this.prisma.jadwalSlotGuru.findMany({
     where: {
       teacherId: { in: candidates.map((c) => c.teacherId) },
       jadwalSlot: {
         hari: params.hari,
         opsiJadwal: { isActive: true },
         id: params.excludeSlotId ? { not: params.excludeSlotId } : undefined,
       },
     },
     include: { jadwalSlot: { include: { opsiJadwal: true, kelas: true } }, teacher: true },
   });
   ```
   Kelompokkan hasil per `teacherId` di memori (JS `Map` atau `groupBy` manual) — bukan query ulang per teacher.

3. **Batch resolve jam** — kumpulkan SEMUA kombinasi unik `(opsiJadwalId, hari, jamKe)` yang perlu di-resolve (baik jam target `params` maupun semua kandidat dari langkah 2), dedupe, lalu resolve SEKALI per kombinasi unik (bukan per baris kandidat). Simpan hasil di `Map<string, {jamMulai, jamSelesai} | null>` dengan key gabungan `${opsiJadwalId}-${hari}-${jamKe}`, reuse dari Map ini saat overlap-check di memori.

   Cek dulu apakah `AlokasiWaktuService.resolveJam()` sendiri sudah efisien (mis. sudah query per-AlokasiWaktu sekali lalu cache internal) — kalau belum, pertimbangkan juga apakah perlu method baru `resolveJamBatch()` di `AlokasiWaktuService` yang menerima banyak kombinasi sekaligus dan return sekali query. **Jangan over-engineer** — cukup dedupe di level pemanggil (Map cache) dulu, itu sudah menghilangkan mayoritas query berulang; baru pertimbangkan ubah `AlokasiWaktuService` kalau ternyata masih jadi bottleneck.

4. **Lakukan overlap-check (deteksi bentrok) di memori JS**, bukan query per candidate — logic pembanding menit (`toMinutes`, overlap formula `mulaiMenit < cSelesai && cMulai < selesaiMenit`) TETAP SAMA PERSIS, cuma sumber datanya sekarang dari data yang sudah di-batch-fetch di langkah 2-3, bukan query on-demand.

5. **Pastikan hasil akhir IDENTIK** dengan versi sebelum refactor — untuk setiap kandidat guru, kategorisasi (tersedia / bentrok pasti / tidak bisa dipastikan, sesuai kontrak task-CORE-002) harus menghasilkan output yang SAMA PERSIS dengan implementasi lama untuk input yang sama. Ini murni optimisasi cara ambil data, BUKAN perubahan logic bisnis.

6. **Tambah test unit** (atau perluas `jadwal-slot.service.spec.ts` yang sudah ada) yang membandingkan skenario dengan banyak kandidat guru (≥5) dan verifikasi jumlah query DB yang terpanggil (kalau ada tooling untuk itu, mis. Prisma query counter/spy) TIDAK bertumbuh linear terhadap jumlah kandidat — atau minimal test functional correctness-nya kalau spy query count tidak feasible di setup test yang ada.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/jadwal-slot/jadwal-slot.service.ts` — `cekKetersediaanGuru()`, `findBentrokConflict()` (kemungkinan di-refactor total jadi helper batch baru, method lama bisa tetap dipertahankan untuk dipakai `create()`/`update()` single-slot yang tidak butuh batch)
- **Kemungkinan modifikasi:** `apps/api/src/alokasi-waktu/alokasi-waktu.service.ts` — HANYA kalau langkah 3 menyimpulkan perlu `resolveJamBatch()` baru. Kalau iya dan ini dipakai service lain juga, pastikan tidak mengubah behavior existing `resolveJam()` singular (harus tetap ada, `resolveJamBatch()` sebagai TAMBAHAN bukan pengganti).
- **Jangan sentuh:** `create()`, `update()` (single-slot validation) di file yang sama — method-method itu memanggil `ensureNoBentrok()` untuk 1 guru saja per call, TIDAK perlu batch (N+1 di sana tidak masalah karena N=1). Fokus HANYA pada `cekKetersediaanGuru()` yang punya banyak kandidat.

**Dilarang dilakukan:**
- Jangan ubah hasil/kontrak logic bentrok — refactor ini murni soal EFISIENSI query, bukan mengubah kapan sesuatu dianggap bentrok/tidak.
- Jangan skip langkah testing (poin 6) — regresi silent di logic bentrok itu risiko tinggi (guru ke-assign dobel tanpa terdeteksi), harus diverifikasi sebelum dianggap selesai.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: 2 kandidat guru berbeda kebetulan share kombinasi `(opsiJadwalId, hari, jamKe)` yang sama di kandidat slot mereka → cache Map harus benar-benar dedupe berdasarkan kombinasi itu, bukan per-guru, supaya tidak resolve jam yang sama berkali-kali.
- Kondisi: `excludeSlotId` (dipakai saat edit slot existing) — pastikan filter ini tetap konsisten diterapkan di query batch, jangan sampai slot yang sedang diedit ikut dihitung sebagai "kandidat bentrok" terhadap dirinya sendiri.
- Kondisi: tidak ada kandidat guru sama sekali (`candidates.length === 0`, mapel belum ada guru terdaftar) → return awal cepat tanpa query batch yang sia-sia (guard clause).

**Edge case:**
- Jumlah kandidat sangat besar (>50 guru untuk 1 mapel, kasus tidak realistis tapi guard tetap baik) → query `IN` dengan array besar tetap 1 round-trip DB, jadi tidak masalah dari sisi jumlah query — cukup dipastikan tidak ada masalah lain (payload besar dsb, di luar scope task ini kalau memang tidak pernah terjadi di skala sekolah ini).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] `cekKetersediaanGuru()` tidak lagi melakukan query di dalam loop per-kandidat-guru — jumlah query DB konstan (tidak bertumbuh linear terhadap jumlah kandidat)
- [ ] Hasil kategorisasi guru (tersedia/bentrok pasti/tidak bisa dipastikan) IDENTIK dengan versi sebelum refactor untuk skenario yang sama
- [ ] Test unit baru/diperluas memverifikasi correctness pada skenario banyak kandidat guru
- [ ] Tidak ada regresi di endpoint `create()`/`update()` JadwalSlot (yang juga pakai `ensureNoBentrok()`/`findBentrokConflict()` versi single)
- [ ] Secara subjektif, user konfirmasi lag saat pertama buka accordion Guru di builder Jadwal Pelajaran berkurang

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; refactor query batch biasanya net-negative LOC meski butuh restrukturisasi)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] **Dependency task-CORE-002 SUDAH selesai** sebelum task ini di-assign — WAJIB, jangan dikerjakan paralel/duluan

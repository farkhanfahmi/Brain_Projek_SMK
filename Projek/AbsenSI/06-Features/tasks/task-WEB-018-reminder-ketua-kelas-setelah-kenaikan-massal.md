# Task-WEB-018: Reminder Assign Ulang Ketua Kelas Setelah Kenaikan Kelas Massal

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit fitur pengaturan admin yang mempengaruhi modul Jurnal Guru, 2026-09-02. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Murni tambahan tampilan (ringkasan hasil + query cek kelas tanpa pengurus) di halaman yang sudah ada — tidak ada perubahan logic bisnis `kenaikanMassal()` itu sendiri.

## 2. Konteks & Tujuan Utama

Audit fitur pengaturan admin, 2026-09-02: `KelasService.kenaikanMassal()` (`apps/api/src/core/kelas/kelas.service.ts`) memindahkan siswa ke kelas tujuan tiap tahun ajaran baru. `KelasPengurus` (ketua/wakil ketua kelas, dipakai QR mulai sesi via `qr-mulai-sesi.service.ts`) di-scope `academicYearId` — DESAIN INI SUDAH BENAR (pengurus lama otomatis tidak berlaku lagi begitu tahun ajaran baru mulai, mencegah ketua kelas lama yang sudah pindah kelas/lulus tetap py akses).

**Gap yang ditemukan**: setelah kenaikan massal selesai, `KenaikanMassalView` (`apps/web/src/app/(admin)/kelas/kenaikan-massal/kenaikan-massal-view.tsx`) HANYA menampilkan ringkasan angka (jumlah naik/lulus/tinggal) — TIDAK ADA reminder ke admin bahwa kelas-kelas yang siswanya baru dipindah/naik **butuh Ketua/Wakil Ketua Kelas baru di-assign** untuk tahun ajaran baru (kelas tanpa pengurus = fitur "Mulai Pembelajaran" via QR TIDAK BISA dipakai kelas itu — dikonfirmasi dari `qr-mulai-sesi.service.ts`, siswa harus `KelasPengurus` aktif untuk generate QR).

Dampak nyata: admin baru sadar masalah ini saat semester baru dimulai dan guru komplain "kelas X tidak bisa mulai presensi QR" — reaktif, bukan proaktif.

**Depends on:** Tidak ada dependency teknis.

## 3. Langkah Eksekusi Detail

1. Di `apps/api/src/core/kelas/kelas.service.ts`, `kenaikanMassal()` — SETELAH transaksi selesai (sebelum `return result`), tambahkan query CEK kelas TUJUAN (bukan kelas asal) yang BELUM punya `KelasPengurus` untuk `academicYearId` tujuan:
   ```ts
   const kelasTujuanIds = [...new Set(dto.proses.filter(p => !p.lulus).map(p => p.kelasTujuanId!))];
   const kelasDenganPengurus = await this.prisma.kelasPengurus.findMany({
     where: { kelasId: { in: kelasTujuanIds }, academicYearId: dto.tahunAjaranTujuanId },
     select: { kelasId: true },
     distinct: ["kelasId"],
   });
   const kelasIdDenganPengurus = new Set(kelasDenganPengurus.map(k => k.kelasId));
   const kelasTanpaPengurus = kelasList.filter(k => kelasTujuanIds.includes(k.id) && !kelasIdDenganPengurus.has(k.id));
   ```
   Tambahkan `kelasTanpaPengurus: Array<{ id: number; nama: string }>` ke `KenaikanMassalResult` interface dan response.

2. **Verifikasi konteks** — kelas TUJUAN yang dicek adalah kelas yang siswa dipindahkan KE SANA (`kelasTujuanId`), BUKAN kelas asal (`kelasAsalId`, yang sudah "usang" setelah kenaikan). Kelas asal TIDAK perlu dicek (irrelevant, siswa sudah pindah).

3. **Kasus kelas tujuan SUDAH punya pengurus dari proses lain** (mis. kelas X sekaligus jadi tujuan kenaikan DUA kelas asal berbeda — bukan skenario umum tapi mungkin terjadi) — pengurus yang sudah ada (kalau memang sudah pernah di-assign manual sebelumnya untuk tahun ajaran itu) TIDAK perlu direset, cukup dicek KEBERADAANNYA saja (query di atas sudah benar, `distinct` mencegah duplikasi).

### Frontend

4. Di `apps/web/src/app/(admin)/kelas/kenaikan-massal/kenaikan-massal-view.tsx` — setelah `hasil` (state existing, hasil response backend) diterima, TAMBAHKAN section baru di area hasil (dekat ringkasan naik/lulus/tinggal existing) menampilkan `hasil.kelasTanpaPengurus` KALAU tidak kosong:
   - Judul jelas: "⚠ X kelas belum punya Ketua Kelas untuk tahun ajaran ini"
   - Daftar nama kelas
   - Link/tombol ke halaman assign pengurus kelas (cek route existing untuk assign `KelasPengurus`, KEMUNGKINAN di halaman detail kelas atau menu terpisah — TENTUKAN saat implementasi, REUSE route yang sudah ada, JANGAN buat halaman baru).
   - Pesan penjelasan singkat: "Kelas tanpa Ketua/Wakil Ketua tidak bisa memakai fitur Mulai Pembelajaran via QR."

5. Kalau `kelasTanpaPengurus` KOSONG (semua kelas tujuan sudah punya pengurus, kemungkinan kecil tapi valid kalau admin sudah assign duluan) — section ini TIDAK ditampilkan sama sekali (jangan tampilkan pesan kosong "semua kelas sudah lengkap", cukup hilang).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/core/kelas/kelas.service.ts` — `kenaikanMassal()`, interface `KenaikanMassalResult`
- **Modifikasi:** `apps/web/src/app/(admin)/kelas/kenaikan-massal/kenaikan-massal-view.tsx` — tampilkan reminder

**Dilarang dilakukan:**
- Jangan otomatis assign ketua kelas — MURNI reminder informasional, keputusan siapa jadi ketua kelas tetap manual oleh admin/wali kelas.
- Jangan cek kelas yang tujuannya "Lulus" (`item.lulus === true`) — kelas lulus tidak butuh pengurus baru (siswa sudah keluar sistem aktif).
- Jangan sentuh logic kenaikan itu sendiri (pemindahan siswa, status lulus/tinggal kelas) — task ini MURNI tambahan reminder di akhir proses, bukan mengubah alur.

**Skenario kegagalan yang WAJIB ditangani:**
- Kenaikan massal HANYA memproses kelas yang diluluskan (`lulus: true` semua) → `kelasTujuanIds` kosong, tidak ada kelas untuk dicek, section reminder tidak tampil (tidak error).
- Kelas tujuan yang SAMA dipakai lebih dari 1 kelas asal (jarang tapi mungkin) → dedupe via `Set`, tidak double-hitung.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Response `kenaikanMassal()` menyertakan daftar kelas tujuan yang belum punya `KelasPengurus` untuk tahun ajaran tujuan
- [ ] UI menampilkan reminder jelas dengan daftar kelas + link assign pengurus, HANYA kalau ada temuan
- [ ] Kelas dengan tujuan "Lulus" tidak dicek/tidak masuk daftar reminder
- [ ] Tidak ada perubahan pada logic kenaikan kelas itu sendiri (murni tambahan read-only setelah proses selesai)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 80 baris perubahan gabungan BE+FE)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada

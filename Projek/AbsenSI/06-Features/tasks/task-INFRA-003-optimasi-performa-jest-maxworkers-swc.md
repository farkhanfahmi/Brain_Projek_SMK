# Task-INFRA-003: Optimasi Performa Jest (maxWorkers + Evaluasi SWC) — Device Dev Lag Saat Test

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk) / INFRA (tooling/infra lintas-app).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low-medium
**Alasan pemilihan:** Langkah 1 (maxWorkers) murni ubah config, low-risk, cepat. Langkah 2 (evaluasi SWC) butuh verifikasi seluruh suite test tetap hijau setelah ganti transform — perlu ketelitian tapi bukan reasoning berat, sebatas "jalankan test, cek tidak ada regresi".

## 2. Konteks & Tujuan Utama

Diskusi 2026-09-02: user melaporkan device dev (Windows, satu-satunya device untuk semua kerja — Docker MySQL/Redis, editor, dst) **selalu lag berat setiap menjalankan jest test** — RAM terpakai ~98%, CPU 100%, meski tidak sampai freeze total.

**Analisa akar masalah** (dari `apps/api/package.json`):
1. Script `"test": "jest"` **tidak membatasi `maxWorkers`** — default jest men-spawn worker sejumlah (core CPU − 1), tiap worker adalah proses Node terpisah.
2. Config jest pakai **`ts-jest`** sebagai transform (`"transform": { "^.+\\.(t|j)s$": "ts-jest" }`) — `ts-jest` menjalankan full TypeScript type-checker per file per worker, jauh lebih berat dibanding transpiler murni (SWC/Babel). Proyek ini **sudah punya `@swc/core`** di devDependencies tapi belum dipakai untuk jest.
3. Kombinasi banyak-worker × type-checker-berat = penyebab RAM/CPU spike yang dilaporkan user.

**Catatan konteks:** test yang dimaksud adalah **unit test** (`*.spec.ts` di `apps/api/src/`, config jest di `package.json` root `apps/api`) — BUKAN e2e (`test:e2e` pakai config terpisah `test/jest-e2e.json`, tidak termasuk scope task ini kecuali user eksplisit minta diperluas). Unit test ini kemungkinan besar mock `PrismaService` (bukan konek ke MySQL/Redis asli), jadi Docker BUKAN penyebab beban — murni jest+ts-jest.

**Keputusan user:** perbaiki performa jest, JANGAN kurangi coverage test untuk mengatasi masalah device — test tetap penting untuk logic bisnis kompleks proyek ini (bentrok jadwal, geofence, RBAC), yang justru pernah terbukti bug baru ketahuan saat live karena kurang tertangkap test otomatis.

**Depends on:** Tidak ada — independen, bisa dikerjakan kapan saja, tidak terkait 6 task audit menu Jadwal sebelumnya.

## 3. Langkah Eksekusi Detail

### Langkah 1 — Batasi maxWorkers (cepat, low-risk, kerjakan dulu)

1. Di `apps/api/package.json`, tambahkan `maxWorkers` ke config jest (baris 70-86):
   ```json
   "jest": {
     "maxWorkers": 2,
     "moduleFileExtensions": [...],
     ...
   }
   ```
   Nilai `2` adalah titik awal aman untuk device dengan RAM terbatas — kalau setelah ini masih terasa berat, boleh turunkan ke `1` (`--runInBand` efeknya serupa, full sequential, paling ringan tapi paling lambat durasi totalnya). **Diskusikan dengan user preferensi trade-off kecepatan vs keringanan device kalau `2` masih berat** — jangan asumsi sepihak turun ke `1` tanpa observasi dulu.

2. **Verifikasi dengan menjalankan test suite penuh sekali** (`pnpm --filter @absensi/api test` atau setara) sambil memantau Task Manager/Resource Monitor Windows — catat perbandingan RAM/CPU peak SEBELUM vs SESUDAH perubahan ini di ringkasan hasil task, supaya user punya angka konkret bukan cuma "kira-kira lebih ringan".

3. Terapkan `maxWorkers` yang sama di `apps/web` dan `apps/kiosk` KALAU mereka juga punya jest config sendiri dengan masalah serupa — cek dulu apakah kedua app itu punya test suite jest aktif (`grep "\"test\":" apps/web/package.json apps/kiosk/package.json`), jangan asumsi otomatis sama seperti `apps/api`.

### Langkah 2 — Evaluasi migrasi ts-jest → @swc/jest (lebih besar, verifikasi ketat)

4. **Install `@swc/jest`** sebagai devDependency di `apps/api` (kompatibel dengan `@swc/core` yang sudah ada).

5. **Ubah config transform** di `package.json` jest block:
   ```json
   "transform": {
     "^.+\\.(t|j)s$": ["@swc/jest", { /* opsi config, sesuaikan dgn kebutuhan decorator NestJS */ }]
   }
   ```
   **PENTING — NestJS pakai decorator TypeScript secara ekstensif** (`@Injectable()`, `@Controller()`, dst) yang butuh `experimentalDecorators`+`emitDecoratorMetadata` aktif. SWC punya dukungan ini tapi behaviornya BISA BEDA dari `tsc`/`ts-jest` di kasus tertentu (terutama `emitDecoratorMetadata` untuk reflection-based DI Nest). **WAJIB riset dulu** apakah `@swc/jest` + NestJS teruji kompatibel (banyak proyek Nest lain sudah migrasi ke SWC untuk build produksi via `@nestjs/cli` opsi `--builder swc`, tapi UNTUK JEST TEST spesifiknya perlu dicek konfigurasi `.swcrc`/opsi jest yang benar) sebelum menerapkan across-the-board.

6. **Jalankan SELURUH suite test yang ada** setelah migrasi transform — bandingkan hasil PASS/FAIL SEBELUM dan SESUDAH. Kriteria sukses: **jumlah test yang lulus harus IDENTIK** (tidak ada test yang tadinya lulus jadi gagal karena decorator metadata salah resolve, atau type-checking yang tadinya menangkap error tipe sekarang lolos diam-diam karena SWC tidak type-check).

7. **Trade-off yang WAJIB dikomunikasikan ke user sebelum PR/selesai:** `ts-jest` melakukan type-checking penuh saat test (kalau ada error tipe, test GAGAL sebelum sempat jalan) — `@swc/jest` TIDAK type-check (murni transpile, lebih cepat tapi tidak menangkap error tipe). Kalau proyek ini mengandalkan `jest` sebagai lapisan deteksi error tipe (selain `tsc --noEmit`/lint terpisah), migrasi ke SWC berarti kehilangan safety net itu DI JEST — perlu dipastikan ada `tsc --noEmit` terpisah di alur kerja (CI/pre-commit/dsb) yang tetap menangkap error tipe, supaya migrasi performa ini tidak diam-diam menurunkan kualitas deteksi bug.

8. **Kalau langkah 4-7 terbukti berisiko/kompleks** (decorator metadata NestJS bermasalah, atau effort verifikasi lebih besar dari perkiraan) — **STOP, laporkan ke user**, cukup terapkan Langkah 1 (`maxWorkers`) saja sebagai perbaikan minimum yang aman. Migrasi SWC bersifat **best-effort improvement**, bukan wajib selesai dalam 1 kali eksekusi task ini.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/package.json` — tambah `maxWorkers`, dan (kalau lanjut Langkah 2) ubah `transform` + tambah `@swc/jest` ke devDependencies
- **Modifikasi (kondisional):** `apps/web/package.json`, `apps/kiosk/package.json` — HANYA kalau langkah 3 menemukan config jest serupa yang perlu disamakan
- **Kemungkinan file baru:** `.swcrc` atau config SWC terpisah — HANYA kalau diperlukan untuk opsi decorator metadata NestJS (langkah 5)
- **Jangan sentuh:** `test/jest-e2e.json` (config e2e terpisah) — di luar scope kecuali user eksplisit minta diperluas ke e2e juga.

**Dilarang dilakukan:**
- Jangan hapus/kurangi test yang sudah ada untuk "mengurangi beban" — scope task ini MURNI soal cara test dijalankan (config), bukan isi/jumlah test.
- Jangan skip verifikasi PASS/FAIL sebelum-sesudah migrasi SWC (langkah 6) — regresi silent di sini (test yang tadinya menangkap bug sekarang lolos) jauh lebih mahal daripada device yang lag.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: setelah `maxWorkers: 2`, device MASIH lag signifikan → Perilaku yang benar: laporkan angka RAM/CPU aktual ke user, tawarkan turun ke `maxWorkers: 1` sebagai langkah lanjutan, JANGAN diam-diam ubah tanpa konfirmasi (durasi test akan jadi lebih lama, user perlu tahu trade-off-nya).
- Kondisi: migrasi ke `@swc/jest` menyebabkan test NestJS yang mengandalkan Dependency Injection (`@Injectable()` constructor injection dst) gagal karena metadata decorator tidak ke-resolve dengan benar → Perilaku yang benar: ROLLBACK transform ke `ts-jest`, laporkan temuan ke user (SWC tidak cocok untuk struktur proyek ini), tetap pertahankan hasil Langkah 1 (`maxWorkers`) yang sudah aman.
- Kondisi: `apps/web`/`apps/kiosk` ternyata TIDAK punya test suite jest aktif sama sekali (kemungkinan besar, berdasarkan `STATUS.md` yang tidak menyebut test FE) → skip langkah 3 untuk app itu, tidak perlu dipaksakan.

**Edge case:**
- Kalau di masa depan proyek ini nambah e2e test yang benar-benar konek ke MySQL/Redis Docker (beda dari unit test sekarang) — task ini TIDAK mencakup optimasi itu, perlu task terpisah kalau device juga lag saat e2e nanti dijalankan.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] `apps/api/package.json` jest config punya `maxWorkers` eksplisit (nilai disepakati dari observasi, mulai dari `2`)
- [ ] Perbandingan RAM/CPU peak sebelum-sesudah `maxWorkers` didokumentasikan di ringkasan hasil task (angka konkret, bukan estimasi)
- [ ] SEMUA test existing tetap PASS setelah perubahan `maxWorkers` (config saja, tidak mengubah hasil test)
- [ ] (Kalau Langkah 2 dilanjutkan) Evaluasi `@swc/jest` didokumentasikan hasilnya — baik "berhasil diterapkan, semua test tetap pass" ATAU "di-rollback, ts-jest dipertahankan, alasan teknisnya dicatat"
- [ ] User dikonfirmasi soal trade-off type-checking-di-jest kalau migrasi SWC benar-benar diterapkan (langkah 7)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan — task ini jauh di bawah itu, murni config)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency

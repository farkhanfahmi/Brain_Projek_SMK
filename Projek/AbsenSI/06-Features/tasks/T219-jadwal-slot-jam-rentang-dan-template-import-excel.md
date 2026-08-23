# T219 — API+Web: JadwalSlot Terima Input "Jam Ke Awal-Akhir" (Expand ke Baris) + Template Import Sama Persis Excel User

## Depends on
**WAJIB SETELAH T206, T207, T208** (butuh `JadwalSlotService`, `MapelGuru`, generate minggu A/B sudah berjalan). Modifikasi/perluasan atas fitur yang sudah dieksekusi (T204-T210 sudah selesai — lihat STATUS.md), BUKAN fitur baru dari nol.

## Konteks — Kebutuhan User (2026-08-18)

User punya file kerja nyata `dariDev/Sistem Jurnal Baru 2026-2027.xlsx` (SMK Kartanegara Wates) berisi 11 sheet dengan data riil (guru, kelas, mapel, alokasi waktu, penjadwalan per-guru) yang SAAT INI diinput 1 baris = 1 jam pelajaran individual — user ingin efisiensi: 1 baris input/import bisa mewakili RENTANG jam ke (misal "Jam Ke Awal 2, Jam Ke Akhir 7" untuk 1 guru mengajar 1 mapel di 1 kelas selama 6 jam pelajaran berturut-turut), TANPA perlu invididual per jam.

**Riset kode (2026-08-18) menemukan dampak MELUAS bila field `JadwalSlot.jamKe` diganti total** (dipakai di ~15 titik: `JadwalSlotService`, `TeachingSessionsService.resolveJamSesi()` dipanggil 6 tempat, `AlokasiWaktuService.resolveJam()`, `journal.service.ts`, `tv-piket.service.ts`, `jadwal-pelajaran.service.ts`, DTO, seluruh UI grid `guru-hari-input.tsx`/`tab-hari-input.tsx` yang mental model-nya "1 baris = 1 jamKe"). Constraint unique DB (`@@unique([opsiJadwalId, kelasId, hari, jamKe, minggu])`) juga TIDAK AKAN mendeteksi overlap rentang parsial (slot A jam 1-3 vs slot B jam 2-4 di kelas+hari sama, nilai `(jamKeAwal,jamKeAkhir)` beda persis, constraint lolos).

## Keputusan Final Dikonfirmasi User (2026-08-18)

1. **JANGAN ubah schema `JadwalSlot.jamKe` jadi 2 kolom** — TETAP `jamKe Int` tunggal seperti sekarang, TIDAK ADA migrasi schema, TIDAK ADA perubahan di 15 titik konsumsi existing, TIDAK ADA redesain UI grid.
2. **Input rentang HANYA sebagai KEMUDAHAN INPUT/IMPORT** — form dan importer TERIMA `jamKeAwal`+`jamKeAkhir`, tapi backend EXPAND jadi BEBERAPA BARIS `JadwalSlot` terpisah (1 baris per jamKe dalam rentang itu), dikirim ke `JadwalSlotService.create()` yang SUDAH ADA apa adanya (tidak diubah).
3. **KRITIKAL — jam selesai SESI (bukan jam per-baris) harus dihitung dari UJUNG AKHIR rentang, bukan dari baris pertama**: Kalau admin input rentang jam 1-3, maka WALAU di-expand jadi 3 baris `JadwalSlot` terpisah (jamKe=1, jamKe=2, jamKe=3), sistem WAJIB tetap tahu bahwa ini SATU sesi mengajar utuh yang jam selesainya adalah jam selesai dari `AlokasiWaktuSlot` jamKe=3 (bukan jamKe=1). **JANGAN sampai 3 baris independen masing-masing dianggap sesi 1-jam terpisah dengan jam selesai jamKe=1 untuk ketiganya** — ini akan salah total untuk `TeachingSession.jamSelesai` (dipakai auto-close sesi, `resolveJamSesi()`).
4. **Template import diseragamkan PERSIS sama dengan struktur 2 sheet Excel asli user** (`Penjadwalan Perguru` dan `Alokasi Waktu`/`Data Alokasi`) — user ingin ke depannya TIDAK PERLU membuat format file baru lagi, cukup ambil sheet dari file Excel yang sudah dia punya sekarang dan upload langsung.

## Struktur Excel Asli User (dikonfirmasi baca langsung file `dariDev/Sistem Jurnal Baru 2026-2027.xlsx`, 2026-08-18)

### Sheet "Penjadwalan Perguru" (3467 baris data riil, kolom PERSIS):
```
No | Minggu | Hari | Nama Guru | Mata Pelajaran | Kelas | Jam Ke Awal | Jam Ke Akhir | Catatan
```
- Kolom **"Minggu"** — berisi `"Minggu A"`/`"Minggu B"` UNTUK KELAS BLOK, TAPI **KOSONG untuk 2216 dari 3467 baris** (mode normal) — file riil ini CAMPURAN normal+blok, KONSISTEN desain `JadwalSlot.minggu` nullable yang SUDAH ADA.
- Kolom **"Jam Ke Awal"/"Jam Ke Akhir"** — INI YANG DIMINTA user, contoh baris riil: `Jam Ke Awal=2, Jam Ke Akhir=7` (rentang 6 jam sekaligus).
- Kolom "Nama Guru" — TEKS NAMA LENGKAP guru (bukan ID) — importer HARUS resolve nama→`teacherId` (KONSISTEN pola importer existing `JadwalSlotImportRow.guru` yang sudah terima nama, cek `jadwal-slot.service.ts` fungsi resolve nama guru yang sudah ada).
- Kolom "Kelas" — teks nama kelas (misal "XI TKR 6") — resolve ke `kelasId` (KONSISTEN pola existing).
- Kolom "Mata Pelajaran" — teks nama mapel — resolve ke `mapelId` (KONSISTEN pola existing).

### Sheet "Alokasi Waktu" / "Data Alokasi" (2 varian ditemukan di file user, PILIH salah satu sebagai acuan — REKOMENDASI "Data Alokasi" karena strukturnya lebih flat/simpel, sudah 1 baris = 1 hari eksplisit tanpa section header manual):
```
No | Hari | Jam Ke | Waktu[Mulai | Selesai | Alokasi | Durasi] | Keterangan
```
- Header 2-level: baris pertama `"Jam Ke"` + `"Waktu"` (merged, sub-kolom Mulai/Selesai/Alokasi/Durasi), baris kedua sub-header.
- Kolom "Hari" — teks nama hari ("Senin", "Selasa", dst) — BUKAN angka DAYOFWEEK — importer HARUS mapping teks→integer (KONSISTEN `hari: Int` 1=Minggu..7=Sabtu di `AlokasiWaktuSlot`).
- Kolom "Jam Ke" — KOSONG untuk baris istirahat (persis sama semantik `AlokasiWaktuSlot.jamKe: Int? // NULL = baris istirahat` yang SUDAH ADA — TIDAK PERLU field baru, importer HARUS translate sel kosong Excel → `null`).
- Kolom "Waktu Mulai"/"Waktu Selesai" — DISIMPAN EXCEL SEBAGAI PECAHAN HARI (fraction of day, misal `0.291666...` = 07:00) — **WAJIB dikonversi ke format `HH:mm` string saat parsing**, BUKAN dipakai mentah (`AlokasiWaktuSlot.jamMulai`/`jamSelesai` bertipe `String` format `HH:mm`, KONSISTEN existing).
- Kolom "Alokasi" (contoh: "7:00 - 7:40") — TEKS GABUNGAN mulai-selesai, REDUNDAN dengan 2 kolom sebelumnya, importer BOLEH ABAIKAN kolom ini (tidak perlu field baru untuk menyimpannya, cukup untuk validasi silang opsional).
- Kolom "Durasi" — pecahan hari juga (representasi lama slot dalam menit/jam) — REDUNDAN, bisa dihitung ulang dari Mulai-Selesai, TIDAK PERLU disimpan sebagai field terpisah.
- Kolom "Keterangan" — teks bebas ("Istirahat Ke-1", dst) — map ke `AlokasiWaktuSlot.keterangan`.
- **PENTING — file sheet "Alokasi Waktu" (varian lain, BUKAN "Data Alokasi") punya SECTION TERPISAH per grup hari** (`"A. SENIN - KAMIS"`, `"B. JUMAT"`, masing-masing dengan jam berbeda — Jumat lebih pendek karena sholat Jumat) — kalau REKOMENDASI "Data Alokasi" (flat per-hari) diikuti, section ini OTOMATIS tidak perlu ditangani khusus (tiap hari SUDAH baris sendiri-sendiri, Jumat punya baris `Hari=Jumat` dengan `jamMulai`/`jamSelesai` sendiri yang natural lebih pendek) — **VERIFIKASI keputusan ini ke user saat implementasi kalau ragu sheet mana yang jadi acuan resmi** (2 sheet mirip ditemukan di file yang sama, `Data Alokasi` dan `Alokasi Waktu`, isinya SEDIKIT beda struktur presentasi tapi data intinya sama).

## Spec Detail

### 1. Backend — `JadwalSlot` tambah field penanda rentang (untuk resolusi jam sesi yang benar)

**TAMBAH 1 field nullable baru** ke `JadwalSlot` (bukan ganti `jamKe`, TAMBAHAN):
```prisma
model JadwalSlot {
  // ... field existing tidak berubah ...
  jamKeAkhirRentang Int? @map("jam_ke_akhir_rentang")
  // NULL = slot tunggal biasa (jam selesai = AlokasiWaktuSlot pada jamKe baris ini sendiri).
  // TERISI = baris ini bagian dari 1 rentang input yang di-expand jadi banyak baris —
  // jam selesai SESI SEBENARNYA harus diresolve dari AlokasiWaktuSlot pada jamKe =
  // jamKeAkhirRentang ini (BUKAN dari jamKe baris itu sendiri). SEMUA baris hasil expand
  // dari 1 rentang input punya nilai jamKeAkhirRentang YANG SAMA (ujung rentang asli).
}
```
- **KENAPA TAMBAH FIELD, BUKAN HITUNG ULANG DARI DATA LAIN**: tidak ada cara lain untuk tahu "baris jamKe=1 ini tadinya bagian dari rentang 1-3 yang sama" tanpa menyimpan penanda eksplisit — kalau HANYA lihat `kelasId+hari+mapelId+teacherId` yang sama pun tidak cukup (bisa saja admin memang sengaja input 2 slot terpisah kebetulan connect jamKe-nya, HARUS dibedakan dari 1 rentang yang sengaja di-expand).
- Migration: `ALTER TABLE jadwal_slots ADD COLUMN jam_ke_akhir_rentang INT NULL` — additive, tidak breaking data existing (semua baris lama otomatis `NULL` = perilaku sama seperti sekarang).

### 2. Backend — `JadwalSlotService` endpoint baru untuk create rentang

TAMBAH method baru `createRentang()` di `jadwal-slot.service.ts` (JANGAN modifikasi `create()` yang sudah ada — method itu TETAP untuk 1 baris tunggal, dipertahankan apa adanya untuk backward-compat form single-jam):

```ts
async createRentang(dto: CreateJadwalSlotRentangDto) {
  const { jamKeAwal, jamKeAkhir, ...rest } = dto;
  if (jamKeAwal > jamKeAkhir) {
    throw new BadRequestException(
      `Jam Ke Awal (${jamKeAwal}) tidak boleh lebih besar dari Jam Ke Akhir (${jamKeAkhir}) — periksa urutan input.`
    );
  }
  const hasilBerhasil = [];
  const hasilGagal = [];
  for (let jamKe = jamKeAwal; jamKe <= jamKeAkhir; jamKe++) {
    try {
      const slot = await this.create({
        ...rest,
        jamKe,
        jamKeAkhirRentang: jamKeAkhir, // SEMUA baris hasil expand simpan ujung akhir rentang yang SAMA
      });
      hasilBerhasil.push(slot);
    } catch (e) {
      hasilGagal.push({ jamKe, error: e.message }); // best-effort per-jam, KONSISTEN pola import existing
    }
  }
  return { berhasil: hasilBerhasil, gagal: hasilGagal };
}
```
- **VALIDASI TAMBAHAN sebelum expand**: setiap `jamKe` dalam rentang WAJIB valid di `AlokasiWaktuSlot` terkait (`ensureJamKeValid()` yang SUDAH ADA dipanggil otomatis lewat `create()` di tiap iterasi) — kalau ADA jamKe di tengah rentang yang ternyata baris ISTIRAHAT (`AlokasiWaktuSlot.jamKe IS NULL` untuk urutan itu) atau tidak ada di Alokasi Waktu, baris itu GAGAL dengan pesan jelas ("Jam Ke 5 dalam rentang 2-7 adalah jam istirahat, tidak bisa dijadikan jadwal mengajar — periksa Alokasi Waktu atau persempit rentang"), TAPI baris LAIN dalam rentang yang valid TETAP di-create (best-effort, KONSISTEN pola import existing) — JANGAN all-or-nothing untuk rentang.
- **VALIDASI OVERLAP KELAS** (celah yang dicatat di riset — constraint DB tidak cukup untuk overlap rentang) — TAMBAH pengecekan baru di `createRentang()`: sebelum expand, cek apakah ADA `JadwalSlot` lain di `kelasId+hari+minggu` yang SAMA dengan `jamKe` manapun DALAM rentang `[jamKeAwal, jamKeAkhir]` yang diminta (query `findMany({ where: { kelasId, hari, minggu, jamKe: { gte: jamKeAwal, lte: jamKeAkhir } } })`) — kalau ADA, TOLAK SELURUH rentang dengan pesan jelas sebut jam yang bentrok ("Kelas ini sudah punya jadwal di Jam Ke 4 (rentang yang diminta 2-7 mencakup jam itu) — periksa jadwal existing kelas ini sebelum input rentang baru").

### 3. Backend — `TeachingSessionsService.resolveJamSesi()` pakai `jamKeAkhirRentang` kalau ada

`resolveJamSesi()` (`teaching-sessions.service.ts:88-97`) — UBAH logic (BUKAN ubah signature, tetap terima `Pick<JadwalSlot, "opsiJadwalId" | "hari" | "jamKe">`, TAMBAH parameter opsional):

```ts
private async resolveJamSesi(
  jadwalSlot: Pick<JadwalSlot, "opsiJadwalId" | "hari" | "jamKe" | "jamKeAkhirRentang">,
) {
  const opsiJadwal = await this.prisma.opsiJadwal.findUniqueOrThrow({ where: { id: jadwalSlot.opsiJadwalId } });
  const jamMulaiResolved = await this.alokasiWaktu.resolveJam(opsiJadwal.alokasiWaktuId, jadwalSlot.hari, jadwalSlot.jamKe);
  const jamKeUntukSelesai = jadwalSlot.jamKeAkhirRentang ?? jadwalSlot.jamKe; // KUNCI PERBAIKAN
  const jamSelesaiResolved = await this.alokasiWaktu.resolveJam(opsiJadwal.alokasiWaktuId, jadwalSlot.hari, jamKeUntukSelesai);
  return { jamMulai: jamMulaiResolved.jamMulai, jamSelesai: jamSelesaiResolved.jamSelesai };
}
```
- **VERIFIKASI SEKSAMA saat implementasi**: pastikan test eksplisit untuk kasus rentang — input rentang jam 1-3, cek `TeachingSession.jamSelesai` (via `resolveJamSesi`) HARUS SAMA DENGAN `AlokasiWaktuSlot` jamKe=3 punya `jamSelesai`, BUKAN jamKe=1. Ini adalah requirement PALING KRITIKAL dari task ini (sesuai penekanan eksplisit user "jangan sampai jam ke 1-3 tapi jam selesainya dihitung jam mulainya jam ke 3... pastikan yang dihitung adalah jam selesainya jam ke 3" — CATATAN: user menyebut "jam mulainya jam ke 3" sebagai kondisi yang SALAH, maksudnya jam selesai TIDAK BOLEH salah ambil titik — pastikan hasil akhir adalah **jam SELESAI dari AlokasiWaktuSlot jamKe akhir rentang**, bukan jam mulai atau jam dari baris awal).
- **6 pemanggil `resolveJamSesi()` existing** (`getMyToday`, `getSesiUntukTanggal`, `startSession`, `autoCloseDueSessions`, `izinSesiSpesifikSudahLewat`, `izinSeharianSudahLewat`) — TIDAK PERLU diubah satu-satu, cukup pastikan mereka SUDAH include `jamKeAkhirRentang` di query `JadwalSlot` yang di-pass ke fungsi ini (cek tiap query Prisma yang select/include `JadwalSlot` fields untuk method ini, tambah `jamKeAkhirRentang` ke `select`/tidak perlu kalau sudah pakai object utuh).

### 4. Backend — Import Excel/CSV format baru (sheet "Penjadwalan Perguru" & "Data Alokasi")

**GANTI TOTAL format kolom importer existing** (`JadwalSlotImportRow` di `jadwal-slot.service.ts`, method `importJadwalSlot()`) dari format lama (`kelas,hari,jam_ke,minggu,mapel,guru,catatan`) ke kolom PERSIS Excel user:

```ts
interface JadwalSlotImportRow {
  no?: string;           // diabaikan (nomor urut Excel, bukan data)
  minggu?: string;       // "Minggu A" / "Minggu B" / KOSONG (mode normal)
  hari?: string;         // "Senin".."Sabtu" — teks, BUKAN angka
  namaGuru?: string;     // teks nama lengkap guru — resolve ke teacherId
  mataPelajaran?: string; // teks nama mapel — resolve ke mapelId
  kelas?: string;        // teks nama kelas — resolve ke kelasId
  jamKeAwal?: string;    // NUMBER as string — WAJIB
  jamKeAkhir?: string;   // NUMBER as string — WAJIB, >= jamKeAwal
  catatan?: string;
}
```
- Header CSV/Excel yang diterima: `No, Minggu, Hari, Nama Guru, Mata Pelajaran, Kelas, Jam Ke Awal, Jam Ke Akhir, Catatan` — **PERSIS SAMA TEKS dengan sheet "Penjadwalan Perguru"** (case-insensitive matching header direkomendasikan untuk toleransi kecil, tapi URUTAN KOLOM dan NAMA KOLOM harus 1:1 cocok supaya user bisa copy-paste sheet apa adanya).
- Parsing `jamKeAwal`/`jamKeAkhir` → panggil `createRentang()` (poin 2) per baris, BUKAN `create()` langsung — SETIAP baris CSV/Excel adalah 1 RENTANG yang di-expand ke N baris `JadwalSlot`.
- Kolom "Minggu" KOSONG → `minggu: null` (mode normal utk `OpsiJadwal` normal) — VALIDASI: kalau `OpsiJadwal.mode === blok` tapi baris punya Minggu kosong, TOLAK baris itu dengan pesan jelas ("Baris ini tidak punya nilai Minggu, tapi Opsi Jadwal '{nama}' bermode Blok — WAJIB isi Minggu A atau Minggu B"); sebaliknya juga (`mode === normal` tapi Minggu terisi → tolak, pesan jelas).
- Resolve "Nama Guru"/"Mata Pelajaran"/"Kelas" dari TEKS ke ID — REPLIKASI logic resolve yang SUDAH ADA di importer lama (cek fungsi lookup existing di `jadwal-slot.service.ts` sebelum tulis ulang) — tambah pesan error actionable kalau nama tidak ketemu persis ("Guru 'xxx' tidak ditemukan di sistem — periksa ejaan nama persis sama dengan data guru terdaftar, atau daftarkan guru ini dulu").

### 5. Backend — Import Excel untuk `AlokasiWaktu`+`AlokasiWaktuSlot` (sheet "Data Alokasi")

**Endpoint import BARU** (belum ada sebelumnya — `AlokasiWaktu` selama ini hanya diinput manual via form T210) — `POST /alokasi-waktu/:id/import` atau `POST /alokasi-waktu/import` (buat baru sekalian dengan `nama` di body, PUTUSKAN saat implementasi):

```ts
interface AlokasiWaktuSlotImportRow {
  no?: string;        // diabaikan
  hari?: string;      // "Senin".."Sabtu" — teks
  jamKe?: string;     // NUMBER as string, KOSONG = baris istirahat
  waktuMulai?: string;  // Excel fraction-of-day ATAU string HH:mm — DETEKSI OTOMATIS format saat parsing
  waktuSelesai?: string;
  keterangan?: string;
}
```
- **PARSING WAKTU EXCEL FRACTION-OF-DAY**: kalau library parsing Excel (cek yang sudah dipakai di modul import lain, kemungkinan `xlsx`/`exceljs`) return angka pecahan (misal `0.2916666...`), WAJIB dikonversi: `jam = Math.floor(fraction * 24)`, `menit = Math.round((fraction * 24 - jam) * 60)`, format `HH:mm` dengan padding 2 digit. **VERIFIKASI dengan data riil**: `0.2916666666666667 * 24 = 7.0 jam` → `"07:00"` (cocok dengan kolom "Alokasi" Excel yang eksplisit tulis "7:00 - 7:40" sebagai validasi silang).
- Kalau cell sudah berupa string `HH:mm` langsung (tergantung cara library parsing baca format cell Excel — Excel bisa simpan sebagai Date/Time serial ATAU text tergantung format sel) — **WAJIB deteksi kedua kemungkinan format saat implementasi**, jangan asumsi salah satu saja.
- Kolom "Jam Ke" KOSONG → `jamKe: null` (baris istirahat, KONSISTEN `AlokasiWaktuSlot.jamKe: Int?` existing).
- Kolom "Alokasi"/"Durasi" Excel — DIABAIKAN saat import (redundan, dihitung ulang dari Mulai-Selesai kalau perlu ditampilkan lagi di UI, TIDAK disimpan sebagai field baru).
- `urutan` (field wajib `AlokasiWaktuSlot.urutan`, dipakai untuk `@@unique([alokasiWaktuId, hari, urutan])`) — DIISI OTOMATIS dari URUTAN BARIS dalam file per hari (baris pertama untuk hari itu = urutan 1, dst) — TIDAK ADA di kolom Excel, importer HARUS generate sendiri berbasis posisi baris.

## Edge Cases

- **Rentang jam melewati batas istirahat** (misal input jamKeAwal=3, jamKeAkhir=6, padahal jamKe=5 di Alokasi Waktu adalah "Istirahat Ke-1" sisipan di antara 4 dan 5 — VERIFIKASI numbering: `AlokasiWaktuSlot.jamKe` untuk istirahat adalah NULL, BUKAN nomor jam biasa yang "dilewati", jadi rentang 3-6 di sisi USER berarti "jam pelajaran ke-3 sampai ke-6" TIDAK termasuk baris istirahat yang jamKe-nya NULL — PASTIKAN interpretasi `jamKe` di sini adalah NOMOR JAM PELAJARAN (bukan nomor urutan baris termasuk istirahat), KONSISTEN definisi existing `AlokasiWaktuSlot.jamKe` yang SUDAH exclude istirahat dari penomoran.
- **Rentang 1 jam saja** (jamKeAwal === jamKeAkhir) — TETAP jalan normal, hasilnya 1 baris `JadwalSlot` dengan `jamKeAkhirRentang` SAMA DENGAN `jamKe` sendiri (tidak masalah, `resolveJamSesi()` tetap benar karena `resolveJam(jamKe)` dan `resolveJam(jamKeAkhirRentang)` mengembalikan hasil yang sama untuk 1 jamKe yang sama).
- **Import rentang yang sebagian bentrok sebagian tidak** — SELURUH rentang ditolak (bukan partial), KONSISTEN keputusan poin 2 spec (constraint validasi overlap kelas dicek SEBELUM expand, all-or-nothing UNTUK 1 baris rentang, tapi best-effort ANTAR baris CSV berbeda).
- **File Excel user py sheet "Alokasi Waktu" (section Senin-Kamis/Jumat) DAN "Data Alokasi" (flat per-hari) SEKALIGUS** — importer HANYA perlu dukung SATU struktur resmi (REKOMENDASI "Data Alokasi", flat) — KONFIRMASI ke user sheet mana yang jadi acuan resmi ke depan sebelum implementasi, supaya tidak dukung 2 varian sekaligus tanpa perlu.

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`JadwalSlot` +`jamKeAkhirRentang`), `apps/api/src/jadwal-slot/jadwal-slot.service.ts` (`createRentang()`, importer format baru), `apps/api/src/jadwal-slot/jadwal-slot.controller.ts` (endpoint baru), `apps/api/src/teaching-sessions/teaching-sessions.service.ts` (`resolveJamSesi()`), `apps/api/src/alokasi-waktu/alokasi-waktu.service.ts` (importer baru).
- **Buat:** migration Prisma baru, DTO `create-jadwal-slot-rentang.dto.ts`, DTO import Alokasi Waktu.
- **Frontend:** form input jadwal (`guru-hari-input.tsx`/`tab-hari-input.tsx`, T212) — TAMBAH opsi "input rentang" sebagai ALTERNATIF cara isi (PUTUSKAN UI saat implementasi — REKOMENDASI: checkbox "Isi beberapa jam sekaligus" yang mengubah 1 baris grid jadi 2 dropdown Jam Ke Awal-Akhir, submit ke `createRentang()` bukan `create()` per baris), halaman import CSV/Excel (T213, update kolom template sesuai poin 4-5 di atas.

## Acceptance Criteria
- [x] `createRentang()` — input jamKeAwal=2, jamKeAkhir=7 menghasilkan 6 baris `JadwalSlot` (jamKe 2,3,4,5,6,7), semua dengan `jamKeAkhirRentang=7`.
- [x] `resolveJamSesi()` untuk SALAH SATU baris hasil expand (misal jamKe=4 dari rentang 2-7) — `jamSelesai` HASIL = jam selesai `AlokasiWaktuSlot` jamKe=7, BUKAN jamKe=4.
- [x] Rentang mencakup jam istirahat/jamKe tidak ada di Alokasi Waktu — baris itu gagal dengan pesan jelas, baris lain dalam rentang tetap sukses.
- [x] Rentang overlap dengan JadwalSlot kelas lain yang sudah ada — SELURUH rentang ditolak, pesan sebut jam yang bentrok.
- [x] Import CSV/Excel format baru — header PERSIS `No,Minggu,Hari,Nama Guru,Mata Pelajaran,Kelas,Jam Ke Awal,Jam Ke Akhir,Catatan` diterima, hasil sama seperti create manual.
- [x] Import Alokasi Waktu baru — waktu format Excel fraction-of-day dikonversi benar ke `HH:mm` (test dengan nilai riil `0.2916666...` → `"07:00"`).
- [x] Kolom Jam Ke kosong di import Alokasi Waktu → tersimpan sebagai baris istirahat (`jamKe: null`).
- [x] Build + type-check hijau, test lengkap untuk skenario rentang + resolusi jam + import format baru (tsc api+web bersih, 494 test jest passing termasuk 37 test baru).

## Validasi Claudian
- [x] **WAJIB test eksplisit** membuktikan jam selesai sesi hasil rentang = jam selesai jamKe AKHIR (bukan jamKe baris individual) — ini requirement paling kritikal, jangan cuma diverifikasi manual. (`teaching-sessions.service.spec.ts`, test "T219 KRITIKAL")
- [x] Konfirmasi ke user sheet Alokasi Waktu MANA yang jadi acuan resmi import (`Data Alokasi` flat vs `Alokasi Waktu` bersection) sebelum implementasi importer ini. (Dikonfirmasi via AskUserQuestion sebelum eksekusi: `Data Alokasi` flat)
- [x] Konfirmasi 6 pemanggil `resolveJamSesi()` existing tidak perlu perubahan signature (hanya perlu pastikan `jamKeAkhirRentang` ikut ke-select di query Prisma yang memberi input ke fungsi ini). (Semua 6 pemanggil pakai `include: { jadwalSlot: true, ... }` — Prisma otomatis select semua kolom termasuk field baru, tidak ada perubahan query diperlukan)

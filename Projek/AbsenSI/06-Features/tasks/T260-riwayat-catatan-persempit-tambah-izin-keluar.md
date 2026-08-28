# T260 — Web+API: Riwayat Catatan — Persempit Scope (Hapus Alfa/Izin/Sakit/Dispen, Duplikat Riwayat Presensi) + Tambah Izin Keluar

## Depends on
**T259** (Riwayat Presensi) — SEBAIKNYA sudah ada dulu, supaya alfa/izin/sakit/dispen yang
dihapus dari sini tidak hilang visibilitasnya sebelum penggantinya siap. Kalau dikerjakan
lebih dulu dari T259, JANGAN hapus dulu — urutan aman: T259 dulu, baru task ini.

## Objective
Riwayat Catatan dipersempit HANYA untuk kejadian yang BUKAN status-kehadiran-harian:
**Terlambat, Izin Keluar (baru), PKL, Terkunci/Dibuka** — entri Izin/Sakit/Dispen/Alfa
(full-day) DIHAPUS dari sini (representasinya sekarang di Riwayat Presensi, T259, supaya
tidak duplikat). Sekaligus jawab pertanyaan desain user: bagaimana 1 tabel menampung jenis
data dengan bentuk detail berbeda (Terlambat vs Izin Keluar) — jawaban: EKSTENSI pola
discriminated union yang SUDAH ADA, bukan redesain tabel.

## Keputusan Diminta User (2026-08-28)
1. Riwayat Catatan akan berisi: **Terlambat, Izin Keluar, dan jenis dari fitur mendatang**.
2. **PKL dan Terkunci/Dibuka TETAP di Riwayat Catatan** (dikonfirmasi eksplisit — bukan
   status kehadiran harian, tidak cocok pindah ke Riwayat Presensi).
3. Izin/Sakit/Dispen (full-day, `permitJenis: "tidak_masuk"`) dan Alfa **DIHAPUS** dari
   Riwayat Catatan — representasinya sekarang HANYA di Riwayat Presensi (T259), hindari
   admin melihat data yang sama 2x di 2 tempat berbeda.

## Konteks — Root Cause & Lokasi Persis (dikonfirmasi via riset 2026-08-28)

`AttendanceReportService.riwayatCatatan()` (`attendance-report.service.ts:596-720-an`) —
3 sumber data digabung jadi `entries[]`:

1. **Terlambat** (`attendanceRecord.findMany({where:{status:terlambat}})`, baris ~607-611)
   — TIDAK DIUBAH.
2. **Permit** (baris ~612-622) — SAAT INI `where: { studentId }` TANPA filter `jenis`,
   jadi permit `tidak_masuk` (izin/sakit/dispen full-day) DAN `keluar` (izin keluar)
   SAMA-SAMA masuk, dipetakan lewat `JENIS_BY_KATEGORI` (baris ~683-687) jadi
   `jenis: "izin"|"sakit"|"dispen"` TANPA membedakan `permitJenis` di label (T245 baru
   nambah label "Keluar" tapi datanya masih include KEDUANYA). **HARUS DIUBAH**: filter
   query jadi `where: { studentId, jenis: "keluar" }` (HANYA izin keluar) — permit
   `tidak_masuk` TIDAK LAGI diambil fungsi ini sama sekali (representasinya pindah ke
   T259's endpoint).
3. **Alfa/PKL** (loop hari wajib, baris ~712-714: `entries.push(pklDates.has(dateKey) ?
   {jenis:"pkl",...} : {jenis:"alfa",...})`) — **UBAH jadi HANYA push kalau PKL**:
   ```ts
   if (pklDates.has(dateKey)) entries.push({ jenis: "pkl", tanggal: dateKey });
   // else: JANGAN push apa pun (alfa TIDAK LAGI relevan di fungsi ini)
   ```
4. **Lock/unlock events** (`terkunci`/`dibuka`) — TIDAK DIUBAH.

## Spec Detail

### 1. Backend — persempit `riwayatCatatan()`
Terapkan 2 perubahan di atas (poin 2 filter `jenis:"keluar"`, poin 3 skip alfa). Field
response untuk permit `keluar` — TAMBAH data yang dibutuhkan format Detail baru (poin 2
Frontend): `jamKeluar`, `jamKembaliDiharapkan`, `statusKembali` (SEMUA field ini SUDAH ADA
di model `Permit`, tinggal di-`select` dan diteruskan ke entry).

### 2. Tipe — tambah varian `izin_keluar` ke discriminated union
`RiwayatCatatanEntry` (`core-types.ts`) — TAMBAH varian baru (BUKAN reuse varian
`izin`/`sakit`/`dispen` yang sudah ada, supaya jelas terpisah dari full-day izin yang sudah
tidak ada lagi di union ini kalau mau bersih — VERIFIKASI SAAT IMPLEMENTASI: BOLEH JUGA
tetap reuse jenis `izin`/`sakit`/`dispen` existing + field `permitJenis` yang T245 sudah
tambahkan sebagai pembeda, TIDAK WAJIB bikin literal baru kalau field pembeda yang sudah
ada cukup — pilih yang paling minim perubahan tipe):
```ts
{
  jenis: "izin" | "sakit" | "dispen"; // existing, sekarang HANYA terisi utk permitJenis="keluar"
  permitJenis: "keluar"; // SELALU "keluar" sekarang (tidak_masuk sudah tidak lewat sini)
  tanggal: string;
  jamKeluar: string | null;
  jamKembaliDiharapkan: string | null;
  statusKembali: "belum" | "sudah" | "pulang";
  alasanDetail: string | null;
  namaPetugas: string;
}
```

### 3. Frontend — format Detail baru untuk Izin Keluar
`riwayat-catatan-table.tsx` — untuk entry `permitJenis === "keluar"` (SATU-SATUNYA kasus
yang lewat cabang izin/sakit/dispen sekarang), ganti isi kolom Detail dari `alasanDetail`
polos jadi format informatif:
- `statusKembali === "sudah"` → `"Keluar {jamKeluar} — Kembali {jamKembaliAktual}"` (VERIFIKASI
  field jam kembali aktual tersedia di response, kalau belum ada tambahkan).
- `statusKembali === "belum"` → `"Keluar {jamKeluar}, diharapkan kembali {jamKembaliDiharapkan}
  (belum kembali)"`.
- `statusKembali === "pulang"` → `"Keluar {jamKeluar} — dianggap pulang (tidak kembali)"`.
- `alasanDetail` (kalau ada) TETAP ditampilkan sebagai info tambahan (baris kedua kecil),
  BUKAN dihapus — cuma bukan lagi SATU-SATUNYA isi kolom Detail.

Label+ikon "Izin Keluar"/`LogOut` dari T245 TIDAK diubah (sudah benar).

### 4. Hapus kode Alfa yang sekarang mati
`riwayat-catatan-table.tsx` — `RIWAYAT_LABEL`/`RIWAYAT_BADGE_CLASS`/`RIWAYAT_ICON` entry
untuk `"alfa"`, dan tombol "Hadir" (T238, baris ~219-226 versi sebelum task ini) — HAPUS
(kode mati, alfa tidak lagi pernah dikirim backend ke fungsi ini). Import
`useAbsenManual`/dialog konfirmasi terkait — hapus juga kalau sudah tidak dipakai di file
ini sama sekali (VERIFIKASI dulu tidak dipakai jenis lain sebelum hapus).

## Edge Cases
- **Siswa dengan histori izin_masuk (tidak_masuk) LAMA sebelum task ini** — data Permit
  lama TIDAK hilang dari database, cuma TIDAK LAGI muncul di Riwayat Catatan — HARUS
  tetap muncul di Riwayat Presensi (T259) sebagai baris Izin/Sakit/Dispen tanggal itu.
  VERIFIKASI silang: total kejadian izin/sakit/dispen historis siswa TIDAK BOLEH hilang
  dari SELURUH aplikasi, cuma pindah lokasi tampilan.
- **PKL bersamaan dengan Terlambat di tanggal yang sama** (siswa PKL tapi entah kenapa ada
  AttendanceRecord terlambat juga — kasus langka) — KEDUA entry tetap muncul terpisah di
  Riwayat Catatan (tidak saling menghapus, konsisten data mentah apa adanya).

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (`riwayatCatatan()`
  — filter permit, skip alfa, tambah field izin-keluar ke response).
- **Modifikasi:** `apps/web/src/lib/core-types.ts` (`RiwayatCatatanEntry` — tambah field
  izin-keluar, HAPUS variant/field yang sudah tidak relevan kalau ada).
- **Modifikasi:** `apps/web/src/components/riwayat-catatan-table.tsx` (format Detail baru
  izin-keluar, hapus kode mati alfa/Hadir-button).
- **Jangan sentuh:** endpoint/logic T259 (Riwayat Presensi) — task ini murni MENGURANGI
  scope Riwayat Catatan, tidak menyentuh implementasi Riwayat Presensi.

## Acceptance Criteria
- [x] Riwayat Catatan TIDAK LAGI menampilkan entri Alfa, Izin (tidak_masuk), Sakit, Dispen
      (full-day) — HANYA Terlambat, Izin Keluar, PKL, Terkunci/Dibuka.
- [x] Entri Izin Keluar tampilkan detail jam keluar/kembali yang informatif (bukan cuma
      alasanDetail polos).
- [x] Data izin/sakit/dispen/alfa historis TIDAK HILANG dari sistem — tetap terlihat di
      Riwayat Presensi (T259) untuk tanggal yang sama.
- [x] Kode mati (tombol Hadir versi lama, label/ikon Alfa) sudah dibersihkan dari
      `riwayat-catatan-table.tsx`.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA data yang hilang total dari aplikasi — cek silang 1 siswa dengan
      histori izin+alfa nyata, pastikan semuanya masih terlihat (di Riwayat Presensi),
      cuma tidak lagi dobel di Riwayat Catatan. (Verifikasi logika, bukan klik-coba live —
      lihat catatan "Belum diverifikasi live" di Implementasi.)
- [x] Konfirmasi PKL dan Terkunci/Dibuka BENAR-BENAR tidak terpengaruh perubahan filter
      permit (keduanya sumber data terpisah, seharusnya aman, tapi verifikasi eksplisit).
- [x] Konfirmasi tombol Hadir/Izin di T259 dan penghapusan di task ini SELARAS (dikerjakan
      dalam sesi eksekusi yang sama secara berurutan, tidak ada celah waktu deploy).

## Implementasi (2026-08-28)

**Backend** (`attendance-report.service.ts`, `riwayatCatatan()`):
- Query `permit` dipersempit `where: { studentId, jenis: PermitJenis.keluar }` (sebelumnya
  tanpa filter `jenis` sama sekali) — `select` ditambah `jamKeluar`, `jamKembaliDiharapkan`,
  `kembaliDikonfirmasiAt`, `statusKembali`.
- Loop `wajibDates` di akhir fungsi HANYA push `{jenis:"pkl"}` — cabang `else push alfa`
  dihapus total.
- `presentOrExcusedDates`/`allRecordDates` (query `AttendanceRecord` tanpa filter status)
  TETAP dipertahankan APA ADANYA — masih dibutuhkan untuk gate PKL (hari yang siswa
  benar-benar hadir/izin tidak boleh salah tampil PKL), dan TIDAK terpengaruh penyempitan
  filter `permits` di atas karena sumber datanya independen (AttendanceRecord, bukan Permit).
- `RiwayatCatatanEntry` (tipe balik fungsi ini): varian `izin|sakit|dispen` diperluas
  `jamKeluar?`, `jamKembaliDiharapkan?`, `jamKembaliAktual?`, `statusKembali?` — varian
  `{jenis:"alfa"}` dihapus total dari union.

**Tipe frontend** (`core-types.ts`): `RiwayatCatatanEntry` di-mirror persis perubahan
backend (field baru + hapus varian alfa).

**Frontend** (`riwayat-catatan-table.tsx`):
- `RIWAYAT_LABEL`/`RIWAYAT_BADGE_CLASS`/`RIWAYAT_ICON` — key `"alfa"` dihapus (TypeScript
  otomatis menolak kalau tidak dihapus, karena union sudah tidak punya varian itu).
- Tombol "Hadir" (T238) + dialog konfirmasi + state `absenTarget`/`saving`/`absenError` +
  handler `handleAbsenManual`/`refetch` — dihapus total (satu-satunya pemakainya adalah
  baris alfa yang sekarang tidak pernah dikirim backend). Prop `studentId`+`canAbsenManual`
  ikut dihapus dari signature komponen — caller `(admin)/siswa/[id]/siswa-detail-view.tsx`
  disesuaikan (props itu dulu HANYA dipakai tombol Hadir; caller wali kelas
  `siswa-detail-wali-view.tsx` sudah tidak pernah mengirim prop itu dari awal, tidak perlu
  diubah).
- Kolom Detail untuk entry `permitJenis === "keluar"` sekarang format 2-baris: baris utama
  dari `formatDetailIzinKeluar()` (cabang `statusKembali`: "sudah" → jam kembali aktual,
  "belum" → jam diharapkan kembali, "pulang" → dianggap pulang tanpa lapor), baris kedua
  kecil `alasanDetail` (kalau ada) — bukan dihapus, cuma bukan lagi satu-satunya isi kolom.
- Label+ikon "Izin Keluar"/`LogOut` (T245) TIDAK diubah.

**Verifikasi:**
- `tsc --noEmit` API dan web — bersih, tanpa error, di kedua kesempatan (setelah backend
  selesai, dan lagi setelah frontend selesai).
- `next build` web — sukses penuh (exit 0), termasuk route `/siswa/[id]` yang memakai
  kedua komponen (`RiwayatCatatanTable` + `RiwayatPresensiTable`).
- `jest attendance-report.service.spec.ts` — 45/45 lulus (dijalankan terisolasi 2x karena
  sempat overlap resource dengan proses lain di sesi ini, hasil isolasi konsisten bersih).
  Tidak ada test suite existing yang menguji `riwayatCatatan()` secara langsung (dicek via
  grep — HANYA 2 baris komentar yang menyinggung nama fungsi ini, bukan `describe`/`it`
  sungguhan) — jadi tidak ada test signature yang perlu diupdate untuk perubahan ini.
- **Belum diverifikasi live** (DB dev naik-turun sepanjang sesi, konsisten keterbatasan yang
  sama seperti T259): tampilan sungguhan kolom Detail izin-keluar di browser, dan cek
  silang manual 1 siswa nyata yang punya histori izin+alfa untuk konfirmasi visual bahwa
  datanya utuh berpindah ke Riwayat Presensi tanpa hilang.

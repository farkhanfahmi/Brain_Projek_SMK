# T217 — API+Web: Grafik Rekap Kehadiran — Persentase, Bukan Count Absolut

## Depends on
Tidak ada dependency ke rangkaian lain. Independen, murni modul `attendance`/`rekap`.

## Konteks — Masalah UX Ditemukan (2026-08-18)

Screenshot user: grafik "Grafik Hadir/Izin/Sakit/Dispen/Alfa" di halaman Rekap Kehadiran Murid (`apps/web/src/app/(admin)/rekap/rekap-view.tsx`) untuk mode rentang tanggal (>1 hari) menampilkan **count absolut** — bisa sampai belasan ribu (contoh screenshot: Hadir ~14.000, Alfa ~1.400). Angka ini adalah hasil `jumlah siswa × jumlah hari dalam rentang`, BUKAN metrik yang mudah dimaknai orang awam ("1.400 alfa itu buruk atau wajar?" tidak terjawab tanpa konteks pembanding).

Root cause teknis: agregasi chart dihitung **murni di frontend** via `reduce()` atas `result.rows` (`rekap-view.tsx:269-280`), sumbu X Chart.js `beginAtZero: true` tanpa batas atas (`rekap-view.tsx:298`) — tidak ada field persentase di response API sama sekali (`ReportRow`/`AttendanceReportRow` di `attendance-report.service.ts:33-51`/`core-types.ts:264-309` semua count integer).

## Keputusan Dikonfirmasi User (2026-08-18)

1. **Tetap bar chart (Chart.js, horizontal, `indexAxis: "y"`)** — TIDAK ganti ke donut/pie/kartu KPI. Perubahan HANYA sumbu X: dari count absolut ke **persentase 0-100%**.
2. **TIDAK perlu angka headline tambahan** (misal "Tingkat Kehadiran: 92%" di atas chart) — cukup breakdown per kategori dalam bentuk persen, tidak menambah elemen UI baru di luar chart yang sudah ada.
3. **Basis persentase = hari wajib yang SUDAH LEWAT saja** (bukan seluruh rentang `[from, to]` apa adanya) — kalau filter "Sampai Tanggal" mencakup hari di masa depan (belum terjadi), hari itu DIKELUARKAN dari basis penyebut, supaya persentase tidak timpang rendah karena hari yang memang belum ada datanya.

## Spec Detail

### 1. Backend — cutoff "sudah lewat" di `resolveHariWajib`

`resolveHariWajib(from, to)` (`apps/api/src/attendance/attendance-report.service.ts:550-612`) SAAT INI tidak punya konsep cutoff hari ini — loop harian dari `rangeStart` ke `rangeEnd` apa adanya, termasuk tanggal masa depan kalau `to` > hari ini.

- **JANGAN ubah signature/behavior `resolveHariWajib` yang sudah dipakai banyak tempat** (`teacher-attendance-report.service.ts:212`, `tv-piket.service.ts:163,205`, `attendance.service.ts:598`) — method ini tetap dipakai untuk `totalHariAktif` existing (basis TIDAK berubah, tetap seluruh rentang, supaya tidak ada regresi ke consumer lain).
- **TAMBAH pemanggilan KEDUA** khusus untuk basis persen chart: `resolveHariWajib(from, effectiveTo)` di mana `effectiveTo = min(to, startOfToday())` (reuse `startOfToday()` yang sudah ada, baris 545-548) — kalau `effectiveTo < from` (seluruh rentang di masa depan), basis persen = 0, tampilkan state "belum ada data" bukan div-by-zero.
- Hasil dari pemanggilan kedua ini dipakai HANYA untuk hitung persentase per kategori chart, TIDAK menggantikan `totalHariAktif` yang sudah ada di setiap `ReportRow` (field itu tetap basis seluruh rentang, dipakai di tempat lain seperti export).

### 2. Backend — tambah agregat persen ke response `reportFlexible` mode "range"

`reportInternal()` (`attendance-report.service.ts:102-270`) — TAMBAH 1 field baru ke return type mode "range" (jangan pecah endpoint baru, cukup tambah field ke response yang sudah ada):

```ts
export type FlexibleReportResult =
  | { mode: "range"; rows: ReportRow[]; peringatan: string | null; agregatPersen: AgregatPersenRekap }
  | { mode: "single-day"; rows: ReportSingleDayRow[] };

export interface AgregatPersenRekap {
  hadirPersen: number;
  izinPersen: number;
  sakitPersen: number;
  dispenPersen: number;
  alfaPersen: number;
  basisHariWajibSudahLewat: number; // penyebut — untuk transparansi di tooltip FE kalau perlu
}
```

- Hitung dari SUM seluruh `rows` (total hadir+terlambat, izin, sakit, dispen, alfa lintas SEMUA siswa hasil filter) dibagi `(basisHariWajibSudahLewat × jumlahSiswa)` — **VERIFIKASI saat implementasi**: penyebut harus dikali jumlah siswa karena pembilang adalah SUM lintas siswa, bukan basis 1 siswa (kalau tidak, persentase bisa >100%).
- `basisHariWajibSudahLewat === 0` (seluruh rentang di masa depan, atau filter tanggal aneh) → semua `*Persen` return `0`, FE tampilkan state kosong bukan chart dengan bar semua 0% (beda pesan: "belum ada data" vs "benar-benar 0%").
- **PERTIMBANGKAN cache/reuse**: `reportInternal()` sudah loop `wajibDates` untuk `totalHariAktif` per baris — pemanggilan `resolveHariWajib` kedua untuk basis persen adalah query TAMBAHAN (beda rentang tanggal `to` vs `effectiveTo`), TIDAK bisa reuse hasil pertama langsung kalau `to` berbeda dari `effectiveTo`. Kalau `to <= startOfToday()` (kasus paling umum — user pilih rentang yang sudah lewat semua), `effectiveTo === to`, BOLEH short-circuit reuse `wajibDates` yang sudah dihitung, hindari query dobel.

### 3. Frontend — sumbu X jadi persen, agregasi dari backend bukan `reduce()` FE

`rekap-view.tsx:269-301` — HAPUS `reduce()` manual di FE (baris 269-280), GANTI ambil langsung dari `result.agregatPersen` (field baru dari API):

```ts
chartRef.current = new Chart(chartCanvasRef.current, {
  type: "bar",
  data: {
    labels: ["Hadir", "Izin", "Sakit", "Dispen", "Alfa"],
    datasets: [{
      data: [
        result.agregatPersen.hadirPersen,
        result.agregatPersen.izinPersen,
        result.agregatPersen.sakitPersen,
        result.agregatPersen.dispenPersen,
        result.agregatPersen.alfaPersen,
      ],
      backgroundColor: ["#4caf50", "#42a5f5", "#ffb300", "#7b4dc6", "#e53935"],
    }],
  },
  options: {
    indexAxis: "y",
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: { display: false },
      tooltip: { callbacks: { label: (ctx) => `${ctx.raw}%` } }, // BARU — tooltip tampil "92%" bukan "92"
    },
    scales: { x: { beginAtZero: true, max: 100, ticks: { callback: (v) => `${v}%` } } }, // BARU — max 100 + label %
  },
});
```

- **State kosong** (`basisHariWajibSudahLewat === 0`) — tampilkan pesan di tempat chart, MISAL "Belum ada hari yang sudah lewat dalam rentang ini" — JANGAN render chart dengan 5 bar kosong tanpa penjelasan (konsisten aturan CLAUDE.md pesan actionable, walau ini bukan error/exception, tetap butuh penjelasan state kosong yang jelas).
- Judul chart (`chartTitle`, baris 428-430) — UPDATE teks jadi sebut "Persentase" (misal "Grafik Persentase Hadir/Izin/Sakit/Dispen/Alfa") supaya user tidak bingung kenapa angka kecil dari sebelumnya.
- Mode **"single-day"** (baris 240-268) — **TIDAK DIUBAH**, tetap count absolut (basisnya jumlah siswa dalam 1 hari, sudah masuk akal sebagai count, bukan rentang panjang yang jadi masalah UX di sini).

### 4. Frontend — `core-types.ts`

Tambah interface `AgregatPersenRekap` dan field `agregatPersen` di `FlexibleReportResult` mode "range" (`apps/web/src/lib/core-types.ts:264-309`), SAMA PERSIS shape backend poin 2.

## Edge Cases

- **Rentang filter SELURUHNYA di masa depan** (misal admin salah pilih tanggal minggu depan) — `basisHariWajibSudahLewat = 0`, semua persen 0, FE tampilkan state kosong "Belum ada hari yang sudah lewat", BUKAN chart 0% yang membingungkan (0% Alfa terlihat seperti "bagus" padahal sebenarnya "belum ada data").
- **Filter tidak ada siswa hasil (hasil `rows` kosong)** — persen semua 0 atau NaN (VALIDASI: hindari div-by-zero kalau `rows.length === 0`, jangan biarkan `NaN%` tampil di chart).
- **`peringatan` sudah ada untuk "tidak ada AcademicYear aktif"** (baris 288-291) — TIDAK diubah, tetap tampil sebagai banner terpisah, independen dari perubahan chart ini.
- **Rentang campur** (sebagian lampau sebagian masa depan, misal filter "Dari 10 Agustus" "Sampai 25 Agustus" sementara hari ini 18 Agustus) — basis persen HANYA hari 10-18 Agustus (yang sudah lewat), data 19-25 Agustus tidak masuk penyebut MAUPUN pembilang (karena `rows` sendiri query attendance yang memang cuma ada sampai hari ini).

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance-report.service.ts` (`reportInternal()`, tambah agregat persen + pemanggilan kedua `resolveHariWajib` dengan cutoff), `apps/web/src/app/(admin)/rekap/rekap-view.tsx` (chart config, hapus `reduce()` FE), `apps/web/src/lib/core-types.ts` (interface baru).
- **Jangan sentuh:** `resolveHariWajib()` signature/behavior existing (dipakai banyak consumer lain, TIDAK boleh berubah basisnya), mode "single-day" chart, `attendance-report-export.service.ts` (PDF/Excel export, formula persen di situ SUDAH ADA dan BEDA basis — tidak perlu disamakan di task ini kecuali user minta terpisah).

## Acceptance Criteria
- [x] Chart mode "range" — sumbu X 0-100%, bukan count absolut.
- [x] Rentang tanggal yang seluruhnya sudah lewat — persen dihitung benar, total tidak melebihi 100% per kategori.
- [x] Rentang tanggal mencakup hari depan — hari depan TIDAK masuk basis penyebut, persen tetap representatif untuk hari yang sudah terjadi.
- [x] Rentang seluruhnya di masa depan — state kosong jelas ditampilkan, BUKAN chart 0% membingungkan.
- [x] Tooltip/label sumbu chart tampilkan simbol "%" eksplisit.
- [x] Mode "single-day" TIDAK berubah (tetap count absolut, diverifikasi tidak ada regresi).
- [x] Build + type-check hijau, test baru untuk `reportInternal()` skenario: rentang lampau penuh, rentang campur lampau+depan, rentang depan penuh (basis 0), rows kosong (no NaN).

## Validasi Claudian
- [x] Konfirmasi penyebut agregat persen dikali jumlah siswa (bukan cuma basis 1 siswa) — `hitungAgregatPersen()`: `penyebut = basisHariWajibSudahLewat * rows.length`, diverifikasi test skenario 2 siswa (total per kategori tidak melebihi 100%).
- [x] Konfirmasi `resolveHariWajib()` existing TIDAK diubah signature/behavior-nya — dipanggil APA ADANYA (signature sama persis), pemanggilan kedua di `reportFlexible()` untuk `effectiveTo` adalah TAMBAHAN murni, consumer lain (`teacher-attendance-report.service.ts`, `tv-piket.service.ts`, `attendance.service.ts`) tidak disentuh sama sekali.

## Implementasi (2026-08-18)

**Backend** (`attendance-report.service.ts`):
- `AgregatPersenRekap` interface baru + field `agregatPersen` di `FlexibleReportResult` mode "range".
- `reportInternal()` (private) — return type diperluas expose `wajibDates`+`from`+`to` (selain `rows`+`adaTahunAjaranAktif` existing), supaya `reportFlexible()` bisa short-circuit reuse tanpa query `resolveHariWajib()` kedua saat `effectiveTo === to` (kasus paling umum — rentang yang dipilih sudah lewat semua).
- `reportFlexible()` — hitung `effectiveTo = min(to, startOfToday())`, reuse `internal.wajibDates` kalau `effectiveTo` sama persis dengan `to` asli, else panggil `resolveHariWajib(from, effectiveTo)` kedua (atau langsung `Set` kosong kalau `effectiveTo < from`, seluruh rentang di masa depan — hindari query sia-sia).
- `hitungAgregatPersen()` (private, baru) — SUM `hadir+terlambat`/`izin`/`sakit`/`dispen`/`alfa` lintas SEMUA `rows`, dibagi `basisHariWajibSudahLewat × rows.length` (WAJIB dikali jumlah siswa, bukan basis 1 siswa). `basisHariWajibSudahLewat === 0` ATAU `rows.length === 0` → semua `*Persen` return `0` langsung (hindari div-by-zero/NaN).
- `resolveHariWajib()` signature/behavior TIDAK disentuh — tetap dipakai apa adanya oleh semua consumer existing.

**Frontend** (`rekap-view.tsx`):
- `reduce()` manual FE atas `result.rows` DIHAPUS untuk mode range — chart langsung pakai `result.agregatPersen.*Persen` dari API.
- Chart config: `scales.x.max: 100` + `ticks.callback` tambah "%", `tooltip.callbacks.label` tampil "92%" bukan "92".
- State kosong baru: `result.agregatPersen.basisHariWajibSudahLewat === 0` (mode range) → pesan "Belum ada hari yang sudah lewat dalam rentang ini..." menggantikan `<canvas>`, effect chart di-skip (tidak render Chart.js dengan data semua 0).
- `chartTitle` mode range diperbarui: "Grafik Persentase Hadir/Izin/Sakit/Dispen/Alfa" (sebut eksplisit "Persentase").
- Mode "single-day" — TIDAK disentuh sama sekali (tetap count absolut).
- `core-types.ts` — `AgregatPersenRekap` + `FlexibleReportResult` mode "range" diperluas, SAMA PERSIS shape backend.

**Verifikasi**: 4 test baru (`attendance-report.service.spec.ts`, dates dihitung RELATIF ke hari ini via helper `mondayOffsetFromToday()` — bukan hardcode, supaya stabil kapan pun dijalankan): rentang lampau penuh (persen benar, total ≤100%), rentang campur lampau+depan (basis < totalHariAktif penuh), rentang depan penuh (basis 0, semua persen 0), rows kosong (no NaN). tsc bersih api+web, `nest build` bersih, `next build` bersih (`/rekap` compile sukses). Dev server direstart bersih pasca build, curl `/rekap` konfirmasi 307 (bukan 500). Live browser verify TIDAK dilakukan.

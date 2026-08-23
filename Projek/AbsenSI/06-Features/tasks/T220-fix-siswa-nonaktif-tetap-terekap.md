# T220 — API: Fix Root Cause — Siswa Nonaktif Tidak Pernah Di-null-kan kelasId, Tetap Terekap

## Depends on
Tidak ada dependency ke rangkaian lain. Independen, murni modul `students`/`kelas`. **PRIORITAS TINGGI** — ini bug data yang sudah berjalan lama, berdampak ke akurasi rekap kehadiran yang dipakai laporan resmi.

## Konteks — Root Cause Ditemukan (2026-08-18)

User laporkan siswa nonaktif tetap muncul di Rekap Kehadiran Murid. Riset kode mengonfirmasi: **desain skema sudah punya niat yang benar** (komentar di `apps/api/prisma/schema.prisma` dekat model `Student`, ditulis 2026-07-30: *"Siswa NONAKTIF (lulus/keluar) juga di-null-kan di sini"*, field `kelasTerakhirNama` disediakan sebagai snapshot teks pengganti) — **TAPI TIDAK ADA SATUPUN KODE SERVICE yang mengimplementasikan niat itu**. Bug ini sudah ada sejak field `kelasTerakhirNama` dibuat, bukan regresi baru.

Ditemukan 2 jalur yang menulis `Student.status = "nonaktif"`, KEDUANYA tidak null-kan `kelasId`:

1. **`StudentsService.update()`** (`apps/api/src/core/students/students.service.ts:396-444`) — PATCH generic dipakai form "Tandai Keluar" (`apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx:459-472`, `TandaiKeluarForm`). Body request `{ status: "nonaktif", alasanNonaktif, tahunLulus }` **tidak pernah menyertakan `kelasId`** — Prisma `update()` karenanya tidak menyentuh kolom itu sama sekali, siswa nonaktif tetap punya `kelasId` mengarah ke kelas lamanya.
2. **`KelasService.kenaikanMassal()` cabang lulus** (`apps/api/src/core/kelas/kelas.service.ts:175-182`) — `updateMany({ data: { status: "nonaktif", alasanNonaktif: "lulus", tahunLulus } })` — `kelasId` **sama sekali tidak disebut** di objek `data`, siswa yang baru "diluluskan" massal tetap terikat ke kelas XII asalnya.

`kelasTerakhirNama` **tidak bisa diisi lewat endpoint manapun** — field ini bahkan tidak ada di `CreateStudentDto`/`UpdateStudentDto` (`apps/api/src/core/students/dto/`), jadi selalu `null` untuk semua siswa yang jadi nonaktif lewat jalur normal aplikasi.

Konsekuensi langsung: `AttendanceReportService.reportInternal()`/`reportSingleDay()` (`attendance-report.service.ts:130-145`, `380-390`) filter `kelasId: query.kelasId ?? { not: null }` **tanpa filter `status`** — siswa nonaktif LOLOS filter ini karena `kelasId`-nya memang tidak pernah jadi `null`.

## Keputusan Dikonfirmasi User (2026-08-18)

**Perbaiki ROOT CAUSE** (alur ubah status ke nonaktif) — BUKAN cuma tempel `status: "aktif"` di where clause query rekap sebagai tambalan. Root cause harus diperbaiki supaya SEMUA fitur lain yang bergantung pada asumsi "siswa nonaktif = kelasId null" (bukan cuma rekap) ikut benar — konsisten prinsip desain yang sudah ada di komentar schema.

## Spec Detail

### 1. Backend — DTO: tambah `kelasTerakhirNama` (agar bisa ditulis)

`apps/api/src/core/students/dto/update-student.dto.ts` — TIDAK PERLU tambah field manual ke DTO (field ini WAJIB diisi OTOMATIS oleh service, BUKAN dikirim dari body request FE — kalau ditambahkan ke DTO, berisiko FE lupa isi/isi asal-asalan). Field `kelasTerakhirNama` HANYA ditulis lewat logic service, TIDAK PERNAH terima dari request body langsung.

### 2. Backend — `StudentsService.update()` — auto null-kan kelasId saat status jadi nonaktif

`apps/api/src/core/students/students.service.ts:396-444` — TAMBAH logic SEBELUM `prisma.student.update()`:

```ts
async update(id: number, dto: UpdateStudentDto) {
  const existing = await this.ensureExists(id); // pastikan return data lengkap termasuk kelas.nama saat ini
  // ... validasi nisn/kelasId existing tidak berubah ...

  let kelasIdUpdate: number | null | undefined = dto.kelasId;
  let kelasTerakhirNamaUpdate: string | undefined;

  const statusBerubahJadiNonaktif = dto.status === "nonaktif" && existing.status !== "nonaktif";
  if (statusBerubahJadiNonaktif) {
    // Snapshot nama kelas SEBELUM di-null-kan — siswa nonaktif kehilangan relasi kelas
    // tapi histori "terakhir di kelas mana" harus tetap terbaca (dipakai riwayat/rekap).
    kelasTerakhirNamaUpdate = existing.kelas?.nama ?? existing.kelasTerakhirNama ?? undefined;
    kelasIdUpdate = null;
  }

  const statusBerubahJadiAktifKembali = dto.status === "aktif" && existing.status !== "aktif";
  if (statusBerubahJadiAktifKembali && dto.kelasId === undefined) {
    // Diaktifkan kembali TANPA kelasId baru eksplisit — TOLAK, jangan biarkan siswa
    // "aktif" tapi kelasId tetap null (state tidak valid, siswa aktif harus py kelas).
    throw new BadRequestException(
      "Siswa yang diaktifkan kembali wajib langsung dipilihkan kelas baru — sertakan kelasId pada request ini."
    );
  }

  return await this.prisma.student.update({
    where: { id },
    data: {
      ...,
      kelasId: kelasIdUpdate,
      kelasTerakhirNama: kelasTerakhirNamaUpdate ?? dto.kelasId !== undefined ? undefined : existing.kelasTerakhirNama,
      status: dto.status,
      alasanNonaktif: dto.alasanNonaktif,
      tahunLulus: dto.tahunLulus,
    },
    include: { kelas: { include: { kampus: true, jurusan: true } } },
  });
}
```
- **VERIFIKASI SAAT IMPLEMENTASI**: pastikan `ensureExists()` (atau query terpisah) mengambil `existing.kelas.nama` SEBELUM update, karena setelah `kelasId` di-null-kan relasi itu hilang — snapshot HARUS diambil dari state SEBELUM mutasi, bukan sesudah.
- **Edge case status TETAP nonaktif** (update lain, misal ubah `nama`, tapi `status` sudah nonaktif sebelumnya dan tidak berubah) — JANGAN re-snapshot `kelasTerakhirNama` (biarkan nilai lama, `statusBerubahJadiNonaktif` HARUS false kalau `existing.status` SUDAH `nonaktif`).
- **Edge case aktifkan kembali siswa nonaktif** — WAJIB sertakan `kelasId` baru di request yang sama (tidak masuk akal siswa "aktif" tapi tanpa kelas) — TOLAK dengan pesan actionable kalau tidak disertakan. Cek FE `handleAktifkanKembali` (disebut di riset, ada di `siswa-detail-view.tsx`) — PASTIKAN alur FE ini SUDAH/akan mengirim `kelasId` bersamaan, kalau belum TAMBAHKAN form pilih kelas saat aktifkan kembali (jangan langsung PATCH tanpa itu).

### 3. Backend — `KelasService.kenaikanMassal()` cabang lulus — null-kan kelasId

`apps/api/src/core/kelas/kelas.service.ts:175-182` — UBAH `updateMany` cabang `if (item.lulus)`:

```ts
if (item.lulus) {
  if (siswaLanjut.length > 0) {
    // updateMany TIDAK BISA pakai nilai per-row berbeda (kelasTerakhirNama beda per siswa
    // asal kelasnya beda) — WAJIB pakai Promise.all individual update, BUKAN updateMany,
    // supaya tiap siswa dapat snapshot kelasTerakhirNama yang BENAR (nama kelas asalnya
    // sendiri, bukan nama kelas item.kelasAsalId yang sama untuk seluruh batch -- VERIFIKASI
    // apakah kenaikanMassal() diproses per-kelas [1 kelasAsalId per `item`] atau lintas-kelas
    // sekaligus; kalau per-kelas, nama kelas SAMA untuk seluruh siswaLanjut dalam 1 `item`,
    // updateMany dengan literal nama kelas itu MASIH VALID dan lebih efisien).
    const kelasAsal = await tx.kelas.findUniqueOrThrow({ where: { id: item.kelasAsalId } });
    await tx.student.updateMany({
      where: { id: { in: siswaLanjut.map((s) => s.id) } },
      data: {
        status: "nonaktif",
        alasanNonaktif: "lulus",
        tahunLulus: dto.tahunLulus,
        kelasId: null,                          // BARU — T220
        kelasTerakhirNama: kelasAsal.nama,       // BARU — T220
      },
    });
  }
  diluluskan = siswaLanjut.length;
}
```
- **VERIFIKASI struktur `item`/`dto` di `kenaikanMassal()`** saat implementasi — pastikan `item.kelasAsalId` (atau field serupa) memang tersedia di scope method untuk lookup nama kelas asal (method lengkap ada di baris 111-218, cek definisi tipe `item` sebelum asumsi nama field).

### 4. Backend — audit method lain yang mungkin set status nonaktif

- Cek `apps/api/src/import/import.service.ts` — dikonfirmasi riset TIDAK menulis `status` sama sekali untuk Student saat ini, TAPI VERIFIKASI ULANG saat implementasi (kemungkinan ada penyesuaian dari task lain yang berjalan paralel/T178-an, cek `git log`/`git diff` file ini sebelum asumsi masih sama seperti temuan riset).
- Tidak ada endpoint "nonaktifkan siswa" terpisah selain 2 jalur di atas — TIDAK PERLU cari jalur ke-3.

### 5. Data lama — perbaikan retroaktif (opsional, WAJIB konfirmasi user)

Siswa yang SUDAH nonaktif sebelum fix ini (data existing di production) TETAP punya `kelasId` terisi dan `kelasTerakhirNama` NULL — fix di atas HANYA berlaku untuk perubahan status BARU ke depan, TIDAK otomatis membetulkan data lama.

**JANGAN jalankan migrasi data retroaktif tanpa konfirmasi eksplisit user** (konsisten protokol pasca-insiden database wipe 2026-07-30 — backup + verifikasi row count wajib untuk operasi UPDATE massal ke data production). Kalau user setuju, script terpisah (BUKAN bagian migration Prisma otomatis):
```sql
-- HANYA contoh konsep, WAJIB backup+dry-run dulu, TIDAK dieksekusi otomatis oleh task ini
UPDATE students s
JOIN kelas k ON s.kelas_id = k.id
SET s.kelas_terakhir_nama = k.nama, s.kelas_id = NULL
WHERE s.status = 'nonaktif' AND s.kelas_id IS NOT NULL;
```

## Edge Cases

- **Siswa dibuat LANGSUNG berstatus nonaktif saat create** (form `SiswaForm`, `siswa-view.tsx:409-438`, `kelasId` tetap `required` di UI) — SKENARIO INI JARANG/TIDAK WAJAR (siswa baru harusnya aktif dulu), TAPI kalau terjadi: `StudentsService.create()` JUGA perlu logic serupa (kalau `dto.status === "nonaktif"` saat create, snapshot `kelasTerakhirNama` dari `kelasId` yang dikirim, lalu simpan `kelasId: null`) — CEK apakah `create()` butuh perbaikan sama, JANGAN cuma `update()`.
- **`kelasTerakhirNama` untuk siswa yang belum PERNAH punya kelas** (kelasId sudah null dari awal, misal siswa baru T072 belum di-plot lalu langsung dinonaktifkan) — `existing.kelas?.nama` akan `undefined`, `kelasTerakhirNama` tetap `null` — INI BENAR, tidak ada kelas untuk di-snapshot.
- **Race kenaikan massal + rekap berjalan bersamaan** — di luar scope task ini (masalah locking/transaksi umum, tidak spesifik ke bug ini).

## Files
- **Modifikasi:** `apps/api/src/core/students/students.service.ts` (`update()`, kemungkinan `create()`), `apps/api/src/core/kelas/kelas.service.ts` (`kenaikanMassal()` cabang lulus), cek FE `siswa-detail-view.tsx` (`handleAktifkanKembali` — pastikan kirim `kelasId` baru).
- **Jangan sentuh:** `attendance-report.service.ts` (query rekap TIDAK perlu tambahan filter `status` kalau root cause ini sudah benar — `kelasId: { not: null }` yang SUDAH ADA otomatis akan exclude siswa nonaktif begitu `kelasId`-nya benar-benar null).

## Acceptance Criteria
- [x] `PATCH /students/:id` dengan `status: "nonaktif"` — `kelasId` OTOMATIS jadi `null`, `kelasTerakhirNama` terisi nama kelas sebelumnya.
- [x] `PATCH /students/:id` dengan `status: "aktif"` (dari nonaktif) TANPA `kelasId` di body — DITOLAK dengan pesan actionable.
- [x] `PATCH /students/:id` dengan `status: "aktif"` + `kelasId` baru — berhasil, siswa aktif kembali dengan kelas baru.
- [x] `kenaikanMassal()` cabang lulus — siswa yang diluluskan `kelasId` jadi `null`, `kelasTerakhirNama` terisi nama kelas asal (XII yang baru dilulus).
- [x] Setelah fix — hasil Rekap Kehadiran Murid (query TIDAK diubah) otomatis TIDAK lagi menampilkan siswa nonaktif BARU (yang statusnya diubah setelah fix ini deploy).
- [x] Update lain ke siswa nonaktif (misal ubah nama) TIDAK re-trigger snapshot `kelasTerakhirNama` kalau status sudah nonaktif sebelumnya (tidak menimpa snapshot lama dengan nilai kosong).
- [x] Build + type-check hijau, test baru: nonaktifkan siswa (cek kelasId null+snapshot benar), aktifkan tanpa kelasId (ditolak), aktifkan dengan kelasId (berhasil), kenaikan massal lulus (kelasId null+snapshot benar), update status tetap sama (tidak re-snapshot).

## Validasi Claudian
- [x] Konfirmasi snapshot `kelasTerakhirNama` diambil dari state SEBELUM mutasi — `existing` di-query dengan `include: { kelas: true }` di baris PERTAMA `update()`, dipakai untuk snapshot SEBELUM `kelasId` di-null-kan di objek `data` yang sama.
- [x] Konfirmasi TIDAK ada migrasi data retroaktif otomatis ke production — TIDAK dijalankan, TIDAK ditanyakan ke user di sesi ini (di luar scope eksekusi task ini, tetap pending keputusan terpisah kalau dibutuhkan nanti).
- [x] Verifikasi struktur `item.kelasAsalId` di `kenaikanMassal()` sesuai kode aktual — dikonfirmasi via baca kode langsung: `kelasAsal` (dgn `.nama`) SUDAH ada di scope (baris 154, lookup dari `kelasById` map), 1 `item` = 1 `kelasAsalId` jadi SEMUA `siswaLanjut` di iterasi itu berasal dari kelas yang sama — `updateMany` literal `kelasAsal.nama` valid, TIDAK perlu `Promise.all` per-siswa seperti draft awal spec.

## Implementasi (2026-08-19)

**Backend** (`students.service.ts`):
- `update()` — di-refactor total: `ensureExists()` (private, cuma cek exists) DIHAPUS, diganti query `existing` yang SEKALIGUS `include: { kelas: true }` di awal method (perlu `kelas.nama` utk snapshot, jadi query 1x lebih kaya daripada 2 query terpisah).
- `statusBerubahJadiNonaktif`/`statusBerubahJadiAktifKembali` — dihitung dari `dto.status` vs `existing.status` (transisi, BUKAN status-sudah-sama) — update lain ke siswa yang KEBETULAN sudah nonaktif (ubah nama dll) tidak re-trigger null-kan/snapshot.
- Transisi nonaktif→aktif TANPA `kelasId` di body → `BadRequestException` SEBELUM `update()` dipanggil.
- `create()` — edge case "dibuat langsung nonaktif" (jarang, tapi didukung): `kelasId` tetap disimpan `undefined` (tidak ditulis), snapshot ke `kelasTerakhirNama` dari `kelas.nama` yang sudah di-query saat validasi `dto.kelasId`.

**Backend** (`kelas.service.ts` — `kenaikanMassal()` cabang lulus):
- `updateMany` tambah `kelasId: null` + `kelasTerakhirNama: kelasAsal.nama` (variable yang SUDAH ada di scope loop, tidak perlu query tambahan).

**Frontend** (`siswa-detail-view.tsx`):
- `handleAktifkanKembali()` (PATCH langsung tanpa kelasId, akan SELALU gagal pasca-fix backend) DIHAPUS, diganti Dialog `AktifkanKembaliForm` (pola sama persis `TandaiKeluarForm`) — admin WAJIB pilih kelas baru dari `Select` sebelum submit PATCH `{status:"aktif", kelasId}`.
- `TandaiKeluarForm` TIDAK diubah — sudah tidak kirim `kelasId` (benar, backend sekarang auto null-kan).

**Verifikasi**: 7 test baru `StudentsService` (nonaktifkan+snapshot benar, siswa tanpa kelas+snapshot tetap null, status-tetap-nonaktif tidak re-snapshot, aktifkan tanpa kelasId ditolak, aktifkan dengan kelasId berhasil, student tidak ditemukan, create langsung-nonaktif) + 2 test baru `KelasService` (file spec BARU, sebelumnya tidak ada test untuk modul ini — cabang lulus kelasId null+snapshot, tidak ada siswa lanjut tidak updateMany). tsc bersih, `nest build` bersih, `next build` bersih (504/504 backend test lulus total). Dev server direstart bersih, curl `/siswa` 307 (bukan 500). Live browser verify TIDAK dilakukan (kredensial login belum tersedia sesi ini).

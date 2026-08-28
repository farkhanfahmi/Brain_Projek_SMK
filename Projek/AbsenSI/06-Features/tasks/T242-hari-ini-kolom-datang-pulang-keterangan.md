# T242 — Web: Dashboard "Hari Ini" Wali Kelas — Kolom Datang/Pulang Terpisah + Keterangan di Kartu Izin/Sakit/Dispen

## Depends on
T241 (Dashboard "Hari Ini" — Tampilkan Jam Pulang di Kartu Hadir). **Sudah selesai** — kode
`waktuPulang` sudah ter-render (dikonfirmasi ke source 2026-08-25, HEAD commit branch `dev`
literal berjudul "...jam pulang dashboard wali kelas"), tapi `STATUS.md` masih tercatat
"Sedang/Belum dikerjakan" — status vault basi, PERBAIKI baris T241 di `STATUS.md` jadi
"Selesai" sekalian saat mengerjakan task ini (bukan task terpisah, cukup 1 baris tabel).

Task ini murni **penyempurnaan visual** dari hasil T241, bukan fitur baru — T241 baru
menambahkan info jam pulang (stacked 2 baris dalam 1 kolom kanan), task ini merapikan jadi
kolom sungguhan + ganti isi kartu Izin/Sakit/Dispen.

## Objective
Baris siswa di kartu yang di-expand (`hari-ini-tab.tsx`) tampil sebagai kolom-kolom jelas:
"Datang" & "Pulang" terpisah untuk kartu Hadir dan Terlambat (bukan digabung 1 kolom kanan
dengan label "Masuk:"/"Pulang:" di dalamnya), dan kartu Izin/Sakit/Dispen tampilkan
**Keterangan** (input petugas piket) alih-alih jam.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-25)

**Backend SUDAH 100% siap, TIDAK PERLU perubahan.** `alasanDetail` (keterangan piket saat
menandai izin/sakit) SUDAH ADA di response `GET /journal/kelas-wali-status-hari-ini` —
`PiketBoardRow.alasanDetail` (`core-types.ts:709`), diisi dari `permit?.alasanDetail ?? null`
di `AttendanceService.resolveStatusHarianSiswa()` (`attendance.service.ts:742`), 1 sumber
kebenaran yang sama dipakai Papan Piket (`piket-board-view.tsx` — kolom "Keterangan" pada
tabel Board Semua Siswa sudah pakai field yang sama persis). Task ini HANYA perlu
menampilkannya di komponen wali kelas, field-nya sudah mengalir sampai frontend.

`apps/web/src/app/(guru)/guru/wali-kelas/components/hari-ini-tab.tsx`, render baris siswa
saat ini (hasil T241, baris ~160-186):

```tsx
<ul className="flex flex-col gap-2">
  {countsByKey.get(expandedKey)!.map((s) => (
    <li key={s.studentId} className="flex items-center justify-between rounded-lg bg-surface-subtle px-3 py-2">
      <span className="text-body text-ink">{s.nama}</span>
      {expandedKey === "hadir" ? (
        <span className="flex flex-col items-end text-caption text-ink-secondary">
          <span>Masuk: {s.waktuMasuk ? formatJam(s.waktuMasuk) : KATEGORI_LABEL[s.kategoriLive]}</span>
          <span>Pulang: {s.waktuPulang ? formatJam(s.waktuPulang) : "Belum Pulang"}</span>
        </span>
      ) : (
        <span className="text-caption text-ink-secondary">
          {s.waktuMasuk ? formatJam(s.waktuMasuk) : KATEGORI_LABEL[s.kategoriLive]}
        </span>
      )}
    </li>
  ))}
</ul>
```

Masalah yang diperbaiki task ini:
1. **Kartu "Hadir"** — Masuk & Pulang digabung 1 `<span>` kanan (2 baris ditumpuk vertikal
   dengan label teks "Masuk:"/"Pulang:") — user minta jadi **2 kolom terpisah** (kolom
   "Datang" dan kolom "Pulang" berdampingan, bukan ditumpuk).
2. **Kartu "Terlambat"** — MASIH masuk cabang `else` (baris tunggal, cuma tampilkan jam
   masuk) — user minta kartu ini JUGA punya kolom Datang + Pulang, SAMA seperti kartu Hadir.
3. **Kartu "Izin/Sakit/Dispen"** — MASIH masuk cabang `else` yang sama (tampilkan jam masuk
   atau label kategori) — user minta jam DIHAPUS, diganti kolom **"Keterangan"** yang
   menampilkan `s.alasanDetail` (isian petugas piket saat menandai izin/sakit siswa).

## Keputusan Diminta User (2026-08-25)

1. Kartu **"Hadir"** — kolom "Datang" dan kolom "Pulang" terpisah (bukan 1 kolom gabungan).
2. Kartu **"Terlambat"** — tambahkan kolom "Datang" dan "Pulang" juga (sekarang cuma 1 nilai).
3. Kartu **"Izin/Sakit/Dispen"** — hapus tampilan jam, ganti kolom **"Keterangan"** berisi
   input keterangan dari petugas piket (`alasanDetail`).

Kartu **"Alfa"** TIDAK disebut user — TIDAK BERUBAH, tetap render label kategori seperti
sekarang (siswa alfa tidak punya `waktuMasuk`/`alasanDetail` yang relevan).

## Spec Detail

### 1. Ubah struktur baris jadi 3 varian berdasarkan `expandedKey`

`apps/web/src/app/(guru)/guru/wali-kelas/components/hari-ini-tab.tsx` — ganti percabangan
2-arah (`hadir` vs lainnya) jadi minimal 3-arah:

- **`hadir` DAN `terlambat`** → render kolom "Datang" + kolom "Pulang" (behavior SAMA persis
  untuk kedua kartu ini, REUSE 1 sub-komponen/fragment yang sama, jangan duplikasi kode).
- **`izin-sakit-dispen`** → render kolom "Keterangan" berisi `s.alasanDetail`.
- **Selain itu (`alfa`)** → TETAP seperti sekarang (label kategori, TIDAK diubah).

Rekomendasi struktur (VERIFIKASI SAAT IMPLEMENTASI — boleh disesuaikan asal hasil visualnya
kolom-kolom sungguhan, bukan cuma teks bertumpuk):

```tsx
{(expandedKey === "hadir" || expandedKey === "terlambat") ? (
  <div className="flex items-center gap-6 text-caption text-ink-secondary">
    <span className="flex w-16 flex-col items-end">
      <span className="text-[10px] uppercase text-ink-tertiary">Datang</span>
      <span>{s.waktuMasuk ? formatJam(s.waktuMasuk) : KATEGORI_LABEL[s.kategoriLive]}</span>
    </span>
    <span className="flex w-20 flex-col items-end">
      <span className="text-[10px] uppercase text-ink-tertiary">Pulang</span>
      <span>{s.waktuPulang ? formatJam(s.waktuPulang) : "Belum Pulang"}</span>
    </span>
  </div>
) : expandedKey === "izin-sakit-dispen" ? (
  <span className="flex flex-col items-end text-caption text-ink-secondary">
    <span className="text-[10px] uppercase text-ink-tertiary">Keterangan</span>
    <span>{s.alasanDetail ?? "-"}</span>
  </span>
) : (
  <span className="text-caption text-ink-secondary">{KATEGORI_LABEL[s.kategoriLive]}</span>
)}
```

Catatan implementasi:
- `formatJam` belum ada sebagai helper terpisah di file ini saat ini (inline
  `new Date(...).toLocaleTimeString(...)` diulang 2x) — kalau kolom ini menambah pengulangan
  ke-3/4, EKSTRAK jadi 1 fungsi lokal `formatJam(value: string | null)` di file yang sama
  (pola ini SUDAH ADA persis di `piket-board-view.tsx` baris 39-42 — replikasi, bukan
  reinvent).
- Untuk kartu "Terlambat", siswa yang tampil di sini per definisi PASTI punya `waktuMasuk`
  terisi (kategori `terlambat` artinya sudah tap masuk tapi telat) — tapi tetap pertahankan
  fallback `KATEGORI_LABEL[s.kategoriLive]` untuk konsistensi kode dengan kartu Hadir
  (defensif, bukan idealnya pernah kepakai untuk kartu ini).
- Header kolom kecil ("Datang"/"Pulang"/"Keterangan") DITAMBAHKAN PER-BARIS di atas nilainya
  (bukan 1 baris header terpisah di atas seluruh list) — konsisten dengan tidak adanya
  struktur `<table>` sungguhan di sini (list `<li>`, bukan `<Table>` shadcn). Kalau saat
  implementasi ternyata heading per-baris terasa berulang secara visual, alternatif: 1 baris
  header non-`<li>` di atas `<ul>` (mirip header tabel) — VERIFIKASI SAAT IMPLEMENTASI, pilih
  yang lebih rapi secara visual, TETAP wajib ada label kolom yang jelas (jangan cuma 2 angka
  berdampingan tanpa keterangan yang mana Datang/mana Pulang).
- Lebar kolom (`w-16`/`w-20` pada contoh) HANYA starting point — sesuaikan biar rata kanan
  rapi untuk isi terpanjang ("Belum Pulang").

### 2. Kartu "Izin/Sakit/Dispen" — `alasanDetail` kosong/null

Kalau `s.alasanDetail` adalah `null` (siswa masuk kategori ini tapi petugas piket TIDAK
mengisi keterangan saat menandai) — tampilkan `"-"`, JANGAN kosong/blank/`null` mentah
(konsisten pola `feedback_pesan_error_sesuai_kondisi_bukan_generik` — state harus eksplisit,
bukan ambigu).

### 3. TIDAK PERLU perubahan lain
- Backend — TIDAK disentuh (field `alasanDetail` sudah lengkap mengalir end-to-end).
- Socket.IO real-time — TIDAK disentuh. **CATATAN**: event patch real-time saat ini
  (`onKampusUpdate`, baris ~64-83) HANYA meng-update `status`/`kategoriLive`/`waktuMasuk`/
  `waktuPulang` — TIDAK meng-update `alasanDetail` saat event masuk. Ini SUDAH BENAR by
  design (alasanDetail diisi lewat aksi piket "Tandai Izin/Sakit", BUKAN lewat event tap
  kartu) — TIDAK PERLU ditambah, di luar scope task ini. Kalau piket menandai izin SETELAH
  dashboard wali kelas dibuka, keterangan itu baru muncul di sesi Socket berikutnya/refresh
  manual — batasan yang sudah ada sebelum task ini, bukan regresi baru.

## Edge Cases
- **`alasanDetail` null** (piket tandai izin/sakit tanpa isi keterangan) → tampil `"-"`.
- **Siswa terlambat tapi belum tap pulang** (kasus normal, sedang jam pelajaran) → kolom
  Pulang tampil "Belum Pulang", BUKAN kosong.
- **Real-time update sedang kartu Hadir/Terlambat di-expand** (siswa tap pulang) → kolom
  Pulang berubah otomatis dari "Belum Pulang" ke jam sebenarnya TANPA reload — infrastruktur
  ini SUDAH ADA dari T241, PASTIKAN tidak regresi saat mengubah struktur JSX-nya.
- **Kartu Alfa** — TIDAK disentuh sama sekali, verifikasi masih render seperti sebelumnya
  setelah refactor percabangan (regression check, bukan perubahan fungsional).

## Files
- **Modifikasi:** `apps/web/src/app/(guru)/guru/wali-kelas/components/hari-ini-tab.tsx`
  (percabangan render baris siswa jadi 3-arah + kolom Datang/Pulang/Keterangan).
- **Modifikasi (vault, bukan kode):** `Projek/AbsenSI/STATUS.md` — perbaiki baris T241 dari
  "Sedang/Belum dikerjakan" jadi "Selesai" (status basi, kode sudah ada di HEAD `dev`).
- **Jangan sentuh:** backend (`attendance.service.ts`, `journal-kelas-wali.controller.ts`),
  Socket.IO gateway, `piket-board-view.tsx` (kolom Keterangan di sana SUDAH benar, jadi
  referensi pola, bukan yang diubah).

## Revisi User Setelah Implementasi Awal (2026-08-25)

Implementasi pertama pakai `<ul>/<li>` dengan label kecil DI ATAS tiap nilai per-baris
("Datang"/"Pulang"/"Keterangan" diulang tiap baris) — user minta diganti jadi **table
sungguhan**: judul kolom SEKALI di header (`<TableHeader>`), bukan diulang per-baris.
Diubah ke `@absensi/ui` Table primitives (REPLIKASI pola `piket-board-view.tsx`):
- Kartu hadir/terlambat → kolom `Nama | Datang | Pulang`.
- Kartu izin-sakit-dispen → kolom `Nama | Keterangan`.
- Kartu alfa → kolom `Nama | Status` (tidak eksplisit diminta user, tapi dibuat konsisten
  table 2 kolom juga alih-alih 1 kolom polos, dikonfirmasi user).

## Acceptance Criteria
- [x] Expand kartu "Hadir" — table dengan header kolom "Datang" dan "Pulang" (bukan label
      diulang per-baris, bukan ditumpuk dalam 1 blok teks).
- [x] Expand kartu "Terlambat" — kolom "Datang" dan "Pulang" tampil sama seperti kartu Hadir.
- [x] Expand kartu "Izin/Sakit/Dispen" — TIDAK ada jam sama sekali, tampil kolom
      "Keterangan" berisi `alasanDetail` (atau `"-"` kalau null).
- [x] Kartu "Alfa" — table `Nama | Status` (revisi user, lihat catatan di atas).
- [x] Update real-time jam pulang (dari T241) tetap berfungsi setelah perubahan struktur JSX
      (state patch `onKampusUpdate` tidak disentuh, hanya JSX render yang berubah).
- [x] Baris T241 di `STATUS.md` diperbarui jadi "Selesai" (sudah dilakukan sesi sebelumnya).
- [x] Build + type-check hijau (`tsc --noEmit` @absensi/web bersih).

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA perubahan backend — task ini murni render field yang sudah ada
      (`waktuMasuk`, `waktuPulang`, `alasanDetail` semua sudah tersedia di `PiketBoardRow`).
- [x] Konfirmasi kartu Izin/Sakit/Dispen benar-benar tidak lagi menampilkan jam sama sekali.
- [x] Konfirmasi kolom Datang/Pulang benar-benar 2 kolom visual terpisah (ada label per
      kolom), bukan sekadar 2 baris teks ditumpuk yang cuma diganti nama variabelnya.
- [x] Konfirmasi tidak ada duplikasi kode antara cabang "hadir" dan "terlambat" (1 sumber
      kebenaran render kolom Datang/Pulang untuk keduanya).

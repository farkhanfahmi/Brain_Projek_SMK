# T245 — Web: "Riwayat Catatan" — Bedakan Visual "Izin Keluar" dari Izin Tidak Masuk Biasa

## Depends on
Tidak ada. Independen, murni frontend (`riwayat-catatan-table.tsx`, shared admin + wali
kelas). **TIDAK PERLU perubahan backend** — data sudah lengkap terkirim.

## Objective
Baris "Izin Keluar" (siswa keluar sementara lalu kembali, bukan absen 1 hari penuh) di
"Riwayat Catatan" tampil BEDA secara visual dari "Izin" biasa (tidak masuk 1 hari) —
supaya wali kelas/admin bisa langsung membedakan tanpa buka detail.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-25, KLARIFIKASI temuan)

**Data TIDAK hilang — cuma tidak dibedakan secara visual.** Root cause BUKAN "izin keluar
tidak tercatat", tapi "izin keluar tercatat TAPI disamarkan jadi terlihat sama dengan izin
tidak-masuk biasa":

1. **Backend `riwayatCatatan()` (`attendance-report.service.ts:596-696`) SUDAH mengambil
   SEMUA Permit tanpa filter jenis** (baris 612-622, `where: { studentId }` — TIDAK ada
   filter `jenis`). Baik `jenis: "tidak_masuk"` (absen 1 hari) MAUPUN `jenis: "keluar"`
   (izin keluar sementara) SAMA-SAMA masuk hasil query.
2. **`permitJenis` SUDAH dikirim ke response** (baris 692: `permitJenis: permit.jenis`) —
   ada di tipe `RiwayatCatatanEntry` (`core-types.ts` sekitar baris 13:
   `permitJenis: "tidak_masuk" | "keluar";`).
3. **MASALAHNYA: frontend `riwayat-catatan-table.tsx` TIDAK PERNAH membaca `permitJenis`
   sama sekali** — render badge "Izin"/"Sakit"/"Dispen" (`RIWAYAT_LABEL`/`RIWAYAT_BADGE_CLASS`/
   `RIWAYAT_ICON`, semua di-key oleh `entry.jenis` yang cuma `"izin"|"sakit"|"dispen"|...`,
   TIDAK ADA percabangan berdasar `permitJenis`). Hasilnya: siswa yang izin keluar 2 jam lalu
   kembali TAMPIL PERSIS SAMA di tabel ini dengan siswa yang izin tidak masuk seharian penuh
   — user (dan siapa pun yang lihat tabel ini) TIDAK BISA membedakan tanpa cek data mentah.

Konfirmasi jalur pembuatan data izin-keluar SUDAH BENAR set `jenis: "keluar"` —
`apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx:252` —
`jenis: "keluar", alasanKategori,` — jadi data-nya memang lengkap sejak dibuat, murni
tampilan yang kurang.

## Keputusan Diminta User (2026-08-25)
Baris izin keluar harus terlihat jelas beda dari izin tidak-masuk biasa di Riwayat Catatan
— user awalnya mengira ini "belum tercatat", padahal sebenarnya "tercatat tapi tersamarkan".

## Spec Detail

`apps/web/src/components/riwayat-catatan-table.tsx` — untuk entry dengan
`jenis === "izin" && permitJenis === "keluar"` (kombinasi SPESIFIK — `sakit`/`dispen`
biasanya bukan lewat izin-keluar TAPI kategori itu SECARA TEKNIS bisa dipilih juga di form
izin-keluar, VERIFIKASI SAAT IMPLEMENTASI apakah "keluar" perlu dibedakan untuk SEMUA 3
kategori atau cuma "izin" yang biasanya dipakai — REKOMENDASI: bedakan untuk permitJenis
"keluar" APAPUN kategorinya, bukan cuma jenis "izin", supaya konsisten):

- **Label berbeda**: "Izin Keluar" (bukan cuma "Izin") saat `permitJenis === "keluar"`.
  Untuk `sakit`/`dispen` yang kebetulan `permitJenis === "keluar"` — label jadi
  "Sakit (Keluar)"/"Dispen (Keluar)" ATAU pola serupa, VERIFIKASI SAAT IMPLEMENTASI teks
  paling jelas.
- **Ikon berbeda** (opsional tapi disarankan, konsisten pola existing "bentuk ikon jadi
  pembeda utama, bukan cuma warna", lihat komentar baris 31-33 file ini) — kandidat: `LogOut`
  atau `DoorOpen` dari `lucide-react` (VERIFIKASI ikon paling representatif "keluar
  sementara").
- **Warna badge BOLEH tetap sama** (primary-soft, konsisten "izin"/"sakit" existing) — yang
  WAJIB beda adalah TEKS/IKON, bukan berarti perlu warna ke-3 baru (DESIGN.md: 1 aksen
  oranye, jangan tambah hue tanpa alasan kuat).

Referensi pola pembeda ikon-vs-warna yang SUDAH ADA di file ini — baris 31-33 komentar:
"izin & sakit berbagi warna (primary-soft), jadi bentuk ikon jadi pembeda utama" — REPLIKASI
prinsip yang sama untuk izin vs izin-keluar.

## Edge Cases
- **Izin keluar yang statusKembali masih "belum"** (siswa belum dikonfirmasi kembali) —
  DI LUAR SCOPE task ini (itu sudah ditangani terpisah di dashboard piket "Belum Kembali").
  Riwayat Catatan ini murni tampilan HISTORIS, tidak perlu status real-time kembali/belum.
- **Kombinasi jenis+permitJenis yang jarang** (misal "dispen" via jalur izin-keluar, kalau
  memang dropdown kategori di form izin-keluar mengizinkan itu) — VERIFIKASI SAAT
  IMPLEMENTASI apakah kombinasi ini realistis terjadi, cek form `izin-keluar-view.tsx` opsi
  dropdown `alasanKategori` yang tersedia di sana.

## Files
- **Modifikasi:** `apps/web/src/components/riwayat-catatan-table.tsx` (label/ikon
  berdasarkan `permitJenis`, SHARED — otomatis berlaku admin + wali kelas, keduanya pakai
  komponen ini lewat `endpoint` berbeda).
- **Jangan sentuh:** backend (`attendance-report.service.ts`) — data sudah lengkap,
  `permitJenis` sudah ada di response, TIDAK PERLU query/field baru.

## Acceptance Criteria
- [x] Entry Riwayat Catatan dengan `permitJenis === "keluar"` tampil label/ikon BEDA dari
      izin tidak-masuk biasa (`permitJenis === "tidak_masuk"`).
- [x] Perubahan berlaku di KEDUA tempat yang pakai komponen ini (Riwayat Catatan admin
      `/siswa/:id` DAN wali kelas `/guru/wali-kelas/siswa/:id`) — 1 komponen shared.
- [x] Tidak ada regresi ke entry jenis lain (terlambat/alfa/pkl/terkunci/dibuka) — hanya
      cabang izin/sakit/dispen yang disentuh.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi TIDAK ADA perubahan backend — task ini murni pakai field `permitJenis`
      yang SUDAH ADA di response, belum pernah dibaca frontend.
- [x] Konfirmasi label/ikon baru benar-benar jelas beda secara visual — label jadi "Izin
      Keluar"/"Sakit Keluar"/"Dispen Keluar" (bukan cuma "Izin" generik) + ikon `LogOut`
      (beda dari `FileText`/`HeartPulse`/`Award` masing-masing jenis biasa).
- [x] Kombinasi kategori+permitJenis diverifikasi via kode: dropdown `izin-keluar-view.tsx`
      hanya punya opsi Izin/Sakit (tidak ada Dispen) — tapi dibedakan untuk SEMUA 3 kategori
      (bukan cuma "izin") sesuai rekomendasi spec, aman kalau nanti Dispen ditambahkan ke
      dropdown izin-keluar.

## Implementasi (2026-08-25)

Helper baru `resolveRiwayatDisplay()` di `riwayat-catatan-table.tsx` — untuk entry
izin/sakit/dispen dengan `permitJenis === "keluar"`, return label `"{Jenis} Keluar"` +
ikon `LogOut` (dari lucide-react); selain itu fallback ke lookup table lama
(`RIWAYAT_LABEL`/`RIWAYAT_ICON`) tanpa perubahan. Warna badge (`RIWAYAT_BADGE_CLASS`)
TIDAK disentuh sesuai spec (cukup label+ikon sebagai pembeda, bukan warna ke-3 baru).
Backend tidak disentuh — field `permitJenis` sudah lengkap mengalir dari sebelumnya. tsc
web bersih.
